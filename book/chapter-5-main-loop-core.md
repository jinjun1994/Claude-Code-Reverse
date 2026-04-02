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
