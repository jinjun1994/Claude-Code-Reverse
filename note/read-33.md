## 第 33 站：`src/hooks/useSwarmPermissionPoller.ts`

### 这是什么文件

`src/hooks/useSwarmPermissionPoller.ts` 是 swarm 权限系统里的 **worker 侧响应轮询器 + 回调分发器**。

上一站 `swarmWorkerHandler.ts` 已经看到：

```text
worker 发起 ask
  -> createPermissionRequest(...)
  -> registerPermissionCallback(...)
  -> sendPermissionRequestViaMailbox(...)
  -> 等 leader 回应
```

这一站补的正是后半段：

```text
leader 的回应回来之后，
worker 这边是谁在收、怎么找回原来的 promise、怎样继续执行
```

所以最准确的一句话是：

```text
useSwarmPermissionPoller.ts = swarm worker 侧权限回包的轮询收件与回调分发中心
```

---

### 先看它解决的核心问题

位置：`src/hooks/useSwarmPermissionPoller.ts:1`

文件头注释写得很直白：

- worker 模式下轮询 leader 的 permission responses
- 收到后调用 `onAllow / onReject`
- 与 `useCanUseTool.ts` 的 worker-side integration 配合

这说明它不是权限判定器，也不是请求发送器，而是：

```text
请求发出去之后，
把“异步 leader 回应”重新接回本地原始权限流程的恢复器
```

所以它在整个 swarm 权限链中的位置可以概括成：

```text
send side: swarmWorkerHandler.ts
receive side: useSwarmPermissionPoller.ts
```

---

### 第一部分：模块级 registry 才是它真正的核心数据结构

位置：`src/hooks/useSwarmPermissionPoller.ts:69`

这里定义了：

- `PermissionResponseCallback`
- `PendingCallbackRegistry = Map<string, PermissionResponseCallback>`
- `const pendingCallbacks = new Map()`

这一层非常关键，因为它说明 swarm 权限回包并不是直接绑在某个 React 组件实例上，而是先进入一个：

```text
requestId -> callback
```

的模块级路由表。

也就是说，真正的运行模型是：

```text
worker 发请求时，
先把“未来 leader 回来后要执行什么”登记到全局 registry；
之后轮询器只要拿到 requestId，就能找到对应 continuation
```

所以这份 registry 本质上就是：

```text
swarm 权限异步协议里的 continuation table
```

---

### 第二部分：`registerPermissionCallback(...)` 把 leader 响应接回原始 ask promise

位置：`src/hooks/useSwarmPermissionPoller.ts:78`

这里暴露出：

- `registerPermissionCallback(...)`
- `unregisterPermissionCallback(...)`
- `hasPermissionCallback(...)`

这说明上一站 `swarmWorkerHandler.ts` 里调用的 `registerPermissionCallback(...)`，并不是简单“留个引用”，而是在做：

```text
为这次 requestId 预注册恢复逻辑
```

一旦 leader 稍后发回：

- approved
- rejected

轮询器就能按 `requestId` 找到那次 ask 对应的：

- `onAllow(...)`
- `onReject(...)`

所以这一步是把分布式异步消息重新映射回本地 promise continuation 的关键桥梁。

---

### 第三部分：它先做 `permissionUpdates` 校验，说明 leader 回包也被视为外部输入

位置：`src/hooks/useSwarmPermissionPoller.ts:30`

`parsePermissionUpdates(raw)` 做的事情很值得注意：

- 只接受数组
- 对每一项跑 `permissionUpdateSchema().safeParse(entry)`
- 合法的留下
- 不合法的丢弃并记 warning debug log

这说明即使 leader / teammate 理论上是“自己人”，这里仍然把 mailbox / disk polling 回来的内容当成：

```text
不可信或至少需要验证的外部输入
```

所以它不是盲信 teammate 进程，而是先做 schema 过滤。

这一点很成熟，因为注释明确说了要防：

- buggy teammate processes
- old teammate processes

所以你应该把这一段记成：

```text
swarm permission response 不是直接注入本地状态，
而是先做结构校验后再进入 callback
```

---

### 第四部分：`processMailboxPermissionResponse(...)` 说明 inbox poller 也能直接复用这套路由表

位置：`src/hooks/useSwarmPermissionPoller.ts:118`

这里提供了一个显式入口：

- `processMailboxPermissionResponse(...)`

它的行为是：

1. 用 `requestId` 找 callback
2. 没找到就返回 `false`
3. 找到后先 `pendingCallbacks.delete(requestId)`
4. 若 approved：校验 `permissionUpdates`，再调 `callback.onAllow(...)`
5. 若 rejected：调 `callback.onReject(...)`

最重要的是文件注释已经点出：

```text
This is called by the inbox poller when it detects a permission_response message.
```

这说明整个系统并不只有“本 hook 自己轮询磁盘”这一种接收路径，
还支持：

```text
其他 inbox / mailbox 消费者在收到 permission_response 后，
直接把消息喂给这个模块统一分发
```

所以这个文件虽然名字叫 poller，但其实还承担了：

```text
permission response dispatcher
```

的角色。

---

### 第五部分：回调在触发前先从 registry 删除，是为了保证一次性消费

位置：

- `processMailboxPermissionResponse(...)` `src/hooks/useSwarmPermissionPoller.ts:144`
- `processResponse(...)` `src/hooks/useSwarmPermissionPoller.ts:245`
- `processSandboxPermissionResponse(...)` `src/hooks/useSwarmPermissionPoller.ts:219`

三个地方都遵循同一模式：

```text
先 delete registry entry
再调用 callback / resolve
```

这说明它明确要保证：

```text
每个 permission response 只能消费一次
```

这样可以防止：

- 重复轮询到同一个回包
- mailbox 和 polling 双通道都碰到同一事件
- callback 内部再次触发其他副作用时造成重复 resolve

所以这一层实际上和 `createResolveOnce(...)` 是同一思路在更上游的体现：

```text
先撤销可再次命中的入口，
再真正交付结果
```

---

### 第六部分：它不仅管 tool permission，还顺手管理 sandbox permission 的第二套 registry

位置：`src/hooks/useSwarmPermissionPoller.ts:158`

文件中间还有一整块：

- `SandboxPermissionResponseCallback`
- `pendingSandboxCallbacks`
- `registerSandboxPermissionCallback(...)`
- `hasSandboxPermissionCallback(...)`
- `processSandboxPermissionResponse(...)`

这说明这个模块的职责已经不只是：

```text
tool use ask 的 leader 响应
```

而是更广一点：

```text
swarm worker 上所有“远端审批请求 -> 回包 -> 恢复本地等待”的通用回调注册中心
```

tool permission 和 sandbox permission 只是它目前承载的两种协议。

所以从架构角度看，这个文件其实在提供：

```text
mailbox approval callback registry
```

而不是仅仅一个单用途 hook。

---

### 第七部分：`processResponse(...)` 是旧/轮询路径上的标准收口

位置：`src/hooks/useSwarmPermissionPoller.ts:228`

这个内部函数消费的是：

- `PermissionResponse`

流程和 `processMailboxPermissionResponse(...)` 几乎平行：

- 查 callback
- 记 debug log
- 删除 registry entry
- approved -> `parsePermissionUpdates(...)` -> `callback.onAllow(...)`
- rejected -> `callback.onReject(...)`

所以你可以看出这个文件故意把“响应来源”和“响应分发”拆开了：

- 来源可以是 mailbox message
- 来源也可以是 `pollForResponse(...)` 读到的 worker response
- 但最终都汇聚到同一套 callback 分发语义

这是一种典型的：

```text
transport-agnostic callback dispatch
```

设计。

---

### 第八部分：真正的 React hook 很薄，它只是驱动轮询时钟

位置：`src/hooks/useSwarmPermissionPoller.ts:268`

真正导出的 hook `useSwarmPermissionPoller()` 本身逻辑并不重，核心是：

- `useRef(false)` 防并发 poll
- `useCallback(async () => { ... })`
- `useInterval(() => void poll(), shouldPoll ? 500 : null)`
- `useEffect(() => { if (isSwarmWorker()) void poll() }, [poll])`

这说明 React hook 这一层只是：

```text
提供定时驱动与生命周期接入
```

而真正的重要状态和逻辑都在模块级 registry + response processing 函数里。

所以不要把这个文件误解成“一个普通 polling hook”。

更准确的说法是：

```text
React hook 只是 transport driver；
真正的核心是 registry 和 response dispatcher
```

---

### 第九部分：轮询前的三个短路条件体现了它的运行边界

位置：`src/hooks/useSwarmPermissionPoller.ts:271`

每次 poll 之前它都会先检查：

1. `!isSwarmWorker()` -> 不轮询
2. `isProcessingRef.current` -> 防止并发轮询
3. `pendingCallbacks.size === 0` -> 没有挂起请求就不做 I/O

这三个条件说明它的运行边界很明确：

```text
只在 worker 身份下运行；
一次只跑一个 poll；
只有真的有待回应请求时才触发外部读取
```

这是一个很节制的轮询器实现，不是无脑定时扫盘。

---

### 第十部分：按 requestId 逐个 `pollForResponse(...)`，说明 worker inbox 是面向请求粒度组织的

位置：`src/hooks/useSwarmPermissionPoller.ts:297`

这里会遍历：

- `for (const [requestId] of pendingCallbacks)`

然后对每个 requestId 调：

- `pollForResponse(requestId, agentName, teamName)`

如果拿到 response：

- `processResponse(response)`
- `removeWorkerResponse(requestId, agentName, teamName)`

这说明底层权限回包存储模型不是“拉一整个通用事件流再自己筛”，而更像：

```text
给定 requestId，检查这个 worker 是否已经收到了对应 leader 响应
```

因此它和 registry 之间是天然配套的：

```text
registry 提供待查的 requestId 集合
poller 按这些 requestId 精准检查响应
```

这让 polling 成本与挂起请求数挂钩，而不是与整个 inbox 历史大小挂钩。

---

### 第十一部分：处理成功后还要 `removeWorkerResponse(...)`，说明磁盘/邮箱层需要显式 ack-cleanup

位置：`src/hooks/useSwarmPermissionPoller.ts:305`

处理成功后它还会调用：

- `removeWorkerResponse(requestId, agentName, teamName)`

这很重要，因为这表明底层 response 不是“消费即消失”的纯内存事件，而是某种需要显式清理的持久/半持久载体。

所以这个 poller 不只是读事件，还承担了：

```text
读到 -> 分发 -> 清理
```

完整收口职责。

换句话说，它扮演的是：

```text
worker inbox response janitor
```

---

### 第十二部分：错误只记 debug，不中断主流程，说明轮询器是辅助恢复层

位置：`src/hooks/useSwarmPermissionPoller.ts:311`

poll 里的异常处理只是：

- `logForDebugging(...)`

并不会抛出，也不会主动清掉 callback。

这说明这个轮询器在产品语义上被视为：

```text
异步恢复层
```

而不是主线程里的硬性关键路径。

短暂轮询失败时，系统选择的是：

```text
先保留 pending callback，等待下一轮再试
```

这和前面权限系统大量采用的思路一致：

- 不因为辅助通道的瞬时异常就直接中止主事务
- 保持后续继续恢复的机会

---

### 第十三部分：它和第 32 站一起构成完整的 worker -> leader -> worker 权限闭环

现在把两站串起来就很清楚了：

#### `swarmWorkerHandler.ts`
```text
worker 发 ask
  -> 注册 callback
  -> 发 permission request 给 leader
  -> 等待 Promise 被回包恢复
```

#### `useSwarmPermissionPoller.ts`
```text
worker 轮询/接收 leader 响应
  -> 按 requestId 找回 callback
  -> 调 onAllow/onReject
  -> 恢复原始 ask promise
  -> 清理 response 和 registry
```

所以你可以把这对文件记成：

```text
swarmWorkerHandler = 发送端 + 本地等待挂起器
useSwarmPermissionPoller = 接收端 + continuation 恢复器
```

---

### 元

问题：**这一站真正想解决的架构问题是什么？**

回答：这一站负责 worker 侧接收 leader 权限回包，并通过模块级 callback registry 把异步响应接回原始 ask promise。关键不是轮询本身，而是 continuation table。

### 反

问题：**如果把这一站的设计反过来，会发生什么？**

回答：如果没有 `requestId -> callback` 这张表，leader 的回包回来了也不知道该恢复哪条权限流程。跨 agent ask 就会断在“请求发出后”的半空。

### 空

问题：**跳出当前文件名，这一站背后更大的问题是什么？**

回答：更大的问题是，异步分布式响应怎样重新接回本地控制流。轮询只是表象，真正重要的是 continuation 的保存与恢复。


### 读完这一站后，你应该抓住的 8 个事实

1. `useSwarmPermissionPoller.ts` 的核心不是 React hook，而是 permission/sandbox 回包的 registry + dispatcher。
2. `pendingCallbacks` 是按 `requestId` 索引的 continuation table，用来把 leader 回应接回原始 ask promise。
3. `registerPermissionCallback(...)` 是 worker 侧“发送请求前先登记恢复逻辑”的关键桥梁。
4. leader 回包里的 `permissionUpdates` 会先做 schema 校验，说明 mailbox 返回值被视为外部输入。
5. `processMailboxPermissionResponse(...)` 说明其他 inbox poller 也可以直接复用这套统一分发逻辑。
6. 回调触发前先删除 registry entry，是为了保证响应一次性消费，避免重复 resolve。
7. 实际 polling 只在 swarm worker、且存在 pending callback 时运行，并用 `isProcessingRef` 防止并发轮询。
8. 处理成功后还会显式 `removeWorkerResponse(...)`，说明底层响应载体需要 ack-cleanup，而不是自动消失。

---

### 现在把第 32-33 站串起来

```text
swarmWorkerHandler.ts
  -> worker 侧发起权限请求，上送 leader，并挂起本地等待
useSwarmPermissionPoller.ts
  -> worker 侧接收 leader 回包，按 requestId 找回 callback，恢复原始权限流程
```

所以 swarm 权限链路现在可以压缩成：

```text
worker ask
  -> mailbox permission request
  -> leader decision
  -> worker poller/dispatcher
  -> callback resume
  -> 本地 PermissionContext 收口
```

---

### 下一站建议

下一站最顺的是：

```text
src/utils/swarm/permissionSync.ts
```

因为现在你已经看懂了：

- worker 怎样发权限请求
- worker 怎样收 leader 回包

下一步最自然就是往下钻到协议层本体：

**permission request / response 的 mailbox 数据结构、目录组织、发送/轮询/清理 API，究竟是怎样在 `permissionSync.ts` 里实现的。**
