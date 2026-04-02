## 第 55 站：`src/hooks/useInboxPoller.ts`

### 这是什么文件

`src/hooks/useInboxPoller.ts` 是 swarm 体系里的 **leader / process-based teammate 的 mailbox 轮询与消息路由器**。

上一站 `teammateMailbox.ts` 已经看懂：

- mailbox 是 per-agent inbox 文件
- 里面既有普通 teammate 文本消息
- 也有 permission / shutdown / plan approval / mode set 等结构化协议消息

但“消息定义好了”不等于“消息真的被系统消费了”。

还需要一个运行时层回答：

- 谁该轮询 inbox
- 轮询频率是多少
- 收到 structured protocol message 时该进哪个控制队列
- 收到普通 teammate message 时什么时候立刻提交成新 turn，什么时候先排队
- shutdown / permission / sandbox permission / plan approval 怎样在 leader 和 teammate 两端分流

所以这一站回答的是：

```text
swarm runtime 到底怎样定期轮询 mailbox，
并把不同类型的消息分别路由到 UI、AppState、权限回调、teammate lifecycle 和模型输入面
```

所以最准确的一句话是：

```text
useInboxPoller.ts = swarm mailbox 的运行时消费器与控制面分发器
```

---

### 先看它的整体定位

位置：`src/hooks/useInboxPoller.ts:1`

这个文件不是：

- mailbox 存储实现
- teammate runner 本体
- backend 实现
- permission callback 注册中心

它的职责是：

1. 定时 poll unread mailbox messages
2. 识别消息类型
3. 把 structured protocol message 路由到对应 handler / queue / state update
4. 把普通 teammate 文本消息在合适时机注入成 Claude 的下一轮输入

所以它是一个很典型的：

```text
runtime mailbox consumer
```

或者更具体点说：

```text
mailbox transport -> app/runtime effects router
```

---

### 第一部分：`getAgentNameToPoll()` 说明 inbox polling 不是所有 swarm 角色都用同一套路径，尤其 in-process teammate 被明确排除

位置：`src/hooks/useInboxPoller.ts:74`

这个函数先分三种角色：

#### 1. in-process teammate
- 返回 `undefined`
- 不使用 `useInboxPoller`

#### 2. process-based teammate
- 用 `CLAUDE_CODE_AGENT_NAME`

#### 3. team lead
- 从 `teamContext.teammates[leadAgentId].name` 找自己的 name
- 否则 fallback `team-lead`

这里最关键的是第一条。

注释明确说：

```text
in-process teammate 应该用 waitForNextPromptOrShutdown()，
不能再走 useInboxPoller，
否则会因为 shared React context / shared AppState 造成路由问题
```

这说明 inbox poller 的适用范围是：

```text
team lead
+
process-based teammates
```

而不是全部 swarm agent。

这和前面 `inProcessRunner.ts` 的独立 polling loop 完全对上。

---

### 第二部分：这里特地容忍“leader re-render 时 ALS 正好处于 in-process teammate context”的情况，说明共享进程执行模型确实会让 UI hook 看到临时错位的身份上下文

位置：`src/hooks/useInboxPoller.ts:81`

注释提到一个很有意思的现象：

- leader 的 REPL re-render
- 可能发生在 in-process teammate 的 AsyncLocalStorage context 活跃期间
- 因为它们共享同一 React/AppState 世界

所以这里不是抛错，
而是安静返回 `undefined`，跳过 polling。

这说明作者清楚意识到：

```text
shared-process teammate execution can transiently expose worker identity to leader-side React code
```

因此这里的设计重点是：

```text
graceful skip over transient context ambiguity
```

这类小判断其实很能体现共享进程模型的成熟度。

---

### 第三部分：`INBOX_POLL_INTERVAL_MS = 1000` 说明 mailbox 消费被建模成低频轮询型控制面，而不是文件监听或实时流

位置：`src/hooks/useInboxPoller.ts:107`

这里每秒 poll 一次。

这说明 swarm 对 mailbox 的预期不是高吞吐、毫秒级响应总线，
而是：

```text
human/agent coordination loop with coarse polling granularity
```

也就是说：

- 1 秒的延迟在 teammate 协作里完全可接受
- 简单、鲁棒比复杂实时监听更重要

这和 mailbox 本身采用文件 JSON + lock 的实现风格是一致的。

---

### 第四部分：`poll()` 的第一段逻辑说明它故意避开 `appState` 的 React 依赖，直接从 store 取当前快照，以避免 hook 自身陷入重渲染/依赖环

位置：`src/hooks/useInboxPoller.ts:139`

这里没有把整个 `appState` 塞进依赖，
而是：

- `const currentAppState = store.getState()`

注释直接说：

- avoid dependency on appState object
- prevents infinite loop

这说明作者非常清楚这个 hook 的副作用特性：

- 它会读 AppState
- 又会写 AppState
- 如果依赖整个对象，很容易造成 poll / state update / rerender / poll 的反馈回路

所以这里真正做的是：

```text
polling side effects should read from store snapshots, not subscribe to full appState identity changes
```

---

### 第五部分：最先处理 `plan approval response`，而且只接受来自 `team-lead` 的消息，说明 plan-mode 退出被视为高优先级安全控制面事件

位置：`src/hooks/useInboxPoller.ts:156`

这里的条件很明确：

- 当前是 teammate
- `isPlanModeRequired()`
- unread 中有 `plan_approval_response`
- 且 `msg.from === 'team-lead'`

然后如果 approved：

- 取 leader 传来的 `permissionMode`
- `applyPermissionUpdate(... setMode ...)`
- 退出 plan mode

这说明：

```text
teammate 从 plan mode 退出，不是靠本地猜测，
而是靠 mailbox 中 team-lead 签发的显式批准消息
```

而且只接受 `team-lead`，
明确防止其他 teammate 伪造批准。

这是一个很关键的安全边界。

---

### 第六部分：中段把 unread messages 拆成 9 类以上队列，说明这个 hook 的核心不是“拿到消息就交给模型”，而是先做严格的协议路由分类

位置：`src/hooks/useInboxPoller.ts:204`

这里把 unread 分成：

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

这正好呼应上一站 `isStructuredProtocolMessage()` 的意义。

也就是说 `useInboxPoller` 真正的工作流不是：

```text
mailbox unread -> submit to Claude
```

而是：

```text
mailbox unread
  -> protocol classification
  -> route to control handlers / queues / state
  -> only leftover regular text goes to model surface
```

这是整个控制面正确性的核心。

---

### 第七部分：permission request 的 leader-side 处理说明 tmux/process teammate 也能获得与 in-process teammate 类似的 ToolUseConfirm UI，只是通过 mailbox 衔接过去

位置：`src/hooks/useInboxPoller.ts:250`

这里当 leader 收到 `permission_request` 时，会：

1. `getLeaderToolUseConfirmQueue()`
2. `findToolByName(getAllBaseTools(), parsed.tool_name)`
3. 构造 `ToolUseConfirm` entry
4. 挂上 `onAllow / onReject / onAbort`
5. 最终回写 `sendPermissionResponseViaMailbox(...)`

这非常关键。

说明 tmux/process-based teammate 的权限请求虽然来自 mailbox，
但 leader 端并不会降级成简陋文本确认。

它仍然被接进统一的：

```text
ToolUseConfirm UI pipeline
```

所以可以把它理解成：

```text
mailbox request transport
+
standard leader permission UI rendering
```

这正是前面 `inProcessRunner.ts` 中“leader UI bridge first”设计在 process-based 路线上的镜像实现。

---

### 第八部分：permission response / sandbox permission response 走的是 callback-based completion 语义，说明 mailbox 在 worker 侧只是异步回复 transport，而真正的等待/恢复点在 callback registry

位置：

- permission response `src/hooks/useInboxPoller.ts:366`
- sandbox response `src/hooks/useInboxPoller.ts:465`

这里的模式都是：

- 先看 `hasPermissionCallback(...)` / `hasSandboxPermissionCallback(...)`
- 再 `processMailboxPermissionResponse(...)` / `processSandboxPermissionResponse(...)`

这说明 worker 端不是每次轮询后自己临时决定怎么恢复执行，
而是：

```text
request side earlier registered a callback/promise continuation,
mailbox poller just resolves it when the response arrives
```

所以 `useInboxPoller` 在这里更像：

```text
response dispatcher into pending control callbacks
```

而不是完整权限系统本体。

---

### 第九部分：sandbox permission request 直接塞进 `workerSandboxPermissions.queue`，说明这类权限流的 leader-side 展示/处理并不复用 `ToolUseConfirm`，而是走另一条专门 UI 队列

位置：`src/hooks/useInboxPoller.ts:399`

这里 leader 收到 sandbox permission request 后，不是转成 `ToolUseConfirm`，
而是：

- 解析 host
- 构造 `newSandboxRequests`
- append 到 `prev.workerSandboxPermissions.queue`

这说明 swarm 权限体系内部其实至少分了两类展示模型：

#### tool permission
走 ToolUseConfirm

#### sandbox network permission
走 workerSandboxPermissions queue

也就是说“权限请求”不是单一 UI 通道，
而是按权限类型拆分的。

---

### 第十部分：`teamPermissionUpdates` 与 `modeSetRequests` 两段说明 inbox poller 不只是转发消息，还会直接改 teammate 本地 permission context 和 team file 状态

位置：

- team permission update `src/hooks/useInboxPoller.ts:497`
- mode set request `src/hooks/useInboxPoller.ts:549`

这里会：

#### team permission update
- `applyPermissionUpdate(... addRules ...)`
- 更新 teammate 自己的 `toolPermissionContext`

#### mode set request
- 仅接受 `team-lead`
- `applyPermissionUpdate(... setMode ...)`
- 还会 `setMemberMode(teamName, agentName, targetMode)`

这说明 `useInboxPoller` 不是纯 transport consumer，
它还是：

```text
runtime state mutation bridge
```

尤其 mode change 同时更新：

- 本地运行态 permission context
- team file 中对外可见的 teammate mode

这再次体现 leader/worker 协作面与持久 team metadata 的联动。

---

### 第十一部分：leader-side `planApprovalRequests` 会被自动批准，而且还故意把原消息继续透传给 regularMessages，说明系统想同时满足“控制面自动流转”与“模型看见 teammate 正在请求批准的上下文”

位置：`src/hooks/useInboxPoller.ts:599`

这里逻辑很特别：

1. leader 收到 `plan_approval_request`
2. 自动构造 `plan_approval_response`
3. 写回 teammate inbox
4. 如果是 in-process teammate，还同步更新 task state
5. 然后 **把原 request 继续 push 到 `regularMessages`**

这意味着：

```text
approval control path can be completed automatically,
while the conversation/model surface still receives the context message
```

所以这里不是“协议 handled 了就完全吞掉”，
而是有选择地：

- 自动处理控制动作
- 但保留对 leader 模型/上下文可见性

这是一个很细的 UX / agent-awareness 设计。

---

### 第十二部分：shutdown request 和 shutdown approval 的分工很清晰——worker 侧把 request 透传给模型决定，leader 侧收到 approval 后再做真正的 pane/team/task 清理

位置：

- shutdown request `src/hooks/useInboxPoller.ts:664`
- shutdown approval `src/hooks/useInboxPoller.ts:677`

#### teammate 侧收到 shutdown request
- 不直接退出
- 只是把消息推回 `regularMessages`
- 让 UI / model 继续处理

这和前面 `inProcessRunner.ts` 完全一致：

```text
shutdown request is advisory control input; the model decides approve/reject
```

#### leader 侧收到 shutdown approval
会继续做真正的 cleanup：

- 如果有 `paneId + backendType`，调用 backend killPane
- `removeTeammateFromTeamFile(...)`
- `unassignTeammateTasks(...)`
- 从 `teamContext.teammates` 删除
- 把对应 task 标成 completed
- 往 inbox UI 加一条 `teammate_terminated` system message

这说明 shutdown approval 在 leader 侧是：

```text
teammate lifecycle finalization trigger
```

而不是单纯的确认提示。

---

### 第十三部分：这里特地把 out-of-process teammate task 标成 completed，说明没有本地 runner 的 process-based teammate 必须靠 inbox poller 补上生命周期收尾，否则 spinner 会永远转下去

位置：`src/hooks/useInboxPoller.ts:749`

注释写得很关键：

- out-of-process (tmux) teammate tasks otherwise stay `status:'running'` forever
- only in-process teammates have a runner that marks `completed`

这说明 `useInboxPoller` 在架构里还有一个很重要的补位角色：

```text
for process-based teammates, mailbox-driven approval handling also closes lifecycle state loops that no local runner can close
```

这正是 leader-side poller 存在的一个重要原因。

---

### 第十四部分：regular teammate messages 只有在“不是结构化控制消息剩余物”时才会进入模型输入面，而且 idle 时直接提交、busy 时先进 AppState inbox 队列

位置：`src/hooks/useInboxPoller.ts:802`

这段是整个 hook 面向模型输入面的核心。

流程是：

1. 若 `regularMessages.length === 0`
   - 只 markRead 然后返回
2. 否则把 regular messages 包成 `<teammate-message>` XML
3. 如果 session idle 且没对话框：
   - 直接 `onSubmitMessage(formatted)`
4. 如果 busy 或提交被拒：
   - 放入 `AppState.inbox.messages`
5. 最后 markRead

这说明普通 teammate 消息进入 Claude 的策略是：

```text
idle -> immediate turn injection
busy -> reliable queued delivery via AppState inbox
```

也就是说 inbox poller 不只是“读取 mailbox”，
它还负责做 turn scheduling 决策。

---

### 第十五部分：消息先“成功提交或可靠入队”后才标记为已读，说明这里明确优先保证不丢消息，而不是优先避免重复投递

位置：`src/hooks/useInboxPoller.ts:860`

注释说得很清楚：

- 只有消息成功 delivered 或 reliably queued 后才 markRead
- 如果 crash 发生在这之前，下次 poll 还��重新读到
- 避免 silent drop

这说明作者在传输语义上的取舍是：

```text
at-least-once-ish delivery semantics over file mailbox
```

允许的副作用是：

- 某些情况下可能重复读/重复排队

但不能接受的是：

- 消息永久丢失

这对协作控制消息显然更合理。

---

### 第十六部分：后半段 `useEffect` 专门处理“session 从 busy 变 idle 之后，把 pending inbox messages 补投成新 turn”，说明 AppState inbox 队列本身就是 mailbox 与 query loop 之间的缓冲层

位置：`src/hooks/useInboxPoller.ts:875`

这个 effect 做了三件事：

1. 清理 `processed` 消息
2. 找出 `pending` 消息
3. 在 session idle 时把它们格式化并再尝试 `onSubmitMessage`

如果成功提交：

- 从 AppState inbox 中删除这些 pending

如果失败：

- 保持排队

这说明 AppState 中的 `inbox.messages` 并不是 UI-only display cache，
而是：

```text
mailbox-to-turn delivery buffer
```

即：

- mailbox 文件是跨 agent transport
- AppState inbox 是本进程内的延迟投递队列
- query loop 变 idle 时再真正交给 Claude 新一轮处理

---

### 第十七部分：这个 hook 的轮询启动条件、初始 poll、以及 useRef 防重入设计，说明作者希望它既能“及时看到新消息”，又不会在 React 生命周期里反复乱触发

位置：

- `useInterval(...)` `src/hooks/useInboxPoller.ts:952`
- initial poll `src/hooks/useInboxPoller.ts:956`

这里的策略是：

- 只有 `enabled && !!getAgentNameToPoll(...)` 才 poll
- mount 时先来一次 initial poll
- 用 `hasDoneInitialPollRef` 防止重复 initial poll

这说明 hook 既考虑到：

#### 及时性
刚挂载就先收一次消息

#### 稳定性
不要因为 rerender / appState 变化不断触发“伪初始 poll”

这也是一个典型的 React runtime hygiene 设计。

---

### 第十八部分：整份文件真正把 mailbox 子系统从“文件消息存储”提升成了“团队控制流转中枢”

如果把整份文件压缩，会发现它其实在做四层事情：

#### 1. 身份与适用范围判定
- 谁该 poll
- 谁不该 poll（in-process teammate）

#### 2. 协议分类与路由
- permission / sandbox / shutdown / mode / plan approval

#### 3. AppState / UI / lifecycle 副作用
- ToolUseConfirm queue
- workerSandboxPermissions queue
- permission context 更新
- team file / task / teammate cleanup
- inbox queue / notifications

#### 4. 模型 turn 注入调度
- idle 立即提交
- busy 排队
- 空闲后再投递

所以最准确的压缩表达是：

```text
useInboxPoller.ts = swarm mailbox 的运行时分拣站：把 per-agent inbox 文件中的协议消息转换成权限队列、状态更新、生命周期动作与新的 Claude turn
```

---

### 读完这一站后，你应该抓住的 10 个事实

1. `useInboxPoller.ts` 不是 mailbox 存储层，而是 mailbox 的运行时消费器与分发器，负责 leader / process-based teammate 对 inbox 的定期轮询与路由。
2. in-process teammate 被明确排除在这个 hook 之外，因为它们使用 `waitForNextPromptOrShutdown()`，共享 React/AppState 时再走 inbox poller 会造成路由混乱。
3. 轮询采用 1s 间隔，说明 mailbox 被建模成低频控制面轮询系统，而不是实时事件流。
4. unread messages 会先被严格分类成 permission、sandbox permission、shutdown、mode set、plan approval、regular text 等多个队列，只有剩余 regular messages 才会进入模型输入面。
5. leader 收到 worker 的 permission request 时，会把它转成标准 `ToolUseConfirm` 队列项，因此 process-based teammate 也能复用 leader 端统一的权限审批 UI。
6. worker 侧收到 permission/sandbox permission response 时，并不是立即自己处理业务，而是通过 callback registry 恢复之前挂起的请求 continuation。
7. team permission update 与 mode set request 会直接修改 teammate 本地的 `toolPermissionContext`，其中 mode change 还会同步写回 team file，体现运行态与持久 team metadata 的联动。
8. leader 收到 plan approval request 时会自动批准并回信，但仍把原消息继续透传给 regularMessages，兼顾控制面自动化与模型上下文可见性。
9. shutdown approval 在 leader 侧会触发真正的 lifecycle 收尾：kill pane、移除 team file 成员、解除 task 分配、更新 AppState 和 system inbox message，尤其补上了 process-based teammate 缺少本地 runner 的 completed 状态收尾。
10. 普通 teammate 消息的投递策略是“idle 立即提交成新 turn，busy 先排进 AppState inbox 队列，等空闲再补投”，并且只有成功提交或可靠入队后才 mark read，以避免消息丢失。

---

### 现在把第 54-55 站串起来

```text
teammateMailbox.ts
  -> 定义 per-agent inbox 文件、已读语义、structured protocol messages 与 XML bridge
useInboxPoller.ts
  -> 定期消费 unread mailbox messages，把它们按协议路由到权限队列、状态更新、lifecycle cleanup 或新的 Claude turn
```

所以现在 swarm mailbox 控制面可以先压缩成：

```text
file mailbox transport
  -> teammateMailbox.ts

runtime consumer / router
  -> useInboxPoller.ts

worker-side callback completion
  -> useSwarmPermissionPoller.ts / inProcessRunner.ts
```

也就是说：

```text
teammateMailbox.ts 回答“消息怎样存、怎样定义、怎样被识别”
useInboxPoller.ts 回答“这些消息进来后，系统到底怎样把它们变成真实的运行时动作与新 turn”
```

---

### 下一站建议

下一站最顺的是：

```text
src/hooks/useSwarmPermissionPoller.ts
```

因为现在你已经看懂了：

- leader / worker 怎样通过 mailbox 来回发送 permission 和 sandbox permission 消息
- `useInboxPoller` 怎样在收到 response 后把它分发给 callback registry
- 但还没看“这些 callback registry 是谁维护的、请求等待态怎样挂起与恢复”的另一半实现

下一步最自然就是把这条权限握手链补完：

**swarm worker 侧到底怎样注册等待中的 permission / sandbox permission callback，leader 的 mailbox 响应回来后又怎样精确唤醒对应请求，把等待中的工具调用继续跑下去。**
