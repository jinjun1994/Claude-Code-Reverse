## 第 30 站：`src/hooks/toolPermission/handlers/interactiveHandler.ts`

### 这是什么文件

`src/hooks/toolPermission/handlers/interactiveHandler.ts` 是权限系统里的**interactive ask 竞态协调器**。

上一站 `PermissionContext.ts` 解决的是事务工具箱；这一站解决的是：

```text
当 ask 最终落到交互式审批时，
本地用户、bridge、channel、hooks、classifier 到底怎样并发竞争一个最终结果
```

所以最准确的一句话是：

```text
interactiveHandler.ts = interactive ask 的多来源竞态协调器
```

---

### 文件头注释已经把本质说透了

位置：`src/hooks/toolPermission/handlers/interactiveHandler.ts:43`

注释明确说它会：

- 往 confirm queue 推一个 `ToolUseConfirm`
- 挂上 `onAbort / onAllow / onReject / recheckPermission / onUserInteraction`
- 后台异步跑 hooks 和 bash classifier
- 让它们与用户交互竞态
- 用 resolve-once guard 防止重复 resolve

这说明它并不是“弹一个框”这么简单，而是：

```text
围绕一次 ask，请把所有可能的批准/拒绝来源组织成一场竞态，
��确保最后只有一个赢家生效
```

---

### 第一部分：先建立 resolve-once 竞态保护

位置：`src/hooks/toolPermission/handlers/interactiveHandler.ts:70`

这里一开始就拿到：

- `resolveOnce`
- `isResolved`
- `claim`

再维护这些局部状态：

- `userInteracted`
- `checkmarkTransitionTimer`
- `checkmarkAbortHandler`
- `bridgeRequestId`
- `channelUnsubscribe`

这直接说明本文件的核心不是 UI 渲染，而是：

```text
处理多个异步来源同时试图结束同一权限请求的并发问题
```

参与竞争的来源至少有：

- 本地用户 allow / reject / abort
- bridge 远端响应
- channel 远端响应
- hook 抢先决定
- bash classifier 抢先自动批准
- recheck 后发现已自动 allow

所以第一结论是：

```text
interactiveHandler.ts 的第一职责是竞态仲裁，而不是界面展示。
```

---

### 第二部分：先把请求推进 permission queue

位置：`src/hooks/toolPermission/handlers/interactiveHandler.ts:92`

这里通过 `ctx.pushToQueue(...)` 推进一条 `ToolUseConfirm`。

里面带的不只是展示字段，还有完整回调：

- `onUserInteraction`
- `onDismissCheckmark`
- `onAbort`
- `onAllow`
- `onReject`
- `recheckPermission`

这说明 queue 里放的不是死数据，而是：

```text
一份可执行的审批事务对象
```

也就是说，UI 组件拿到的不是“请自己实现逻辑”的静态模型，而是已经绑定好行为的请求实体。

---

### 第三部分：`onUserInteraction()` 体现用户优先级切换

位置：`src/hooks/toolPermission/handlers/interactiveHandler.ts:108`

这个回调在用户开始操作 dialog 时触发。

它会：

- 前 200ms 内忽略，作为 grace period
- 超过 200ms 后设 `userInteracted = true`
- `clearClassifierChecking(...)`
- 清掉 classifier UI indicator

这里的设计非常细：

```text
一旦用户真的开始自己处理审批，后台 classifier 就不该再“代替用户决定”
```

同时 200ms 宽限期是为了避免刚弹窗时的误按过早取消自动批准机会。

所以这一段体现的是：

```text
interactive ask 的优先级会从“允许自动化抢跑”切换到“用户已接管，自动化退场”
```

---

### 第四部分：本地用户三条主回调

#### A. `onAbort()`
位置：`src/hooks/toolPermission/handlers/interactiveHandler.ts:137`

它会：

- `claim()` 抢占唯一结果权
- 向 bridge 回传 deny + cancel
- 取消 channel 订阅
- `ctx.logCancelled()`
- 记���条 `user_abort`
- `resolveOnce(ctx.cancelAndAbort(undefined, true))`

所以 abort 不是“关掉窗口”，而是一次正式的拒绝/中止事务。

#### B. `onAllow()`
位置：`src/hooks/toolPermission/handlers/interactiveHandler.ts:154`

它会：

- 先 `claim()`，防止 await 期间被别人抢先
- 给 bridge 回传 allow + `updatedPermissions`
- 取消 channel 订阅
- 调 `ctx.handleUserAllow(...)`
- 最终 `resolveOnce(...)`

这说明用户点 allow 后，并不是本文件自己拼返回值，而是复用 `PermissionContext` 里的标准用户批准事务。

#### C. `onReject()`
位置：`src/hooks/toolPermission/handlers/interactiveHandler.ts:183`

它会：

- `claim()`
- 给 bridge 回传 deny
- 取消 channel 订阅
- 记录 `user_reject`
- `resolveOnce(ctx.cancelAndAbort(...))`

所以 reject 同样是正式权限事务，而不是 UI 私有行为。

---

### 第五部分：`recheckPermission()` 说明弹出的审批仍然是“活”的

位置：`src/hooks/toolPermission/handlers/interactiveHandler.ts:204`

这里会重新调用：

- `hasPermissionsToUseTool(...)`

如果 fresh result 已经变成 allow，就：

- `claim()`
- cancel bridge request
- 取消 channel 订阅
- 移除 queue 项
- 记一次 `source: 'config'` 的 accept
- `resolveOnce(ctx.buildAllow(...))`

这说明 interactive dialog 弹出来之后，并不是静态等待用户点击，而是：

```text
在对话框存活期间，这个请求仍然可以根据最新权限状态被动态重新判定
```

这是非常典型的 runtime revalidation 设计。

---

### 第六部分：bridge 是正式审批通道，不是附加功能

位置：`src/hooks/toolPermission/handlers/interactiveHandler.ts:234`

如果有 `bridgeCallbacks`，这里会：

1. `sendRequest(...)` 把审批请求发到 CCR / claude.ai
2. `onResponse(...)` 监听 bridge 侧回应
3. 回应回来后与本地用户 / hook / classifier 竞争 `claim()`

如果 bridge allow：

- 持久化 `updatedPermissions`（如有）
- 记录 user accept
- `resolveOnce(ctx.buildAllow(...))`

如果 bridge deny：

- 记录 user reject
- `resolveOnce(ctx.cancelAndAbort(...))`

所以 bridge 和本地 CLI 的关系不是主从，而是：

```text
并行竞态的两个审批终端，谁先响应谁赢
```

这点非常关键。

---

### 第七部分：channel relay 是另一条远端审批副通道

位置：`src/hooks/toolPermission/handlers/interactiveHandler.ts:300`

这里会把 permission prompt 发往：

- Telegram
- iMessage
- 其他 channel relay client

流程是：

- 从 MCP clients 里筛可用 channel relay
- 组装 `ChannelPermissionRequestParams`
- 发 `CHANNEL_PERMISSION_REQUEST_METHOD` 通知
- 订阅 `channelCallbacks.onResponse(...)`
- 收到 yes/no 后继续和其他来源抢 `claim()`

这里最重要的理解是：

```text
channel relay 不是唯一审批入口，
而是在本地 dialog 之外增加的远端 yes/no 副通道
```

并且它始终以本地 UI 为兜底。

---

### 第八部分：hooks 在 interactive ask 里是后台竞态者

位置：`src/hooks/toolPermission/handlers/interactiveHandler.ts:410`

当 `awaitAutomatedChecksBeforeDialog` 为假时，hooks 不会先跑完再弹框，而是：

- dialog 先出现
- hooks 在后台异步执行
- 若 hook 先返回决定，就抢 `claim()`
- 取消 bridge / channel
- 移除 queue 项
- `resolveOnce(hookDecision)`

也就是说：

```text
interactive ask 不是“先 UI，后自动化”，
而是 UI 与 hooks 并行竞争
```

这和 coordinator 路径正好形成对照。

---

### 第九部分：bash classifier 在这里也是后台竞态者

位置：`src/hooks/toolPermission/handlers/interactiveHandler.ts:433`

满足条件时：

- `setClassifierChecking(...)`
- `executeAsyncClassifierCheck(...)`
- 传入 `shouldContinue: () => !isResolved() && !userInteracted`

这里最重要的是 `shouldContinue`：

```text
只要已有赢家，或者用户已经开始亲自操作，classifier 就必须停止抢跑
```

如果 classifier 抢先 allow：

- `claim()`
- 取消 bridge / channel
- 清理 checking 状态
- 更新 queue item 为 auto-approved checkmark
- 同步 classifier approval UI 状态
- 记录 classifier accept
- `resolveOnce(ctx.buildAllow(...))`

之后还会有一个纯展示态过渡：

- 终端聚焦时 checkmark 留 3 秒
- 非聚焦时留 1 秒
- Esc 可提前 dismiss
- abort 时也会清掉这个展示态 queue 项

这说明 interactiveHandler 不只负责逻辑正确，还负责：

```text
用户如何感知“这次 ask 被 classifier 自动批准了”
```

---

### 读完这一站后，你应该抓住的 8 个事实

1. `interactiveHandler.ts` 是 interactive ask 的多来源竞态协调器。
2. permission queue 里放的是可执行审批事务对象，而不是纯展示数据。
3. `claim()` 是整份文件最关键的同步原语，保证最终只会有一个赢家。
4. 用户一旦开始交互，classifier 应主动退场，这由 `userInteracted` + grace period 保证。
5. bridge 是正式审批终端，与本地 CLI 并���竞争，不是附加功能。
6. channel relay 是远端 yes/no 副通道，本地 UI 仍然是兜底。
7. hooks 与 classifier 在 interactive ask 中都是后台竞态者，而不是先后串行步骤。
8. 该文件还负责 classifier auto-approve 的展示过渡，因此同时协调逻辑结果与用户感知。

---

### 现在把第 28-30 站串起来

```text
useCanUseTool.tsx
  -> 作为总闸门，决定 ask 落到哪条运行时分支
PermissionContext.ts
  -> 提供日志 / persist / abort / queue / hooks / classifier 工具箱
interactiveHandler.ts
  -> 当 ask 进入 interactive 分支时，协调本地用户、bridge、channel、hooks、classifier 的并发竞态
```

所以三站分工可以压缩成：

```text
useCanUseTool.tsx = 总闸门
PermissionContext.ts = 事务工具箱
interactiveHandler.ts = interactive ask 竞态协调器
```

---

### 下一站建议

下一站最顺的是：

```text
src/hooks/toolPermission/handlers/coordinatorHandler.ts
```

因为现在你已经看懂主 agent 的 interactive ask，下一步最自然就是对照看：

**coordinator worker 为什么选择“先自动化、后打扰用户”，以及它和 interactive 分支在调度策略上的本质区别。**
