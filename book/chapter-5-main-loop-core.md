# 第 5 章：主循环核心

## 5.1 消息处理管线：queryLoop 状态机

`src/query.ts`（约 1,730 行）是 Claude Code 的主循环引擎。它不叫"Agent Loop"，也不叫"Chat Loop"——它叫 `queryLoop`。这个名字本身就是一种工程立场：主循环不是"智能体行为模拟器"，而是"查询引擎"，每一轮迭代都是对模型的一次新查询，携带更新后的上下文集。

---

### query → queryLoop 的双层生成器

`query()` 是公共出口，`queryLoop()` 是内部引擎。两者都是 async generator，通过 `yield*` 委托：

```typescript
// query.ts:219-239
export async function* query(params: QueryParams): AsyncGenerator<
  StreamEvent | RequestStartEvent | Message | TombstoneMessage | ToolUseSummaryMessage,
  Terminal  // return 类型：调用方通过 .return() 获取 Terminal 实例
> {
  const consumedCommandUuids: string[] = []
  const terminal = yield* queryLoop(params, consumedCommandUuids)
  // 成功路径：通知所有消费命令完成（失败则跳过，started-without-completed 信号）
  for (const uuid of consumedCommandUuids) {
    notifyCommandLifecycle(uuid, 'completed')
  }
  return terminal
}
```

**为何拆成两个函数**——`consumedCommandUuids` 是一个在循环中收集的数组。`query()` 捕获这个数组的引用，在 `queryLoop` 正常返回后执行生命周期通知。如果 `queryLoop` 抛出异常或被 `.return()` 提前终止，通知不会触发。这种不对称性是一种诊断信号：系统可以检测到"开始但从未完成"的命令。

---

### State 类型：可变状态的集中载体

```typescript
// query.ts:204-217
type State = {
  messages: Message[]                           // 当前消息历史
  toolUseContext: ToolUseContext                // 工具上下文（含 AbortController）
  autoCompactTracking: AutoCompactTrackingState | undefined
  maxOutputTokensRecoveryCount: number          // max_output_tokens 重试计数
  hasAttemptedReactiveCompact: boolean          // reactive compact 是否已尝试
  maxOutputTokensOverride: number | undefined   // 临时覆盖 max_output_tokens
  pendingToolUseSummary: Promise<ToolUseSummaryMessage | null> | undefined
  stopHookActive: boolean | undefined           // stop hook 是否活跃
  turnCount: number                             // 已执行轮次数
  transition: Continue | undefined              // 上一轮 continue 的原因
}
```

**为何用集中 State 而非分散变量**——`queryLoop` 有 7 个 `continue` 站点。如果每个变量单独赋值，每个 continue 站点需要 7-10 行赋值语句，共 50-70 行分散的重复代码。State 对象使得每个 continue 站点只需构造一个新对象（`state = next`），将 7x10 缩减为 7x1。

---

### 单轮生命周期时序

```mermaid
sequenceDiagram
    participant Loop as queryLoop
    participant Memory as Memory Prefetch
    participant API as Claude API (Streaming)
    participant Tools as Streaming Tool Executor
    participant Hooks as Stop/Post Hooks

    Loop->>Memory: startRelevantMemoryPrefetch() [fire-and-forget]
    Loop->>Loop: skillDiscoveryPrefetch [异步后台]
    Loop->>API: yield 'stream_request_start'
    API-->>Loop: Assistant stream chunks
    Loop->>API: Snip (超长消息截断)
    Loop->>API: MicroCompact (单轮消息精简)
    Note over Loop,API: 如果 30 行无文本工具：contextCollapse<br/>drain + retry

    API-->>Loop: Assistant response (含 tool_use blocks)
    Loop->>Memory: consume pendingMemoryPrefetch [已就绪则立即，否则跳过]
    Loop->>Tools: runTools / streamingToolExecutor
    Tools-->>Loop: Tool results (逐批)

    alt 用户中断 (Ctrl+C)
        Loop->>Tools: 清理 CU 锁定
        Loop-->>Loop: yield userInterruption
        Loop->>Loop: return { reason: 'aborted_streaming' }
    else 无工具调用 (needsFollowUp=false)
        Loop->>Hooks: executePostSamplingHooks()
        alt max_output_tokens exceeded
            Loop->>API: escalate 8k → 64k (一次)
            Loop->>API: 注入 recovery message (最多 N 次)
        else prompt-too-long
            Loop->>Loop: contextCollapse.drain + retry
            Loop->>API: reactiveCompact + retry (仅一次)
        else stop hook blocks
            Loop->>Loop: 注入 hook blocking error → continue
        else normal
            Loop->>Loop: return { reason: 'completed' }
        end
    else 有工具调用
        Loop->>Loop: 收集附件 (memory, skill, queued commands)
        opt maxTurns 未超限
            Loop->>Loop: 构造新 State → continue
        else maxTurns 超限
            Loop->>Loop: yield max_turns_reached → return
        end
    end
```

---

### 循环的七个 Continue 站点

`queryLoop` 的核心是一个 `while (true)`，只有 `return` 能退出。每次 `continue` 都是状态机的一个转换：

| # | Transition Reason | 触发条件 | 新状态 |
|---|------------------|---------|-------|
| 1 | `collapse_drain_retry` | prompt-too-long，context collapse 成功 drain | 精简消息，保持同一 turn |
| 2 | `reactive_compact_retry` | collapse 不够/媒体超限，reactive compact 介入 | 编译后消息，设 `hasAttemptedReactiveCompact=true` |
| 3 | `max_output_tokens_escalate` | 输出超限，首次命中 cap | `maxOutputTokensOverride = 64K` |
| 4 | `max_output_tokens_recovery` | 输出超限，注入 recovery message | 追加 meta message，`recoveryCount++` |
| 5 | `stop_hook_blocking` | stop hook 返回 blocking error | 追加 blocking error 到消息 |
| 6 | `token_budget_continuation` | Token 预算超限但仍有 diminishing returns | 注入 nudge message |
| 7 | `next_turn` | 正常工具执行完成，进入下一轮 | 追加 `messages + assistantMessages + toolResults` |

**为何用 `state = next; continue` 而非函数调用**——每个 continue 站点都需要修改不同的 State 子集。如果用函数，需要传递可变引用或返回值。State 对象模式更清晰：每次构造不可变快照，明确写入下一个状态。

---

## 5.2 Tool 抽象类与工具调度

### Tool 接口：900 行定义的工程哲学

`src/Tool.ts` 的 `Tool` 类型约 900 行。这不是接口膨胀——而是一个工具需要满足 12 个关注点的横切需求：

| 关注点 | 接口示例 |
|--------|---------|
| 行为定义 | `call()`, `description()`, `prompt()` |
| 类型安全 | `inputSchema: z.infer<Input>`, `outputSchema` |
| 权限控制 | `checkPermissions()`, `validateInput()`, `isReadOnly()`, `isDestructive()` |
| 并发控制 | `isConcurrencySafe()`, `interruptBehavior()` |
| UI 渲染 | `renderToolUseMessage()`, `renderToolResultMessage()`, `renderGroupedToolUse()` |
| 交互反馈 | `getActivityDescription()`, `getToolUseSummary()` |
| 安全分类 | `toAutoClassifierInput()`, `isOpenWorld()` |
| 工具发现 | `searchHint`, `shouldDefer`, `alwaysLoad` |
| 错误展示 | `renderToolUseErrorMessage()`, `renderToolUseRejectedMessage()` |
| 进度展示 | `renderToolUseProgressMessage()`, `renderToolUseQueuedMessage()` |
| 转录搜索 | `extractSearchText()` |
| MCP 集成 | `mcpInfo`, `inputJSONSchema`, `isMcp` |

**为何不是基类**——如果 `Tool` 是一个抽象类，每个子类被迫继承所有方法，即使是空实现（`abstract class` 强制 override，或 `class` 默认继承）。TypeScript 的结构化接口允许实现者只提供相关的方法，而 `buildTool` 函数填充默认值。

---

### buildTool 工厂：默认值的集中管理

```typescript
// Tool.ts:757-792
const TOOL_DEFAULTS = {
  isEnabled: () => true,
  isConcurrencySafe: (_input?: unknown) => false,  // 默认不安全（fail-closed）
  isReadOnly: (_input?: unknown) => false,           // 默认是写操作
  isDestructive: (_input?: unknown) => false,
  checkPermissions: () => Promise.resolve({ behavior: 'allow' }),
  toAutoClassifierInput: () => '',                  // 默认跳过分类
  userFacingName: () => '',
}

export function buildTool<D extends AnyToolDef>(def: D): BuiltTool<D> {
  return { ...TOOL_DEFAULTS, userFacingName: () => def.name, ...def } as BuiltTool<D>
}
```

**fail-closed 默认值分析**——`isConcurrencySafe` 默认为 `false` 是安全设计。新工具开发者不需要理解并发模型；如果他们忘记设置，工具默认不会并发执行。同样，`isReadOnly` 默认为 `false` 意味着新工具默认需要写权限。这降低了安全漏洞的风险——默认值偏向保守。

---

### 工具调度：流式执行 vs 批处理

在主循环中，工具执行有两种路径：

```typescript
// query.ts:1380-1408
const toolUpdates = streamingToolExecutor
  ? streamingToolExecutor.getRemainingResults()
  : runTools(toolUseBlocks, assistantMessages, canUseTool, toolUseContext)

for await (const update of toolUpdates) {
  if (update.message) {
    yield update.message  // 实时产出工具结果
  }
  if (update.newContext) {
    updatedToolUseContext = { ...update.newContext, queryTracking }
  }
}
```

**为何有 StreamingToolExecutor**——传统 `runTools` 是批处理：收集所有 `tool_use` blocks，按依赖顺序执行，完成后返回结果。StreamingToolExecutor 允许工具在流式到达时就执行——先到达的工具先开始，不需要等整个 assistant message 完成采样。这在工具执行时间长（如 Bash 命令运行 10 秒）的场景下，可以将端到端延迟从 `stream_duration + tool_duration` 降至 `max(stream_duration, tool_duration)`。

---

### ToolUseContext：回调式的上下文传递

```typescript
// Tool.ts（简化）
interface ToolUseContext {
  agentId?: string                      // 所属子 Agent ID
  abortController: AbortController      // 全局中止控制器
  options: {                            // 查询选项
    tools: Tools                        // 当前工具集
    refreshTools?: () => Tools           // 可选的动态刷新
    isNonInteractiveSession: boolean
  }
  readFileState: ReadFileState          // 本轮已读文件追踪（去重内存）
  queryTracking: { depth: number }     // 嵌套查询深度
}
```

`ToolUseContext` 不是简单的配置对象——它是工具与主循环之间的通信通道。`AbortController` 允许工具检测 Ctrl+C 中断，`readFileState` 让上下文压缩系统知道哪些文件已被读取（从而可以安全地丢弃它们的内容，因为模型已经"看过"了）。

---

## 5.3 Turn 管理与状态追踪

### Turn 边界的确定

Turn 是 Claude Code 的核心计量单元。与"对话"（整个 session 的消息序列）不同，turn 是"一轮完整的 query 调用——从用户输入到最终响应"。

```
Session: [Turn 1] ─→ [Turn 2] ─→ [Turn 3] ─→ ...
Turn:    用户输入 → 模型采样 → 工具执行 → 结果回送 → continue?
```

`turnCount` 在 `queryLoop` 初始化时为 1，每次 `next_turn` 转换时递增。它驱动了多个下游功能：

- **Token 预算**：每次 turn 检查是否超过 turn-level budget
- **任务摘要**：`claude ps` 依赖 turn 边界生成摘要
- **自动压缩追踪**：turn 计数器重置时标记紧凑边界

---

### Withheld 模式：延迟暴露的 API 错误

主循环最复杂的工程决策之一是 **withheld error** 机制。某些 API 错误（`max_output_tokens`、`prompt_too_long`）在流式响应中被"扣留"——不在 yield 中暴露给调用方，只在恢复尝试耗尽后才 yield。

```
┌─────────────────────────────────────────────────────────┐
│                    Stream Response                      │
│                                                         │
│  API: 200 OK (content: "...")                          │
│       → 流式产出给调用方                                │
│                                                         │
│  API: 413 (prompt too long, withheld)                  │
│       → 不产出，触发 recovery                           │
│       → contextCollapse.drain retry (先试)             │
│       → reactive compact retry (再试)                  │
│       → 仍失败 → 产出 withheld error → return          │
│                                                         │
│  API: max_output_tokens (withheld)                     │
│       → 不产出                                          │
│       → escalate 8k → 64k (一次)                       │
│       → injection recovery message (最多 N 次)         │
│       → 仍超限 → 产出 withheld error → return          │
└─────────────────────────────────────────────────────────┘
```

**为何不立即暴露**——如果 `prompt_too_long` 立即 yield 给调用方（如 SDK 消费者），消费者会认为请求已完成（因为 stream 结束了），但实际错误是可恢复的。Withheld 机制保证调用方只看到最终结果：要么成功恢复的响应，要么确实无法恢复的错误。这是一个状态封装决策。

---

### Stop Hook：主循环的扩展点

```typescript
// query.ts:1267-1306
const stopHookResult = yield* handleStopHooks(
  messagesForQuery, assistantMessages, systemPrompt,
  userContext, systemContext, toolUseContext, querySource, stopHookActive,
)

if (stopHookResult.preventContinuation) {
  return { reason: 'stop_hook_prevented' }
}

if (stopHookResult.blockingErrors.length > 0) {
  // 注入 blocking error → continue，允许下一次迭代尝试修复
  state = {
    messages: [...messagesForQuery, ...assistantMessages, ...stopHookResult.blockingErrors],
    hasAttemptedReactiveCompact,  // 保留 guard —— 重置会导致无限循环
    // ...
    transition: { reason: 'stop_hook_blocking' },
  }
  state = next
  continue
}
```

**为什么保留 `hasAttemptedReactiveCompact` guard**——注释记录了一个真实的 bug：如果 stop hook 返回 blocking error 时将 `hasAttemptedReactiveCompact` 重置为 `false`，循环会进入无限重试：compact → 仍然 too long → error → stop hook blocking → compact → … 消耗数千次 API 调用。这是通过修复状态 guard 而非修改 hook 逻辑来解决的——根因在状态重置，不在 hook 执行。

---

### 附件注入：Memory、Skills 和命令队列

在每个 turn 进入递归调用之前，主循环会收集多种来源的"附件"消息，将它们追加到 tool results 中：

```typescript
// query.ts:1580-1590
for await (const attachment of getAttachmentMessages(
  null, updatedToolUseContext, null, queuedCommandsSnapshot,
  [...messagesForQuery, ...assistantMessages, ...toolResults],
  querySource,
)) {
  yield attachment
  toolResults.push(attachment)
}
```

附件来源包括：
1. **Memory 预取结果**——`pendingMemoryPrefetch.consumedOnIteration` 控制每轮只消费一次
2. **Skill 发现**——`skillPrefetch.collectSkillDiscoveryPrefetch()` 异步返回隐藏的技能
3. **命令队列**——queued commands（优先级排序）转换为 attachment 注入
4. **文件变更通知**——`edited_text_file` 附件通知模型文件已被外部编辑

**为什么附件在工具执行之后**——如果附件先于工具结果注入，API 会拒绝交织的 `tool_result` 和 `user` 消息。附件是用户消息类型，必须在所有工具结果之后。

---

### 多 Agent 隔离：主线程 vs 子 Agent

```typescript
// query.ts:1567-1578
const isMainThread =
  querySource.startsWith('repl_main_thread') || querySource === 'sdk'
const currentAgentId = toolUseContext.agentId
const queuedCommandsSnapshot = getCommandsByMaxPriority(
  sleepRan ? 'later' : 'next',
).filter(cmd => {
  if (isSlashCommand(cmd)) return false           // 排除 slash 命令
  if (isMainThread) return cmd.agentId === undefined  // 主线程只收无 agentId 的命令
  return cmd.mode === 'task-notification' && cmd.agentId === currentAgentId
})
```

**命令队列的隔离设计**——命令队列是一个进程全局单例，由协调器和所有进程内子 Agent 共享。每个循环只消费属于自己的命令：主线程消费 `agentId === undefined` 的命令，子 Agent 消费 `agentId` 匹配自己的通知。用户提示（`mode: 'prompt'`）始终只发送给主线程，子 Agent 永远不会看到用户的 prompt 流。

---

### 模型降级：FallbackTriggeredError 与签名剥离

```typescript
// query.ts（在 catch 块中）
if (error instanceof FallbackTriggeredError && fallbackModel) {
  // 清除所有累积的消息（降级模型可能不兼容 thinking 块签名）
  state.messages = state.messages.filter(block => !isThinkingBlock(block))
  // 切换到 fallback 模型
  currentModel = fallbackModel
  // 重新发送请求
  continue
}
```

**为什么清除所有消息**——thinking 模型（如 Opus 4.5+）在 assistant message 中注入 `<anththinking>` 块。如果 fallback 模型（如 Bedrock 上的 Sonnet）不支持 thinking 格式，这些块会被 API 拒绝。`stripSignatureBlocks` 递归移除所有 thinking 签名块，保留核心内容。这是一个兼容性桥接决策——牺牲上下文完整性换取跨模型兼容性。


---

## 5.5 Query 状态机的 Continue 模式

```typescript
// query.ts:203-217
type State = {
  messages: Message[]
  toolUseContext: ToolUseContext
  autoCompactTracking: AutoCompactTrackingState | undefined
  maxOutputTokensRecoveryCount: number
  hasAttemptedReactiveCompact: boolean
  maxOutputTokensOverride: number | undefined
  pendingToolUseSummary: Promise<ToolUseSummaryMessage | null> | undefined
  stopHookActive: boolean | undefined
  turnCount: number
  transition: Continue | undefined  // WHY the previous iteration continued
}
```

**`transition` 字段的设计意图**——记录上一轮继续的原因。这允许子系统检查是否已经触发过恢复路径（如 `state.transition?.reason !== 'collapse_drain_retry'`）。没有这个字段，系统可能在同一轮中反复触发同恢复，造成循环。

### State 更新模式

注释明确指出（line 266-267）：
> Continue sites write `state = { ... }` instead of 9 separate assignments.

```typescript
// 每轮底部的原子状态更新：
state = {
  messages: [...messagesForQuery, ...assistantMessages, ...toolResults],
  toolUseContext: toolUseContextWithQueryTracking,
  autoCompactTracking: tracking,
  turnCount: nextTurnCount,
  maxOutputTokensRecoveryCount: 0,
  hasAttemptedReactiveCompact: false,
  pendingToolUseSummary: nextPendingToolUseSummary,
  maxOutputTokensOverride: undefined,
  stopHookActive,
  transition: { reason: 'next_turn' },
}
// Falls through to while(true) top -- next iteration
```

这是函数式状态管理的核心——整个 `State` 对象被替换，而不是逐个字段修改。这保证了状态的一致性：如果某一轮在更新一半时出错，不会有部分更新的状态泄漏。

---

## 5.6 输入提交管线：HandlePromptSubmit

```typescript
// handlePromptSubmit.ts:120-387
async function handlePromptSubmit():
  1. 如果提供 queuedCommands → 直接 executeUserInput
  2. 空输入 → 立即返回
  3. 处理退出命令 (exit, quit, :q 等)
  4. 展开文本引用
  5. 处理活动查询中的即时 local-jsx 斜杠命令
  6. 如果 query guard 活跃（有查询在运行）:
     - 如果所有执行中的工具可中断 → abort 当前轮
     - 将命令加入队列（带优先级）
  7. 如果不活跃 → 创建 QueuedCommand 并传递给 executeUserInput
```

### Query Guard：并发查询防护

```typescript
// QueryGuard 状态机
class QueryGuard {
  active: boolean
  reserve(): boolean   // 如果活跃返回 false
  end(): void
}
```

**为何需要 guard**——如果用户快速连续按 Enter，可能触发多次并发查询。Guard 阻止这种情况——如果查询已在运行，guard 返回 false，新命令被排队而不是并发执行。

### ExecuteUserInput 的 Abort 语义

```typescript
// handlePromptSubmit.ts:396-610
async function executeUserInput(commands):
  1. 创建新的 AbortController
  2. 保留 query guard
  3. 遍历所有队列命令，通过 processUserInput() 处理:
     - 处理斜杠命令
     - 处理 bash 命令
     - 处理文本提示
     - 返回 { messages, shouldQuery, allowedTools, model, effort, ... }
  4. 获取文件历史快照
  5. 调用 onQuery() 运行模型
  6. finally: 取消 guard 预留 + 清理占位符
```

**文件历史快照**——`takeFileHistorySnapshot()` 在查询前拍摄当前文件系统状态。这使得工具执行后可以回滚文件变更。快照不捕获文件内容——只捕获文件的存在/不存在、mtime 等元数据。

---

## 5.7 终止条件全貌

`queryLoop` 通过以下路径退出（返回 `Terminal`）：

| 原因 | 条件 | 行为 |
|------|------|------|
| `blocking_limit` | Token 数超过硬阻塞限制 | 自动压缩关闭时的最终防线 |
| `image_error` | 图像大小/调整错误或有保留媒体 | 无法处理的媒体错误 |
| `model_error` | 流式传输期间未处理异常 | 致命 API 错误 |
| `aborted_streaming` | Abort controller 在流式传输期间触发 | 用户中断 |
| `prompt_too_long` | 所有恢复路径耗尽 | 413 错误无法恢复 |
| `completed` | 无 tool_use + 无 stop hook 阻塞 | 正常完成 |
| `stop_hook_prevented` | Stop hook 阻止继续 | Hook 主动停止 |
| `aborted_tools` | 用户取消工具执行 | 用户中断 |
| `max_turns` | 轮次超过可配置最大值 | 安全限制 |
| `max_output_tokens_escalate` | 从 8k 升级到 64k | OTK 升级恢复 |

### 恢复路径（不退出，继续循环）

| 原因 | 效果 |
|------|------|
| `reactive_compact_retry` | Reactive compact 触发 → 继续 |
| `collapse_drain_retry` | 排出暂存压缩 → 继续 |
| `token_budget_continuation` | 预算允许 → 注入 nudge 继续 |
| `next_turn` | 正常工具结果 → 下一轮 |
| `max_output_tokens_recovery` | 超限 → 注入恢复消息继续 |

**`max_turns` 安全网**——默认没有限制，但可以配置。这是防止无限循环的关键安全网。如果模型进入工具调用循环（如反复尝试同一个失败的 Bash 命令），`max_turns` 最终会终止循环。

---

## 5.8 工具执行管线

### Streaming Tool Executor vs runTools

```typescript
// query.ts:1360-1409
// 两种工具执行路径:
if streamingToolExecutor active (tengu_streaming_tool_execution2):
  toolResults = streamingToolExecutor.getRemainingResults()
else:
  toolResults = runTools(toolUseBlocks, assistantMessages, canUseTool, toolUseContext)
```

**Streaming Tool Executor**——当 `tengu_streaming_tool_execution2` gate 启用时，工具在流式到达时就开始执行（不等完整响应）。这使工具执行时间与模型流式输时间重叠，降低端到端延迟。

**`getRemainingResults()`**——获取尚未产出结果的工具的最终结果。这些工具可能在流式传输结束前已经开始执行了。

### 工具更新的事件循环

```typescript
// 遍历工具结果:
for await (const update of toolUpdates):
  yield result message         // 转发到 UI
  accumulate toolResults[]     // 累积用于下一轮
  track updated toolUseContext  // 跟踪文件/内存变更
```

`toolUseContext` 的跟踪是关键——它记录哪些文件被读取、编辑、删除。这使得压缩后系统可以重新注入最近访问的文件。

---

## 5.9 轮次附件与后处理

### Phase 7: Post-Tool Attachments

```typescript
// query.ts:1547-1671
// 工具完成后:
1. 排空队列命令（任务通知优先级更高，如果没有 Sleep 工具）
2. 获取附件消息（文件差异、编辑、IDE 上下文等）
3. 消费预取内存预取结果
4. 注入预取技能发现结果
5. 消费并移除已转换为附件的队列命令
6. 刷新工具（新连接的 MCP 服务器可用）
```

### 预取机制

| 预取类型 | 启动时机 | 注入时机 |
|---------|---------|---------|
| Session memory 预取 | 查询入口 | 转结束时 |
| Skill discovery 预取 | 查询入口前 | Phase 7 |
| Skill listing 预取 | 新工作目录 | Phase 7 |

**为何用预取而非同步加载**——这些操作是 I/O 密集型的（读文件系统、扫描技能目录）。如果同步执行，它们会阻塞 API 调用。预取让它们在 API 调用和工具执行的空闲时间内运行。

### 轮次指标重置与消费

```typescript
// REPL.tsx:2790-2792 - 查询开始时重置
resetTurnHookDuration()
resetTurnToolDuration()
resetTurnClassifierDuration()

// REPL.tsx:2826-2843 - 查询结束后消费
const hookMs = getTurnHookDurationMs()     // SDK hook 执行时间
const hookCount = getTurnHookCount()       // Hook 执行次数
const toolMs = getTurnToolDurationMs()     // 工具执行时间
const toolCount = getTurnToolCount()       // 工具调用次数
const classifierMs = getTurnClassifierDurationMs()
const classifierCount = getTurnClassifierCount()
const turnMs = Date.now() - loadingStartTimeRef.current
```

这些指标被渲染为 `ApiMetricsMessage`（内部版本）——用户可以看到每轮的详细性能分析。

---

## 5.10 Thinking 在 Turn Loop 中的传递

```
main.tsx:2457-2488
  └── REPL props
        └── getToolUseContext options (REPL.tsx)
              └── query() params
                    └── deps.callModel call (query.ts:662)
```

**Thinking 配置类型**：
- `{ type: 'disabled' }` — 关闭思考
- `{ type: 'adaptive' }` — 模型自动决定
- `{ type: 'enabled', budgetTokens: N }` — 明确预算

### Thinking 的规则（query.ts:152-163）

1. 带有 thinking/redacted_thinking 块的消息需要 `max_thinking_length > 0`
2. 思考块不能是最后一条消息
3. 思考块必须保留给助手轨迹（单轮，或如果有 tool_use 则也包括其 tool_result 和后面的助手消息）

**流式回退时的 Thinking 剥离**（行 928，内部版本）：
```typescript
// 内部构建中，Thinking 签名在重试前被剥离
// 这是为了确保回退模型不会收到不兼容的 thinking 配置
```

---

## 5.11 Stream-End 中止处理

```typescript
// query.ts:1011-1052
// 如果 abort controller 在流式传输期间触发:
1. 消耗 streaming executor 中的剩余结果（生成合成 tool_results）
2. 清理 computer-use 锁（Chicago MCP 特性）
3. 生成用户中断消息
4. 返回 { reason: 'aborted_streaming' }
```

**合成 tool_results**——如果用户在中途取消，某些工具可能已经开始但未完成。`executePostSamplingHooks` 生成合成的错误结果，确保 API 消息序列有效（每个 `tool_use` 都有配对的 `tool_result`）。

**Computer-use 锁清理**——Chicago MCP 的 computer-use 特性使用分布式锁来协调屏幕/键盘/鼠标访问。如果会话被中止，锁必须被释放，否则它会导致其他会话或后续尝试被永久阻塞。

---

## 5.12 命令队列机制

```typescript
// messageQueueManager.ts
// 命令队列支持优先级调度
// 当查询运行时，新命令被排队而不是立即执行
```

**带优先级的队列**——某些命令（如任务通知）优先级高于用户输入。这确保了系统在工具执行期间仍能处理高优先级事件。

**队列处理时机**——在 turn 之间（REPL 的 `useQueueProcessor` hook）。这意味着队列在模型响应完成后、下一轮开始前被处理。

**可中断工具**——如果所有执行中的工具都是可中断的，用户输入可以 abort 当前轮。不可中断工具（如长运行的 Bash 命令）阻止中断——用户必须等待工具完成或手动 kill 进程。

---

## 5.13 三层嵌套架构

Turn loop 不是单层循环，而是三层嵌套结构：

**Sk（Agent 感知 turn 包装器）**——`QueryEngine.ts`：解析模型、构建系统提示、构建工具权限上下文、加载 hooks/skills/MCP 工具、过滤悬挂的 fork-context `tool_use` 块、调用 OS、用侧链转录写入产出过滤后的事件、执行 finally 清理。

**PF（核心 while-loop 状态机）**——`query.ts` 的 `queryLoop()`：跨迭代维护可变状态、处理预采样归一化、microcompact、autocompact、采样、工具分支。

**RG8（异步任务包装器）**——消费 Sk 的事件、更新用量追踪、管理任务状态、输出 completed/failed/killed。

---

## 5.14 QueryGuard 三态并发保护

`QueryGuard` 类是同步状态机，恰好三个状态：

```
idle --> dispatching  (reserve())
dispatching --> running (tryStart())
idle --> running (tryStart(), 直接用户提交)
running --> idle (end(generation) / forceEnd())
dispatching --> idle (cancelReservation())
```

**关键方法**：
- `reserve()`：`idle → dispatching`。如非 idle 返回 false
- `tryStart()`：成功返回 generation 号，已在运行返回 null。接受从 idle 和 dispatching 的转换
- `end(generation)`：返回布尔值——generation 是否仍是当前。返回 false 表示新查询已启动，过时的 finally 块跳过清理
- `forceEnd()`：由 `onCancel` 使用。递增 generation 使过时的 finally 块看到不匹配

**`_generation` 计数器**防止竞争条件——被取消查询的 finally 块不会清理更新的替换查询。

---

## 5.15 QueryLoop 状态机

核心循环状态定义：

```typescript
type State = {
  messages: Message[]
  toolUseContext: ToolUseContext
  autoCompactTracking: AutoCompactTrackingState
  maxOutputTokensRecoveryCount: number   // 最多 3 次
  hasAttemptedReactiveCompact: boolean
  maxOutputTokensOverride: number
  pendingToolUseSummary: Promise<ToolUseSummaryMessage | null>
  stopHookActive: boolean
  turnCount: number
  transition: Continue                    // 前一轮为何继续
}
```

**Transition 类型**：

| Transition | 原因 |
|-----------|------|
| `next_turn` | 正常递归 |
| `max_output_tokens_recovery` | 重试（最多 3 次） |
| `max_output_tokens_escalate` | 单次升级到 64k |
| `reactive_compact_retry` | 反应式压缩重试 |
| `collapse_drain_retry` | 塌陷排水重试 |
| `stop_hook_blocking` | Stop hook 阻塞 |
| `token_budget_continuation` | Token 预算继续 |

**Terminal 原因**：`completed`、`aborted_streaming`、`aborted_tools`、`prompt_too_long`、`image_error`、`blocking_limit`、`model_error`、`stop_hook_prevented`、`max_turns`。

---

## 5.16 每轮预采样管线

每次循环迭代运行有序管线：

1. **Skill discovery prefetch**——`skillPrefetch.startSkillDiscoveryPrefetch()`
2. **产出 `stream_request_start`**
3. **Query tracking** 初始化：chainId 和 depth
4. **`getMessagesAfterCompactBoundary(messages)`**——切片最后压缩边界后的消息
5. **工具结果预算执行**——`applyToolResultBudget()`——在微观压缩前强制执行每消息预算
6. **Snip compact**（特性门控）——`snipModule.snipCompactIfNeeded()`
7. **Microcompact**——`deps.microcompact()`——轻量消息减少
8. **Context collapse**（特性门控）——`contextCollapse.applyCollapsesIfNeeded()`——在 autocompact 前运行，使塌陷能将我们降到 autocompact 阈值以下
9. **Autocompact**——`deps.autocompact()`——完全对话摘要
10. **工具上下文更新**——`toolUseContext = { ...toolUseContext, messages: messagesForQuery }`
11. **Token 阻塞限制检查**——未压缩时检查 `calculateTokenWarningState()`，超过则返回 `'blocking_limit'`

---

## 5.17 Streaming 工具执行器

**并发规则**（`canExecuteTool()`）：
- 如无工具正在执行，可执行
- 或新工具**和**所有当前执行的工具都是并发安全的
- 非并发安全工具按顺序执行，阻塞后续工具

**Sibling abort cascade**——仅 Bash 工具错误取消兄弟（line 359）：
```typescript
if (tool.block.name === BASH_TOOL_NAME) {
  this.hasErrored = true
  this.siblingAbortController.abort('sibling_error')
}
```

Read/WebFetch 失败是独立的——一个失败不会取消其余。

**Abort 控制器层级**：
```
main query abortController
  └── siblingAbortController (工具批次级)
        └── toolAbortController (逐工具级)
```

使用 `WeakRef` 防止内存泄漏。父级中止级联到子级。子级中止不影响父级。

---

## 5.18 Streaming 内工具执行

工具在流式响应中**立即**执行：

```
while (attemptWithFallback) {
  // 每个流式消息检查 tool_use 块
  if (tool_use found) {
    toolUseBlocks.push(toolBlock)
    needsFollowUp = true
    // 立即送入 streamingToolExecutor
    streamingToolExecutor.addTool(toolBlock, message)
  }
}
```

这是并发工具执行的关键——工具在 API 仍在流式传输结果时就开始执行。

**事件扣留用于可恢复错误**——`withheld` 标志门控 yield：
- 检查：context collapse、reactive compact、media size error、`max_output_tokens`
- 被扣留的错误**不**发送给 SDK 调用者（SDK 在 `error` 字段上终止）
- 错误**仍**推入 `assistantMessages` 用于流后恢复检查

**流式回退处理**：
- `onStreamingFallback` 回调设置 `streamingFallbackOccurred = true`
- 检测时：标记所有 `assistantMessages`、丢弃 `streamingToolExecutor`、创建新执行器、重置累加器
- 如模型变更，重试前剥离 thinking 块

---

## 5.19 工具调用管线：完整链

从 tool_use 到 tool_result 的完整链：

1. **工具查找**——`findToolByName()`，遗留工具的别名回退
2. **取消检查**——启动前检查 abortController
3. **`streamedCheckPermissionsAndCallTool()`**：
   a. **Zod 验证**——`tool.inputSchema.safeParse(input)`
   b. **值验证**——`tool.validateInput(parsedInput.data, toolUseContext)`
   c. **投机分类器检查**（bash）——与 hooks 并行
   d. **纵深剥离**——从模型提供的输入中剥离 `_simulatedSedEdit`
   e. **回填可观察输入**——浅克隆带扩展路径
   f. **PreToolUse hooks**——异步生成器产出消息、hookPermissionResult、hookUpdatedInput、preventContinuation、stopReason
   g. **权限决策解析**——`resolveHookPermissionDecision()`——组合 hook 结果与 canUseTool
   h. **OTel 日志**——`tool_decision` 事件
   i. **权限拒绝处理**——创建带 tool_result 的用户消息
   j. **tool.call()**——`await tool.call(callInput, ...)`
   k. **PostToolUse hooks**——MCP 工具可修改输出
   l. **错误路径**——`runPostToolUseFailureHooks` 带 isInterrupt 标志

---

## 5.20 Tool Hook 系统

**`runPreToolUseHooks`** 产出 7 种判别联合变体：

| 类型 | 用途 |
|------|------|
| `message` | 进度或附件消息 |
| `hookPermissionResult` | allow/deny/ask 带 decisionReason |
| `hookUpdatedInput` | 输入修改无权限决策 |
| `preventContinuation` | 阻止继续 |
| `stopReason` | 停止原因 |
| `additionalContext` | 附加上下文消息 |
| `stop` | 完全中止执行 |

**`resolveHookPermissionDecision`** 是关键连接点：
- Hook 允许 → **仍**检查 `checkRuleBasedPermissions()`（拒绝规则始终适用，即使在 hook 允许时）
- Hook 拒绝 → 最终
- Hook 询问或无决策 → 正常 `canUseTool()` 流程

**`runPostToolUseHooks`**：
- 可修改 MCP 工具输出
- 可产出 `blockingError` 附件
- 可设置 `preventContinuation` 终止 turn
- 期间 hook 错误产出 `hook_error_during_execution` 附件

---

## 5.21 Post-Streaming 决策树

流结束后，代码分支通过几个决策路径：

**中止检查**——消费 `getRemainingResults()` 生成孤立工具的合成 tool_results，清理 Chicago MCP（computer use 锁），产出中断消息。

**Pending tool use summary**——前一轮异步 Haiku 生成的摘要，解析并产出。

**无 follow-up 路径**（`!needsFollowUp`）：
- 被扣留的 prompt-too-long：先尝试 context collapse drain，再 reactive compact
- 被扣留的 media size error：仅 reactive compact（塌陷不剥离图像）
- 被扣留的 max_output_tokens：
  - 单次升级：8k 默认 → 64k `ESCALATED_MAX_TOKENS`
  - 多轮恢复：注入 "Resume directly" 消息，最多 3 次

**Follow-up 路径**（`needsFollowUp`）：
- 两种执行模式：`StreamingToolExecutor.getRemainingResults()`（异步生成器）或 `runTools`（同步模式）

---

## 5.22 `runTools` 分区算法

`toolOrchestration.ts` 中的工具分区：
- 将工具调用分组为批次
- 连续并发安全工具批处理在一起
- 非并发安全工具串行运行（每批一个）
- 最大并发：`parseInt(process.env.CLAUDE_CODE_MAX_TOOL_USE_CONCURRENCY || '10')`

---

## 5.23 Stop Hooks 管线

`handleStopHooks()` 在每次 turn 完成后无 tool_use 时运行：

1. 为旁观者查询缓存安全参数
2. Job 分类器（特性门控，60 秒超时）
3. 后台记账（非 bare 模式）：
   - Prompt suggestion
   - Memory extraction
   - Auto-dream
4. Computer use 清理（Chicago MCP）
5. `executeStopHooks()`——产出进度、阻塞错误、preventContinuation 信号
6. **TeammateIdle hooks**——如为 teammate，运行 TaskCompleted 和 TeammateIdle hooks

Stop hook 阻塞错误创建 meta 用户消息作为工具结果推入，导致下一轮迭代将它们视为用户输入。

---

## 5.24 Turn 续接与状态重置

循环续接点（行 1714）：

```typescript
const next: State = {
  messages: [...messagesForQuery, ...assistantMessages, ...toolResults],
  turnCount: nextTurnCount,           // 递增
  maxOutputTokensRecoveryCount: 0,    // 重置
  hasAttemptedReactiveCompact: false, // 重置
  maxOutputTokensOverride: undefined,  // 升级后重置
  transition: { reason: 'next_turn' }
}
```

**Thinking 块规则**——四种不变量：
1. 带 thinking/redacted_thinking 块的消息必须在 `max_thinking_length > 0` 的查询中
2. Thinking 块**不能**是块中的最后一条消息
3. Thinking 块必须在 assistant 轨迹期间保留（单 turn，或 turn + tool_use + tool_result + 后续 assistant 消息）
4. `MAX_OUTPUT_TOKENS_RECOVERY_LIMIT = 3`

---

## 5.25 错误恢复路径汇总

| 错误 | 恢复机制 | 最大尝试 |
|------|---------|---------|
| 上下文过长（主动） | autocompact | 无限（熔断：连续 3 次失败） |
| 上下文过长（反应式/PTL） | collapse drain → reactive compact | 各 1 次 |
| Media size error | 仅 reactive compact | 1 次 |
| max_output_tokens（首次） | 升级 8k → 64k | 1 次 |
| max_output_tokens（回退） | 注入 "resume directly" | 3 次 |
| FallbackTriggeredError | 切换到 fallbackModel | 1 次 |
| Stop hook blocking error | 注入结果消息作为用户输入 | 无界 |
| API 错误（不可恢复） | 产出错误消息，退出 | N/A |
| 流式中止 | 所有 pending 工具的合成 tool_results | N/A |

---
