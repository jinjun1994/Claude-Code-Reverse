## 第 36 站：`src/hooks/useInboxPoller.ts`

### 这是什么文件

`src/hooks/useInboxPoller.ts` 是 swarm 邮箱系统里的 **运行时消息消费中心**。

上一站 `teammateMailbox.ts` 已经看懂：

```text
每个 agent 都有 inbox 文件
各种 protocol message / regular teammate message 都会写进去
```

这一站要解决的是：

```text
谁来轮询 inbox
怎样区分 permission / sandbox / shutdown / mode set / plan approval / 普通消息
哪些要走程序处理链
哪些要变成 LLM 新一轮输入或 UI 队列
```

所以最准确的一句话是：

```text
useInboxPoller.ts = teammate mailbox 的协议分流器 + 运行时投递器
```

---

### 先看它在系统里的位置

位置：`src/hooks/useInboxPoller.ts:118`

文件头注释已经明确：

1. 每 1 秒轮询 unread messages
2. 空闲时立即提交成新 turn
3. 忙碌时先存进 `AppState.inbox`，等 turn 结束再投递

这说明它不是单纯“收邮件”的 hook，而是同时负责两件大事：

```text
A. 把 structured mailbox message 路由到正确的程序处理器
B. 把普通 teammate message 路由到 Claude 的对话输入面
```

所以它正好站在 mailbox 底层和 query/runtime 上层之间。

---

### 第一部分：`getAgentNameToPoll(...)` 先决定“这个会话到底该不该轮询 inbox”

位置：`src/hooks/useInboxPoller.ts:74`

这个函数的逻辑非常关键：

- in-process teammate -> `undefined`
- process-based teammate -> 自己的 `agentName`
- team lead -> lead 的名字
- standalone session -> `undefined`

这说明 inbox poller 不是所有会话都开，而是只在这些角色上工作：

```text
需要独立 inbox 的进程型 teammate
需要消费团队消息的 lead
```

而 in-process teammate 被明确排除，因为它们通过：

- `waitForNextPromptOrShutdown()`

走另一套机制。

这里最该抓住的是：

```text
useInboxPoller 不是“所有 swarm 成员共享的统一接收器”，
而是只服务于真正拥有独立 mailbox 身份的执行体
```

---

### 第二部分：它先读 unread inbox，再做统一分流

位置：`src/hooks/useInboxPoller.ts:139`

主 poll 流程一开始就是：

- `readUnreadMessages(agentName, teamName)`
- 没消息就返回
- 有消息就进入分类逻辑

然后建出 10 类桶：

- `permissionRequests`
- `permissionResponses`
- `sandboxPermissionRequests`
- `sandboxPermissionResponses`
- `shutdownRequests`
- `shutdownApprovals`
- `teamPermissionUpdates`
- `modeSetRequests`
- `planApprovalRequests`
- `regularMessages`

这说明它真正的核心不是定时器，而是：

```text
把 inbox 中的混合消息流拆成多个协议子流
```

所以这一站最核心的理解是：

```text
useInboxPoller = mailbox control-plane router
```

---

### 第三部分：permission request 在 leader 侧会被重新转成标准 `ToolUseConfirm` UI 事务

位置：`src/hooks/useInboxPoller.ts:250`

这是整份文件最关键的一段之一。

leader 收到 `permission_request` 后，并不是自己临时拼个简陋 prompt，而是：

- `getLeaderToolUseConfirmQueue()`
- `findToolByName(getAllBaseTools(), parsed.tool_name)`
- 构造标准 `ToolUseConfirm`
- 放入 leader 的 confirm queue

并绑定：

- `onAbort()` -> `sendPermissionResponseViaMailbox(...rejected...)`
- `onAllow()` -> `sendPermissionResponseViaMailbox(...approved...)`
- `onReject()` -> `sendPermissionResponseViaMailbox(...rejected...)`

这说明 mailbox permission request 到达 leader 后，会被桥接回主权限 UI 的标准事务对象。

也就是说：

```text
tmux / out-of-process worker 的权限请求
最终仍然复用主 agent 那套 ToolUseConfirm 审批 UI
```

所以 `useInboxPoller.ts` 在这里其实就是：

```text
worker-mailbox protocol -> leader interactive permission UI bridge
```

这一步把分布式权限链路接回了本地统一审批界面。

---

### 第四部分：permission response 在 worker 侧不会进 LLM，而是直接恢复挂起回调

位置：`src/hooks/useInboxPoller.ts:366`

worker 收到 `permission_response` 后：

- 先 `hasPermissionCallback(parsed.request_id)`
- 如果存在 callback
- 就 `processMailboxPermissionResponse(...)`

approved 时传：

- `updatedInput`
- `permissionUpdates`

rejected 时传：

- `feedback`

这说明 permission response 完全属于控制面消息：

```text
它不是给模型看的，不进入 regular teammate message 流；
它直接恢复上一站注册的 worker-side continuation
```

所以 mailbox 在这里不是“消息通知”，而是一次真正的远程 RPC 响应通道。

---

### 第五部分：sandbox permission 在这里也形成 leader-side queue / worker-side callback 的镜像结构

位置：

- leader 处理 request：`src/hooks/useInboxPoller.ts:399`
- worker 处理 response：`src/hooks/useInboxPoller.ts:465`

收到 `sandbox_permission_request` 时，leader 侧会：

- 校验 `hostPattern.host`
- 追加到 `workerSandboxPermissions.queue`
- 必要时发桌面通知

收到 `sandbox_permission_response` 时，worker 侧会：

- `hasSandboxPermissionCallback(requestId)`
- `processSandboxPermissionResponse(...)`
- 清掉 `pendingSandboxRequest`

这说明 tool permission 和 sandbox permission 在控制面形状上是高度平行的：

```text
worker request
  -> leader queue/UI
  -> leader decision
  -> worker callback resume
```

所以 inbox poller 不是只服务于某一个协议，而是统一承接多个 approval sub-protocol。

---

### 第六部分：`team_permission_update` 和 `mode_set_request` 是 leader 对 teammate 的状态广播/控制通道

位置：

- team permission update：`src/hooks/useInboxPoller.ts:497`
- mode set request：`src/hooks/useInboxPoller.ts:549`

#### `team_permission_update`
worker 收到后会：

- 验证 `permissionUpdate.rules / behavior`
- `applyPermissionUpdate(...)`
- 直接改本地 `toolPermissionContext`

这说明它不是一个“通知消息”，而是 leader 对 worker 权限上下文的远程同步机制。

#### `mode_set_request`
worker 收到后会：

- 只接受 `m.from === 'team-lead'`
- `permissionModeFromString(parsed.mode)`
- 本地改 `toolPermissionContext.mode`
- 再 `setMemberMode(teamName, agentName, targetMode)` 更新 team config

这说明 inbox poller 还在执行另一类职责：

```text
把 leader 的团队控制命令落地成 teammate 本地状态更新
```

所以它并不仅仅是消息投递器，也是在做分布式状态同步。

---

### 第七部分：plan approval 是一个很特殊的链路：既自动批准，又保留对话上下文

位置：`src/hooks/useInboxPoller.ts:599`

leader 收到 `plan_approval_request` 后，这里做了一个很有意思的双重动作：

1. **程序上立即 auto-approve**
   - 生成 `plan_approval_response`
   - `writeToMailbox(...)` 发回 teammate
   - 如果是 in-process teammate，还 `handlePlanApprovalResponse(...)`

2. **语义上仍把原请求塞回 `regularMessages`**
   - `regularMessages.push(m)`

注释直接解释了原因：

```text
approval is already sent
but the model still gets context about what the teammate is doing
```

这说明 inbox poller 在这里明确区分了两层：

```text
控制面处理：自动给 teammate 放行
对话语义层：把 teammate 的计划内容继续暴露给模型作为上下文
```

这是整份文件中非常典型的“双通道处理”案例。

---

### 第八部分：shutdown request / shutdown approval 展示了 mailbox 控制面不只做权限，还能驱动团队生命周期管理

位置：

- teammate 侧 shutdown request：`src/hooks/useInboxPoller.ts:664`
- leader 侧 shutdown approval：`src/hooks/useInboxPoller.ts:677`

#### shutdown request
teammate 侧并不会直接程序化吞掉，而是：

- 保留原 JSON
- 走 `regularMessages.push(m)`

因为注释说：

- UI 会把它渲染得更友好
- 模型也会通过工具文档知道该如何响应

#### shutdown approval
leader 收到后则会真正执行系统动作：

- 如有 paneId/backendType，则 kill pane
- `removeTeammateFromTeamFile(...)`
- `unassignTeammateTasks(...)`
- 更新 `teamContext.teammates`
- 把对应 teammate task 标记完成
- 追加一条 system inbox message

这说明 shutdown approval 在这个模块里已经不再是“消息”，而是一次：

```text
teammate termination state transition
```

所以 inbox poller 不只是 mailbox reader，它还是团队运行时状态机的执行点。

---

### 第九部分：普通 teammate message 的处理策略是“空闲立即投递，忙时排队”

位置：`src/hooks/useInboxPoller.ts:802`

对于 `regularMessages`，逻辑是：

- 先格式化成 `<teammate_message ...>` XML
- 如果当前不在 loading 且没有 dialog：
  - `onSubmitTeammateMessage(formatted)`
  - 成功则立即变成新 turn
  - 失败则排队
- 如果当前忙：
  - 直接写入 `AppState.inbox.messages`

这说明 inbox poller 对普通 teammate 消息的核心策略是：

```text
不要打断当前主流程；
但一旦空闲，尽快把 teammate 消息提升成 Claude 的新输入轮次
```

所以它既不是纯实时抢占式，也不是只能下轮再说，而是一个带背压的即时投递器。

---

### 第十部分：`markRead()` 放在“成功投递或可靠排队之后”体现了很成熟的防丢消息设计

位置：`src/hooks/useInboxPoller.ts:860`

这里的注释非常重要：

```text
只有在消息成功投递，或已经可靠排入 AppState 队列后，才标记为已读。
否则如果忙的时候提前标记已读，崩溃后消息会永久丢失。
```

这说明它对 inbox 消费语义的定义是：

```text
read != seen
read = 已经被可靠接管
```

这是一种接近消息队列 ack 语义的设计，而不是 UI 层面的“看过了就算已读”。

因此 `markRead()` 在整个文件里承担的是：

```text
mailbox ack
```

角色。

---

### 第十一部分：第二个 `useEffect` 负责“会话空闲后的延迟投递与清理”

位置：`src/hooks/useInboxPoller.ts:875`

这段 effect 做两件事：

#### 1. 清理 `processed` 消息
这些消息已经作为附件中途投递过，不需要继续留在 inbox queue。

#### 2. 投递 `pending` 消息
如果当前 session 已空闲：

- 重新把 pending messages 格式化成 XML
- 再次 `onSubmitTeammateMessage(formatted)`
- 成功才从 `AppState.inbox.messages` 移除
- 失败则继续保留

所以整个 inbox 交付模型不是一步完成，而是两阶段：

```text
poll 时：立即投递或可靠排队
idle 时：再尝试把排队消息真正交给 Claude 新 turn
```

这使它具备了很强的运行时韧性。

---

### 第十二部分：它把“structured protocol messages”和“regular teammate messages”真正分成了两条不同的处理平面

如果把整份文件压缩，会发现所有消息最终都落到两大类：

#### A. 控制面消息
直接在 poller 中执行程序逻辑：

- permission request / response
- sandbox permission request / response
- team permission update
- mode set request
- 部分 plan approval
- 部分 shutdown approval

#### B. 对话面消息
进入 `regularMessages`，再：

- 直接提交成 turn
- 或排进 `AppState.inbox`
- 最终以 XML attachment 形式喂给模型

所以 `useInboxPoller.ts` 可以被最简洁地概括为：

```text
mailbox 的 control-plane / conversation-plane 分流执行器
```

这正好呼应上一站 `isStructuredProtocolMessage(...)` 的设计初衷。

---

### 第十三部分：它和前几站串起来后，swarm 协作总线已经闭环了

现在把几站连起来：

```text
teammateMailbox.ts
  -> 提供 inbox 存储、写入、协议消息格式与解析基础设施
permissionSync.ts
  -> 把 swarm permission 审批适配成 mailbox message 往返
useSwarmPermissionPoller.ts
  -> worker 侧把 permission response 接回 callback continuation
useInboxPoller.ts
  -> 轮询 inbox，分流 structured protocol messages，并把 regular messages 投递到 Claude turn / UI queue
```

所以这一站真正补完的是：

```text
mailbox message 在运行时到底怎样被“消费”
```

到这里 swarm mailbox 已经形成完整链路了。

---

### 元

问题：**这一站真正想解决的架构问题是什么？**

回答：这一站是 teammate mailbox 的协议分流器和运行时投递器。它决定谁该轮询 inbox，把 structured messages 送给程序处理链，把普通消息转成新 turn 输入或暂存到 AppState。

### 反

问题：**如果把这一站的设计反过来，会发生什么？**

回答：如果所有消息都一视同仁塞给模型，权限、shutdown、mode set 这类控制信号会被错误地当作普通聊天。若全都程序处理，又会丢掉队友自然沟通。

### 空

问题：**跳出当前文件名，这一站背后更大的问题是什么？**

回答：更大的问题是，同一条消息管道里如何共存控制流和内容流。真正难的不是读取消息，而是正确分流。


### 读完这一站后，你应该抓住的 10 个事实

1. `useInboxPoller.ts` 的核心不是简单轮询，而是对 teammate inbox 做协议分流与运行时投递。
2. 它只在真正需要独立 inbox 的执行体上工作；in-process teammate 明确走另一套通道。
3. inbox 中的 unread message 会先被拆成 permission、sandbox、shutdown、mode、plan approval、regular message 等多个子流。
4. leader 收到 worker 的 `permission_request` 后，会把它桥接回标准 `ToolUseConfirm` 队列，而不是另做一套权限 UI。
5. worker 收到 `permission_response` 后，会直接恢复挂起的 permission callback，而不会把这类消息交给模型。
6. `team_permission_update` 和 `mode_set_request` 说明 inbox poller 还承担 leader -> teammate 的远程状态同步职责。
7. `plan_approval_request` 采用“双通道处理”：程序上自动批准，但语义上仍保留为 regular message 供模型获得上下文。
8. `shutdown_approved` 在 leader 侧会触发真实的团队生命周期操作，包括 kill pane、移出 team file、回收任务与更新 teamContext。
9. 普通 teammate 消息采用“空闲立即提交，忙时排队，空闲后再投递”的背压式交付策略。
10. `markRead()` 只在成功投递或可靠排队后执行，体现了接近消息队列 ack 的防丢消息语义。

---

### 现在把第 35-36 站串起来

```text
teammateMailbox.ts
  -> 定义 mailbox 存储、消息信封、协议类型与结构化消息识别
useInboxPoller.ts
  -> 轮询 unread inbox，按协议类型分流，并把控制面消息与对话面消息分别接入运行时
```

所以 mailbox 子系统现在可以压缩成：

```text
writeToMailbox -> inbox file
readUnreadMessages -> protocol classification
structured message -> program handler / state sync / callback resume
regular message -> Claude turn or AppState inbox queue
```

---

### 下一站建议

下一站最顺的是：

```text
src/utils/inProcessTeammateHelpers.ts
```

因为这一站你已经看懂了：

- 进程外 / tmux teammate 通过 mailbox + inbox poller 通信
- in-process teammate 被明确排除在 `useInboxPoller` 之外

下一步最自然就是补齐那条被排除掉的并行分支：

**in-process teammate 的 plan approval / task mapping / 响应恢复，是怎样通过 `inProcessTeammateHelpers.ts` 和共享 AppState 走另一条内存内协作链路的。**
