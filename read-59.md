## 第 59 站：`src/hooks/toolPermission/handlers/swarmWorkerHandler.ts`

### 这是什么文件

`src/hooks/toolPermission/handlers/swarmWorkerHandler.ts` 是工具权限 ask-path 里的 **swarm worker 上抛 leader 审批处理器**。

上一站 `useCanUseTool.tsx` 已经看懂：

- ask 分支不会立刻本地弹框
- 会先尝试 `handleSwarmWorkerPermission(...)`
- 但还没落到这条 handler 自己到底做了什么：
  - 怎样创建 permission request
  - 怎样注册 pending callback
  - 怎样发给 leader
  - 等待期间 UI 怎么显示
  - leader 回来后又怎样接回原来的工具调用

所以这一站回答的是：

```text
当一个 swarm worker 进入 ask-mode 权限路径时，
系统怎样把本地待审批工具调用转成 leader 审批请求，
并把 leader 的决定重新接回这次工具调用主链
```

所以最准确的一句话是：

```text
swarmWorkerHandler.ts = ask-mode tool permission 的 worker->leader 上抛与恢复桥
```

---

### 先看它的整体定位

位置：`src/hooks/toolPermission/handlers/swarmWorkerHandler.ts:1`

这个文件不是：

- 底层 permission rule evaluator
- mailbox router
- worker callback registry 本体
- interactive permission UI

它的职责是：

1. 判断当前 ask-path 是否适合走 swarm worker 审批
2. 先尝试 classifier 自动通过
3. 构造 permission request
4. 注册 leader response callback
5. 发请求到 leader mailbox
6. 在等待期间更新 pending indicator
7. leader 回复后把 decision 重新 resolve 回本次 tool use

所以它本质上是一个：

```text
worker permission escalation handler
```

---

### 第一部分：最外层 `!isAgentSwarmsEnabled() || !isSwarmWorker()` 直接返回 `null`，说明它不是通用 ask handler，而是一个严格的条件分支处理器

位置：`src/hooks/toolPermission/handlers/swarmWorkerHandler.ts:40`

这里一开始就 gate：

- swarm feature 必须启用
- 当前运行体必须真的是 swarm worker

否则返回 `null`。

这说明它的契约不是：

```text
我一定处理 ask
```

而是：

```text
如果当前上下文适合走 worker->leader 审批，我来处理；
否则我不接，交回上层继续走别的 ask 路径
```

这和上一站 `useCanUseTool.tsx` 的设计完全对上：

- handler 返回 `PermissionDecision` -> 已处理
- 返回 `null` -> fall through 到后续 interactive handling

所以这里的关键语义是：

```text
conditional interception, not mandatory ownership
```

---

### 第二部分：worker 侧和主 agent 的 classifier 策略并不一样——这里是“先等 classifier 完整出结果”，而不是主 agent 那种先继续往下、后面再给 2 秒宽限窗口

位置：`src/hooks/toolPermission/handlers/swarmWorkerHandler.ts:49`

注释写得很清楚：

- 对 bash commands 先试 classifier auto-approval
- agents 会 **await classifier result**
- 不像 main agent 那样和用户交互 race

代码也是：

- `ctx.tryClassifier?.(...)`
- 有结果就直接 return classifierResult

这说明 worker 场景的策略偏向：

```text
background worker should prefer silent automated resolution before escalating to the leader
```

而主 agent 则更偏向：

```text
don't block the user too long; allow a short race before showing UI
```

这是一个非常细但很关键的角色差异：

- 主 agent 重视交互速度
- worker 重视少打扰、少上抛

---

### 第三部分：真正进入 swarm 审批时，核心结构是 `new Promise<PermissionDecision>`，说明这个 handler 不是同步发消息后立刻返回，而是把“等待 leader 回答”直接内联进当前 tool use 的 Promise 生命周期

位置：`src/hooks/toolPermission/handlers/swarmWorkerHandler.ts:67`

这里的核心不是简单：

- 发个 request
- 然后返回某个 pending 标记

而是：

- 创建一个 Promise
- 直到 leader onAllow/onReject/abort 其中之一发生才 resolve

这说明从 `useCanUseTool(...)` 往上看，
这个 handler 的语义其实是：

```text
I will eventually yield the final PermissionDecision for this tool use,
possibly after remote leader approval
```

也就是说 remote leader 审批被压平进了一次普通异步 permission decision 中。

这正是整个 swarm permission 设计最漂亮的一点：

```text
distributed approval,
local Promise-shaped result
```

---

### 第四部分：`createResolveOnce(resolve)` 很关键，它把 onAllow / onReject / abort 三条竞争路径收敛成单次结算，防止重复 resolve 同一 tool use

位置：`src/hooks/toolPermission/handlers/swarmWorkerHandler.ts:68`

这里先拿到：

- `resolveOnce`
- `claim`

后面三条路径都会先：

- `if (!claim()) return`

然后再继续：

- onAllow
- onReject
- abort signal

这说明作者非常清楚这里存在真实竞态：

- leader 可能正好回复 allow
- 同时本地 abort 也可能触发
- 或某种异常路径导致重复回调

所以这里不是靠“希望不会重复”，
而是明确引入：

```text
single-settlement guard for the permission promise
```

这和前面 callback registry 的 one-shot continuation 哲学完全一致。

---

### 第五部分：permission request 是在这里真正从当前 tool use 上下文实例化出来的，说明 `swarmWorkerHandler` 是 tool world 到 transport world 的第一座桥

位置：`src/hooks/toolPermission/handlers/swarmWorkerHandler.ts:70`

这里调用：

- `createPermissionRequest({ ... })`

填入的字段来自当前上下文：

- `toolName: ctx.tool.name`
- `toolUseId: ctx.toolUseID`
- `input: ctx.input`
- `description`
- `permissionSuggestions: suggestions`

这说明前面 `permissionSync.ts` 里定义的领域模型，
在这里第一次和真实 tool invocation 接上。

也就是说：

```text
this exact tool invocation
-> materialized as SwarmPermissionRequest
```

因此这个 handler 的一个核心职责就是：

```text
convert local permission question into a routable permission request artifact
```

---

### 第六部分：先 `registerPermissionCallback(...)` 再 `sendPermissionRequestViaMailbox(...)`，这是整条链最关键的竞态修复点之一

位置：`src/hooks/toolPermission/handlers/swarmWorkerHandler.ts:79`

注释直接写：

```text
Register callback BEFORE sending the request to avoid race condition
where leader responds before callback is registered
```

这非常重要。

因为这条链是跨 agent 的：

- worker 发 request
- leader 很快看到并批准
- response 可能立刻通过 mailbox 回来

如果顺序反了：

```text
send request
-> leader responds fast
-> inbox poller sees response
-> no callback registered yet
-> response loses its consumer
```

所以这里的正确顺序是：

```text
prepare continuation locally first
-> then emit distributed request
```

这是整个 distributed permission flow 的关键正确性点。

---

### 第七部分：callback 里存的是当前 tool use 的 continuation 逻辑，而不是简单的“收到允许就返回 true”，说明恢复时还要重新走一层本地 permission result 组装

位置：`src/hooks/toolPermission/handlers/swarmWorkerHandler.ts:81`

注册进去的 callback 包含：

- `requestId`
- `toolUseId`
- `onAllow(...)`
- `onReject(...)`

而 `onAllow(...)` 不是简单 resolve allow，
它会：

1. claim
2. clear pending UI
3. 计算 `finalInput`
4. `await ctx.handleUserAllow(...)`
5. 再 `resolveOnce(...)`

这说明 leader 批准并不是权限链的终点。

还要把 leader decision 转回本地工具执行兼容的标准结果，
包括：

- 更新后的 input
- permission updates
- feedback
- contentBlocks

所以 continuation 的真正语义是：

```text
resume local permission completion logic,
not merely mark remote approval arrived
```

---

### 第八部分：`allowedInput` 与 `ctx.input` 的合并逻辑说明 leader 审批不只是 yes/no，还可能重写工具输入，因此 worker 恢复时必须重新决定最终执行参数

位置：`src/hooks/toolPermission/handlers/swarmWorkerHandler.ts:93`

这里逻辑是：

- 如果 `allowedInput` 存在且非空，使用它
- 否则 fallback 原始 `ctx.input`

这说明 leader 审批消息不仅能表达：

- 批准
- 拒绝

还允许：

```text
approved, but with updated tool input
```

因此 swarm permission flow 不是单纯远程确认框，
而是一个支持 resolver-side input mutation 的审批协议。

这和前面 `permissionSync.ts` 的 `updatedInput` 完全对上。

---

### 第九部分：`ctx.handleUserAllow(...)` 非常有意思——即使真正批准者是 leader，worker 侧仍复用“本地用户允许”这一套后续逻辑，说明 remote approval 在恢复时被折叠进本地 allow pipeline

位置：`src/hooks/toolPermission/handlers/swarmWorkerHandler.ts:99`

这里没有单独写一个：

- `handleLeaderAllow(...)`

而是直接走：

- `ctx.handleUserAllow(...)`

这说明作者的抽象取舍是：

```text
once approval reaches the worker,
what matters locally is just that the request has been approved,
regardless of whether the approver was local user or team leader
```

也就是说 remote approval 被本地化处理了。

这大大减少了权限链下游分叉复杂度。

---

### 第十部分：`onReject(...)` 不只是结束等待，还会记录 `user_reject` 来源，说明 leader 的拒绝在本地日志语义上被当作一种“用户拒绝”类决策

位置：`src/hooks/toolPermission/handlers/swarmWorkerHandler.ts:109`

reject 分支会：

- claim
- clear pending UI
- `ctx.logDecision({ decision:'reject', source:{ type:'user_reject', hasFeedback: !!feedback } })`
- `resolveOnce(ctx.cancelAndAbort(...))`

这里很有意思：

虽然拒绝来自 leader，
但本地记录 source 时仍然归类为：

```text
user_reject
```

这说明在权限语义上，
leader 审批被视为代表用户意志的上级审批面。

对于 worker 本地而言，
它不需要再区分：

- 本地用户点拒绝
- leader 代表用户拒绝

这又是一个“remote decision local-normalization”的设计。

---

### 第十一部分：`pendingWorkerRequest` 写进 AppState，说明等待 leader 审批期间，系统不只是 Promise 挂起，还会把这个等待态显式投影到 UI/runtime 状态里

位置：`src/hooks/toolPermission/handlers/swarmWorkerHandler.ts:125`

发送请求后立刻：

- `setAppState(prev => ({ ... pendingWorkerRequest: { toolName, toolUseId, description } }))`

这说明“等待 leader 审批”不只是 invisible async state，
而是一个用户/系统需要看见的运行时状态：

```text
there is currently a worker request pending leader approval
```

这类状态通常会支撑：

- spinner / waiting indicator
- 状态栏展示
- 防重复触发
- 更清晰的中断语义

所以这里实现的是：

```text
Promise-level pending
+
AppState-level pending projection
```

两层同步存在。

---

### 第十二部分：`clearPendingRequest()` 被 onAllow / onReject / abort 共用，说明 pendingWorkerRequest 被建模成严格的一次性等待态，不允许留脏状态

位置：

- 定义 `src/hooks/toolPermission/handlers/swarmWorkerHandler.ts:61`
- onAllow 调用 `src/hooks/toolPermission/handlers/swarmWorkerHandler.ts:91`
- onReject 调用 `src/hooks/toolPermission/handlers/swarmWorkerHandler.ts:111`
- abort 调用 `src/hooks/toolPermission/handlers/swarmWorkerHandler.ts:141`

三条终局路径都会清：

- `pendingWorkerRequest: null`

这说明作者非常明确地把这个字段当成：

```text
one outstanding leader-approval wait slot
```

而不是历史记录。

一旦等待结束：

- 批准
- 拒绝
- 中止

都必须立即清掉。

这和 callback registry 的 one-shot、classifier checking 的 one-shot 完全统一。

---

### 第十三部分：abort signal listener 是这条链非常关键的“不让 Promise 永远挂住”保险丝

位置：`src/hooks/toolPermission/handlers/swarmWorkerHandler.ts:135`

这里显式监听：

- `ctx.toolUseContext.abortController.signal.addEventListener('abort', ...)`

然后：

- claim
- clear pending UI
- `ctx.logCancelled()`
- `resolveOnce(ctx.cancelAndAbort(undefined, true))`

注释写得也很直白：

```text
If the abort signal fires while waiting for the leader response,
resolve the promise with a cancel decision so it does not hang.
```

这说明作者知道 remote approval flow 的一个天然风险：

```text
request sent,
but caller no longer cares,
while leader response may never arrive or arrive too late
```

如果没有这条 abort resolve，
当前 tool use 的 Promise 就会一直悬空。

所以这是整条 remote approval path 的关键 liveness 保证。

---

### 第十四部分：发送 request 时用了 `void sendPermissionRequestViaMailbox(request)`，说明 transport 发送被故意 fire-and-forget 化，而真正的控制权留给后续 callback / abort 这两个终局

位置：`src/hooks/toolPermission/handlers/swarmWorkerHandler.ts:123`

这里没有：

- `await sendPermissionRequestViaMailbox(...)`

而是：

- `void sendPermissionRequestViaMailbox(request)`

这说明这条链里更重要的是：

- 本地 callback 已经注册好
- pending UI 已经建立
- 之后就进入“等待终局”状态

而不是同步等待“消息发送确认”再继续。

也就是说这里的 transport 发送被当成：

```text
asynchronous emission side effect
```

真正决定 Promise 何时结束的不是 send 完成，
而是：

- onAllow
- onReject
- abort

这种设计让权限等待主链更简单，
但也意味着 send failure 更多落到后续 fallback / error path 上。

---

### 第十五部分：外层 `try/catch` 把“swarm permission submission 失败”降级成 `return null`，说明 remote approval 是 ask-path 的优先分支，但不是唯一生路

位置：`src/hooks/toolPermission/handlers/swarmWorkerHandler.ts:150`

catch 里做的是：

- `logError(toError(error))`
- `return null`

注释也明确说：

```text
If swarm permission submission fails, fall back to local handling
```

这说明作者不想把远程审批通路的异常变成整个 ask-path 的 hard failure。

而是：

```text
try leader escalation first;
if that path is unavailable, let the caller fall back to local interactive handling
```

这点非常实用：

- mailbox 暂时坏了
- team state 异常
- transport helper 抛错

都不应该把整次工具调用卡死。

---

### 第十六部分：`handleSwarmWorkerPermission(...)` 真正把前几站的三条链收束到了一个点上：permission context、permission transport、permission continuation recovery

如果把依赖串起来，这个 handler 正好是三条线的交叉点：

#### 来自 `useCanUseTool.tsx`
- ask path orchestration
- `ctx`
- `description`
- suggestions / updatedInput

#### 来自 `permissionSync.ts`
- `createPermissionRequest(...)`
- `sendPermissionRequestViaMailbox(...)`
- `isSwarmWorker()`

#### 来自 `useSwarmPermissionPoller.ts`
- `registerPermissionCallback(...)`
- 后续 response 恢复

所以它可以被非常准确地描述成：

```text
tool permission ask-path
  -> instantiate swarm request
  -> register one-shot continuation
  -> emit to leader
  -> resume tool decision on response
```

这份文件就是这条链在 worker 侧的实装入口。

---

### 第十七部分：整份文件的核心价值，在于把“分布式审批”包装成对上层几乎透明的 `PermissionDecision`

上层 `useCanUseTool(...)` 调这个 handler 时，
看到的只是：

- 返回 `PermissionDecision`
- 或返回 `null`

它不需要知道：

- request 是怎么生成的
- mailbox 怎么发的
- callback 怎么注册的
- leader 什么时候回来的
- abort 怎么处理的

也就是说这个 handler 做到的是：

```text
hide distributed permission choreography behind a local async decision boundary
```

这正是分布式控制流封装得好的标志。

---

### 读完这一站后，你应该抓住的 10 个事实

1. `swarmWorkerHandler.ts` 是 ask-mode 权限路径里的 worker->leader 审批 handler，不是通用 permission handler，只在 agent swarms 启用且当前是 swarm worker 时接管。
2. worker 侧的 classifier 策略和主 agent 不同：这里会先完整 await classifier 结果，优先避免不必要地把权限请求上抛给 leader。
3. 真正进入 leader 审批时，这个 handler 会把当前 tool invocation 实例化成 `SwarmPermissionRequest`，因此它是 tool-use world 到 transport world 的第一座桥。
4. `registerPermissionCallback(...)` 被刻意放在 `sendPermissionRequestViaMailbox(...)` 之前，以修复“leader 回得太快，response 先到而 callback 还没注册”的竞态问题。
5. `createResolveOnce(...)` 让 onAllow、onReject、abort 三条竞争路径共享单次结算保护，避免同一个权限 Promise 被重复 resolve。
6. leader 批准后，worker 侧不会直接返回 true，而是继续走 `ctx.handleUserAllow(...)`，说明 remote approval 在本地被折叠进统一 allow pipeline。
7. leader 拒绝在本地被记录成 `user_reject` 类来源，体现了“leader 审批在 worker 看来就是代表用户的上级审批”这一语义归一化。
8. `pendingWorkerRequest` 被写入 AppState，说明等待 leader 审批不仅是 Promise 挂起态，还是一个显式投影到 UI/runtime 的等待状态。
9. abort signal listener 是这条 remote approval path 的关键 liveness 保障：如果等待期间请求被取消，Promise 会以 cancel decision 收束，而不会无限挂住。
10. 如果 swarm permission submission 失败，这个 handler 不会把 ask-path 整体打死，而是返回 `null` 让上层继续走本地 interactive handling，体现了 remote approval 的“优先通道但非唯一通道”定位。

---

### 现在把第 58-59 站串起来

```text
useCanUseTool.tsx
  -> ask-path 统一调度
  -> 先尝试 swarm worker permission handler
swarmWorkerHandler.ts
  -> 把 ask-mode tool use 转成 leader permission request
  -> 注册 continuation
  -> 等 leader 回复后恢复本地 PermissionDecision
```

所以现在 worker 侧 distributed permission path 可以先压缩成：

```text
ask-mode tool use
  -> useCanUseTool routes into swarmWorkerHandler
  -> create permission request
  -> register one-shot callback
  -> send request to leader mailbox
  -> pending indicator shown
  -> leader response arrives
  -> callback resumes and returns final PermissionDecision
```

也就是说：

```text
useCanUseTool.tsx 回答“ask-mode 权限在全局上该走哪条审批路径”
swarmWorkerHandler.ts 回答“如果选择走 worker->leader 审批，这条分布式权限链在本地到底怎样启动、等待、恢复和收尾”
```

---

### 下一站建议

下一站最顺的是：

```text
src/hooks/toolPermission/PermissionContext.ts
```

因为现在你已经看懂了：

- `useCanUseTool.tsx` 怎样做总分流
- `swarmWorkerHandler.ts` 怎样用 `ctx` 去处理上抛与恢复
- 但还没真正拆开 `ctx` 本身：
  - `ctx.buildAllow(...)`
  - `ctx.handleUserAllow(...)`
  - `ctx.cancelAndAbort(...)`
  - `ctx.logDecision(...)`
  - `createResolveOnce(...)`
  - queue ops / messageId / abort 辅助到底怎样组织

下一步最自然就是把这个共享权限上下文层补齐：

**PermissionContext 到底怎样把一次工具审批流需要的状态、日志、队列操作、一次性 resolve 保护和 allow/reject 结果构造统一封装起来，供 coordinator/swarm/interactive 三条 handler 共用。**
