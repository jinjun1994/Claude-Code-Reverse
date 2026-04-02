## 第 115 站：`src/tools/SendMessageTool/`（4 个文件，~88K）

### 这是什么

SendMessageTool 是多代理（agent swarm）之间的消息传递工具——支持单播、广播、shutdown 请求/响应、plan 审批四种消息类型。这是 agent team 协作的核心通信机制：teammate 之间通过 mailbox 异步通信。

---

### 输入 Schema

```ts
{
  to: string             // 收件人：teammate 名、"*" 广播、"uds:..." 跨会话、"bridge:..." 远程控制
  summary?: string       // UI 预览摘要（5-10 字，纯文本消息时必填）
  message: string | StructuredMessage
}

// StructuredMessage 是三种类型的 discriminated union：
type StructuredMessage =
  | { type: 'shutdown_request', reason?: string }
  | { type: 'shutdown_response', request_id: string, approve: boolean, reason?: string }
  | { type: 'plan_approval_response', request_id: string, approve: boolean, feedback?: string }
```

### 输出类型

```ts
type MessageOutput     = { success, message, routing?: MessageRouting }
type BroadcastOutput   = { success, message, recipients: string[], routing?: MessageRouting }
type RequestOutput     = { success, message, request_id: string, target: string }
type ResponseOutput    = { success, message, request_id?: string }
```

---

### 五种消息路由路径

#### 1. 跨会话桥接（bridge）

```ts
if (addr.scheme === 'bridge') {
  postInterClaudeMessage(addr.target, input.message)
}
```

通过 Anthropic 服务器在两个远程 Claude 会话之间传递纯文本消息。需用户显式确认（`safetyCheck`，bypass-immune）。仅在 REPL Bridge 活跃时可用。

#### 2. UDS 本地 Socket

```ts
if (addr.scheme === 'uds') {
  sendToUdsSocket(addr.target, input.message)
}
```

通过 Unix Domain Socket 向本地另一个 Claude 进程发送消息。不走权限检查的直接发送。

#### 3. In-process 子代理（LocalAgentTask）

```ts
if (isLocalAgentTask(task) && !isMainSessionTask(task)) {
  if (task.status === 'running') {
    queuePendingMessage(agentId, input.message, ...)  // 排队等下一次工具轮次
  } else {
    resumeAgentBackground({ agentId, prompt: message })  // 自动恢复停止的代理
  }
}
```

这是最常见的路径——向正在运行的子代理发消息：
- **运行中**：消息排队，在子代理下一次工具轮次时投递（不做完整 fork，直接排队到 agent 的输入队列）
- **已停止**：自动从磁盘 transcript 恢复并在后台运行

#### 4. Team Mailbox（单播）

```ts
await writeToMailbox(recipientName, { from, text, summary, timestamp, color }, teamName)
```

消息写入团队共享的文件级 mailbox——接收方下次轮询时读取。带颜色标记（`senderColor`、`recipientColor`）用于 UI 渲染。

#### 5. Team Mailbox（广播）

```ts
for (const member of teamFile.members) {
  if (member.name !== senderName) {
    await writeToMailbox(memberName, message, teamName)
  }
}
```

向团队文件中所有成员发送消息（排除发送者自己）。

---

### Shutdown 协议

#### Request → Response 握手

```
Teammate A                      Team Lead
    |                               |
    |-- shutdown_request (id=X) --> |  (写入 mailbox)
    |                          [轮询检查]
    |                               |
    |                               | [决定是否批准]
    |                               |
    |   <-- shutdown_response ----  |
    |      (id=X, approve=true)     |
    |                               |
    | [abort controller / graceful shutdown]
```

- **请求方**：发送 `shutdown_request`，包含 `request_id` 和可选 `reason`
- **响应方**：发送 `shutdown_response` 到 `TEAM_LEAD_NAME`，包含 `approve: boolean`
- **批准时**：in-process 代理直接 `abortController.abort()`；其他路径调用 `gracefulShutdown()`
- **拒绝时**：返回 reason 并继续工作

#### 安全约束

- shutdown_response 必须发给 `TEAM_LEAD_NAME`——不能发给其他 teammate
- 拒绝时必须提供 reason

---

### Plan 审批协议

```ts
// 只有 team lead 可以批准/拒绝方案
if (!isTeamLead(appState.teamContext)) {
  throw new Error('Only the team lead can approve plans.')
}

// 批准时继承 permission mode
const modeToInherit = leaderMode === 'plan' ? 'default' : leaderMode
```

审批响应包含即将继承的 permission mode——这让 teammate 知道审批通过后进入什么模式��default、auto、等）。

---

### Backfill Observable Input

```ts
backfillObservableInput(input) {
  if (input.to === '*') {
    input.type = 'broadcast'
  } else if (typeof input.message === 'string') {
    input.type = 'message'
    input.recipient = input.to
  } else {
    // structured message → extract type, request_id, etc.
  }
}
```

这个钩子将输入转换为标准化的可观察格式——用于分析分类器（`toAutoClassifierInput`）和日志。`backfillObservable` 是 Claude Code 工具特有的机制：将原始输入转化为结构化的分析格式。

---

### 权限桥接安全

```ts
async checkPermissions(input) {
  if (parseAddress(input.to).scheme === 'bridge') {
    return {
      behavior: 'ask',
      decisionReason: { type: 'safetyCheck', classifierApprovable: false },
    }
  }
  return { behavior: 'allow' }
}
```

Bridge 消息永远是 `ask`（无论什么 mode）且 `classifierApprovable: false`（自动分类器不允许自动批准）。这是跨机器 prompt injection 防护——不能因为处在 auto 模式就允许向另一台机器发送消息。

---

### UDS_INBOX 特性门

`feature('UDS_INBOX')` 控制跨会话消息路径：
- 开启时：`to` 支持 `uds:<socket-path>` 和 `bridge:<session-id>` 地址格式
- 关闭时：只支持队友名字和 `*` 广播

---

### 读完后应抓住的 5 个事实

1. **五种消息路由路径**：SendMessageTool 不是简单的 mailbox 写入——它处理五种完全不同的场景：(1) bridge 跨远程会话、(2) UDS 跨本地进程、(3) in-process 子代理、(4) team mailbox 单播、(5) team mailbox 广播。每种路径有不同的协议和验证规则。

2. **三种结构化消息类型**：不只是文本——`shutdown_request`、`shutdown_response`、`plan_approval_response` 是 teammate 间的正式协议消息。shutdown 有 request/response 握手机制，只有 team lead 能决定批准或拒绝。

3. **自动恢复代理机制**：向已停止的代理发消息时，`resumeAgentBackground()` 自动从磁盘 transcript 恢复代理运行。这是容错设计——不需要显式 "重启代理" 命令。

4. **Bridge 安全是 bypass-immune**：跨机器 bridge 消息即使在 auto 模式下也需要用户显式确认。`classifierApprovable: false` 阻止自动分类器绕过权限。这是跨会话 prompt injection 防护的核心。

5. **Backfill Observable Input 模式**：SendMessageTool 将原始输入格式（文本/结构化）规范化为标准化的分析格式（`type`、`recipient`、`content` 字段）。这让分析分类器（`toAutoClassifierInput`）可以在不感知原始格式的情况下对输入做分类。
