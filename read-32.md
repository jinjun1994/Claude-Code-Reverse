## 第 32 站：`src/hooks/toolPermission/handlers/swarmWorkerHandler.ts`

### 这是什么文件

`src/hooks/toolPermission/handlers/swarmWorkerHandler.ts` 是权限系统里的 **swarm worker 权限上送处理器**。

前两站已经看到：

```text
interactiveHandler.ts
  -> main agent 的 interactive ask 竞态协调器
coordinatorHandler.ts
  -> coordinator worker 的 automated-first 预处理器
```

而这一站处理的是第三种场景：

```text
当当前执行者不是主 agent，而是 swarm / teammate worker 时，
ask 不再优先弹本地交互，而是尽量把权限请求上送给 leader 来裁决
```

所以最准确的一句话是：

```text
swarmWorkerHandler.ts = swarm worker 把权限请求转交给 leader 的转发收口器
```

---

### 先看它解决的产品问题

位置：`src/hooks/toolPermission/handlers/swarmWorkerHandler.ts:26`

文件头注释已经把主流程说清楚了：

1. 先尝试 classifier auto-approval
2. 如果不能自动批准，就把权限请求发给 leader
3. 注册 leader 响应回调
4. 等待期间设置 pending indicator

这说明 swarm worker 的 ask 策略不是：

```text
worker 自己弹审批框自己问用户
```

而是：

```text
worker 尽量不上本地交互，
而是把审批权提升给 leader 所在的主控制面
```

所以它本质上是在做：

```text
worker -> leader 的权限升级转发
```

---

### 第一部分：只有在真正的 swarm worker 身份下才生效

位置：`src/hooks/toolPermission/handlers/swarmWorkerHandler.ts:40`

一开始先判断：

- `isAgentSwarmsEnabled()`
- `isSwarmWorker()`

只要任一不满足，就直接返回 `null`。

这说明它不是通用 ask handler，而是一个条件分支处理器：

```text
只有启用了 agent swarms，且当前执行体确实是 worker 时，
这条“上送 leader”路径才成立
```

这里的 `null` 语义和 coordinatorHandler 一样关键：

```text
不是 deny
不是 error
而是“这条分支不适用，请调用方继续走别的 ask 处理链”
```

所以在 `useCanUseTool.tsx` 里，它的位置是：

```text
swarm worker 特殊分支探测器
```

---

### 第二部分：worker 也会先试 classifier，但方式和 interactive 不同

位置：`src/hooks/toolPermission/handlers/swarmWorkerHandler.ts:49`

这里先做：

- `feature('BASH_CLASSIFIER')`
- `await ctx.tryClassifier?.(params.pendingClassifierCheck, updatedInput)`

注释点得很清楚：

- 对 bash 命令才尝试 classifier
- agent 会等待 classifier 结果
- 不像 main agent 那样把 classifier 和用户交互做并发竞态

所以 swarm worker 在 classifier 这一层更接近 coordinator，而不是 interactive：

```text
先等自动化；
自动化若能批准，就不必把请求送给 leader
```

因此你应该把这里记成：

```text
swarm worker = classifier first, leader second
main interactive = user/UI 与 classifier 竞态
```

这再次说明不同运行角色会采用不同的权限产品策略。

---

### 第三部分：真正的核心是把 permission request 封装后发进 mailbox

位置：`src/hooks/toolPermission/handlers/swarmWorkerHandler.ts:70`

如果 classifier 没解决，接下来会创建请求：

- `createPermissionRequest(...)`

里面带上：

- `toolName`
- `toolUseId`
- `input`
- `description`
- `permissionSuggestions`

然后通过：

- `sendPermissionRequestViaMailbox(request)`

发给 leader。

这里最该抓住的不是“发了个消息”，而是：

```text
权限审批在 swarm 模型里被当成一种标准 mailbox 协议消息
```

也就是说，worker 和 leader 之间不是共享一个本地 confirm queue，
而是通过团队通信基础设施做跨执行体权限同步。

所以这一站把权限系统和 swarm 基础设施真正接上了。

---

### 第四部分：先注册回调，再发送请求，是为了关掉跨进程竞态窗口

位置：`src/hooks/toolPermission/handlers/swarmWorkerHandler.ts:79`

这里非常关键的一句注释是：

```text
Register callback BEFORE sending the request to avoid race condition
where leader responds before callback is registered
```

实际顺序是：

1. `registerPermissionCallback(...)`
2. 然后才 `sendPermissionRequestViaMailbox(request)`

这说明这里防的不是普通函数回调顺序问题，而是：

```text
跨 worker / leader 的异步消息系统中，
响应可能比你想象得更快回来
```

如果先 send 再 register，就可能发生：

- leader 已经批了
- 响应已经回来了
- 但本地还没挂上监听器
- 导致本次 ask 永远悬空

所以这一步的本质是：

```text
先占住回包接收口，再发出请求
```

这是典型的异步协议竞态修复手法。

---

### 第五部分：`createResolveOnce(...)` 在这里继续充当唯一裁决保护

位置：`src/hooks/toolPermission/handlers/swarmWorkerHandler.ts:67`

它同样先建：

- `resolveOnce`
- `claim`

这说明即使在 swarm worker 场景里，仍然存在多个可能结束这次 ask 的来源：

- leader allow
- leader reject
- 本地 abort signal

因此这里依然需要：

```text
只有第一个到达的终止来源可以真正结束这次权限流程
```

所以 `createResolveOnce(...)` 不是 interactive 专用，而是整个 permission runtime 的统一同步原语。

---

### 第六部分：leader allow 最终仍然走 `ctx.handleUserAllow(...)`

位置：`src/hooks/toolPermission/handlers/swarmWorkerHandler.ts:84`

当 leader 回来的是 allow 时：

- 先 `claim()`
- 清掉 pending 状态
- 计算最终输入 `finalInput`
- 调 `ctx.handleUserAllow(...)`
- `resolveOnce(...)`

这里一个很重要的设计是：

```text
虽然批准来自 leader，
但在 worker 本地仍然复用“用户批准事务”的标准收口函数
```

这意味着 leader 的审批结果并不会绕开本地权限事务模型。

它仍然要统一经过：

- permission updates 持久化
- accept 日志
- feedback/contentBlocks 处理
- 标准 allow decision 构造

所以更准确地说：

```text
leader 只是远端决策者；
真正的事务落地仍由 worker 本地 PermissionContext 统一收口
```

---

### 第七部分：leader reject 也不是特殊返回，而是复用标准 reject/cancel 语义

位置：`src/hooks/toolPermission/handlers/swarmWorkerHandler.ts:109`

当 leader 返回 reject：

- `claim()`
- 清 pending 状态
- `ctx.logDecision(...)`
- `resolveOnce(ctx.cancelAndAbort(...))`

这里能看出两个层次：

#### 1. reject 会被明确记日志
来源被记录成：

- `decision: 'reject'`
- `source: { type: 'user_reject', hasFeedback: !!feedback }`

这说明 leader 的拒绝在语义上仍然被视为：

```text
一次正式的人工拒绝
```

#### 2. 返回值仍走 `cancelAndAbort(...)`
说明 swarm worker 分支最后对上层返回的仍是统一的 interruption / ask-style 中止结果，
而不是另起一种“leader rejected”专有协议。

所以它保持了 permission runtime 很一致的一点：

```text
不同来源可以不同，
但最终返回给上层的 PermissionDecision 结构尽量统一
```

---

### 第八部分：`pendingWorkerRequest` 暴露的是“正在等待 leader”这一运行时状态

位置：`src/hooks/toolPermission/handlers/swarmWorkerHandler.ts:61`

发送请求后，它会：

- `setAppState(...)`
- 写入 `pendingWorkerRequest`

内容包括：

- `toolName`
- `toolUseId`
- `description`

在 allow / reject / abort 时，又会统一清掉它。

这说明 swarm worker ask 期间，运行时会显式暴露一个状态：

```text
当前 worker 正在等待 leader 审批
```

所以这里不只是逻辑通道，还有 UI / 状态层面的可观测性。

你可以把它理解成：

```text
interactiveHandler 用 confirm queue 暴露“正在等本地/远端审批”
swarmWorkerHandler 用 pendingWorkerRequest 暴露“正在等 leader 回应”
```

两者都是把权限等待态显式同步进 app state，只是表示形式不同。

---

### 第九部分：abort 监听保证 worker 不会因为 leader 没回而永久挂住

位置：`src/hooks/toolPermission/handlers/swarmWorkerHandler.ts:135`

这里给 `abortController.signal` 挂了一个一次性监听：

- 如果等待 leader 期间发生 abort
- 就 `claim()`
- 清 pending 状态
- `ctx.logCancelled()`
- `resolveOnce(ctx.cancelAndAbort(undefined, true))`

这段非常关键，因为它解决的是一个典型分布式等待问题：

```text
如果请求方自己都已经取消了，
那就不能继续无限期等待远端 leader 的回包
```

所以这一段的真正意义是：

```text
leader mailbox 只是异步审批通道，
绝不能让它成为本地 ask promise 的永久悬挂点
```

这和 coordinatorHandler 的思想是一致的：

- 自动化可以失败
- 远端审批可以慢
- 但权限流不能因此卡死

---

### 第十部分：发送失败会回落本地处理，而不是直接失败

位置：`src/hooks/toolPermission/handlers/swarmWorkerHandler.ts:150`

如果 swarm permission submission 失败：

- `logError(toError(error))`
- 返回 `null`

这再次说明：

```text
swarm worker 上送 leader 是优先路径，但不是唯一通道
```

`null` 在这里的含义仍然是：

```text
这条分支没有成功解决 ask；
请调用方继续回落到本地 interactive handling
```

所以整体策略并不是：

```text
worker 模式下必须依赖 leader 才能继续
```

而是：

```text
优先 leader；
leader 通道不可用时，仍保留本地兜底能力
```

这是一个很稳健的容错设计。

---

### 第十一部分：它和 interactive / coordinator 的真正区别

现在把三条 ask 路径并排看：

#### `interactiveHandler.ts`
```text
主 agent 场景
先把交互入口建立起来
本地用户 / bridge / channel / hooks / classifier 并发竞态
```

#### `coordinatorHandler.ts`
```text
coordinator worker 场景
先顺序跑 hooks -> classifier
不能解决再回落 interactive dialog
```

#### `swarmWorkerHandler.ts`
```text
swarm worker 场景
先 classifier
再把 ask 通过 mailbox 上送 leader
leader 不可用或失败时再回落本地处理
```

所以这三者不是简单的“不同 handler”，而是三种运行角色对应的三套 ask 策略：

- main agent：本地/远端多终端并发竞态
- coordinator worker：自动化优先，尽量少打扰用户
- swarm worker：leader 集中裁决优先，worker 自己尽量不上本地审批

---

### 读完这一站后，你应该抓住的 8 个事实

1. `swarmWorkerHandler.ts` 是 swarm worker 的权限上送处理器，不是普通 interactive handler。
2. 它只在启用 agent swarms 且当前身份是 swarm worker 时生效。
3. 它会先尝试 classifier auto-approval，再决定是否把请求发给 leader。
4. 权限请求通过 mailbox 协议发往 leader，说明 swarm 权限同步是团队通信的一部分。
5. 先注册回调再发送请求，是为了防止 leader 过快响应造成的回包竞态丢失。
6. leader 的 allow / reject 最终仍通过本地 `PermissionContext` 统一收口，而不是走一套独立返回协议。
7. `pendingWorkerRequest` 表示 worker 当前正等待 leader 审批，是显式的运行时状态。
8. leader 通道失败或中断时会回落本地处理，不会把 ask 流卡死。

---

### 现在把第 30-32 站串起来

```text
interactiveHandler.ts
  -> main agent 的 interactive ask 竞态协调器
coordinatorHandler.ts
  -> coordinator worker 的 automated-first ask 预处理器
swarmWorkerHandler.ts
  -> swarm worker 的 leader-forwarded ask 转发收口器
```

所以现在已经可以把 ask 的三种主要运行时策略并列记住：

```text
main agent = 本地/远端并发竞态审批
coordinator worker = 先自动化，再决定是否交互
swarm worker = 先 classifier，再上送 leader 审批
```

---

### 下一站建议

下一站最顺的是：

```text
src/hooks/useSwarmPermissionPoller.ts
```

因为这一站你已经看懂了：

- worker 怎样把权限请求发给 leader
- 本地怎样注册 leader 响应回调

下一步最自然就是补齐另一半：

**leader / worker 之间的 permission mailbox 响应是怎样被轮询、分发并回调到 `registerPermissionCallback(...)` 这条链上的。**
