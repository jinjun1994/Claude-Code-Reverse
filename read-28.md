## 第 28 站：`src/hooks/useCanUseTool.tsx`

### 这是什么文件

`src/hooks/useCanUseTool.tsx` 是权限系统里的**运行时总闸门**。

如果说上一站 `src/utils/permissions/permissions.ts` 解决的是：

```text
规则上到底应该 allow / deny / ask
```

那么这一站解决的是：

```text
这个判定怎样接进真实 tool 调用前的最后一道运行时门，
并把确认弹窗、用户反馈、permission updates、worker 转发、classifier 快路径都收口进去
```

所以最准确的一句话是：

```text
useCanUseTool.tsx = tool 调用前的权限运行时总闸门
```

---

### 先看入口签名

位置：`src/hooks/useCanUseTool.tsx:27`

这里导出的核心类型是：

- `CanUseToolFn`

输入包括：

- `tool`
- `input`
- `toolUseContext`
- `assistantMessage`
- `toolUseID`
- 可选 `forceDecision`

输出是：

- `Promise<PermissionDecision<Input>>`

这说明在 query / tool execution 看来，这不是一个 UI hook，而是：

```text
给定一次 tool_use，请返回最终权限决定的异步能力接口
```

---

### 主函数只做一件大事：把“判定”升级成“可执行流程”

位置：`src/hooks/useCanUseTool.tsx:28`

`useCanUseTool(...)` 返回的函数并不是简单包一下 `hasPermissionsToUseTool(...)`。

它做的完整主链路是：

```text
收到 tool_use
  -> createPermissionContext(...)
  -> 先检查是否已 abort
  -> forceDecision ? 直接采用 : 调 hasPermissionsToUseTool(...)
  -> allow -> 直接放行并记录来源
  -> deny -> 记录拒绝并做用户可见通知
  -> ask -> 分流给 coordinator / swarm worker / speculative classifier / interactive dialog
  -> finally 清理 classifier checking 状态
```

所以这里最关键的理解是：

```text
permissions.ts 产出的是规则裁决；
useCanUseTool.tsx 产出的是可在真实运行时执行的审批流程。
```

---

### 第一部分：先创建 `PermissionContext`

位置：`src/hooks/useCanUseTool.tsx:33`

一上来先调用：

- `createPermissionContext(...)`
- `createPermissionQueueOps(setToolUseConfirmQueue)`

这一步很重要，因为它说明这个文件并不亲自管理所有副作用细节，而是先把本次权限事务所需的能力打包成 `ctx`。

从当前文件里能看到 `ctx` 至少承担：

- `resolveIfAborted(...)`
- `logDecision(...)`
- `buildAllow(...)`
- `cancelAndAbort(...)`

也就是说：

```text
useCanUseTool.tsx 自己是总闸门，
而 PermissionContext 是它用来执行业务副作用的事务工具箱。
```

---

### 第二部分：它怎样接 `hasPermissionsToUseTool(...)`

位置：`src/hooks/useCanUseTool.tsx:37`

这里的关键代码是：

- 有 `forceDecision` 时直接 `Promise.resolve(forceDecision)`
- 否则调用 `hasPermissionsToUseTool(tool, input, toolUseContext, assistantMessage, toolUseID)`

这说明 `forceDecision` 是一个明确的运行时覆盖口，允许上层在某些场景跳过常规计算，直接把决定灌进后续流程。

而正常路径则是：

```text
hasPermissionsToUseTool(...) = 给出决定
useCanUseTool.tsx = 消费这个决定并转成真实动作
```

所以两层关系很清楚：

```text
permissions.ts = decide
useCanUseTool.tsx = execute the decision path
```

---

### 第三部分：allow 分支不是“直接返回”这么简单

位置：`src/hooks/useCanUseTool.tsx:39`

当 `result.behavior === 'allow'` 时，这里会：

1. 再检查一次 abort
2. 如果是 auto-mode classifier 的批准，调用 `setYoloClassifierApproval(...)`
3. `ctx.logDecision({ decision: 'accept', source: 'config' })`
4. `resolve(ctx.buildAllow(...))`

这说明 allow 分支仍然要处理两类事：

- 记录来源与日志
- 把批准原因同步到 classifier/UI 状态

所以它不是裸 allow，而是：

```text
一个已经被运行时产品化的 allow 收口分支
```

---

### 第四部分���deny 分支会进入用户可见反馈链

位置：`src/hooks/useCanUseTool.tsx:64`

deny 之前，它会先算：

- `description = await tool.description(...)`

然后在 `case 'deny'` 里做：

- `logPermissionDecision(...)`
- 如果是 auto-mode classifier 拒绝：
  - `recordAutoModeDenial(...)`
  - `toolUseContext.addNotification(...)`
  - 通知里带 `/permissions`
- 最后 `resolve(result)`

这说明 deny 并不是静默失败，而是会显式进入：

- 审计日志
- auto mode 拒绝历史
- 用户通知层

最该记住的是：

```text
deny 在 useCanUseTool.tsx 这一层被“产品化”成用户可见、可追踪的拒绝事件。
```

---

### 第五部分：为什么 `ask` 分支最复杂

位置：`src/hooks/useCanUseTool.tsx:93`

`ask` 才是真正把权限系统接进运行时的核心。

这里的顺序是：

#### A. coordinator worker 优先
位置：`src/hooks/useCanUseTool.tsx:95`

如果 `awaitAutomatedChecksBeforeDialog` 为真，就先走：

- `handleCoordinatorPermission(...)`

也就是：

```text
先等自动化检查；
如果自动化已经能解决，就不要马上打断用户。
```

#### B. swarm worker 再尝试 leader 转发
位置：`src/hooks/useCanUseTool.tsx:113`

然后会走：

- `handleSwarmWorkerPermission(...)`

也就是：

```text
如果当前是 swarm worker，优先把 ask 交给 leader 审批链处理。
```

#### C. 主 agent 的 speculative classifier grace period
位置：`src/hooks/useCanUseTool.tsx:126`

如果是 Bash 且带 `pendingClassifierCheck`，这里会：

- `peekSpeculativeClassifierCheck(...)`
- 最多等 2 秒
- 命中高置信 prompt rule 时：
  - `consumeSpeculativeClassifierCheck(...)`
  - `setClassifierApproval(...)`
  - `ctx.logDecision({ source: { type: 'classifier' } })`
  - `resolve(ctx.buildAllow(...))`

这说明在真正弹出确认前，主 agent 还会给 speculative classifier 一个短暂抢跑窗口。

也就是说：

```text
ask 不一定马上弹窗；
对于 Bash，有一个“再给 classifier 2 秒试试”的运行时宽限期。
```

#### D. 最后才落到 interactive dialog
位置：`src/hooks/useCanUseTool.tsx:160`

如果前面都没解决，才调用：

- `handleInteractivePermission(...)`

并把这些东西一起传下去：

- `ctx`
- `description`
- `result`
- `awaitAutomatedChecksBeforeDialog`
- `bridgeCallbacks`
- `channelCallbacks`

所以真正的确认弹窗并不是孤立 UI，而是 ask 分支最后的兜底收口点。

---

### 第六部分：确认弹窗、用户反馈、permission updates 是怎样接进来的

从这个文件本身可以清楚看到：

- 弹窗不是在这里直接画，而是委托给 `handleInteractivePermission(...)`
- 用户反馈和 permission updates 也不是在这里手搓，而是通过 `ctx` 和下层 handler 回流
- allow / deny / ask 三种结果最后都统一 resolve 成 `PermissionDecision`

这说明 `useCanUseTool.tsx` 的职责不是具体实现每一种 UI，而是：

```text
把确认弹窗、用户反馈、permission updates、worker 转发、classifier 快路径接到同一个总入口上
```

所以它是典型 orchestration 层，不是具体实现层。

---

### 第七部分：异常和 abort 如何收口

位置：`src/hooks/useCanUseTool.tsx:171`

这里统一处理：

- `AbortError`
- `APIUserAbortError`
- 其他普通错误

对于 abort：

- `ctx.logCancelled()`
- `resolve(ctx.cancelAndAbort(undefined, true))`

对于其他错误：

- `logError(error)`
- 同样 `resolve(ctx.cancelAndAbort(undefined, true))`

最后在 `finally` 里总会：

- `clearClassifierChecking(toolUseID)`

这说明这个文件还有一个经常被忽略的职责：

```text
无论权限流程在哪条支路结束，useCanUseTool.tsx 都负责做最后的异常/取消清理收口。
```

---

### 读完这一站后，你应该抓住的 7 个事实

1. `src/hooks/useCanUseTool.tsx:28` 是 tool 调用前的权限运行时总闸门。
2. 它通过 `hasPermissionsToUseTool(...)` 接规则引擎，但自己负责把结果转成真实审批流程。
3. `forceDecision` 提供了直接覆盖常规权限计算的运行时入口。
4. `allow` / `deny` 在这里都不是裸返回，而是会进入日志、通知、classifier UI 状态等产品化收口。
5. `ask` 分支会依次尝试 coordinator 自动化、swarm worker 转发、speculative classifier，再回落 interactive dialog。
6. 确认弹窗、用户反馈、permission updates 并不在这里直接实现，而是由它统一编排进下层 handler / context。
7. 它还负责 abort / error / classifier checking 的最后清理，因此是权限系统的最终运行时收口层。

---

### 现在把权限链再串一次

```text
permissions.ts
  -> 给出 allow / deny / ask 规则裁决
useCanUseTool.tsx
  -> 把裁决接进运行时主流程，成为 tool 调用前最后一道门
PermissionContext.ts
  -> 提供日志 / queue / abort / persist 等事务工具箱
coordinatorHandler / swarmWorkerHandler / interactiveHandler
  -> 分别处理不同 ask 运行时分支
```

到这里你应该把它记成：

```text
permissions.ts = 判定规则
useCanUseTool.tsx = 运行时总闸门
```

---

### 下一站建议

下一站最顺的是：

```text
src/hooks/toolPermission/PermissionContext.ts
```

因为现在你已经看懂总闸门如何分流，下一步最自然就是看：

**这个 `ctx` 到底封装了哪些权限事务能力，以及它怎样统一处理 queue、日志、取消、permission updates 持久化。**
