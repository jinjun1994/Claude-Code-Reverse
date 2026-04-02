## 第 78 站：`src/utils/hooks/hookHelpers.ts`

### 这是什么文件

`src/utils/hooks/hookHelpers.ts` 是 hooks 子系统里的 **执行器共享工具层：定义了 prompt/agent hooks 共用的响应 schema、`$ARGUMENTS` 模板展开、StructuredOutput 工具构造，以及用于强制 agent hooks 返回结构化输出的 session function hook 注册。**

这一站之前已经看懂了三大执行器，而它们都复用了这里定义的辅助函数。

所以最准确的一句话是：

```text
hookHelpers.ts = hook 执行器的共享工具层：定义 hook 响应 schema、prompt 模板参数替换、StructuredOutput 工具工厂，以及用于强制 agent 返回结构化输出的 session-level function hook
```

---

### 先看它的整体定位

位置：`src/utils/hooks/hookHelpers.ts:1`

这个文件不是：

- hook runtime 执行器
- hook 配置 schema
- hook 事件层

它的职责非常集中：

1. 定义 prompt/agent hooks 共用的 `hookResponseSchema`
2. 定义 `addArgumentsToPrompt` 模板参数替换
3. 定义 `createStructuredOutputTool` 工具工厂
4. 定义 `registerStructuredOutputEnforcement` 注册 function hook 来强制 agent 返回结构化输出

所以它本质上是一个：

```text
shared verification primitives for LLM-based hooks
```

---

### 第一部分：`hookResponseSchema` 是 prompt hooks 和 agent hooks 的通用响应协议

位置：`src/utils/hooks/hookHelpers.ts:16`

```ts
const hookResponseSchema = lazySchema(() =>
  z.object({
    ok: z.boolean().describe('Whether the condition was met'),
    reason: z.string().describe('Reason, if the condition was not met').optional(),
  }),
)
```

这个 schema 定义了：

```json
{
  "ok": true,
  "reason": "optional reason"
}
```

它被以下地方使用：

- `execPromptHook.ts` — 解析模型响应文本
- `execAgentHook.ts` — 解析 structured output 附件数据

这说明 LLM-based hooks 共享同一个验证响应协议，
而不是各自定义不同格式。

---

### 第二部分：`addArgumentsToPrompt` 委托给一个独立的参数替换模块

位置：`src/utils/hooks/hookHelpers.ts:30`

```ts
export function addArgumentsToPrompt(prompt: string, jsonInput: string): string {
  return substituteArguments(prompt, jsonInput)
}
```

它支持的功能：

- `$ARGUMENTS` — 整个 JSON 替换
- `$ARGUMENTS[0]`, `$ARGUMENTS[1]` — 索引参数
- `$0`, `$1` 等 — 快捷写法

这说明 hook prompt 模板系统支持类似于 shell 脚本的位置参数。

`substituteArguments` 定义在 `argumentSubstitution.ts`，
不在本文件，
但本文件是它的 hook 特定入口。

---

### 第三部分：`createStructuredOutputTool` 是 StructuredOutput 钩子工具工厂

位置：`src/utils/hooks/hookHelpers.ts:41`

```ts
export function createStructuredOutputTool(): Tool {
  return {
    ...SyntheticOutputTool,
    inputSchema: hookResponseSchema(),
    inputJSONSchema: {
      type: 'object',
      properties: {
        ok: { type: 'boolean', description: '...' },
        reason: { type: 'string', description: '...' },
      },
      required: ['ok'],
      additionalProperties: false,
    },
    async prompt(): Promise<string> {
      return `Use this tool to return your verification result. You MUST call this tool exactly once at the end of your response.`
    },
  }
}
```

这说明它创建的是一个：

- 基于 `SyntheticOutputTool` 的工具
- 输入 schema 是 `{ ok: boolean, reason?: string }`
- prompt 说明告诉 agent 必须调用这个工具一次

这个工具被 `execAgentHook.ts` 使用。

注意 Zod 的 `.describe(...)` 和 JSON Schema 的 `description` 字段需要分别设置，
因为 Zod schema 不直接导出给模型提示文本。

---

### 第四部分：`registerStructuredOutputEnforcement` 是整个文件最有意思的函数

位置：`src/utils/hooks/hookHelpers.ts:70`

```ts
export function registerStructuredOutputEnforcement(
  setAppState: SetAppState,
  sessionId: string,
): void {
  addFunctionHook(
    setAppState,
    sessionId,
    'Stop',
    '', // No matcher - applies to all stops
    messages => hasSuccessfulToolCall(messages, SYNTHETIC_OUTPUT_TOOL_NAME),
    `You MUST call the ${SYNTHETIC_OUTPUT_TOOL_NAME} tool to complete this request. Call this tool now.`,
    { timeout: 5000 },
  )
}
```

这说明它注册了一个：

- 事件：`Stop`
- matcher：`''`（空，匹配所有）
- 检查函数：消息中是否成功调用过 `SYNTHETIC_OUTPUT_TOOL_NAME`
- 失败消息：告诉 agent 必须调用这个工具
- 超时：5000ms

这意味着：

```text
在每个 Stop 事件触发时，检查 agent 是否调用了 StructuredOutput 工具
如果没有 -> 返回失败，强制 agent 重试
```

这个设计非常巧妙：

1. 注册一个 Stop 事件的 function hook
2. 这个 hook 检查 agent 是否做了必要的事情
3. 如果没做，返回失败并带错误消息
4. agent 收到错误消息后被强制重试

这就是 `execAgentHook.ts` 中：

```ts
registerStructuredOutputEnforcement(
  toolUseContext.setAppState,
  hookAgentId,
)
```

的原理：确保 agent hook 必须返回结构化输出。

---

### 元

问题：**这一站真正想解决的架构问题是什么？**

回答：这一站关心的是，prompt hook 和 agent hook 共享的那些原语为什么不能散落在执行器里。通用响应 schema、`$ARGUMENTS` 展开、structured output 工具和强制输出 hook，本质上是同一套验证基础设施。

### 反

问题：**如果把这一站的设计反过来，会发生什么？**

回答：如果这些共享逻辑各写一份，prompt/agent 两条路线就会逐渐漂移。最先失去一致性的，不是功能，而是“什么叫合法 hook 结果”这件事。

### 空

问题：**跳出当前文件名，这一站背后更大的问题是什么？**

回答：更大的问题，是相似扩展路径怎样共享同一组低层原语。辅助层的意义，正是在重复还没变坏之前先把它收束掉。

### 读完这一站后，你应该抓住的 5 个事实

1. `hookResponseSchema` 定义 `{ ok: boolean, reason?: string }` 的通用验证响应 schema，被 prompt hooks 和 agent hooks 共用。
2. `addArgumentsToPrompt` 委托给 `substituteArguments`，支持 `$ARGUMENTS`、`$ARGUMENTS[0]`、`$0` 等模板语法。
3. `createStructuredOutputTool` 基于 `SyntheticOutputTool` 创建工具，输入 schema 是 `hookResponseSchema`，prompt 告诉 agent 必须调用一次。
4. `registerStructuredOutputEnforcement` 注册一个 Stop 事件 function hook，检查消息中是否有 `SyntheticOutputTool` 调用；没有则返回失败强制重试。
5. 这个机制确保 `execAgentHook` 中的 agent 总是返回结构化输出，否则被 Stop function hook 强制重试。

---

### 下一站建议

接下来最顺的是：

**`src/utils/hooks/ssrfGuard.ts` — SSRF 防护底层实现。**

前面 `execHttpHook.ts` 提到了 `ssrfGuardedLookup` 但没有展开，
这个文件应该定义了 DNS 级别的内网地址防护逻辑。
