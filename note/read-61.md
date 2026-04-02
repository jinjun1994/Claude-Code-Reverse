## 第 61 站：`src/hooks/toolPermission/handlers/interactiveHandler.ts`

### 这是什么文件

`src/hooks/toolPermission/handlers/interactiveHandler.ts` 是工具权限 ask-path 里的 **本地交互审批处理器与多路审批竞态协调器**。

上一站 `PermissionContext.ts` 已经看懂：

- handler 都围绕 `ctx` 共享能力层来工作
- `ctx` 负责 logging、abort、allow/deny 构造、hooks、classifier、queue 操作
- 但 ask-path 最终兜底那条“本地确认框”自己到底怎样工作，还没拆开

这一站回答的是：

```text
当 ask-mode 最终进入本地交互权限路径时，
系统怎样把请求推入确认队列，
又怎样让本地用户、bridge、channel relay、hooks、classifier
这些不同审批源并发竞争，
最后只让一个胜出并收尾
```

所以最准确的一句话是：

```text
interactiveHandler.ts = ask-mode 权限对话框的队列接入点 + 多审批源竞态仲裁器
```

---

### 先看它的整体定位

位置：`src/hooks/toolPermission/handlers/interactiveHandler.ts:1`

这个文件不是：

- 底层 permission policy evaluator
- swarm worker 上抛 handler
- bridge / channel transport 本体
- classifier 算法本体

它的职责是：

1. 把 ask 请求包装成 `ToolUseConfirm` queue 项
2. 挂上所有本地用户交互回调
3. 同时启动 remote bridge / channel relay / hooks / classifier 这些异步审批源
4. 用 one-shot resolve 机制确保只允许一个审批结果落地
5. 统一处理 UI 清理、queue 移除、classifier indicator 清理、bridge/channel 取消订阅

所以它本质上是一个：

```text
interactive permission race coordinator
```

而不只是“弹一个确认框”。

---

### 第一部分：文件头注释已经点明它的核心不是渲染，而是“设置 callbacks，让未来某个时刻 resolve 外层 Promise”

位置：`src/hooks/toolPermission/handlers/interactiveHandler.ts:43`

注释里明确写：

- push 一个 `ToolUseConfirm` entry 到 queue
- 挂 `onAbort / onAllow / onReject / recheckPermission / onUserInteraction`
- 异步启动 hooks 和 bash classifier
- race against user interaction
- 使用 resolve-once guard
- **本函数不返回 Promise**

这说明它的真实定位是：

```text
wire up an approval race,
not synchronously decide anything itself
```

所以这份文件其实是“审批 orchestration 的 UI-facing wiring 层”。

---

### 第二部分：`createResolveOnce(resolve)` + `userInteracted` + 一堆 cleanup handle，说明这个 handler 一上来就在准备处理并发审批竞争，而不是只准备本地点击事件

位置：`src/hooks/toolPermission/handlers/interactiveHandler.ts:70`

一开始就准备了：

- `resolveOnce / isResolved / claim`
- `userInteracted`
- `checkmarkTransitionTimer`
- `checkmarkAbortHandler`
- `bridgeRequestId`
- `channelUnsubscribe`

这说明 ask-mode interactive path 的真实复杂度不在“允许/拒绝按钮”，
而在：

```text
many possible approvers / finishers may race:
local user, remote bridge, channel relay, hooks, classifier, abort
```

因此这个 handler 的首要任务是建立竞态控制与收尾句柄，
而不是直接显示 UI。

---

### 第三部分：`displayInput = result.updatedInput ?? ctx.input` 说明进入 interactive path 时，界面展示给用户看的输入，已经允许是 ask 前面流程改写过的版本

位置：`src/hooks/toolPermission/handlers/interactiveHandler.ts:83`

这里很细但很关键：

- 如果之前某个阶段已经给了 `updatedInput`
- 本地交互 UI 显示的就是那个版本

这说明 ask-path 不是线性的“原始输入 -> 最终审批”，
而是允许前置阶段先改写输入，
之后 UI 再基于改写后的输入继续审批。

也就是说 interactive handler 接的不是纯原始 tool input，
而是：

```text
current best candidate input for this permission request
```

这和前面 `useCanUseTool.tsx`、`swarmWorkerHandler.ts` 里一路传 `updatedInput` 完全一致。

---

### 第四部分：`ctx.pushToQueue(...)` 说明 interactive path 的第一步不是弹窗 API，而是把请求建模成可持久操作的 queue item；权限 UI 是队列驱动的

位置：`src/hooks/toolPermission/handlers/interactiveHandler.ts:92`

这里会把一个完整 `ToolUseConfirm` entry push 进去，包含：

- `assistantMessage`
- `tool`
- `description`
- `input`
- `toolUseContext`
- `toolUseID`
- `permissionResult`
- `permissionPromptStartTimeMs`
- classifier 相关状态
- 各种 callbacks

这说明权限对话框不是“函数调用弹出一次 modal”，
而是：

```text
append a permission-request model object to a shared confirmation queue
```

这使得权限 UI 能支持：

- 队列化展示
- 中途 update
- transition/checkmark 状态
- 同时被本地与远程审批源驱动更新

所以 ask UI 本质上是 queue-driven state machine。

---

### 第五部分：`onUserInteraction()` 的 200ms grace period 很精细，说明作者在保护 classifier auto-approval 不被用户的“惯性按键”过早打断

位置：`src/hooks/toolPermission/handlers/interactiveHandler.ts:108`

这里逻辑是：

- 用户一旦真的开始交互
- 就 `userInteracted = true`
- `clearClassifierChecking(...)`
- `clearClassifierIndicator()`

但前 200ms 内的交互直接忽略。

注释解释得很清楚：

- 防 accidental keypress
- 防 classifier 被过早 cancel

这说明 ask UI 和 classifier 之间不是简单互斥，
而是有一个 UX 优化窗口：

```text
give auto-approval a tiny head start,
so the dialog doesn't lose to incidental user input immediately
```

这是一个很细的交互层打磨。

---

### 第六部分：`onAbort / onAllow / onReject` 三个本地回调，不只是处理本地用户操作，还顺手同步清理 bridge/channel 远程审批面，说明本地审批和远程审批是同一个竞态空间里的参与者

位置：

- abort `src/hooks/toolPermission/handlers/interactiveHandler.ts:137`
- allow `src/hooks/toolPermission/handlers/interactiveHandler.ts:154`
- reject `src/hooks/toolPermission/handlers/interactiveHandler.ts:183`

这三个回调都有共同模式：

- `if (!claim()) return`
- 若 bridge 存在，发 response / cancelRequest
- `channelUnsubscribe?.()`
- 记录 decision / cancel
- `resolveOnce(...)`

这说明本地 UI 并不是孤立确认框。

它和：

- bridge (claude.ai/code 等远端 UI)
- channel relay (Telegram/iMessage 等)

处在同一个审批 race 里。

谁先赢，其他通道都要被撤销或静默失效。

所以这里体现的是：

```text
local approval callbacks are global race winners,
not merely local UI events
```

---

### 第七部分：`recheckPermission()` 很关键，它允许外部状态变化后重新跑 `hasPermissionsToUseTool(...)`，说明权限对话框在显示期间并不是静态的

位置：`src/hooks/toolPermission/handlers/interactiveHandler.ts:204`

这里会重新执行：

- `hasPermissionsToUseTool(...)`

如果 freshResult 变成 allow：

- claim
- cancel bridge request
- unsubscribe channel
- remove queue item
- 记 accept/config
- `resolveOnce(ctx.buildAllow(...))`

这说明 ask dialog 不是“只等用户点按钮”。

它还支持：

```text
while dialog is open,
external state changes may make the tool allowed without further manual action
```

注释也提到一个具体场景：

- CCR 触发 mode switch
- 然后本地 recheck
- 工具其实已可执行

这说明 permission dialog 是一个动态可重判的等待态，
不是静态 modal snapshot。

---

### 第八部分：bridge block 说明 interactive path 可以把权限请求镜像到 CCR/远端 UI，并与本地点击并发竞速，谁先回应谁赢

位置：`src/hooks/toolPermission/handlers/interactiveHandler.ts:234`

这里当存在 `bridgeCallbacks + bridgeRequestId` 时：

- `sendRequest(...)` 到 bridge
- 订阅 `onResponse(...)`
- response 到来后：
  - `claim()`
  - 清 classifier / queue / channel
  - allow -> persist updates + log + buildAllow
  - deny -> log + cancelAndAbort

注释写得很清楚：

```text
Whichever side (CLI or CCR) responds first wins via claim().
```

这说明 bridge 不是备用通道，
而是 ask 审批里的并行参与者。

更关键的是它还能返回：

- `updatedInput`
- `updatedPermissions`

所以 bridge 审批语义比 channel relay 更丰富，
几乎等价于一个远程完整审批 UI。

---

### 第九部分：bridge allow 路径里会 `ctx.persistPermissions(response.updatedPermissions)`，说明远端 UI 的“always allow”类操作会被本地正式接纳并落到持久权限上下文里

位置：`src/hooks/toolPermission/handlers/interactiveHandler.ts:266`

这里远端 allow 时：

- 如果有 `updatedPermissions`
- 直接 `void ctx.persistPermissions(...)`
- log source 为 `user/permanent`
- `resolveOnce(ctx.buildAllow(...))`

这说明 remote bridge 审批不是只能点一次 yes/no，
而是能产生和本地对话框同等级的 permission update 副作用。

也就是说在权限语义上：

```text
remote bridge user is treated as a first-class approver
```

而不是低配代理按钮。

---

### 第十部分：channel relay block 表明 ask-path 还支持通过 MCP channel client 把权限请求转发到消息通道，但这条路径被刻意限制成纯 yes/no relay

位置：`src/hooks/toolPermission/handlers/interactiveHandler.ts:300`

这里的设计和 bridge 不同：

- 只在 `!ctx.tool.requiresUserInteraction?.()` 时启用
- 通过 channel MCP notification 发结构化请求
- response 只有 allow/deny
- 没有 `updatedInput` 路径

注释直接说：

- channel replies are pure yes/no
- today 某些需要复杂 UI 的工具根本不会走这里

这说明 channel relay 的定位是：

```text
lightweight remote approval relay for simple permission decisions
```

而不是全功能远程审批面。

---

### 第十一部分：channel relay 的 outbound 请求采用结构化 params 而不是旧的 text/content/message hack，说明这一层已经从“猜插件参数名”升级成稳定协议

位置：`src/hooks/toolPermission/handlers/interactiveHandler.ts:328`

注释里直接点出：

- server owns platform formatting
- CC sends raw structured parts
- old `send_message(text/content/message)` triple-key hack is gone

这里发送的参数是：

- `request_id`
- `tool_name`
- `description`
- `input_preview`

这说明 channel relay 不再依赖每个插件的私有参数习惯，
而是：

```text
permission relay now has a canonical MCP-level outbound schema
```

这是一个很重要的协议成熟信号。

---

### 第十二部分：`channelUnsubscribe` 的包装修复了 abort listener 泄漏问题，说明作者在这条多竞态审批链里不仅关注功能正确，还关注 session 级 listener 生命周期卫生

位置：`src/hooks/toolPermission/handlers/interactiveHandler.ts:356`

注释非常具体：

- 以前 local/hook/classifier 赢时只删 map entry
- dead closure 还挂在 session-scoped abort signal 上
- 功能上没坏，但 closure 活得太久

所以这里把 `channelUnsubscribe` 设计成同时做：

- map delete
- abort listener remove

这说明 ask-path 的 remote relay 不是“能跑就行”，
作者还在认真清理：

```text
non-functional listener retention / closure lifetime leaks
```

这是很成熟的长期运行时 hygiene。

---

### 第十三部分：hooks 在 interactive path 里是异步后台 racer，而不是进入 dialog 前必须完成的前置步骤；这说明 ask UI 显示后，hook 仍有机会抢先结束整条权限流

位置：`src/hooks/toolPermission/handlers/interactiveHandler.ts:410`

条件是：

- `!awaitAutomatedChecksBeforeDialog`

然后后台异步：

- `ctx.runHooks(...)`
- 若 hookDecision 出来且 `claim()` 成功
- cancel bridge
- unsubscribe channel
- remove queue
- `resolveOnce(hookDecision)`

这说明 interactive dialog 显示后，
并不意味着本地用户一定要亲自点。

hook 仍然可以后来居上，
直接把请求解决掉。

所以 ask dialog 的真实语义是：

```text
visible fallback surface while automated racers keep running in background
```

这正是 race coordinator 的特征。

---

### 第十四部分：bash classifier 在 interactive path 里也是一个后台 racer，而且它不仅能 auto-allow，还能驱动 UI 进入“checkmark 过渡态”而不是瞬间消失

位置：`src/hooks/toolPermission/handlers/interactiveHandler.ts:433`

这一段做了很多细事：

- `setClassifierChecking(toolUseID)`
- `executeAsyncClassifierCheck(...)`
- `shouldContinue: () => !isResolved() && !userInteracted`
- `onComplete` 清 indicator
- `onAllow` 时：
  - claim
  - cancel bridge/channel
  - 记 classifier approval
  - 更新 queue item 为 auto approved/checkmark 状态
  - `resolveOnce(ctx.buildAllow(...))`
  - 保留一段 checkmark 可见时间后再 remove dialog

这说明 classifier 在 interactive path 的角色不是简单“后台算一下”。

它还是 UI transition driver：

```text
classifier win
-> dialog shows visual auto-approved transition
-> then disappears after a short delay
```

这是非常注重交互感知的一层设计。

---

### 第十五部分：`shouldContinue: () => !isResolved() && !userInteracted` 说明 classifier 不只是怕别的审批源先赢，也会在用户真正开始交互后主动失去继续资格

位置：`src/hooks/toolPermission/handlers/interactiveHandler.ts:449`

这里的 continue 条件同时依赖：

- 这场 race 还没有人赢
- 用户还没真正开始交互

这说明 ask-path 在 UX 上的原则是：

```text
once the user is actively engaging with the dialog,
stop trying to auto-approve behind their back
```

所以 classifier 和用户交互之间不是简单速度竞赛，
还有一个“尊重用户接管”的语义边界。

---

### 第十六部分：classifier auto-allow 后保留 1s/3s checkmark，再 remove queue，说明自动批准不想表现得像“对话框闪一下就没了”，而是明确给用户一个可感知的成功过渡

位置：`src/hooks/toolPermission/handlers/interactiveHandler.ts:495`

这里：

- terminal focused -> 3s
- not focused -> 1s
- 期间用户可通过 Esc 早退
- abort 时也会清这个过渡态

这说明作者在意的是：

```text
auto-approved permission should still feel legible,
not invisible or jarring
```

所以这里不是纯功能逻辑，
而是很强的权限 UX 设计。

---

### 第十七部分：这份文件最核心的架构价值，是把 ask-mode 权限 dialog 从“单一本地确认框”扩展成“多审批源共享的竞态中枢”

如果把整个流程压缩，会发现这里真正协调的是六类可能终局：

1. 本地用户 allow
2. 本地用户 reject
3. 本地用户 abort
4. bridge 远端审批响应
5. channel relay 通道审批响应
6. hook 决策
7. classifier auto-allow
8. recheck 后 config 已 allow

它们都在竞争同一个：

```text
final PermissionDecision for this tool use
```

而 `claim()` / `resolveOnce()` / queue cleanup / bridge/channel cancel 正是在保证：

```text
many racers,
one winner,
clean teardown
```

这才是整份文件的真正主线。

---

### 第十八部分：整份文件把前面 PermissionContext 提供的能力几乎全部串起来了，是 ask-path 中最“总装厂”式的一个 handler

它实际用到了 `ctx` 的这些能力：

- `pushToQueue`
- `updateQueueItem`
- `removeFromQueue`
- `persistPermissions`
- `logDecision`
- `logCancelled`
- `handleUserAllow`
- `cancelAndAbort`
- `runHooks`
- `buildAllow`

这说明 interactive handler 就像 PermissionContext 的最大消费者。

也就是说它不是一个轻量 handler，
而是 ask-path 本地审批半边的：

```text
integration hub
```

把：

- queue UI
- local callbacks
- bridge
- channels
- hooks
- classifier
- dynamic recheck

全部总装在一起。

---

### 元

问题：**这一站真正想解决的架构问题是什么？**

回答：这一站的元问题，是本地交互审批为什么不是“弹个框”这么简单，而是一场多审批源竞态。它真正编排的是用户点击、bridge、channel、hooks、classifier 这些并发决定谁先结算。

### 反

问题：**如果把这一站的设计反过来，会发生什么？**

回答：如果把 interactive path 理解成单纯 UI 组件，系统就会忽略远端审批和自动审批与本地交互的竞争关系。结果往往是重复结算、队列残留或状态回收不干净。

### 空

问题：**跳出当前文件名，这一站背后更大的问题是什么？**

回答：更大的问题，是当多个决策源同时存在时，谁来做终局裁决。这个文件说明，权限对话框本质上是一个竞态协调器，而不只是一个界面。

### 读完这一站后，你应该抓住的 10 个事实

1. `interactiveHandler.ts` 不是单纯“弹一个确认框”，而是 ask-mode 权限路径里的本地交互审批 handler 与多审批源竞态协调器。
2. 它的第一步是把请求变成 `ToolUseConfirm` queue item，因此权限 UI 是队列驱动的，而不是直接 imperative 地弹 modal。
3. 本地用户 allow/reject/abort、bridge 远端审批、channel relay、hooks、classifier、recheck-config 都会竞争同一个最终 `PermissionDecision`，通过 `claim()` / `resolveOnce()` 只允许一个胜出。
4. `onUserInteraction()` 的 200ms grace period 用来防止惯性按键过早打断 classifier auto-approval，体现了很细的交互优化。
5. bridge 审批是全功能远端审批面，可以返回 `updatedInput` 和 `updatedPermissions`，因此在权限语义上被当作一等审批源。
6. channel relay 则是更轻量的 yes/no 审批通道，走 MCP 结构化 notification 协议，但没有 `updatedInput` 这类 richer mutation 路径。
7. `recheckPermission()` 说明 dialog 显示期间权限并不是静态的；外部 mode/rule 变化后，tool 可能在不经过用户点击的情况下直接变成 allow。
8. hooks 在 interactive path 中仍然是后台 racer，dialog 出现以后它们依然可能抢先结束整条权限流。
9. bash classifier 也是后台 racer，而且 auto-allow 后不会瞬间移除 dialog，而是进入一个可感知的 checkmark 过渡态，再延时移除。
10. 这份文件真正把 ask-mode 的本地审批路径从“单一 CLI 对话框”升级成了“本地 UI + 远端 bridge + channel relay + hooks + classifier”共同参与的统一竞态中枢。

---

### 现在把第 60-61 站串起来

```text
PermissionContext.ts
  -> 提供共享能力层：logging / abort / hooks / classifier / queue / result factories
interactiveHandler.ts
  -> 用这些共享能力把 ask-mode 本地交互审批、远端审批和后台自动化审批串成一个竞态系统
```

所以现在本地 ask-path 可以先压缩成：

```text
create PermissionContext
  -> push ToolUseConfirm queue item
  -> attach local callbacks
  -> start bridge/channel/hook/classifier racers
  -> first winner claims resolution
  -> clean up queue + remote listeners + indicators
```

也就是说：

```text
PermissionContext.ts 回答“ask-path 各 handler 共用哪些能力与标准副作用”
interactiveHandler.ts 回答“这些能力在本地交互权限链里到底怎样被组装成一个多审批源并发竞态系统”
```

---

### 下一站建议

下一站最顺的是：

```text
src/hooks/toolPermission/handlers/coordinatorHandler.ts
```

因为现在你已经看懂了：

- `interactiveHandler.ts` 是 ask-path 的本地兜底与 race coordinator
- `swarmWorkerHandler.ts` 是 worker->leader 远端审批 handler
- `PermissionContext.ts` 是共享能力层
- 但还没补 ask-path 最前面的“自动检查优先”分支：
  - coordinator 模式为什么要先 await automated checks
  - hooks / classifier 在这个分支里怎样提前消化权限请求
  - 为什么它会改变 dialog 是否立即出现

下一步最自然就是把 ask-path 的最前置自动化分支补齐：

**coordinatorHandler.ts 到底怎样在真正打断用户前先运行自动化审批检查，并把可自动解决的 ask 请求提前收束，只有在无结论时才放行进入 interactiveHandler。**
