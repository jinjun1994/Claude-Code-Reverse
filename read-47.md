## 第 47 站：`src/utils/swarm/inProcessRunner.ts`

### 这是什么文件

`src/utils/swarm/inProcessRunner.ts` 是 swarm 体系里的 **in-process teammate 主运行循环 + idle mailbox 协调器**。

上一站 `InProcessBackend.ts` 已经看懂：

```text
in-process backend 会先：
- spawnInProcessTeammate(...) 做注册
- 再 startInProcessTeammate(...) 真正启动执行
```

但那时还没落到真正的运行内核：

- teammate 是怎样在 ALS context 里跑起来的
- 权限 ask 怎样接到 leader UI
- 历史消息怎样跨多轮 prompt 保持
- idle 后怎样继续存活而不是退出
- shutdown request 怎样交给模型决定
- task list 怎样在 idle 态自动 claim 下一项工作

所以这一站回答的是：

```text
in-process teammate 真正的执行循环、权限桥接、idle 等待、消息收发、收尾回收，是怎样在一个共享进程里被组织起来的
```

所以最准确的一句话是：

```text
inProcessRunner.ts = in-process teammate 的常驻运行时内核
```

---

### 先看它的整体定位

位置：`src/utils/swarm/inProcessRunner.ts:1`

文件头注释直接点出它负责：

- ALS context isolation
- progress tracking
- AppState updates
- idle notification
- plan mode approval flow
- cleanup on completion or abort

如果和之前几站对照：

- `spawnInProcess.ts` 负责生/死注册边界
- `teammateContext.ts` 负责 ALS 身份隔离容器
- `InProcessBackend.ts` 负责把 backend 接口接到 spawn/start

那么这份文件就是：

```text
真正把“已注册的 in-process teammate”跑起来的地方
```

也就是说它不是 helper，也不是 facade，而是：

```text
runtime loop implementation
```

---

### 第一部分：`createInProcessCanUseTool(...)` 说明 in-process teammate 的权限 ask 会优先桥接到 leader 的 ToolUseConfirm UI，而不是退化成“无法弹窗”

位置：`src/utils/swarm/inProcessRunner.ts:128`

这是全文件非常关键的一段。

它创建了一份给 `runAgent(...)` 用的 `canUseTool` 函数，逻辑大致是：

1. 先正常调用 `hasPermissionsToUseTool(...)`
2. 如果结果不是 `ask`，直接返回
3. 如果是 bash ask，先尝试 classifier auto-approval
4. 如果仍然要 ask：
   - 如果 leader 的 `ToolUseConfirm` 队列可用，直接走 leader UI 对话框
   - 否则退回 mailbox permission request/response

这说明 in-process teammate 虽然是“异步后台协作者”，
但在权限交互能力上并不弱化成简单拒绝。

它的真实语义是：

```text
只要 leader UI bridge 可用，就给 in-process teammate 复用 leader 那套完整工具审批界面
```

包括：

- BashPermissionRequest
- FileEditToolDiff
- worker badge
- onAllow/onReject/recheckPermission

所以这一段最重要的理解是：

```text
in-process teammate 的 ask permission，不是 mailbox first；
而是 leader UI bridge first，mailbox second
```

---

### 第二部分：权限桥接里专门保留了 bash classifier auto-approval，说明 in-process teammate 与主 agent 一样复用“先模型/分类器，后人工确认”的路径

位置：`src/utils/swarm/inProcessRunner.ts:156`

这里如果：

- feature `BASH_CLASSIFIER` 开启
- tool 是 Bash
- 有 pending classifier check

就会先 `await awaitClassifierAutoApproval(...)`。

而且注释还明确说：

```text
agent 会等待 classifier 结果，不像 main agent 那样与用户交互 race
```

这说明 in-process teammate 的权限语义并不是简化版，
而是仍然接在正式 Bash 权限体系上。

也就是说：

```text
subagent/in-process teammate 也能享受 classifier-based fast-path
```

只是交互时序更偏“先等自动批准结果，再决定是否上浮到 leader UI”。

---

### 第三部分：leader UI bridge 路径不仅允许 approve/reject，还会把 permission updates 回写到 leader 的共享权限上下文

位置：`src/utils/swarm/inProcessRunner.ts:195`

走 `ToolUseConfirm` 队列时，这里不只是 resolve 一个允许/拒绝结果。

在 `onAllow(...)` 里它还会：

- `persistPermissionUpdates(permissionUpdates)`
- 调 `getLeaderSetToolPermissionContext()`
- `applyPermissionUpdates(...)`
- `setToolPermissionContext(updatedContext, { preserveMode: true })`

这里最关键的是 `preserveMode: true` 的注释：

```text
防止 worker 侧变换后的 acceptEdits context 泄漏回 coordinator
```

这说明权限桥接不是简单“leader 点同意，worker继续”，
而是还在维护：

```text
leader-side shared permission context 的一致性
```

这是一种非常成熟的控制面设计。

---

### 第四部分：当 leader UI bridge 不可用时，fallback mailbox 权限流在 in-process teammate 里仍然是完整闭环，而不是单向请求

位置：`src/utils/swarm/inProcessRunner.ts:336`

fallback 分支做的是：

1. `createPermissionRequest(...)`
2. `registerPermissionCallback(...)`
3. `sendPermissionRequestViaMailbox(request)`
4. 每 500ms poll 自己 mailbox
5. 找到匹配的 permission response 后：
   - `markMessageAsReadByIndex(...)`
   - `processMailboxPermissionResponse(...)`
6. callback resolve promise

这说明 in-process teammate 在没有 UI bridge 时，
会临时退回到和 process-based teammate 非常相似的 mailbox ask/response 协议。

所以这一段本质上是：

```text
in-process teammate 以 leader UI bridge 为首选，
但保留一整套 transport-compatible mailbox fallback
```

这也正是 swarm “同一协议语义，不同执行后端”设计思想的体现。

---

### 第五部分：`formatAsTeammateMessage(...)` 说明 in-process teammate 在消息呈现上会主动伪装成 process-based teammate 的 XML 语义

位置：`src/utils/swarm/inProcessRunner.ts:457`

这个函数会把消息格式化成：

```xml
<teammate-message teammate_id="..." color="..." summary="...">
...
</teammate-message>
```

注释直接说明：

```text
这样模型看到的格式就和 tmux teammates 一样
```

这说明 in-process 和 process-based teammate 在 transport/backend 层虽然不同，
但在送给模型的会话语义层，系统刻意维持：

```text
统一 teammate message surface
```

也就是说模型不需要知道：

- 这条消息来自 in-process teammate
- 还是 tmux teammate

因为进入 prompt 时已经统一封装成同一类 teammate-message XML。

---

### 第六部分：`sendMessageToLeader()` / `sendIdleNotification()` 说明 in-process teammate 对 leader 的外部可见状态仍通过 mailbox 通知，而不是直接写 leader UI

位置：

- `sendMessageToLeader()` `src/utils/swarm/inProcessRunner.ts:547`
- `sendIdleNotification()` `src/utils/swarm/inProcessRunner.ts:569`

这两个函数都落到：

- `writeToMailbox(TEAM_LEAD_NAME, ...)`

而 idle 通知会先用：

- `createIdleNotification(...)`

说明即便 leader 与 teammate 共处同一进程，
系统仍然在很多“对外协调信号”上保持 mailbox 作为统一控制通道。

所以这里再次坐实：

```text
in-process teammate 的执行态是 shared AppState/ALS，
但对 leader 的团队协调事件仍尽量通过 mailbox family 暴露
```

这样 leader 侧的处理逻辑就可以和 process-based teammate 尽量共用。

---

### 第七部分：`findAvailableTask()` / `tryClaimNextTask()` 揭示了 in-process teammate 在 idle 态不仅等消息，还会主动从团队 task list 里拉活

位置：

- `findAvailableTask()` `src/utils/swarm/inProcessRunner.ts:595`
- `tryClaimNextTask()` `src/utils/swarm/inProcessRunner.ts:624`

这里的逻辑非常重要：

#### 可用 task 定义
- `status === 'pending'`
- 没有 owner
- `blockedBy` 全部已解决

#### claim 流程
- `listTasks(taskListId)`
- 找第一个可用 task
- `claimTask(...)`
- 再 `updateTask(..., { status: 'in_progress' })`
- 把 task 格式化成 prompt

这说明 in-process teammate 并不是单纯被动等待 leader 发消息。

它在 idle loop 里会主动参与：

```text
shared task list 的自助领活机制
```

这非常关键，因为它把 agent team 从“leader 单点调度”推进到了：

```text
leader + task list + teammate autonomous claiming
```

的协作模型。

---

### 第八部分：`waitForNextPromptOrShutdown(...)` 才是 teammate 常驻不退出的关键——它把 idle 态建模成一个轮询驱动的等待循环，而不是 terminal state

位置：`src/utils/swarm/inProcessRunner.ts:689`

这个函数会一直循环，直到：

- abort
- 收到 shutdown request
- 收到新消息
- 或 claim 到新 task

而它的检查顺序很值得注意：

1. 先看内存态 `pendingUserMessages`
2. 再看 abort
3. 再读 mailbox
4. mailbox 里先找 shutdown request
5. 再找 team-lead 消息
6. 再找其他 unread message
7. 最后去 team task list 里尝试 claim 下一项任务

这说明 idle 态并不是：

```text
任务完成 -> 进程结束
```

而是：

```text
任务完成 -> teammate 进入常驻待命循环
```

直到明确被 kill 或 shutdown approved。

这正是“常驻 teammate”成立的基础。

---

### 第九部分：`pendingUserMessages` 的优先级最高，说明 transcript 视图里注入的用户消息会绕过 mailbox，直接走本地快速通道

位置：`src/utils/swarm/inProcessRunner.ts:705`

idle loop 的第一检查就是：

- 当前 task 是否有 `pendingUserMessages`

如果有：

- 取出第一条
- 从队列里 pop
- 立即返回 `type: 'new_message', from: 'user'`

这说明从 transcript / UI 视图里给 in-process teammate 补消息时，
系统会走一条比 mailbox 更直接的路径：

```text
AppState pendingUserMessages queue
```

所以同样是“给 teammate 送消息”，
在 in-process runner 里其实已经有优先级分层：

#### 最高优先级
本地内存注入消息（最快）

#### 其次
mailbox unread messages

#### 最后
主动 claim task

这是一个非常实用的本地协作优化。

---

### 第十部分：mailbox polling 专门把 shutdown request 提到最高优先级，并让 team-lead 消息压过 peer 消息，说明 idle loop 明确有控制面优先级设计

位置：`src/utils/swarm/inProcessRunner.ts:760`

这里的顺序设计非常成熟：

#### 第一级：shutdown request
先扫描所有 unread，优先找 shutdown

#### 第二级：team-lead message
如果没有 shutdown，再优先拿 leader 发来的消息

#### 第三级：其他 unread peer message
最后才按 FIFO 处理普通 peer 消息

注释还特别说明了原因：

- 防止 shutdown 被 peer chatter 饥饿
- leader 代表用户意图与协调控制，不应被 peer chatter 压住

所以这一段真正表达的是：

```text
idle mailbox 不是简单 FIFO，
而是 control-plane-priority queue semantics
```

这对 swarm 协调正确性非常关键。

---

### 第十一部分：`runInProcessTeammate(...)` 先构造 `AgentContext` 和系统 prompt，再进入 while-loop，说明真正的 in-process teammate 既是一个 agent，也是一条长期会话

位置：`src/utils/swarm/inProcessRunner.ts:883`

一进入主函数，先做两件大事：

#### 1. 构造 `AgentContext`
字段包括：

- `agentId`
- `parentSessionId`
- `agentName`
- `teamName`
- `agentColor`
- `planModeRequired`
- `isTeamLead: false`
- `agentType: 'teammate'`
- `invokingRequestId`
- invocation metadata

这说明 analytics / tracing 里，in-process teammate 被正式当成独立 agent 节点对待。

#### 2. 构造 teammateSystemPrompt
来源包括：

- 默认系统 prompt
- `TEAMMATE_SYSTEM_PROMPT_ADDENDUM`
- custom agent definition prompt
- append/replace 模式下的额外 systemPrompt

这说明 in-process teammate 不是简单“调用 runAgent 一下”，
而是有自己正式独立的 agent persona / instruction stack。

---

### 第十二部分：`resolvedAgentDefinition` 强制注入团队关键工具，说明 teammate 即使被显式工具列表限制，也必须保留最小协作能力

位置：`src/utils/swarm/inProcessRunner.ts:972`

这里很关键。

如果 custom agent definition 给了工具列表，
系统会额外并入：

- `SendMessage`
- `TeamCreate`
- `TeamDelete`
- `TaskCreate`
- `TaskGet`
- `TaskList`
- `TaskUpdate`

注释直接说明：

```text
即使 explicit tool list 存在，也要注入这些 team-essential tools，
否则 teammate 无法响应 shutdown、发消息、走 task list 协作
```

这说明在 swarm 设计里，某些工具不是可选能力，而是：

```text
teammate minimum viability tools
```

也就是说再定制也不能删掉这些协作基元。

这是一个非常重要的系统约束。

---

### 第十三部分：`permissionMode: 'default'` 的注释很关键——teammate 不继承 leader 的 permissionMode，而是总拿 full tool access 再在运行时受 task state mode 约束

位置：`src/utils/swarm/inProcessRunner.ts:973`

这里注释写得非常明确：

```text
IMPORTANT: Set permissionMode to 'default' so teammates always get full tool
access regardless of the leader's permission mode.
```

这看起来有点反直觉，但结合后面每轮会从 task state 读取 `currentPermissionMode` 就能看懂：

- 定义层不要把 teammate 永久锁死在 leader 当前 mode
- 真正生效的当前 mode，在每轮 iteration 前从 task state 动态读取

所以这里的建模是：

```text
agent definition 层保持工具全集能力
实际 permission mode 在运行时由 task state 动态控制
```

这能让 leader 之后改 teammate mode 时，
不必重建整个 agent definition。

---

### 第十四部分：`allMessages` + compaction 逻辑说明 in-process teammate 是一条长期会话，不是一次性 prompt execution

位置：`src/utils/swarm/inProcessRunner.ts:1003`

这里有：

- `allMessages: Message[] = []`
- `currentPrompt`
- `while (!abort && !shouldExit)`

每轮都会：

- 把当前 user prompt 追加进去
- 跑一轮 `runAgent(...)`
- 把产出的消息累积进 `allMessages`
- 下一轮把这些消息当作 `forkContextMessages`

而且还会在 token 超阈值时：

- `compactConversation(...)`
- `buildPostCompactMessages(...)`
- reset microcompact state
- reset content replacement state
- 同步更新 `task.messages`

这说明 in-process teammate 被明确建模成：

```text
可跨多轮继续工作的长寿命 agent conversation
```

不是单轮做完就丢弃的 background task。

---

### 第十五部分：compaction 时专门克隆 `readFileState` 并关掉 UI callbacks，说明 teammate compaction 被设计成与主会话相互隔离

位置：`src/utils/swarm/inProcessRunner.ts:1081`

这里为了 compact，会创建一个 `isolatedContext`：

- `readFileState: cloneFileStateCache(...)`
- `onCompactProgress: undefined`
- `setStreamMode: undefined`

注释写得很明确：

```text
不要让 teammate 的 compact 清掉主 session 的 readFileState cache，
也不要触发主 session 的 UI callbacks
```

这说明即使 teammate 与 leader 共进程，
作者仍然很小心地避免：

```text
subagent internal maintenance operation
-> 污染 main session runtime/UI state
```

这是共享进程架构里非常成熟的一种隔离意识。

---

### 第十六部分：每轮都有独立 `currentWorkAbortController`，说明 in-process teammate 明确区分“停止当前工作”与“杀死整个 teammate”两个取消域

位置：`src/utils/swarm/inProcessRunner.ts:1053`

这里每轮 turn 都会创建：

- `currentWorkAbortController`

而外层还有：

- `abortController`

注释明确说：

```text
Escape 只停止当前工作，不杀死整个 teammate；
生命周期 abort 才杀整个 teammate
```

这说明 in-process teammate 的取消模型被分成两层：

#### work abort
- 仅当前 turn
- 对应 UI 中断当前工作

#### lifecycle abort
- 整个 teammate 死亡
- 对应 kill / whole-agent termination

这是非常重要的运行时细节，
也是共享进程里用户体验能更细腻的原因之一。

---

### 第十七部分：`runAgent(...)` 是真正的内核调用，而 `runWithTeammateContext(...)` + `runWithAgentContext(...)` 说明 in-process teammate 的运行会同时挂上身份隔离和分析归因上下文

位置：`src/utils/swarm/inProcessRunner.ts:1159`

核心运行片段是：

- `runWithTeammateContext(teammateContext, ...)`
- `runWithAgentContext(agentContext, ...)`
- `for await (const message of runAgent(...))`

这说明 in-process teammate 的一次 turn 并不是裸跑 `runAgent`，
而是在两层上下文下运行：

#### teammateContext
提供 ALS 身份隔离

#### agentContext
提供 analytics / lineage / tracing attribution

然后才进入共享的 `runAgent` 主循环。

所以 in-process teammate 的真正运行模型可以压缩成：

```text
ALS teammate identity
+
agent attribution context
+
shared runAgent core loop
```

这就是为什么它既像独立 agent，又不需要独立进程。

---

### 第十八部分：每条 `runAgent` 产出的 message 都会同步进 progress、task.messages、inProgressToolUseIDs，说明 task state 就是 in-process teammate 的实时 UI 镜像层

位置：`src/utils/swarm/inProcessRunner.ts:1221`

每条 message 进来后，会做三件事：

1. `iterationMessages.push(message)`
2. `allMessages.push(message)`
3. `updateProgressFromMessage(...)`
4. 更新 task state：
   - `progress`
   - `messages`
   - `inProgressToolUseIDs`

尤其 `inProgressToolUseIDs` 的维护很有意思：

- assistant 发出 `tool_use` -> 加入集合
- user `tool_result` 回来 -> 从集合删除

这说明 transcript view 里的“某工具正在执行”的动画状态，并不是另一个独立系统，
而是直接从 agent loop 消息流增量维护出来的。

所以：

```text
AppState 的 in-process teammate task
= 运行内核对 UI/状态层的实时投影
```

---

### 第十九部分：一轮结束后并不会把结果自动发给 leader，这说明 teammate 产出默认是私有的，必须显式 SendMessage 才能上浮

位置：`src/utils/swarm/inProcessRunner.ts:1328`

这里注释写得很明确：

```text
We do NOT automatically send the teammate's response to the leader.
Teammates should use the Teammate tool to communicate with the leader.
This matches process-based teammates where output is not visible to the leader.
```

这点非常重要。

因为它说明在 swarm 里，teammate 完成一轮工作后：

```text
leader 不会自动看到完整输出
```

teammate 必须：

- 主动调用 SendMessage/Teammate tool
- 或通过 task/idle summary 等更轻量通道报告

这正是 swarm 协作模型中“主动汇报”而非“透明共享 transcript”的核心原则之一。

---

### 第二十部分：idle transition 会触发 `onIdleCallbacks` + idle mailbox notification，说明本地等待者与 leader 观察者两类消费者都被照顾到了

位置：`src/utils/swarm/inProcessRunner.ts:1317`

一轮结束后，runner 会：

1. 更新 task 为 `isIdle: true`
2. 调所有 `onIdleCallbacks`
3. 必要时发送 `sendIdleNotification(...)`

这说明 idle 事件有两类受众：

#### 本地进程内等待者
例如前一站 `waitForTeammatesToBecomeIdle(...)` 注册的 callback

#### leader / 外部协调面
通过 mailbox 接收 idle notification

所以 idle 不是单纯状态位翻转，
而是一次：

```text
local callback broadcast + cross-agent mailbox notification
```

的复合事件。

---

### 第二十一部分：`waitForNextPromptOrShutdown(...)` 返回 shutdown request 后，不是直接自动退出，而是把 shutdown request 包成 teammate-message 再交还给模型决定

位置：`src/utils/swarm/inProcessRunner.ts:1363`

收到 shutdown request 后：

- 不自动 approve
- 不直接退出
- 而是 `formatAsTeammateMessage(...)`
- 再把它 append 到 transcript
- 让模型下一轮处理

注释明确说：

```text
模型会用 approveShutdown 或 rejectShutdown tool 做决定
```

这说明 in-process teammate 的 shutdown 不是外部强行拍板的，
而是保留了一层：

```text
agent-self-determination over graceful shutdown
```

也就是：

- 当前工作是否已完成
- 是否可安全退出
- 是否应拒绝 shutdown

这些仍由 teammate 模型自己判断。

这是很关键的协作设计。

---

### 第二十二部分：退出收尾逻辑非常谨慎，专门避免和 `killInProcessTeammate(...)` 双重终结冲突

位置：`src/utils/swarm/inProcessRunner.ts:1419`

正常退出时，代码先检查：

- `if (task.status !== 'running') alreadyTerminal = true`

注释明确说明：

```text
killInProcessTeammate 可能已经把 status 设成 killed + notified true；
这里不能再把 killed 覆盖成 completed，也不能重复 emit SDK 终止事件
```

这说明作者非常清楚这里有并发 race：

- runner 正常收尾
- 外部 kill 同时发生

如果不加这层判断，就会出现：

- 状态回滚/覆盖
- 双重终止通知
- UI 与 SDK 语义错乱

所以这一段封装的是非常关键的并发正确性。

---

### 第二十三部分：失败路径同样会发 idle notification，而且 `completedStatus: 'failed'`，说明失败也被纳入统一的“teammate 已闲置/已结束”对外通知语义

位置：`src/utils/swarm/inProcessRunner.ts:1465`

catch 分支里会：

- task 标成 `failed`
- evict output / evict terminal task
- emitTaskTerminatedSdk(..., 'failed')
- `sendIdleNotification(... { idleReason: 'failed', completedStatus: 'failed', failureReason })`

这说明对 leader/团队观察面来说，
失败并不是一个完全独立的信号通道，
而仍然被纳入：

```text
idle/availability notification family
```

只是 reason/status 不同。

因此 idle notification 其实不只是“我空闲了”，
更像：

```text
我这一轮/这一生命周期已经进入某种可协调的稳定状态
```

---

### 第二十四部分：`startInProcessTeammate(...)` 看似只是 fire-and-forget 包装，但它还专门避免 closure 长期持有完整 config，说明作者非常在意长寿命 teammate 的内存保持问题

位置：`src/utils/swarm/inProcessRunner.ts:1544`

最后这个函数只是：

- 提前抽出 `agentId`
- `void runInProcessTeammate(config).catch(...)`

但注释非常关键：

```text
先把 agentId 抽出来，避免 catch closure 持有完整 config（尤其 toolUseContext）很多小时
```

这说明作者明确把 in-process teammate 看成：

```text
可能持续很久的长寿命对象
```

因此即使是一个小小的 `.catch(...)` 闭包，
也会考虑是否把整包 config 长时间 pin 住。

这和前面清空 `toolUseContext.messages` 的做法完全呼应：

```text
共享进程模型里，内存生命周期控制是一级公民问题
```

---

### 第二十五部分：这份文件真正把 in-process teammate 的完整模型串起来了——“共享 runAgent 内核 + 本地 task 状态机 + mailbox 协议兼容 + 常驻 idle loop”

如果把整份文件压缩，你会发现它不是单一机制，而是四层一起工作：

#### 1. 共享 agent execution core
- `runAgent(...)`
- compact / token budget / permission system

#### 2. 本地身份与运行时隔离
- `runWithTeammateContext(...)`
- `runWithAgentContext(...)`
- `AbortController`
- task state

#### 3. 协议与协调兼容层
- mailbox send/receive
- permission request/response
- shutdown request
- idle notification

#### 4. 常驻 teammate loop
- 完成一轮后不退出
- idle 等待下一条 prompt / shutdown / task claim
- 再进入下一轮

所以最准确的压缩表达就是：

```text
inProcessRunner.ts = 在共享 Node.js 进程里，把普通 runAgent 提升成可常驻、可协调、可被团队控制的 teammate runtime
```

---

### 读完这一站后，你应该抓住的 12 个事实

1. `inProcessRunner.ts` 是 in-process teammate 的真正运行时内核，负责常驻循环、权限桥接、AppState 同步、idle 等待与收尾回收。
2. `createInProcessCanUseTool(...)` 优先把 ask permission 桥接到 leader 的 `ToolUseConfirm` UI；只有 bridge 不可用时才退回 mailbox permission 流。
3. in-process teammate 同样复用 bash classifier auto-approval，说明它并没有脱离正式权限体系。
4. teammate 消息会被包装成统一的 `<teammate-message>` XML，使 in-process 与 process-based teammate 在模型视角下保持同一消息表面语义。
5. `sendIdleNotification()` 等协调信号仍通过 mailbox 发给 leader，说明执行态虽是 shared AppState，但对外协调面仍尽量复用 mailbox family。
6. idle loop 不仅等待新消息，还会主动从共享 task list 里 claim 可用任务，说明 teammate 具备一定的自主领活能力。
7. `waitForNextPromptOrShutdown(...)` 明确实现了控制面优先级：shutdown request > team-lead message > peer message > task list claim。
8. `runInProcessTeammate(...)` 会构造独立的 `AgentContext`、system prompt、resolved agent definition，说明 in-process teammate 是正式独立 agent，而不是 leader turn 的内联 helper。
9. team-essential tools（如 `SendMessage`、`Task*`、`Team*`）会被强制注入 agent definition，保证 teammate 无论工具列表如何都保留最小协作能力。
10. 每轮 turn 都有独立 `currentWorkAbortController`，与生命周期 `abortController` 分离，分别对应“中断当前工作”和“杀死整个 teammate”。
11. 一轮工作结束后，runner 会进入 idle 而不是退出；只有 abort 或 shutdown 获批才真正结束整个 teammate 生命周期。
12. 该文件到处都在做长寿命内存边界控制，如清空 `toolUseContext.messages`、使用 isolated compact context、避免 catch 闭包长期持有完整 config，说明共享进程模型下的内存管理是核心设计考虑。

---

### 现在把第 46-47 站串起来

```text
InProcessBackend.ts
  -> 负责把 spawn/start/terminate/kill/isActive 接到 in-process task/runtime 模型上
inProcessRunner.ts
  -> 负责真正跑起 teammate 的长期执行循环、权限桥接、idle 等待、消息与 shutdown 协调
```

所以现在 in-process teammate 这条主链可以先压缩成：

```text
backend facade
  -> InProcessBackend

spawn/register lifecycle
  -> spawnInProcess.ts

runtime identity isolation
  -> teammateContext.ts

long-running execution loop
  -> inProcessRunner.ts
```

也就是说：

```text
InProcessBackend 决定“怎么把 teammate 启动起来”
inProcessRunner 决定“启动后它怎样长期活着并参与团队协作”
```

---

### 下一站建议

下一站最顺的是：

```text
src/utils/swarm/spawnUtils.ts
```

因为现在你已经看懂了：

- pane backend 会通过 `PaneBackendExecutor` 构造 CLI 启动命令
- in-process backend 则绕过 CLI 直接进本地 runner
- `spawnUtils.ts` 正好是 pane/process teammate 启动命令构造链里的公共拼装层

下一步最自然就是回到 process-based teammate 的启动拼装细节：

**`getTeammateCommand()`、`buildInheritedCliFlags()`、`buildInheritedEnvVars()` 这些公共 spawn helper 到底怎样决定 process-based teammate 启动时继承哪些 leader 配置、注入哪些身份与运行时参数。**
