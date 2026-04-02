## 第 56 站：`src/hooks/useSwarmPermissionPoller.ts`

### 这是什么文件

`src/hooks/useSwarmPermissionPoller.ts` 是 swarm 权限链路里的 **worker 侧 permission / sandbox permission 等待态回调注册表与响应恢复器**。

上一站 `useInboxPoller.ts` 已经看懂：

- leader / process-based teammate 会轮询 mailbox
- inbox poller 会把 `permission_response` / `sandbox_permission_response` 路由出来
- 收到 response 后会去查 callback registry
- 但“callback registry 自己是谁维护的、请求挂起后怎样恢复”还没补上

这一站正好补的是这半边：

```text
worker 发出权限请求之后，
本地怎样把“等待 leader 回答”的 continuation 挂起来，
leader 响应回来后又怎样精确唤醒对应请求，把工具执行继续跑下去
```

所以最准确的一句话是：

```text
useSwarmPermissionPoller.ts = swarm worker 权限请求的 pending-callback registry + response continuation bridge
```

---

### 先看它的整体定位

位置：`src/hooks/useSwarmPermissionPoller.ts:1`

这个文件不是：

- permission policy 判定器
- mailbox 消息定义层
- leader 端审批 UI
- 具体工具执行器

它的职责是：

1. 在 worker 侧注册等待中的 permission / sandbox permission continuation
2. 轮询 leader 返回的 permission response
3. 把 response 解析后分发给正确 callback
4. 在恢复完成后清掉 pending 状态与响应文件

所以它本质上是一个：

```text
worker-side async permission continuation manager
```

更具体一点：

```text
permission request pending state
  -> callback registry
  -> poll for response
  -> resume suspended execution
```

---

### 第一部分：文件头注释直接说明它不是权限系统本体，而是 worker 侧“等待 leader 批复”的恢复层

位置：`src/hooks/useSwarmPermissionPoller.ts:1`

头部注释已经点明：

- 这个 hook 只在 swarm worker 场景使用
- 它轮询 team leader 的 permission response
- 收到后调用 `onAllow/onReject`
- 用来继续原来被挂起的执行

这说明它要解决的核心问题不是：

```text
这次工具调用该不该允许？
```

而是：

```text
这个权限请求先前已经发出去了，
现在 leader 回复到了，
怎么把原来暂停的那段执行接回去？
```

所以这个文件的关键词其实是：

```text
continuation / resumption
```

而不是 authorization policy。

---

### 第二部分：`POLL_INTERVAL_MS = 500` 说明 worker 侧权限恢复比普通 inbox 消息更高频，意图是尽量缩短工具调用卡住等待审批的停顿感

位置：`src/hooks/useSwarmPermissionPoller.ts:28`

这里轮询间隔是 500ms。

上一站 `useInboxPoller.ts` 的普通 mailbox polling 是 1000ms，
这里更快一倍。

这说明作者对两类控制面延迟有明显区分：

#### 普通 teammate mailbox
- 1 秒粒度足够
- 偏协作消息 / 控制路由

#### 权限等待恢复
- 延迟更敏感
- 工具调用正卡在中间等 leader 回复

所以这里的设计语义是：

```text
permission continuation resumption deserves lower latency than general mailbox routing
```

也就是尽量减少 worker 在 ask 模式下“悬停等待”的体感时长。

---

### 第三部分：`parsePermissionUpdates()` 的重点不是做完整协议解析，而是把来自外部通道的 `permissionUpdates` 逐条做 schema 过滤，避免脏数据直接流入 continuation

位置：`src/hooks/useSwarmPermissionPoller.ts:35`

这里先判断：

- 不是数组直接返回 `[]`

然后对每个 entry：

- 用 `permissionUpdateSchema()` 做 `safeParse`
- 成功才保留
- 失败只记 debug warn，不抛错

这说明作者对 `permissionUpdates` 的信任模型是：

```text
response payload comes from an external-ish transport boundary,
so validate each update before handing it to the suspended execution path
```

也就是说 mailbox / disk polling 回来的 payload，
虽然来自系统内部 agent，
但仍然被视为：

- 可能来自旧版本 teammate
- 可能来自 buggy process
- 可能字段不全或格式漂移

因此这里的目标不是“只要一条坏了就整包失败”，
而是：

```text
best-effort keep valid permission updates, drop malformed ones
```

这是很典型的恢复路径健壮性设计。

---

### 第四部分：`PermissionResponseCallback` 把恢复点建模成 `requestId + toolUseId + onAllow/onReject`，说明真正等待中的不是“消息”，而是一次工具调用 continuation

位置：`src/hooks/useSwarmPermissionPoller.ts:58`

这个类型最关键的字段是：

- `requestId`
- `toolUseId`
- `onAllow(updatedInput, permissionUpdates, feedback?)`
- `onReject(feedback?)`

这里很值得注意：

#### 为什么有 `requestId`
用于把外部响应和本地等待项一一匹配

#### 为什么还有 `toolUseId`
说明这个 callback 不是抽象审批而已，
而是挂在某次具体 tool use 上的 continuation

#### 为什么是 `onAllow / onReject`
说明 leader 的返回不会直接“替你执行工具”，
只是告诉 worker：

- 继续执行
- 或中断并拒绝

所以这里真正挂起的是：

```text
a suspended tool invocation awaiting leader decision
```

而不是一个通用通知订阅。

---

### 第五部分：`pendingCallbacks` 做成 module-level `Map`，说明等待态必须跨 render 持久存在；它不是 React state，而是运行时 registry

位置：`src/hooks/useSwarmPermissionPoller.ts:73`

这里定义：

- `type PendingCallbackRegistry = Map<string, PermissionResponseCallback>`
- `const pendingCallbacks = new Map()`

注释还明确说：

```text
Module-level registry that persists across renders
```

这非常关键。

因为这个等待态的本质不是 UI 展示态，
而是运行时挂起 continuation：

- worker 发出权限请求时注册 callback
- 可能几百毫秒后 leader 才回复
- 期间 React 可以反复 render
- callback 不能跟着组件重建而丢失

所以作者没有把它塞进：

- `useState`
- `AppState`
- 组件局部闭包

而是直接放成模块级 registry。

这说明设计重点是：

```text
pending permission continuations are runtime process state, not UI state
```

---

### 第六部分：`register/unregister/hasPermissionCallback` 三个 API 说明这个文件既服务 polling hook，也服务外部请求发起方；它是一个显式 continuation registry 接口面

位置：

- register `src/hooks/useSwarmPermissionPoller.ts:82`
- unregister `src/hooks/useSwarmPermissionPoller.ts:94`
- has `src/hooks/useSwarmPermissionPoller.ts:104`

这几个函数组合起来说明，
这个模块不是单纯内部 hook 实现，
而是对外暴露了完整的挂起/查询/取消接口：

#### `registerPermissionCallback(...)`
请求发起方在发送 permission request 后注册 continuation

#### `unregisterPermissionCallback(...)`
本地提前结束、超时、cleanup 时主动移除

#### `hasPermissionCallback(...)`
外部 router 判断“这个 response 还有没有对应等待者”

这和上一站 `useInboxPoller.ts` 完全对上：

- inbox poller 先 `hasPermissionCallback(...)`
- 再 `processMailboxPermissionResponse(...)`

所以这个文件实际提供的是：

```text
worker-side permission continuation registry API
```

而 hook 本身只是这个 registry 的一个消费者。

---

### 第七部分：`clearAllPendingCallbacks()` 同时清 permission 与 sandbox callback，说明 `/clear` 清的不只是 UI transcript，也包括挂起的控制面 continuation

位置：`src/hooks/useSwarmPermissionPoller.ts:108`

注释写得很明确：

- `/clear` 时的 `clearSessionCaches()` 会调用这里
- tests 也会用它做隔离

而这个函数会：

- `pendingCallbacks.clear()`
- `pendingSandboxCallbacks.clear()`

这说明系统对“清 session” 的理解不仅是：

- 清消息
- 清可见上下文

还包括：

```text
reset hidden pending control continuations so stale approvals can't resume abandoned work later
```

这点很重要。

否则会出现危险情况：

- 用户已经 `/clear`
- 旧工具调用 continuation 其实还挂着
- leader 的旧 response 晚到
- 竟然把已放弃的执行重新唤醒

因此这里是一个关键的 session hygiene 边界。

---

### 第八部分：`processMailboxPermissionResponse()` 说明 mailbox router 和 continuation registry 的分工非常清楚——router 负责识别消息，registry 负责恢复挂起执行

位置：`src/hooks/useSwarmPermissionPoller.ts:124`

这个函数的流程是：

1. 用 `requestId` 查 `pendingCallbacks`
2. 没找到就返回 `false`
3. 找到后先从 registry 删除
4. `approved` -> 校验 `permissionUpdates` + 取 `updatedInput` + 调 `onAllow`
5. `rejected` -> 调 `onReject`

这正好把职责切开：

#### `useInboxPoller`
- 发现 mailbox 里有 `permission_response`
- 提取结构化字段
- 调到这里

#### `useSwarmPermissionPoller`
- 根据 requestId 找 continuation
- 真正恢复被挂起的工具调用

所以这里本质上是：

```text
mailbox transport response
  -> registry lookup
  -> suspended tool continuation resumes
```

而不是 inbox poller 自己直接掌控恢复逻辑。

---

### 第九部分：callback 在调用前先从 registry 删除，说明恢复语义是 one-shot continuation；作者优先避免重复恢复，而不是保留“失败可重试 callback”

位置：

- permission response `src/hooks/useSwarmPermissionPoller.ts:144`
- generic response `src/hooks/useSwarmPermissionPoller.ts:245`
- sandbox response `src/hooks/useSwarmPermissionPoller.ts:219`

这里三条路径都遵守同一个模式：

```text
lookup callback
-> delete callback from registry
-> invoke callback
```

这说明 continuation 被建模成：

```text
single-consumption resumption token
```

而不是：

- 可重复触发监听器
- 失败可自动重放的 handler
- 长生命周期订阅

这样做的最大价值是避免：

- 同一个 response 被重复消费
- inbox / disk poll 重读后再次唤醒同一 continuation
- callback 内部再触发时形成双重执行

所以它体现的是：

```text
approval callbacks are one-shot by design
```

---

### 第十部分：sandbox permission 回调注册表单独分离，说明 sandbox network permission 与普通 tool permission 在 worker 侧也不是一套 continuation 语义

位置：`src/hooks/useSwarmPermissionPoller.ts:158`

这里单独定义了：

- `SandboxPermissionResponseCallback`
- `pendingSandboxCallbacks`
- `registerSandboxPermissionCallback`
- `hasSandboxPermissionCallback`
- `processSandboxPermissionResponse`

而且 sandbox callback 结构明显更简单：

- `requestId`
- `host`
- `resolve(allow: boolean)`

这说明 sandbox permission 的等待模型是：

```text
promise-style yes/no network permission gate
```

而不是 tool permission 那种：

- 可能带 `updatedInput`
- 可能附带 `permissionUpdates`
- 可能有更复杂的工具恢复语义

这和上一站 leader 侧 UI 分流也完全一致：

- tool permission -> `ToolUseConfirm`
- sandbox permission -> `workerSandboxPermissions.queue`

因此这里再次体现：

```text
permission system is split by permission kind all the way through the continuation layer
```

---

### 第十一部分：`processSandboxPermissionResponse()` 只 resolve allow 布尔值，说明 sandbox permission continuation 被刻意压扁成最小恢复面，不传播额外控制上下文

位置：`src/hooks/useSwarmPermissionPoller.ts:201`

这里的恢复逻辑非常简洁：

- 根据 `requestId` 找 callback
- 删除 pending
- `callback.resolve(params.allow)`

注意这里不会做：

- `updatedInput`
- `permissionUpdates`
- `feedback`
- host 重写或更复杂补丁

这说明 sandbox permission 这条链路被作者刻意建模成：

```text
can this host be reached or not?
```

而不是更一般化的审批系统。

也就是说这类请求的 continuation 接口面被压得很窄，
从而避免把 network permission 复杂化成另一套 tool-use 协商协议。

---

### 第十二部分：`processResponse()` 和 `processMailboxPermissionResponse()` 同时存在，说明 worker 侧故意兼容两条响应来源：磁盘/response polling 路径与 mailbox 路径

位置：`src/hooks/useSwarmPermissionPoller.ts:228`

这里有两个非常像的函数：

#### `processMailboxPermissionResponse(...)`
输入是 mailbox router 已经拆好的参数对象

#### `processResponse(response: PermissionResponse)`
输入是 `permissionSync.ts` 提供的 `PermissionResponse`

它们最终都做同一件事：

- 查 registry
- 删除 pending
- approved 调 `onAllow`
- rejected 调 `onReject`

这说明作者有意识地支持两种 transport：

```text
mailbox-delivered permission response
vs
worker-response polling file/disk path
```

也就是说 continuation 层被设计成：

```text
transport-agnostic resumption core
```

输入来源可以不同，
但恢复挂起工具调用的逻辑尽量收敛一致。

---

### 第十三部分：`useSwarmPermissionPoller()` 只在 `isSwarmWorker()` 时启用，说明这是严格的 worker-side 机制，leader 不会参与这条等待恢复 loop

位置：`src/hooks/useSwarmPermissionPoller.ts:268`

hook 里第一层 gating 非常清楚：

- poll 时先 `if (!isSwarmWorker()) return`
- `shouldPoll = isSwarmWorker()`
- `useEffect` 初始 poll 也再判断一次

这说明这份 hook 完全属于：

```text
worker-side waiting-for-leader-decision runtime
```

leader 不需要它，
因为 leader 在这条链路里的职责是：

- 展示审批 UI
- 产出 response
- 写回 transport

真正要“悬停并等待恢复”的只有 worker。

---

### 第十四部分：`isProcessingRef` 的作用是避免并发 poll 重入，说明这里最怕的不是漏轮询，而是一次 response 被两个并发 poll 周期同时消费

位置：`src/hooks/useSwarmPermissionPoller.ts:269`

这里不是用 state，
而是 `useRef(false)` 做 reentrancy guard。

然后：

- 进入 poll 前检查 `isProcessingRef.current`
- 开始处理时置 `true`
- finally 里再恢复 `false`

这说明作者认定这里的关键风险是：

```text
overlapping polling cycles causing duplicate processing / duplicate cleanup
```

尤其考虑到：

- 轮询频率 500ms
- 单次 poll 内部有 async 文件读取
- 还会遍历多个 pending request
- 最后还会删 response 文件

如果没有重入保护，
很容易出现两个 poll 周期同时看到同一 response。

所以这里体现的是典型的：

```text
serialize polling side effects, not polling schedule itself
```

---

### 第十五部分：没有 pending callback 就直接不 poll，说明这个 hook 的轮询不是“长期看 mailbox”，而是“只在存在挂起 continuation 时才激活”

位置：`src/hooks/useSwarmPermissionPoller.ts:282`

这里很关键的一句是：

- `if (pendingCallbacks.size === 0) return`

这说明它和 `useInboxPoller` 的设计哲学不同。

#### `useInboxPoller`
- 长期开着
- 因为要接收各种 mailbox 协议消息

#### `useSwarmPermissionPoller`
- 只有本地确实有未决 permission continuation 时才有意义

所以这条 hook 的本质更像：

```text
demand-driven polling for suspended permission requests
```

而不是常驻消息总线消费者。

这能减少无意义磁盘检查，
也更符合它“只负责恢复挂起执行”的定位。

---

### 第十六部分：轮询循环按 `requestId` 遍历 `pendingCallbacks`，说明这里的响应匹配模型是 request-centric，而不是“扫一遍 inbox 后再反向归并”

位置：`src/hooks/useSwarmPermissionPoller.ts:297`

这里做的是：

- `for (const [requestId, _callback] of pendingCallbacks)`
- 对每个 requestId 单独 `pollForResponse(requestId, agentName, teamName)`

这说明作者选择的恢复策略不是：

```text
scan all unread responses
-> classify them
-> map onto local requests
```

而是：

```text
for each currently suspended request,
ask whether its response exists yet
```

这种 request-centric polling 的优点是：

- 本地只关心自己确实在等的那些 request
- 匹配逻辑简单
- continuation registry 天然就是主索引

也就是说这里的核心索引不是 inbox 消息列表，
而是：

```text
pending request registry keyed by requestId
```

---

### 第十七部分：`processed -> removeWorkerResponse(...)` 说明响应文件清理发生在“成功恢复 continuation 之后”，和 mailbox mark-read 一样遵守先投递后确认的可靠性顺序

位置：`src/hooks/useSwarmPermissionPoller.ts:301`

这里流程是：

1. `pollForResponse(...)` 找到 response
2. `processResponse(response)`
3. 若 `processed` 为真
4. `removeWorkerResponse(...)`

这个顺序非常像上一站 inbox poller 的：

- 成功提交或可靠入队后才 `markRead`

这里对应成：

- 成功找到 continuation 并恢复后才删 response 文件

所以同样体现的是：

```text
ack/delete only after successful handoff to the consumer
```

其背后取舍仍然是：

- 可以接受某些重复读取风险
- 不能接受响应被提前删掉导致挂起请求永远无法恢复

这是很一致的可靠性哲学。

---

### 第十八部分：`agentName` / `teamName` 缺失就直接 return，说明 continuation 恢复不仅依赖 requestId，还依赖稳定的 worker 身份上下文

位置：`src/hooks/useSwarmPermissionPoller.ts:289`

poll 里先取：

- `getAgentName()`
- `getTeamName()`

如果任一缺失就返回。

这说明恢复 continuation 不是全局查表，
而是需要在：

```text
specific worker identity + specific team namespace
```

下去找 response。

也就是说 response transport 不是一个全局共享总线，
而是和前面 mailbox / swarm team 设计保持一致：

- team-scoped
- agent-scoped

这再次证明整个 swarm 权限恢复系统仍然建立在稳定的 teammate/team 身份模型之上。

---

### 第十九部分：`useInterval + mount initial poll` 的组合，说明这个 hook 也遵循“先立即尝试，再定期轮询”的控制面常见模式

位置：

- interval `src/hooks/useSwarmPermissionPoller.ts:320`
- initial poll `src/hooks/useSwarmPermissionPoller.ts:324`

这里和前面多个 poller 一样：

- 先 `useInterval(..., 500ms)`
- 再在 `useEffect` mount 时主动 `poll()` 一次

它解决的是两个不同问题：

#### initial poll
刚挂载时立刻看一次，避免白等半秒

#### interval poll
后续持续等待 leader 回复

所以这里延续的是一套典型 swarm runtime hygiene：

```text
immediate first check + stable periodic follow-up polling
```

---

### 第二十部分：整份文件真正把“leader 审批后 worker 如何继续执行”这半条链从隐式概念变成了显式运行时机制

如果把整份文件压缩，会发现它其实在做四层事情：

#### 1. continuation registry
- permission callbacks
- sandbox permission callbacks

#### 2. external payload validation
- `parsePermissionUpdates()`

#### 3. response-to-callback dispatch
- `processMailboxPermissionResponse()`
- `processSandboxPermissionResponse()`
- `processResponse()`

#### 4. worker polling loop
- `useSwarmPermissionPoller()`

所以最准确的压缩表达是：

```text
useSwarmPermissionPoller.ts = swarm worker 侧的权限等待恢复中枢：把 leader 返回的 permission / sandbox permission 响应精确映射回挂起中的工具调用 continuation，并在成功恢复后完成一次性消费与清理
```

---

### 元

问题：**这一站真正想解决的架构问题是什么？**

回答：这一站关心的是，worker 发起权限请求后，挂起的 continuation 到底在哪里等 leader 的回复。它把“等待审批”从抽象 Promise 变成了本地可注册、可恢复的 callback registry。

### 反

问题：**如果把这一站的设计反过来，会发生什么？**

回答：如果不单独维护这层挂起恢复机制，权限请求发出去以后就只剩消息，没有执行上下文。那样 leader 回答再准确，也接不回原来的工具调用。

### 空

问题：**跳出当前文件名，这一站背后更大的问题是什么？**

回答：更大的问题，是分布式审批怎样把“异步决定”重新接回“正在执行的本地调用栈”。这正是 continuation bridge 这类设计长期要回答的事。

### 读完这一站后，你应该抓住的 10 个事实

1. `useSwarmPermissionPoller.ts` 不是权限判定器，而是 worker 侧等待 leader 审批结果时的 continuation registry 与恢复层。
2. 普通 tool permission 和 sandbox permission 在这里仍然分成两套 pending callback registry，说明两类权限从 UI 到恢复语义都被分流处理。
3. `pendingCallbacks` / `pendingSandboxCallbacks` 都是 module-level `Map`，因为挂起 continuation 必须跨 render 持久存在，不能依赖 React state。
4. `registerPermissionCallback`、`unregisterPermissionCallback`、`hasPermissionCallback` 让这个模块成为显式的运行时 registry API，而不仅是一个内部 hook。
5. `parsePermissionUpdates()` 会对外部 transport 带回的 `permissionUpdates` 做逐条 schema 过滤，坏条目只丢弃并记录 warn，不会把整个恢复流程打断。
6. 无论是 mailbox 路径还是 `permissionSync.ts` 路径，真正恢复挂起工具调用时都遵循同一个模式：先按 `requestId` 找 callback，再从 registry 删除，最后调用 `onAllow/onReject`。
7. callback 在调用前先删除，说明权限响应 continuation 被明确建模成 one-shot resumption token，以避免重复恢复同一工具调用。
8. `useSwarmPermissionPoller()` 只在 `isSwarmWorker()` 且存在 pending callback 时才有意义，说明它是 demand-driven 的 worker-side polling，而不是常驻总线消费者。
9. poll 成功处理响应后才 `removeWorkerResponse(...)`，与 `useInboxPoller.ts` 的 mark-read 语义一样，都遵循“先成功交付，再确认删除”的可靠性顺序。
10. 这份文件真正补上的是 swarm 权限链路里最关键的半环：worker 发出 ask 请求后，本地如何把暂停中的工具调用挂起，并在 leader 回应后精确恢复执行。

---

### 现在把第 55-56 站串起来

```text
useInboxPoller.ts
  -> 轮询 mailbox unread messages
  -> 把 permission / sandbox permission response 路由到 callback registry
useSwarmPermissionPoller.ts
  -> 维护 worker 侧 pending callbacks
  -> 收到响应后恢复被挂起的工具调用 continuation
```

所以现在 swarm 权限恢复链可以先压缩成：

```text
permission request sent by worker
  -> pending callback registered

leader response transport
  -> mailbox / permissionSync

response dispatch
  -> useInboxPoller / useSwarmPermissionPoller

suspended execution resumes
  -> onAllow / onReject / sandbox resolve
```

也就是说：

```text
useInboxPoller.ts 回答“leader/worker 运行时怎样把 mailbox 里的权限响应识别并分流”
useSwarmPermissionPoller.ts 回答“这些响应落到 worker 本地后，系统到底怎样找到原来挂起的那次工具调用并继续执行”
```

---

### 下一站建议

下一站最顺的是：

```text
src/utils/swarm/permissionSync.ts
```

因为现在你已经看懂了：

- worker 本地怎样注册 pending permission continuation
- poller 怎样按 requestId 去等 response
- `processResponse(...)` / `pollForResponse(...)` / `removeWorkerResponse(...)` 这套恢复逻辑明显依赖 `permissionSync.ts`
- 但还没看“响应到底落在哪、怎样被 worker 轮询到、磁盘/文件命名与清理语义是什么”

下一步最自然就是把 transport 这一半补齐：

**swarm worker 的 permission response 到底怎样通过 `permissionSync.ts` 落盘、被 requestId 精确检索、被消费后删除，以及它与 mailbox 路径之间的边界到底如何划分。**
