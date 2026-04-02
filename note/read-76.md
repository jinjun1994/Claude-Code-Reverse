## 第 76 站：`src/utils/hooks/execPromptHook.ts`

### 这是什么文件

`src/utils/hooks/execPromptHook.ts` 是 hooks 子系统里的 **LLM prompt hook 执行器：负责把 prompt hook 模板展开 `$ARGUMENTS`、构造一个轻量 LLM 查询来判定 hook 条件是否满足（返回 `{"ok": true/false}`），并根据 LLM 返回值构造 blocking/success/error 结果。**

这一站是三大特殊 hook 执行器中的第二站。此前已经看懂了 HTTP hook 执行器的安全策略。

所以最准确的一句话是：

```text
execPromptHook.ts = prompt hook 的 LLM 执行层：负责 prompt 模板展开、用小模型快速判定 hook 条件是否满足、并把 LLM 响应翻译为 hook 的 blocking/success/非阻塞错误结果
```

---

### 先看它的整体定位

位置：`src/utils/hooks/execPromptHook.ts:1`

这个文件不是：

- prompt hook schema
- prompt hook 配置
- 通用 LLM 查询 wrapper

它的职责是：

1. 替换 `$ARGUMENTS` 占位符
2. 构造用户消息
3. 拼接对话历史
4. 用小模型查询 LLM
5. 解析 JSON 响应
6. 校验响应 schema
7. 根据条件是否满足返回 blocking/success
8. 处理超时/取消/错误

所以它本质上是一个：

```text
LLM-based condition evaluator for prompt hooks
```

---

### 第一部分：`effectiveToolUseID` 处理

位置：`src/utils/hooks/execPromptHook.ts:32`

```ts
const effectiveToolUseID = toolUseID || `hook-${randomUUID()}`
```

这说明如果没有提供 toolUseID（因为 prompt hook 本身不是 tool use），
仍然需要生成一个唯一 ID 来创建：

- attachment messages
- error reporting
- 结果展示

所以 prompt hook 虽然是 LLM 调用，
但对外暴露的结果形式被包装成跟 tool hook 一样的结构。

---

### 第二部分：`addArgumentsToPrompt(hook.prompt, jsonInput)` 说明 prompt hook 的模板语言支持 `$ARGUMENTS` 占位符，运行时被替换为序列化后的 hook 输入 JSON

位置：`src/utils/hooks/execPromptHook.ts:35`

注释说：

```text
Replace $ARGUMENTS with the JSON input
```

所以 prompt hook 配置的 prompt 字段里可以用 `$ARGUMENTS` 来引用
hook 触发时的完整 JSON 输入。

---

### 第三部分：注释关于无限递归的说明

位置：`src/utils/hooks/execPromptHook.ts:40`

```ts
// Create user message directly - no need for processUserInput which would
// trigger UserPromptSubmit hooks and cause infinite recursion
```

这说明 prompt hook 不能走正常用户输入处理流水线，
否则：

```
prompt hook -> UserPromptSubmit -> prompt hook -> ...
```

所以这里绕开 `processUserInput`，
直接构造消息。

这是一个很实际的反递归设计。

---

### 第四部分：查询消息构建

位置：`src/utils/hooks/execPromptHook.ts:45`

```ts
const messagesToQuery =
  messages && messages.length > 0
    ? [...messages, userMessage]
    : [userMessage]
```

这意味着 prompt hook 可以被传入：

- 只有 prompt（单轮查询）
- 包含对话历史 + prompt（多轮上下文）

---

### 第五部分：`queryModelWithoutStreaming` 是整个函数的核心

位置：`src/utils/hooks/execPromptHook.ts:62`

调用它时有几个特别关键的配置：

#### 系统提示

```ts
systemPrompt: asSystemPrompt([
  `You are evaluating a hook in Claude Code.

Your response must be a JSON object matching one of the following schemas:
1. If the condition is met, return: {"ok": true}
2. If the condition is not met, return: {"ok": false, "reason": "Reason for why it is not met"}`,
])
```

这说明 prompt hook 的 LLM 被严格约束为：

```text
boolean condition + optional reason
```

而不是自由文本回答。

#### 思考模式

```ts
thinkingConfig: { type: 'disabled' }
```

prompt hook 不使用 extended thinking。

#### 模型

```ts
model: hook.model ?? getSmallFastModel()
```

默认使用小快模型，不是主会话模型。

#### 工具

```ts
tools: toolUseContext.options.tools
toolChoice: undefined
mcpTools: []
agents: []
```

prompt hook可以带工具但禁用 MCP 和 agent。

#### 输出格式

```ts
outputFormat: {
  type: 'json_schema',
  schema: {
    type: 'object',
    properties: {
      ok: { type: 'boolean' },
      reason: { type: 'string' },
    },
    required: ['ok'],
    additionalProperties: false,
  },
}
```

这意味着 LLM 被要求严格按 JSON schema 输出。

#### `isNonInteractiveSession: true`

这说明 prompt hook 的 LLM 查询不触发交互式会话流程。

#### `hasAppendSystemPrompt: false`

不追加额外的系统提示。

#### `querySource: 'hook_prompt'`

追踪查询来源。

---

### 第六部分：超时至多 30 秒

位置：`src/utils/hooks/execPromptHook.ts:55`

```ts
const hookTimeoutMs = hook.timeout ? hook.timeout * 1000 : 30000
```

默认 30 秒超时。

---

### 第七部分：响应解析采用双层验证

位置：`src/utils/hooks/execPromptHook.ts:113`

```ts
const json = safeParseJSON(fullResponse)
if (!json) {
  return { outcome: 'non_blocking_error', ... }
}

const parsed = hookResponseSchema().safeParse(json)
if (!parsed.success) {
  return { outcome: 'non_blocking_error', ... }
}
```

先做宽松 JSON 解析
再做 schema 校验

这比单次解析更安全。

---

### 第八部分：条件不满足时的 blocking 处理

位置：`src/utils/hooks/execPromptHook.ts:154`

当 `ok: false` 时返回：

```ts
{
  outcome: 'blocking',
  blockingError: {
    blockingError: `Prompt hook condition was not met: ${reason}`,
    command: hook.prompt,
  },
  preventContinuation: true,
  stopReason: reason
}
```

这说明 prompt hook 条件不满足时：

- 阻止继续执行
- 提供错误原因
- 记录哪个 prompt 触发的

而如果 LLM 返回 `ok: true` 则只是 success，不阻塞流。

---

### 第九部分：错误处理分为三层

位置：`src/utils/hooks/execPromptHook.ts:183-210`

#### 内部 catch
处理 abort：

```ts
if (combinedSignal.aborted) {
  return { outcome: 'cancelled' }
}
throw error
```

#### 外部 catch
捕获模型本身的错误：

```ts
return {
  outcome: 'non_blocking_error',
  message: createAttachmentMessage({...})
}
```

这说明：

- 取消 = 立即返回 cancelled
- 模型错误 = 非阻塞错误
- 其他异常 = 向上抛

---

### 第十部分：整份文件最核心的架构价值

是：

```text
prompt hook = LLM-powered condition gate
```

它利用小模型快速判定某个条件是否满足，
而不像 command hook 需要 fork 进程，
也不像 HTTP hook 需要发网络请求。

---

### 读完这一站后，你应该抓住的 10 个事实

1. `src/utils/hooks/execPromptHook.ts` 不是通用 LLM 请求，而是专门用于 LLM-based hook condition 判定的执行器。
2. prompt hook 的 prompt 模板通过 `$ARGUMENTS` 占位符接收完整 hook 输入 JSON，
3. 模型使用默认的小快模型 而非主会话模型，除非 hook 配置了自定义 `model`。
4. LLM 被约束返回 `{"ok": boolean, "reason"?: string}` JSON schema，而不是自由文本回答。
5. prompt hook 默认 30 秒超时，可通过 hook 配置覆盖。
6. prompt hook 绕过 `processUserInput` 直接构造消息，防止触发 `UserPromptSubmit` hooks 导致无限递归。
7. 响应解析先宽松（`safeParseJSON`），再严格（Zod schema 验证），确保不规范的 LLM 输出被捕获。
8. `ok: false` 返回 blocking error + `preventContinuation: true` 阻止继续执行；`ok: true` 返回 success 不阻止。
9. prompt hook 可以带工具（tools）但不允许 MCP 工具和 agent。
10. prompt hook 错误处理：取消 = cancelled，模型错误 = 非阻塞错误，其他异常 = 向上抛。

---

### 下一站建议

第三类特殊 hook 执行器是：

**`src/utils/hooks/execAgentHook.ts` 怎样执行 agentic verifier hooks。**
