## 第 60 站：`src/hooks/toolPermission/PermissionContext.ts`

### 这是什么文件

`src/hooks/toolPermission/PermissionContext.ts` 是工具权限系统里的 **共享审批上下文封装层**。

上一站 `swarmWorkerHandler.ts` 已经看懂：

- ask-mode worker 会拿到一个 `ctx`
- 用它来：
  - `handleUserAllow(...)`
  - `cancelAndAbort(...)`
  - `logDecision(...)`
- 但 `ctx` 自己到底封装了什么、为什么所有 handler 都围绕它转，还没拆开

这一站回答的是：

```text
一次 tool permission 流里，
那些跨 handler 共享的状态、日志、持久化、abort、queue 操作、
one-shot resolve 保护和 allow/deny 结果构造，
到底怎样被统一封装成 PermissionContext
```

所以最准确的一句话是：

```text
PermissionContext.ts = 工具权限编排流的共享操作面与结果构造器
```

---

### 先看它的整体定位

位置：`src/hooks/toolPermission/PermissionContext.ts:1`

这个文件不是：

- 具体 permission policy evaluator
- 某一个 ask handler
- React hook 本体
- mailbox transport 层

它的职责是：

1. 为一次具体 tool permission 流构造共享 `ctx`
2. 提供统一的 logging / cancel / allow / deny / hooks / classifier / queue 操作
3. 提供 one-shot `resolveOnce` 原语
4. 把 React queue setter 适配成通用 queueOps 接口

所以它本质上是：

```text
permission-flow shared runtime context
```

或者更具体地说：

```text
one tool-use permission attempt
  -> one frozen orchestration context
  -> all handlers operate through it
```

---

### 第一部分：`PermissionApprovalSource` / `PermissionRejectionSource` 把“是谁批准/拒绝的”显式类型化，说明系统非常在意权限决策来源的可解释性

位置：`src/hooks/toolPermission/PermissionContext.ts:45`

这里把批准来源分成：

- `{ type: 'hook'; permanent?: boolean }`
- `{ type: 'user'; permanent: boolean }`
- `{ type: 'classifier' }`

拒绝来源分成：

- `{ type: 'hook' }`
- `{ type: 'user_abort' }`
- `{ type: 'user_reject'; hasFeedback: boolean }`

这说明权限系统并不满足于只知道：

- allow
- deny

它还要知道：

```text
why / by whom / with what permanence semantics
```

这些来源信息后面会影响：

- logging
- UI 呈现
- analytics
- permission updates 持久化语义

所以这里是在给整条权限链建立“决策 provenance vocabulary”。

---

### 第二部分：`PermissionQueueOps` 很关键，因为它把 queue 操作抽象成通用接口，让 PermissionContext 不直接依赖 React state 细节

位置：`src/hooks/toolPermission/PermissionContext.ts:55`

接口只有三个动作：

- `push(item)`
- `remove(toolUseID)`
- `update(toolUseID, patch)`

注释直接说：

- 这是 generic interface
- 在 REPL 里由 React state 支撑

这说明作者不想让 `PermissionContext` 自己绑定：

- `setState`
- React component tree
- 某种具体 UI 容器

而是把 queue 看成一个可替换的宿主能力：

```text
permission queue as an abstract mutable collection interface
```

这样后面 handler 操作 queue 时，
就不用关心底层是不是 React。

---

### 第三部分：`ResolveOnce<T>` / `createResolveOnce()` 是这份文件最容易忽略但极关键的并发原语，它把多条异步终局路径收敛成单次结算语义

位置：`src/hooks/toolPermission/PermissionContext.ts:63`

这个类型提供：

- `resolve(value)`
- `isResolved()`
- `claim()`

而 `claim()` 的注释很关键：

```text
Atomically check-and-mark as resolved.
Use this in async callbacks BEFORE awaiting.
```

也就是说它专门解决的是：

```text
多个异步回调同时竞争结束同一个 permission flow
```

例如：

- onAllow
- onReject
- abort
- hook interrupt
- classifier completion

如果没有这个原语，
单靠 `isResolved()` 再 `resolve()` 会有竞态窗口。

所以 `createResolveOnce()` 本质上是：

```text
permission-flow settlement mutex
```

这是整个权限编排链稳定性的关键基础件。

---

### 第四部分：`createPermissionContext(...)` 把一次 tool permission 流的核心身份锚点固定成 `tool + input + toolUseContext + assistantMessage + toolUseID + messageId`

位置：`src/hooks/toolPermission/PermissionContext.ts:96`

这个函数一开始收进来的就是一次完整上下文：

- `tool`
- `input`
- `toolUseContext`
- `assistantMessage`
- `toolUseID`
- `setToolPermissionContext`
- `queueOps?`

再从 `assistantMessage.message.id` 推出：

- `messageId`

这说明 `ctx` 不是某种全局权限单例，
而是：

```text
one immutable orchestration context for one specific tool-use attempt
```

所有后续：

- log
- allow/reject result
- hooks
- classifier
- queue mutation
- abort

都绑定在这个固定身份锚点上。

---

### 第五部分：`logDecision(...)` 说明 PermissionContext 的一个核心职责是把日志参数标准化，避免每个 handler 自己重复组装 tool/input/messageId/toolUseID

位置：`src/hooks/toolPermission/PermissionContext.ts:113`

这里 `ctx.logDecision(...)` 内部会统一调用 `logPermissionDecision(...)`，
并自动带上：

- `tool`
- `input`（可被 opts 覆盖）
- `toolUseContext`
- `messageId`
- `toolUseID`

这说明作者很明确地把 logging 当成 `ctx` 的基础服务之一。

也就是说 handler 不需要每次都记得补全日志上下文，
只要提供：

- accept/reject
- source
- 可选耗时

就行。

所以这里实现的是：

```text
centralized permission decision logging envelope
```

---

### 第六部分：`logCancelled()` 单独走 analytics，而不是只写普通 debug/log，说明“工具权限被取消”在产品分析上是一个一等事件

位置：`src/hooks/toolPermission/PermissionContext.ts:132`

这里会发：

- `logEvent('tengu_tool_use_cancelled', ...)`
- 附带 `messageID`、`toolName`

这说明取消审批不只是内部控制流，
还是产品需要观测的交互事件。

也就是说从产品视角，
用户/系统为什么中途放弃工具调用，
是值得统计的。

这也解释了为什么前面那么多地方都认真区分：

- user_abort
- user_reject
- hook deny
- classifier allow

因为这些都关系到权限系统的实际使用行为分析。

---

### 第七部分：`persistPermissions(...)` 把“接受这次工具调用”和“把 permission updates 写回长期上下文”合并成一个原子辅助动作，说明 permanent allow 是权限流的正式副作用，而不是 UI 附带选项

位置：`src/hooks/toolPermission/PermissionContext.ts:139`

这里的逻辑是：

1. 没 updates 就直接 false
2. `persistPermissionUpdates(updates)`
3. 从 `appState` 取当前 `toolPermissionContext`
4. `applyPermissionUpdates(...)`
5. `setToolPermissionContext(...)`
6. 返回是否有支持 persistence 的 destination

这说明 permission updates 的语义是双层的：

#### 持久层
写入长期规则

#### 运行层
立即更新当前 appState permission context

也就是说“Always allow”这类选择不是等下次会话才生效，
而是：

```text
persist now + apply now
```

这就是为什么它被做成 `ctx` 里的标准能力。

---

### 第八部分：`resolveIfAborted(...)` 和 `cancelAndAbort(...)` 两个方法组合起来，说明 PermissionContext 统一收口了“权限流被取消时该怎样构造结果、怎样中断 abortController、怎样生成最终消息”

位置：

- `resolveIfAborted()` `src/hooks/toolPermission/PermissionContext.ts:148`
- `cancelAndAbort()` `src/hooks/toolPermission/PermissionContext.ts:154`

`resolveIfAborted(...)` 负责：

- 检查 signal 是否已 aborted
- 记录 cancelled
- 立刻 `resolve(cancelAndAbort(...))`

而 `cancelAndAbort(...)` 负责：

- 生成 reject message
- 判断是不是 subagent
- 需要时补 `withMemoryCorrectionHint(...)`
- 在某些条件下触发 `abortController.abort()`
- 返回 `{ behavior:'ask', message, contentBlocks }`

这说明取消不是随便抛个异常，
而是一个有标准产物的权限结果构造流程。

特别是它返回的仍然是：

```text
PermissionDecision-shaped object
```

而不是错误。

这再次体现整条权限系统的设计哲学：

```text
normalize control-flow interruptions into decision objects
```

---

### 第九部分：subagent 与主 agent 的拒绝消息模板在这里统一分流，说明“同样是拒绝”，给主 agent 和子 agent 的提示语义并不相同

位置：`src/hooks/toolPermission/PermissionContext.ts:159`

这里会根据：

- `!!toolUseContext.agentId`

判断是不是 subagent。

然后选用：

- `SUBAGENT_REJECT_MESSAGE`
- `SUBAGENT_REJECT_MESSAGE_WITH_REASON_PREFIX`
- `REJECT_MESSAGE`
- `REJECT_MESSAGE_WITH_REASON_PREFIX`

而且只有主 agent 的 message 才会再套：

- `withMemoryCorrectionHint(...)`

这说明作者明确认为：

```text
permission rejection messaging is role-sensitive
```

主 agent 和 subagent 的后续行动空间不同，
因此拒绝提示也不同。

这里实际上是权限结果文案层面的 role normalization。

---

### 第十部分：`tryClassifier(...)` 被放进 PermissionContext，而不是散落在 handler 里，说明 classifier auto-approval 被视为一项标准共享能力，而不是 Bash 专属临时逻辑

位置：`src/hooks/toolPermission/PermissionContext.ts:174`

虽然这个方法只在 feature `BASH_CLASSIFIER` 下存在，
且内部确实只处理 `tool.name === BASH_TOOL_NAME`，
但它仍被做成 `ctx.tryClassifier(...)`。

这说明作者的组织方式是：

```text
classifier auto-approval is part of the permission orchestration toolkit
```

而不是让：

- swarm handler 自己拼 classifier 流
- coordinator handler 自己再写一遍 classifier 流

这样各 handler 就能统一调用：

- 试 classifier
- 成功就得到标准 `PermissionDecision`
- 失败就继续自己的 ask-path

这是一种很好的共享能力抽象。

---

### 第十一部分：`tryClassifier(...)` 内部直接记日志并构造 allow result，说明一旦 classifier 命中，它在语义上已经被视为一个完整的 accept source，而不是前置建议

位置：`src/hooks/toolPermission/PermissionContext.ts:183`

这里 classifier 命中后会：

- `setClassifierApproval(...)`（某些场景）
- `logPermissionDecision(... accept/classifier ...)`
- 返回完整的 `behavior:'allow'` decision

这说明 classifier 在这套架构里不是“给用户建议一下”，
而是：

```text
a full-fledged approval source
```

它和：

- hook allow
- user allow

一样，都会最终生成标准 allow decision。

---

### 第十二部分：`runHooks(...)` 很关键，因为它把 PermissionRequest hooks 也收进 PermissionContext，说明 hook 审批和 user/classifier/swarm 审批在架构上是同级能力

位置：`src/hooks/toolPermission/PermissionContext.ts:216`

这里会：

- 调 `executePermissionRequestHooks(...)`
- 遍历 hook result
- 如果有 `permissionRequestResult`
  - allow -> `handleHookAllow(...)`
  - deny -> `buildDeny(...)`

这说明 hook 并不是权限系统的外围插件，
而是已经被纳入标准审批路径。

也就是说在架构上：

```text
hook approval/rejection is a first-class decision source
```

和：

- classifier
- user
- swarm leader

一样，都能直接决定一条工具调用的 fate。

---

### 第十三部分：`runHooks(...)` 对 hook deny 的处理暴露了一个重要语义：hook 不仅能 deny，还能 interrupt 整个工具调用链

位置：`src/hooks/toolPermission/PermissionContext.ts:240`

当 hook deny 时：

- `logDecision(reject/hook)`
- 若 `decision.interrupt`
  - `abortController.abort()`
- `buildDeny(...)`

这说明 hook deny 有两层强度：

#### 普通 deny
这次工具不用了

#### interrupt deny
把整个当前控制流也中断掉

所以 hook 在这里不是只会“给一个建议”，
而是真正能驱动执行层 abort 的强控制源。

这也是为什么 hook 路径必须放进 `ctx` 的统一 abort 语义面里。

---

### 第十四部分：`buildAllow(...)` / `buildDeny(...)` 说明 PermissionContext 的另一个关键职责是统一结果对象形状，避免各 handler 各自手写 PermissionDecision 结构

位置：

- `buildAllow()` `src/hooks/toolPermission/PermissionContext.ts:264`
- `buildDeny()` `src/hooks/toolPermission/PermissionContext.ts:285`

`buildAllow(...)` 会统一组装：

- `behavior: 'allow'`
- `updatedInput`
- `userModified`
- `decisionReason`
- `acceptFeedback`
- `contentBlocks`

`buildDeny(...)` 则统一组装：

- `behavior: 'deny'`
- `message`
- `decisionReason`

这说明 `ctx` 不只是共享状态工具箱，
还是：

```text
canonical PermissionDecision factory
```

这样所有 handler 的输出 shape 都由同一个地方保证。

---

### 第十五部分：`handleUserAllow(...)` 和 `handleHookAllow(...)` 把“批准后要做的副作用”统一封装了，说明 allow 真正复杂的不是返回 allow，而是随之而来的持久化、日志和输入差异计算

位置：

- user allow `src/hooks/toolPermission/PermissionContext.ts:291`
- hook allow `src/hooks/toolPermission/PermissionContext.ts:319`

这两个函数都会：

- `persistPermissions(...)`
- `logDecision(accept, source...)`
- 最后 `buildAllow(...)`

但它们又保留了差异：

#### `handleUserAllow(...)`
- 计算 `userModified`
- 带 `feedback`
- 可带 `contentBlocks`
- 可带任意 `decisionReason`

#### `handleHookAllow(...)`
- 固定 source 为 hook
- 固定 decisionReason 为 `hook: PermissionRequest`

这说明作者不是简单抽一个“大 allow helper”，
而是在共享副作用之上保留不同批准源的语义差异。

---

### 第十六部分：`tool.inputsEquivalent?.(...)` 被用于判断 `userModified`，说明权限系统不仅关心“允许了没有”，还关心批准时用户是否实质改写了工具输入

位置：`src/hooks/toolPermission/PermissionContext.ts:308`

这里会：

- 如果 tool 提供 `inputsEquivalent`
- 就比较原始 `input` 与 `updatedInput`
- 得出 `userModified`

这很重要。

因为同样是 allow，
其实有至少两种语义：

```text
allow as-is
vs
allow after human adjustment
```

这个差异会影响：

- logging
- UI 回显
- 后续分析
- 可能的模型理解

所以权限系统追踪的不是单一 accept，
而是更细粒度的 accept semantics。

---

### 第十七部分：`pushToQueue/removeFromQueue/updateQueueItem` 被放进 ctx，说明 PermissionContext 同时也是权限 UI queue 的统一操作面

位置：`src/hooks/toolPermission/PermissionContext.ts:337`

这里只是简单代理：

- `queueOps?.push(item)`
- `queueOps?.remove(toolUseID)`
- `queueOps?.update(toolUseID, patch)`

但意义很大。

因为这表示 handler 不需要知道：

- queue 是 React state
- queue item 如何按 toolUseID 匹配
- queue update 的底层细节

它们只需要操作：

```text
current permission request's queue representation
```

这使得 PermissionContext 同时统一了：

- 决策结果构造
- abort 语义
- queue mutation

成为一个真正完整的审批共享上下文。

---

### 第十八部分：`Object.freeze(ctx)` 暗示 PermissionContext 被设计成只读协作面，而不是可随手被 handler 动态改写的共享可变对象

位置：`src/hooks/toolPermission/PermissionContext.ts:347`

最后返回前：

- `return Object.freeze(ctx)`

这说明作者希望 handler 把 `ctx` 当成：

```text
stable capability bundle
```

而不是：

- 动态塞字段
- 运行时猴补状态
- 随手覆写共享属性

这能减少 handler 之间的隐式耦合。

所以 `ctx` 在架构上更像 capability object，
而不是 mutable session bag。

---

### 第十九部分：`createPermissionQueueOps(...)` 是 React -> generic queue interface 的桥，说明这份文件既抽象化权限流，又负责把抽象重新落回当前 REPL 宿主

位置：`src/hooks/toolPermission/PermissionContext.ts:357`

这里把：

- `setToolUseConfirmQueue`

包装成：

- `push`
- `remove`
- `update`

内部都是标准 React setState 变换。

这说明整份文件的定位非常巧妙：

#### 向上
给 handler 暴露不依赖 React 的通用 ctx/queueOps

#### 向下
把当前 REPL 宿主的 React state 适配进来

也就是说它兼顾了：

```text
abstraction
+
current host integration
```

---

### 第二十部分：整份文件真正把权限系统从“多个 helper 到处传参”提升成了一个统一的 capability layer

如果把整份文件压缩，会发现它其实在做五层事：

#### 1. decision provenance typing
- approval/rejection source types

#### 2. concurrency primitive
- `createResolveOnce()`

#### 3. shared orchestration context
- `createPermissionContext()`

#### 4. result factories + side effects
- `buildAllow/buildDeny`
- `handleUserAllow/handleHookAllow`
- `persistPermissions`
- `cancelAndAbort`
- `runHooks`
- `tryClassifier`

#### 5. host integration bridge
- `createPermissionQueueOps()`

所以最准确的压缩表达是：

```text
PermissionContext.ts = 工具权限系统的共享能力层：把日志、abort、hook/classifier 处理、permission update 持久化、queue 操作和标准结果构造收口成一个冻结的单次审批上下文
```

---

### 读完这一站后，你应该抓住的 10 个事实

1. `PermissionContext.ts` 不是某个具体 handler，而是工具权限编排流的共享能力层，负责把一次 tool-use permission attempt 所需的共用操作统一封装进 `ctx`。
2. `PermissionApprovalSource` / `PermissionRejectionSource` 说明权限系统非常重视决策来源的可解释性，不仅要知道 allow/deny，还要知道是谁批准/拒绝、是否永久生效、是否带反馈。
3. `createResolveOnce()` 是整条权限异步链的关键并发原语，用来让多个竞争的终局路径（allow/reject/abort 等）只允许一次结算。
4. `createPermissionContext()` 会把 tool、input、toolUseContext、assistantMessage、toolUseID、messageId 固定成单次权限流的身份锚点，并返回一个 `Object.freeze(...)` 的只读 capability object。
5. `persistPermissions()` 体现了 permanent allow 的双重语义：既把 updates 写回持久规则，又立即更新当前运行态的 `toolPermissionContext`。
6. `resolveIfAborted()` 与 `cancelAndAbort()` 统一了权限流取消时的标准行为：记录取消、必要时触发 abortController、并返回规范化的 PermissionDecision，而不是直接抛异常。
7. `tryClassifier()` 表明 classifier auto-approval 被纳入 PermissionContext 的标准共享能力，一旦命中就直接产出完整 allow decision，而不是仅作为建议存在。
8. `runHooks()` 说明 PermissionRequest hooks 也是一等审批源，既可以 allow，也可以 deny，甚至可以通过 `interrupt` 直接中断整条工具调用链。
9. `buildAllow/buildDeny` 与 `handleUserAllow/handleHookAllow` 把结果对象形状、日志、副作用和持久化统一收口，避免各 handler 手写不同风格的 decision 结果。
10. `createPermissionQueueOps()` 则把 React `setToolUseConfirmQueue` 适配为通用 queue interface，使 PermissionContext 能同时保持抽象化和对当前 REPL 宿主的落地集成。

---

### 现在把第 59-60 站串起来

```text
swarmWorkerHandler.ts
  -> 用 ctx 把 worker->leader 审批链接回当前 tool use
PermissionContext.ts
  -> 定义这个 ctx 到底有哪些共享能力与标准结果构造逻辑
```

所以现在 permission handler 层可以先压缩成：

```text
one tool-use permission flow
  -> create frozen PermissionContext
  -> handlers use ctx for classifier/hooks/queue/logging/abort/persistence
  -> final PermissionDecision returned in a normalized shape
```

也就是说：

```text
swarmWorkerHandler.ts 回答“worker->leader ask-path 怎样利用 ctx 启动并恢复远程审批”
PermissionContext.ts 回答“这个 ctx 本身到底提供了哪些共享能力，怎样把整个权限流的共性副作用和结果构造统一起来”
```

---

### 下一站建议

下一站最顺的是：

```text
src/hooks/toolPermission/handlers/interactiveHandler.ts
```

因为现在你已经看懂了：

- `useCanUseTool.tsx` 如何总分流
- `swarmWorkerHandler.ts` 如何处理 worker->leader 审批
- `PermissionContext.ts` 如何给各 handler 提供共享能力
- 但还没看 ask-path 最终兜底的那条本地交互权限链：
  - `ToolUseConfirm` queue 项怎么构造
  - `onAllow / onReject / onAbort` 怎样落地
  - hooks / classifier / queue 更新怎样在 interactive dialog 中配合

下一步最自然就是把本地交互审批这半边补齐：

**interactiveHandler.ts 到底怎样把一次 ask-mode 权限请求放进确认队列、连接用户操作回调，并最终用 PermissionContext 统一收尾成 allow/reject/abort 的标准结果。**
