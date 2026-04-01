## 第 29 站：`src/hooks/toolPermission/PermissionContext.ts`

### 这是什么文件

`src/hooks/toolPermission/PermissionContext.ts` 是权限系统里的**事务工具箱**。

上一站 `src/hooks/useCanUseTool.tsx` 已经看到，它会先创建一个 `ctx`，然后把 allow / deny / ask 各种运行时分支都交给这个 `ctx` 去做日志、取消、queue、persist 等副作用。

所以这一站回答的核心问题是：

```text
这个 PermissionContext 到底封装了哪些权限事务能力，
为什么 useCanUseTool / handlers 不直接自己写这些副作用
```

最准确的一句话是：

```text
PermissionContext.ts = permission flow 的事务执行工具箱
```

---

### 先看最关键的小工具：`createResolveOnce(...)`

位置：`src/hooks/toolPermission/PermissionContext.ts:75`

这个函数非常关键。

它返回三样东西：

- `resolve(value)`
- `isResolved()`
- `claim()`

其中 `claim()` 的意义最重要：

```text
原子地抢占“这次权限流程最终由谁来 resolve”
```

源码注释已经明确说了，它就是为了关掉 async callback 之间的竞态窗口。

也就是说，这个模块从设计上就把权限流程视为：

- 多个异步来源竞争同一个结果
- 必须严格防止重复 resolve

所以第一结论是：

```text
PermissionContext 不只是副作用工具箱，
它还内建了权限竞态的基础同步原语。
```

---

### 第二部分：`createPermissionContext(...)` 真正做了什么

位置：`src/hooks/toolPermission/PermissionContext.ts:96`

这个工厂函数接收：

- `tool`
- `input`
- `toolUseContext`
- `assistantMessage`
- `toolUseID`
- `setToolPermissionContext`
- `queueOps`

然后返回一个冻结的 `ctx` 对象。

这一步的含义不是“塞几个 helper 进去”，而是：

```text
把这一次 tool_use 对应的权限事务上下文固定下来，
让后续所有 handler 都基于同一份上下文工作。
```

它统一固定了：

- 当前 tool / input / messageId / toolUseID
- 当前 appState / abortController 访问入口
- 当前 queue 操作入口
- 当前 permission context 更新入口

因此上层 handler 不必重复传递这些公共字段。

---

### 第三部分：统一日志入口 `logDecision(...)`

位置：`src/hooks/toolPermission/PermissionContext.ts:113`

这里的作用是：

- 补齐 `tool`
- 补齐 `input`
- 补齐 `toolUseContext`
- 补齐 `messageId`
- 补齐 `toolUseID`
- 再调用 `logPermissionDecision(...)`

也就是说：

```text
PermissionContext 把权限日志需要的公共字段统一注入，
避免每个分支自己拼装。
```

这就是典型的事务上下文价值：

- 上层只关心“记一条 accept/reject”
- 不再关心日志参数如何完整拼起来

---

### 第四部分：取消不是普通异常，`logCancelled()` 单独建模

位置：`src/hooks/toolPermission/PermissionContext.ts:132`

这里会发一个 analytics event：

- `tengu_tool_use_cancelled`

说明取消在权限系统里被当成独立产品事件，而不是普通错误。

也就是说：

```text
abort / cancel 是权限流程的一等公民，不是 incidental failure。
```

这也解释了为什么后面的多个 handler 一旦遇到 abort，都会专门调 `ctx.logCancelled()`。

---

### 第五部分：`persistPermissions(...)` 是“落盘 + 内存刷新”双写收口

位置：`src/hooks/toolPermission/PermissionContext.ts:139`

它做的事很关键：

1. `persistPermissionUpdates(updates)`
2. 读取当前 `appState`
3. `applyPermissionUpdates(appState.toolPermissionContext, updates)`
4. `setToolPermissionContext(...)`
5. 返回这批更新里是否有支持持久化的 destination

所以这不是简单“保存一下规则”，而是：

```text
把 permission updates 的磁盘持久化和当前运行时内存态刷新绑成一个统一步骤
```

这正是用户批准“always allow”这类行为真正落地的地方。

因此你应该把它记成：

```text
persistPermissions(...) = 权限更新的双写事务入口
```

---

### 第六部分：`resolveIfAborted(...)` 和 `cancelAndAbort(...)` 是统一中止出口

位置：
- `resolveIfAborted(...)` `src/hooks/toolPermission/PermissionContext.ts:148`
- `cancelAndAbort(...)` `src/hooks/toolPermission/PermissionContext.ts:154`

#### `resolveIfAborted(...)`
作用很直接：

- 如果 signal 已经 aborted
- 先 `logCancelled()`
- 再 `resolve(this.cancelAndAbort(...))`

这说明上层在关键节点反复调用它，是为了防止：

```text
异步等待期间请求已经失效，但后续逻辑还继续跑下去
```

#### `cancelAndAbort(...)`
这个函数更值得细看。

它会根据是否 subagent、是否有 feedback、是否有 contentBlocks 来决定：

- 返回给用户/agent 的拒绝消息文本
- 是否要补 `withMemoryCorrectionHint(...)`
- 是否要真的触发 `abortController.abort()`

而且它最终返回的不是 `deny`，而是：

- `{ behavior: 'ask', message, contentBlocks }`

这说明这里的“取消”语义更像：

```text
把当前审批流程中止，并回给上层一个带解释信息的 ask-style interruption
```

这也是一个很有产品语义的设计。

---

### 第七部分：classifier 在这里被统一封装成 `tryClassifier(...)`

位置：`src/hooks/toolPermission/PermissionContext.ts:174`

当开启 `BASH_CLASSIFIER` 时，`ctx` 上会动态带一个：

- `tryClassifier(...)`

它会：

- 检查是否是 Bash 工具
- 调 `awaitClassifierAutoApproval(...)`
- 若 classifier 命中，再同步 `setClassifierApproval(...)`
- 记一条 classifier accept 日志
- 返回标准 `allow` decision

这说明 classifier 在 PermissionContext 这一层已经被标准化成：

```text
一个可以被 coordinator / swarm worker / interactive flow 复用的自动审批能力
```

也就是说，classifier 不再是 scattered special case，而是正式进入 `ctx` 能力模型。

---

### 第八部分：hooks 也在这里被标准化成 `runHooks(...)`

位置：`src/hooks/toolPermission/PermissionContext.ts:216`

`runHooks(...)` 会：

- 调 `executePermissionRequestHooks(...)`
- 逐个消费 hook ���果
- 若 hook 返回 allow：
  - 合并 `updatedInput`
  - 走 `handleHookAllow(...)`
- 若 hook 返回 deny：
  - 记 reject 日志
  - 必要时触发 interrupt abort
  - 返回标准 deny decision
- 若没有 hook 决定：
  - 返回 `null`

这说明在这个模块里，hooks 被统一建模成：

```text
一个可能提前结束权限流程的自动化审批来源
```

并且 hook allow / deny 都会被翻译回标准 PermissionDecision。

所以它不是“外部插件回调”，而是权限事务中的正式分支。

---

### 第九部分：`buildAllow(...)` / `buildDeny(...)` / `handleUserAllow(...)`

位置：
- `buildAllow(...)` `src/hooks/toolPermission/PermissionContext.ts:264`
- `buildDeny(...)` `src/hooks/toolPermission/PermissionContext.ts:285`
- `handleUserAllow(...)` `src/hooks/toolPermission/PermissionContext.ts:291`
- `handleHookAllow(...)` `src/hooks/toolPermission/PermissionContext.ts:319`

这里体现的是另一层统一：

#### `buildAllow(...)`
把 allow 结果对象统一组装出来，支持：

- `updatedInput`
- `userModified`
- `decisionReason`
- `acceptFeedback`
- `contentBlocks`

#### `buildDeny(...)`
统一构造 deny 决定。

#### `handleUserAllow(...)`
这是最关键的用户批准事务：

- 持久化 permissionUpdates
- 记录 accept 日志（含 permanent 与否）
- 判断 `userModified`
- trim feedback
- 返回标准 allow decision

#### `handleHookAllow(...)`
和上面类似，但来源改成 hook。

所以这一组函数的价值是：

```text
不同批准来源共享同一套 allow/deny 结果构造骨架，
但保留各自的来源语义。
```

---

### 第十部分：queue 操作为什么也放进 `ctx`

位置：
- `pushToQueue(...)` `src/hooks/toolPermission/PermissionContext.ts:337`
- `removeFromQueue(...)` `src/hooks/toolPermission/PermissionContext.ts:340`
- `updateQueueItem(...)` `src/hooks/toolPermission/PermissionContext.ts:343`

这三项看起来普通，但意义很大。

说明 permission queue 在架构上不是 React 组件私有状态，而是：

```text
权限事务的一部分副作用接口
```

也就是说，interactive handler 之类的上层逻辑不必直接依赖 `setState`，只需要对 `ctx` 说：

- push 一个请求
- remove 当前请求
- update 当前请求

这让 queue 成为通用接口，而不是 UI 私货。

---

### 第十一部分：`createPermissionQueueOps(...)` 是 React 适配器

位置：`src/hooks/toolPermission/PermissionContext.ts:357`

这里很清楚地把 React state setter 适配成：

- `push(item)`
- `remove(toolUseID)`
- `update(toolUseID, patch)`

所以正确理解是：

```text
PermissionContext 不依赖 React；
React 只是 queueOps 的一个后端实现。
```

这就是它为什么能被看作真正的事务工具箱，而不是 hook 辅助文件。

---

### 读完这一站后，你应该抓住的 8 个事实

1. `PermissionContext.ts` 是 permission flow 的事务执行工具箱。
2. `createResolveOnce(...)` 提供了多异步来源竞争同一结果时的基础同步原语。
3. `createPermissionContext(...)` 把一次 tool_use 的权限事务上下文固定下来，供所有 handler 共享。
4. `persistPermissions(...)` 统一处理 permission updates 的落盘与内存态刷新。
5. `resolveIfAborted(...)` / `cancelAndAbort(...)` 提供了统一的取消与中止出口。
6. classifier 和 hooks 都在这里被标准化成可复用的自动审批能力。
7. `handleUserAllow(...)` / `handleHookAllow(...)` 统一了不同来源的批准事务语义。
8. queue 操作被抽象成通用接口，React 只是其适配器实现。

---

### 现在把第 28-29 站串起来

```text
useCanUseTool.tsx
  -> 作为运行时总闸门，决定本次权限流程走哪条分支
PermissionContext.ts
  -> 为这些分支提供统一的日志 / persist / queue / abort / classifier / hooks 工具箱
```

所以两站分工可以压缩成：

```text
useCanUseTool.tsx = 权限运行时总闸门
PermissionContext.ts = 权限事务工具箱
```

---

### 下一站建议

下一站最顺的是：

```text
src/hooks/toolPermission/handlers/interactiveHandler.ts
```

因为现在你已经看懂了：

- 总闸门怎样分流
- 事务工具箱怎样提供能力

下一步最自然就是看：

**当结果最终落到 interactive ask 时，本地 UI、bridge、channel、hooks、classifier 是怎样竞争并收口到一个最终决定上的。**
