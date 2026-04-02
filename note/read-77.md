## 第 77 站：`src/utils/hooks/execAgentHook.ts`

### 这是什么文件

`src/utils/hooks/execAgentHook.ts` 是 hooks 子系统里的 **agentic verifier hook 执行器：负责把一个 hook prompt 交给一个临时 mini-agent 循环执行，该 agent 可以使用现有工具（除受限工具外）来验证某个条件是否满足，并通过结构化输出工具返回 `{ok: true/false, reason?: string}` 结果。**

这一站是三大特殊 hook 执行器中的第三站。此前已经看懂了：

- HTTP hook -> `execHttpHook.ts`
- Prompt hook -> `execPromptHook.ts`

Agent hook 与前两种完全不同：它不是一个简单的单次 LLM 调用，而是一个完整的多轮 agent 循环。

所以最准确的一句话是：

```text
execAgentHook.ts = agent hook 的多轮验证执行层：负责构造一个临时 mini-agent，传入工具集和 transcript 读取权限，在最大 50 轮内通过结构化输出返回验证结果
```

---

### 先看它的整体定位

位置：`src/utils/hooks/execAgentHook.ts:36`

这个文件不是：

- agent hook schema
- 通用 agent 框架
- 完整的 agent worker

它的职责是：

1. 展开 `$ARGUMENTS` prompt 模板
2. 构造一个 mini-agent 执行上下文
3. 配置工具集（可用工具 - disallowed + structured output tool）
4. 给 agent 设置读取 transcript 的权限
5. 运行 agent query 循环（最多 50 turns）
6. 从结构化输出附件中提取验证结果
7. 根据结果返回 blocking/success/cancelled
8. 清理 session hooks 和信号

所以它本质上是一个：

```text
multi-turn agentic verifier for hook conditions
```

---

### 第一部分：Agent hook 与 prompt hook 的关键区别

位置：函数签名和注释对比

#### prompt hook
- 一次 LLM 查询
- 用 system prompt 约束 JSON 返回
- 不需要工具
- 默认 30s 超时

#### agent hook
- 多轮 agent 循环（最多 50 turns）
- 可以使用所有可用工具
- 需要用结构化输出工具来正式返回结果
- 默认 60s 超时
- 可以读取对话 transcript

这说明 agent hook 的设计目标是：

```text
multi-turn verification using available tools
```

而不是简单的单次判定。

---

### 第二部分：消息与 prompt 模板

位置：`src/utils/hooks/execAgentHook.ts:60`

```ts
const processedPrompt = addArgumentsToPrompt(hook.prompt, jsonInput)
```

和 prompt hook 一样，agent hook 的 prompt 模板也通过 `$ARGUMENTS` 占位符接收
hook 输入 JSON。

然后：

```ts
const userMessage = createUserMessage({ content: processedPrompt })
const agentMessages = [userMessage]
```

注意这里不追加历史对话消息，
因为 agent hook 是独立的多轮验证循环，
不是查询已有上下文。

---

### 第三部分：系统提示

位置：`src/utils/hooks/execAgentHook.ts:107`

```ts
const systemPrompt = asSystemPrompt([
  `You are verifying a stop condition in Claude Code.
Your task is to verify that the agent completes the given plan.
The conversation transcript is available at: ${transcriptPath}
You can read this file to analyze the conversation history if needed.

Use the available tools to inspect the codebase and verify the condition.
Use as few steps as possible - be efficient and direct.

When done, return your result using the ${SYNTHETIC_OUTPUT_TOOL_NAME} tool with:
- ok: true if the condition is met
- ok: false with reason if the condition is not met`,
])
```

这说明 agent hook 的系统提示定义了：

1. agent 的角色：验证器
2. agent 的资源：transcript 文件路径
3. agent 的行为准则：高效直接
4. agent 的结束方式：通过结构化输出工具返回结果

---

### 第四部分：权限配置是 agent hook 的关键架构点

位置：`src/utils/hooks/execAgentHook.ts:125`

```ts
const agentToolUseContext: ToolUseContext = {
  ...toolUseContext,
  agentId: hookAgentId,
  abortController: hookAbortController,
  options: {
    ...toolUseContext.options,
    tools,
    mainLoopModel: model,
    isNonInteractiveSession: true,
    thinkingConfig: { type: 'disabled' },
  },
  setInProgressToolUseIDs: () => {},
  getAppState() {
    const appState = toolUseContext.getAppState()
    const existingSessionRules = appState.toolPermissionContext.alwaysAllowRules.session ?? []
    return {
      ...appState,
      toolPermissionContext: {
        ...appState.toolPermissionContext,
        mode: 'dontAsk',
        alwaysAllowRules: {
          ...appState.toolPermissionContext.alwaysAllowRules,
          session: [...existingSessionRules, `Read(/${transcriptPath})`],
        },
      },
    }
  },
}
```

这段代码做了三件事：

1. 创建一个唯一的 hook agent ID
2. 设置 `mode: 'dontAsk'` 不询问权限（因为它不应该弹权限对话框）
3. 追加一条 session 规则：总是允许读取 transcript 文件

这很关键：

```text
agent hook 的 agent 不能弹权限对话框，
但需要能够读取当前会话的 transcript
```

---

### 第五部分：StructuredOutput 工具

位置：`src/utils/hooks/execAgentHook.ts:89`

```ts
const structuredOutputTool = createStructuredOutputTool()
```

然后在过滤工具后添加：

```ts
const tools: Tool[] = [
  ...filteredTools.filter(
    tool => !ALL_AGENT_DISALLOWED_TOOLS.has(tool.name)
  ),
  structuredOutputTool,
]
```

这说明：

1. agent 可以使用除受限工具外的所有可用工具
2. 结构化输出工具是额外附带的专门工具
3. 需要过滤掉已有的同名工具避免冲突

---

### 第六部分：Agent 主循环

位置：`src/utils/hooks/execAgentHook.ts:167`

```ts
for await (const message of query({
  messages: agentMessages,
  systemPrompt,
  userContext: {},
  systemContext: {},
  canUseTool: hasPermissionsToUseTool,
  toolUseContext: agentToolUseContext,
  querySource: 'hook_agent',
})) {
```

这是真正的 agent 循环：

1. 调用 `query(...)` 获取消息流
2. 处理每个消息：
   - 跳过流事件
   - 计数 assistant turns
   - 检查是否达到 50 turn 限制
   - 检查是否收到 `structured_output` 附件
3. 收到结构化输出 -> abort 并退出循环
4. 超过 turn 限制 -> abort 并退出循环

---

### 第七部分：结构化输出提取

位置：`src/utils/hooks/execAgentHook.ts:212`

```ts
if (
  message.type === 'attachment' &&
  message.attachment.type === 'structured_output'
) {
  const parsed = hookResponseSchema().safeParse(message.attachment.data)
  if (parsed.success) {
    structuredOutputResult = parsed.data
    hookAbortController.abort()
    break
  }
}
```

这说明 agent hook 的合法返回方式是：

```text
通过 structured_output attachment 返回 { ok, reason }
```

而不是纯文本 JSON（像 prompt hook 那样）。

这也解释了为什么前面需要 `createStructuredOutputTool()` 和
`registerStructuredOutputEnforcement(...)`——
这些机制确保 agent 被强制使用结构化输出工具来结束验证。

---

### 第八部分：Turn 限制

位置：`src/utils/hooks/execAgentHook.ts:119`

```ts
const MAX_AGENT_TURNS = 50
```

这个值防止 agent hook 无限制地运行。

如果达到 50 turns：

```ts
hitMaxTurns = true
hookAbortController.abort()
```

然后返回 `cancelled`。

---

### 第九部分：清理

位置：`src/utils/hooks/execAgentHook.ts:233`

```ts
clearSessionHooks(toolUseContext.setAppState, hookAgentId)
```

这说明 agent hook 在执行期间可能注册了 session-level hooks
（比如用于强制结构化输出），
完成后必须全部清理。

---

### 第十部分：结果返回策略

位置：`src/utils/hooks/execAgentHook.ts:236-303`

#### 没有结构化输出
- 如果 hit max turns -> `cancelled`
- 否则（未完成）-> `cancelled`

#### 有结构化输出
- `ok: true` -> `success`
- `ok: false` -> `blocking` + `blockingError`

这与 prompt hook 的结果类型一致。

---

### 第十一部分：Analytics 事件

位置：多处

```ts
logEvent('tengu_agent_stop_hook_max_turns', {...})  // 达到最大 turns
logEvent('tengu_agent_stop_hook_error', {...})     // 一般错误
logEvent('tengu_agent_stop_hook_success', {...})   // 成功
```

这说明 agent hook 的执行被详细跟踪用于分析。

---

### 第十二部分：信号处理

位置：`src/utils/hooks/execAgentHook.ts:76-85`

```ts
const hookAbortController = createAbortController()
const { signal: parentTimeoutSignal, cleanup: cleanupCombinedSignal } =
  createCombinedAbortSignal(signal, { timeoutMs: hookTimeoutMs })
const onParentTimeout = () => hookAbortController.abort()
parentTimeoutSignal.addEventListener('abort', onParentTimeout)
const combinedSignal = hookAbortController.signal
```

这里的信号链比较复杂：

1. 创建一个 agent 专用的 abort controller
2. 父信号加超时组合成一个组合信号
3. 当组合信号 abort 时，触发 agent controller abort
4. agent 使用 agent controller 的 signal

这比简单使用组合信号更复杂，
可能是为了 agent 内部需要独立的 controller 来控制。

---

### 元

问题：**这一站真正想解决的架构问题是什么？**

回答：这一站真正要解释的是，为什么有些 hook 条件检查必须升级成一个临时 mini-agent，而不是单次 prompt。它需要多轮工具使用、结构化输出和上限控制，才能完成更复杂的验证任务。

### 反

问题：**如果把这一站的设计反过来，会发生什么？**

回答：如果把 agent hook 压回单轮模型调用，很多需要查文件、读 transcript、逐步验证的条件都无法可靠成立。相反若不加约束地放开，又会把 hook 变成无限扩张的小代理。

### 空

问题：**跳出当前文件名，这一站背后更大的问题是什么？**

回答：更大的问题，是系统何时该从“回答问题”转向“执行验证过程”。`execAgentHook.ts` 的价值，是把这种升级控制在正式框架里。

### 读完这一站后，你应该抓住的 10 个事实

1. `src/utils/hooks/execAgentHook.ts` 不是简单的单次 LLM 查询，而是一个完整的多轮 agent 循环执行器。
2. agent hook 通过 `$ARGUMENTS` 占位符接收 hook 输入 JSON，与 prompt hook 使用相同的模板系统。
3. agent hook 默认 60 秒超时，比 prompt hook（30s）多一倍，因为需要多轮工具执行。
4. agent hook 的系统提示定义 agent 角色为"验证器"，可以使用 transcript 文件来验证条件是否满足。
5. agent hook 的 `mode: 'dontAsk'` 确保不会弹权限对话框，并且为 transcript 读取添加了 session-level always-allow 规则。
6. agent hook 可以使用除受限工具外的所有可用工具，外加一个专门的结构化输出工具来正式返回结果。
7. Agent 循环最多 50 turns；达到 turn 限制时 abort 并返回 `cancelled`。
8. agent hook 通过 `structured_output` 附件提取结果，而不是解析响应文本中的 JSON。
9. agent hook 执行后清理 `clearSessionHooks(hookAgentId)`，避免注册的任何 session hooks 泄漏到后续操作。
10. agent hook 有详细的 analytics 事件：`max_turns`、`error`（类型 1/2）、`success`，都带有 `durationMs` 和 `turnCount`。

---

### 现在把第 75-77 站串起来

三大特殊 hook 执行器已经全部看完：

```text
execHttpHook.ts    -> 安全 HTTP POST 请求，带白名单/SSRF/代理策略
execPromptHook.ts  -> 单次 LLM 判定，用小模型评估 prompt 条件
execAgentHook.ts   -> 多轮 agent 循环，用工具和 transcript 进行验证
```

这三种 hook 执行器的共同点是：

- 都返回 `HookResult` 类型
- 都使用 `$ARGUMENTS` 模板
- 都支持自定义 `model` 和 `timeout`
- 都有 `ok: true/false` 的判定结果
- `ok: false` 都返回 blocking error
- `ok: true` 都返回 success
- 错误都遵循非阻塞错误/取消的处理逻辑

区别是：

| | HTTP hook | Prompt hook | Agent hook |
|---|---|---|---|
| 执行模型 | POST 请求 | 单次 LLM 查询 | 多轮 agent 循环 |
| 默认超时 | 10 分钟 | 30 秒 | 60 秒 |
| 结果来源 | HTTP 响应体 | JSON 文本中的 `ok` | StructuredOutput 附件 |
| 工具 | 无 | 有（但无 MCP/agent） | 有（除受限工具） |
| 安全特性 | URL 白名单、SSRF guard、header 清洗 | 小模型、JSON schema 输出 | Turn 限制、dontAsk 模式 |

---

### 下一站建议

三大执行器已经全部完成。接下来还有：

- `src/utils/hooks/hookHelpers.ts` - 共享辅助函数
- `src/utils/hooks/ssrfGuard.ts` - SSRF 防护底层
- `src/utils/hooks/registerFrontmatterHooks.ts` - frontmatter hook 注册
- `src/utils/hooks/registerSkillHooks.ts` - skill hook 注册
- `src/utils/hooks/hooksConfigSnapshot.ts` - 配置快照
- `src/utils/hooks/fileChangedWatcher.ts` - 文件变更 watcher
- `src/utils/hooks/skillImprovement.ts` - skill 改进
- `src/utils/hooks/apiQueryHookHelper.ts` - API 查询辅助函数

最自然的下一站是：

**`src/utils/hooks/hookHelpers.ts` 定义 `addArgumentsToPrompt`、`createStructuredOutputTool`、`hookResponseSchema`、`registerStructuredOutputEnforcement` 等所有执行器都复用的共享辅助函数。**
