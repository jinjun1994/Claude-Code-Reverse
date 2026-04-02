## 第 58 站：`src/hooks/useCanUseTool.tsx`

### 这是什么文件

`src/hooks/useCanUseTool.tsx` 是 Claude Code 工具权限链路里的 **统一 permission gating 入口与分流器**。

上一站已经看懂：

- `permissionSync.ts` 负责 swarm permission request/response 的 transport 与持久化桥接
- `useSwarmPermissionPoller.ts` 负责 worker 侧 pending callback registry 与响应恢复
- 但还差最前线那个入口：
  - 一次 tool use 到底从哪里开始做 permission check
  - 什么时候直接 allow/deny
  - 什么时候走自动检查
  - 什么时候交给 leader
  - 什么时候弹本地交互式确认框

所以这一站回答的是：

```text
一次工具调用在真正执行前，
系统怎样统一做权限判定、自动化审批、swarm worker 上抛、
以及最终的人机交互确认分流
```

所以最准确的一句话是：

```text
useCanUseTool.tsx = tool execution 前的统一权限闸门与审批路径调度器
```

---

### 先看它的整体定位

位置：`src/hooks/useCanUseTool.tsx:1`

这个文件不是：

- permission rule parser
- tool executor
- leader 审批 UI 本体
- swarm mailbox transport 层

它的职责是：

1. 接收一次具体 tool use
2. 调用底层 `hasPermissionsToUseTool(...)` 产出 permission result
3. 根据 `allow / deny / ask` 分流
4. 在 ask 场景下继续调度：
   - coordinator 自动检查
   - swarm worker 上抛 leader
   - classifier 宽限自动通过
   - 最终 interactive dialog
5. 统一处理 abort / cleanup / classifier 状态清理

所以它本质上是一个：

```text
permission orchestration entrypoint
```

或者更具体一点：

```text
tool use request
  -> permission result
  -> route to the right approval path
```

---

### 第一部分：`CanUseToolFn` 的签名说明这个 hook 面向的不是抽象“权限问题”，而是一次完整的 tool invocation 上下文

位置：`src/hooks/useCanUseTool.tsx:27`

函数签名里包含：

- `tool`
- `input`
- `toolUseContext`
- `assistantMessage`
- `toolUseID`
- `forceDecision?`

这说明权限判断并不是只看：

- tool name
- 或 permission mode

而是绑定在一次具体工具调用上。

尤其几个字段很关键：

#### `toolUseContext`
说明权限路径会读 runtime 状态、工具集合、通知接口等

#### `assistantMessage`
说明权限系统可能和当前 assistant turn / transcript 绑定

#### `toolUseID`
说明后续 classifier、UI 队列、continuation 恢复都要精确对应到这次 tool use

#### `forceDecision`
说明调用方可以注入一个“已经算好的 permission result”，绕过底层重新判定

所以这里真正处理的是：

```text
one concrete tool-use attempt with full runtime context
```

---

### 第二部分：最外层返回的是 `new Promise(resolve => ...)`，说明权限系统从这里开始就被建模成“可能异步等待审批”的决策过程，而不是同步布尔判断

位置：`src/hooks/useCanUseTool.tsx:32`

这里不是直接：

- `return boolean`
- `return PermissionResult`

而是显式 new 一个 Promise，
然后在不同分支里 `resolve(...)`。

这说明作者从入口层就接受了一个事实：

```text
permission check may suspend for UI / hooks / swarm leader / classifier work
```

也就是说 `can use tool` 的真实语义不是：

```text
现在能不能？
```

而是：

```text
最终这次 tool use 会以什么 permission decision 收束？
```

这里已经把“等待审批”纳入了主模型。

---

### 第三部分：`createPermissionContext(...)` 很关键，因为它把一次工具审批流的共享操作面统一包装成 `ctx`，避免后面各分支重复碰 AppState / queue / abort 细节

位置：`src/hooks/useCanUseTool.tsx:33`

这里先创建：

- `createPermissionContext(...)`
- `createPermissionQueueOps(setToolUseConfirmQueue)`

这说明 `useCanUseTool` 虽然是分流器，
但不想自己到处分散操作：

- queue
- permission context 更新
- log
- abort
- messageId
- allow/reject result 构造

所以作者先把这些东西压成一个共享 `ctx` 对象，
后续不同 handler 都基于同一语义面工作。

也就是说这个 hook 的组织方式不是：

```text
if/else 里各自直接碰状态
```

而是：

```text
build one permission orchestration context
-> all approval paths operate through it
```

这是后面 handler 分拆成立的基础。

---

### 第四部分：`ctx.resolveIfAborted(resolve)` 在多个阶段反复检查，说明权限流不是“一旦开始就必须走完”，而是整个审批链都支持中途取消

位置：

- 初始检查 `src/hooks/useCanUseTool.tsx:34`
- allow 后检查 `src/hooks/useCanUseTool.tsx:40`
- description 后检查 `src/hooks/useCanUseTool.tsx:61`
- automated checks 后检查 `src/hooks/useCanUseTool.tsx:110`
- classifier race 后检查 `src/hooks/useCanUseTool.tsx:132`

这一点特别关键。

作者不是只在入口判断一次 abort，
而是每跨过一个异步阶段就重查：

- 先看有无已取消
- 算完 permission result 再看
- 生成 description 再看
- 跑完 coordinator checks 再看
- classifier grace period 后再看

这说明权限链被视为：

```text
multi-stage async workflow vulnerable to stale completion
```

所以这里强调的是：

```text
never surface stale approval UI or stale decisions for an already-aborted tool use
```

这类重复 abort check 非常像运行时卫生层。

---

### 第五部分：`forceDecision ?? hasPermissionsToUseTool(...)` 说明这里把“权限规则求值”和“权限路径编排”严格分层了

位置：`src/hooks/useCanUseTool.tsx:37`

这里先得到 `decisionPromise`：

- 有 `forceDecision` 就直接用它
- 否则调用 `hasPermissionsToUseTool(...)`

这说明 `useCanUseTool` 并不自己判规则，
它只消费一个 permission result：

```text
allow / deny / ask + 附加上下文
```

也就是说模块边界很清晰：

#### `hasPermissionsToUseTool(...)`
负责规则求值、配置判断、生成 permission result

#### `useCanUseTool(...)`
负责把这个 result 变成真实运行时动作

这是一个很干净的：

```text
decision computation
vs
decision orchestration
```

分层。

---

### 第六部分：`behavior === 'allow'` 分支说明“允许”并不是简单返回 true，而是还会记录 classifier/UI 语义并构造标准 allow result

位置：`src/hooks/useCanUseTool.tsx:39`

allow 分支会做三件事：

1. 若是 transcript classifier 的 auto-mode 批准，调用 `setYoloClassifierApproval(...)`
2. `ctx.logDecision({ decision:'accept', source:'config' })`
3. `resolve(ctx.buildAllow(...))`

这说明即便是最简单的 allow，
这里也不是裸返回。

系统仍然要保留：

- 决策来源
- UI 展示信息
- classifier 解释
- 标准化的 permission decision 对象

所以 allow 在这里的语义其实是：

```text
accept + record provenance + normalize output
```

而不是布尔通过。

---

### 第七部分：进入非 allow 分支后，先用 `tool.description(...)` 生成人类可读描述，说明 deny/ask 路径的核心输入不是原始 JSON，而是对用户/leader 可展示的审批语义文本

位置：`src/hooks/useCanUseTool.tsx:55`

这里在处理 deny/ask 前，
先算：

- `const description = await tool.description(...)`

而且传进去的上下文有：

- `isNonInteractiveSession`
- `toolPermissionContext`
- `tools`

这说明 description 不是 UI 附带品，
而是审批链的重要输入：

- deny 时拿来记录 auto-mode denial display
- ask 时给 coordinator / swarm / interactive handler 使用

所以作者的设计是：

```text
permission escalation should carry a human-readable action summary,
not just raw tool input
```

这对 leader 审批和本地确认都很关键。

---

### 第八部分：`deny` 分支说明拒绝也分“普通配置拒绝”和“classifier/auto-mode 拒绝”两种展示语义，说明 deny 不只是终止，还会反馈给用户界面

位置：`src/hooks/useCanUseTool.tsx:65`

这里除了常规：

- `logPermissionDecision(... reject/config)`
- `resolve(result)`

还额外处理 auto-mode classifier deny：

- `recordAutoModeDenial(...)`
- `toolUseContext.addNotification(...)`
- UI 里显示 `denied by auto mode · /permissions`

这说明 deny 不是纯内部状态，
而是用户需要感知的一种自动决策结果。

所以这里体现的是：

```text
some automated denials deserve explicit user-facing explanation and follow-up affordance
```

也就是为什么还会带 `/permissions` 提示。

---

### 第九部分：`ask` 分支的第一个优先级是 coordinator 自动检查，说明系统在真正打断用户前，会先尝试用后台自动化通道把审批问题消化掉

位置：`src/hooks/useCanUseTool.tsx:93`

这里的条件是：

- `appState.toolPermissionContext.awaitAutomatedChecksBeforeDialog`

然后会走：

- `handleCoordinatorPermission(...)`

并传入：

- `pendingClassifierCheck`
- `updatedInput`
- `suggestions`
- `permissionMode`

这说明 ask 并不等价于“立刻弹框”。

它其实是：

```text
not auto-allowed by config,
but maybe still resolvable by automated checks before bothering the user
```

这对后台 worker / coordinator 场景尤其重要：

- 尽量不打断前台用户
- 先让自动化检查或规则建议尝试兜底

---

### 第十部分：`handleSwarmWorkerPermission(...)` 出现在 interactive dialog 之前，说明 swarm worker 的 leader 上抛在 ask 路径里优先于本地弹框，是一个正式的一等审批通道

位置：`src/hooks/useCanUseTool.tsx:113`

这里在 coordinator 检查之后，
立刻尝试：

- `handleSwarmWorkerPermission(...)`

传入：

- `ctx`
- `description`
- `pendingClassifierCheck`
- `updatedInput`
- `suggestions`

如果返回 `swarmDecision`，就直接 resolve。

这非常关键。

说明在 swarm worker 场景里，
ask-mode 的正常路线不是：

```text
worker 自己直接本地弹框
```

而是：

```text
worker 先尝试把权限请求上抛给 leader
```

也就是说 swarm leader 审批不是外围补丁，
而是 ask 路径里的正式优先分支。

这正好把前面几站的 mailbox / permissionSync / callback registry 全部接上了。

---

### 第十一部分：Bash classifier 的 speculative grace period 很有意思——它说明主 agent 在展示权限对话框前，愿意再等 2 秒给高置信 classifier 一个“免打断批准”的机会

位置：`src/hooks/useCanUseTool.tsx:126`

这里条件很细：

- feature `BASH_CLASSIFIER`
- 存在 `pendingClassifierCheck`
- `tool.name === BASH`
- 不是 `awaitAutomatedChecksBeforeDialog` 模式

然后会：

1. `peekSpeculativeClassifierCheck(command)`
2. `Promise.race([speculativePromise, timeout 2s])`
3. 若高置信命中：
   - `consumeSpeculativeClassifierCheck(command)`
   - `setClassifierApproval(toolUseID, matchedRule)`
   - `ctx.logDecision(accept/classifier)`
   - `resolve(ctx.buildAllow(...))`

这说明 ask 流程中还有一个很细的 UX 优化：

```text
before showing a blocking dialog,
wait briefly to see if the classifier can confidently auto-allow
```

也就是说系统愿意为了避免不必要弹框，
承担一个非常短的审批延迟窗口。

---

### 第十二部分：interactive permission 被放在最后兜底，说明本地确认框不是 ask 的默认第一选择，而是所有自动化/上抛通路都没解决时的最终 fallback

位置：`src/hooks/useCanUseTool.tsx:160`

最后才调用：

- `handleInteractivePermission(...)`

并把这些东西都交进去：

- `ctx`
- `description`
- `result`
- `awaitAutomatedChecksBeforeDialog`
- `bridgeCallbacks`
- `channelCallbacks`

这说明 interactive dialog 在架构里的真实定位是：

```text
last-mile human approval fallback
```

前面几条路径优先级更高：

1. config allow/deny
2. coordinator automated checks
3. swarm leader approval
4. bash classifier grace-period auto allow
5. local interactive dialog

也就是说“弹框”并不是 ask 的定义，
只是 ask 走到最后还没收束时的终局方案。

---

### 第十三部分：`bridgeCallbacks` / `channelCallbacks` 被传进 interactive handler，说明权限 UI 不只服务本地 REPL，还能桥接到其他外部交互通道

位置：`src/hooks/useCanUseTool.tsx:164`

这里根据 feature 开关取：

- `appState.replBridgePermissionCallbacks`
- `appState.channelPermissionCallbacks`

再交给 `handleInteractivePermission(...)`。

这说明 interactive permission surface 不是单一终端弹框组件，
而是一个可桥接的审批出口：

```text
local UI
or
bridge mode callbacks
or
channel-based permission callbacks
```

所以这个 hook 的职责不只是决定要不要弹框，
还包括把 ask 流路由到正确的“审批呈现通道”。

---

### 第十四部分：整个函数没有直接操作具体 UI 组件，而是全交给 handler，说明 `useCanUseTool` 的核心定位是 orchestration，不是 rendering

位置：

- `handleCoordinatorPermission(...)`
- `handleSwarmWorkerPermission(...)`
- `handleInteractivePermission(...)`

这说明作者有意识把 ask 分支拆成多个专职 handler：

#### coordinator handler
自动检查优先路径

#### swarm worker handler
leader 审批上抛路径

#### interactive handler
人机交互兜底路径

而 `useCanUseTool` 只做：

```text
given one permission result,
choose which handler pipeline should run next
```

这是典型的 orchestration 层组织方式。

---

### 第十五部分：异常处理把 `AbortError` / `APIUserAbortError` 单独识别出来，说明权限流中的“取消”被视为正常控制流，而不是普通失败

位置：`src/hooks/useCanUseTool.tsx:171`

这里区分两类：

#### Abort 类错误
- debug log
- `ctx.logCancelled()`
- `resolve(ctx.cancelAndAbort(...))`

#### 其他错误
- `logError(error)`
- 也 `resolve(ctx.cancelAndAbort(...))`

这说明至少在最终返回语义上：

- 取消
- 非预期错误

都会转换成统一的 abort/cancel decision，
避免权限流把异常继续往工具执行主链上乱抛。

但作者仍然保留了取消的专门日志语义，
说明“用户或系统主动中止审批”是一个重要且常见的合法分支。

---

### 第十六部分：`finally { clearClassifierChecking(toolUseID) }` 说明 classifier 状态被视为临时审批过程态，必须在任何结局下清理，避免污染后续工具调用

位置：`src/hooks/useCanUseTool.tsx:180`

无论最后是：

- allow
- deny
- swarm leader 解决
- interactive 解决
- abort
- exception

最终都会：

- `clearClassifierChecking(toolUseID)`

这说明 classifier checking 是严格绑定单次 toolUseID 的临时态，
不能泄漏到后面。

所以这里体现的是：

```text
classifier evaluation state must be one-shot and per-tool-use scoped
```

这和前面 callback registry、permission queue 的 one-shot 风格很一致。

---

### 第十七部分：这份文件虽然是 React hook，但本质上在实现一个多阶段权限状态机

把整体流程压缩一下，其实就是：

```text
start
  -> compute permission result
  -> allow ? accept
  -> deny ? reject
  -> ask ? automated checks
      -> coordinator resolved ? done
      -> swarm worker resolved ? done
      -> classifier grace period resolved ? done
      -> interactive approval ? done
  -> any abort/error ? cancel/abort
  -> cleanup classifier state
```

也就是说它虽然写成 hook，
但真正实现的是：

```text
tool permission orchestration state machine
```

而不是简单的 React 辅助函数。

---

### 第十八部分：整份文件真正把前面分散的权限子系统收敛成了单一入口

如果把整份文件压缩，会发现它把很多前面分散的能力统一收口到了一个入口：

#### 1. 底层 permission result 计算
- `hasPermissionsToUseTool(...)`

#### 2. orchestration context
- `createPermissionContext(...)`
- `createPermissionQueueOps(...)`

#### 3. automated approval lanes
- `handleCoordinatorPermission(...)`
- speculative bash classifier

#### 4. distributed approval lane
- `handleSwarmWorkerPermission(...)`

#### 5. human approval lane
- `handleInteractivePermission(...)`

#### 6. cleanup / cancellation
- abort handling
- classifier state cleanup

所以最准确的压缩表达是：

```text
useCanUseTool.tsx = Claude Code 工具权限系统的总入口：把规则求值结果转成自动化检查、leader 审批、交互确认或直接拒绝/允许等真实运行时动作
```

---

### 元

问题：**这一站真正想解决的架构问题是什么？**

回答：这一站的元问题，是一次工具调用在真正执行前，为什么需要一个统一权限闸门，而不是每个工具各自 ask/allow/deny。它把底层判定结果与后续 coordinator、worker、interactive 路径接成一条总入口。

### 反

问题：**如果把这一站的设计反过来，会发生什么？**

回答：如果没有统一入口，工具权限会按工具类型各自生长，模式切换、abort、自动审批和人工审批都无法保持一致。那样系统很快会出现“同一种 ask，不同工具完全不同行为”的裂缝。

### 空

问题：**跳出当前文件名，这一站背后更大的问题是什么？**

回答：更大的问题，是复杂产品怎样把权限理解为“编排问题”而不只是“规则问题”。`useCanUseTool` 的意义正在这里，它决定的是整条审批流怎么被组织。

### 读完这一站后，你应该抓住的 10 个事实

1. `useCanUseTool.tsx` 不是权限规则求值器，而是 tool use 前的统一权限编排入口，负责把 permission result 变成真实运行时动作。
2. 这个 hook 处理的是一次完整 tool invocation 上下文，而不是抽象布尔权限判断，因此签名里包括 `tool`、`input`、`toolUseContext`、`assistantMessage`、`toolUseID` 等完整上下文。
3. 入口先通过 `createPermissionContext(...)` 建立共享 orchestration context，再让后续各审批路径都通过同一语义面工作。
4. 权限链是显式异步多阶段 workflow，因此 `ctx.resolveIfAborted(resolve)` 会在多个异步边界反复检查，避免已取消请求继续弹旧对话框或提交旧结果。
5. `forceDecision ?? hasPermissionsToUseTool(...)` 表明“权限规则求值”和“权限路径调度”被严格分层：前者负责算结果，后者负责决定下一步怎么走。
6. ask 路径的优先顺序不是“直接弹框”，而是先尝试 coordinator 自动检查，再尝试 swarm worker 上抛给 leader，再尝试 classifier 宽限自动通过，最后才落到 interactive dialog。
7. `handleSwarmWorkerPermission(...)` 出现在 interactive handler 前，说明 swarm leader 审批是 ask 流中的正式一等通道，而不是附加补丁。
8. Bash classifier 的 speculative 2 秒 grace period 体现了一个很细的 UX 取舍：系统愿意短暂等待高置信自动批准，以换取更少的阻塞式权限对话框。
9. interactive permission surface 并不只服务本地 REPL，还可以通过 bridge/channel callbacks 路由到其他审批呈现通道。
10. 无论最后是允许、拒绝、leader 审批、人工确认还是取消，`finally` 都会清理 `toolUseID` 绑定的 classifier checking 状态，说明这些审批过程态都被设计成单次工具调用范围内的一次性状态。

---

### 现在把第 57-58 站串起来

```text
permissionSync.ts
  -> 定义 permission request / resolution 模型
  -> 提供旧文件同步路径与新 mailbox transport 路径
useCanUseTool.tsx
  -> 在 tool use 前统一做 permission routing
  -> 决定这次 ask 是自动解决、leader 审批还是本地交互确认
```

所以现在工具权限链可以先压缩成：

```text
tool invocation begins
  -> useCanUseTool computes/routs permission flow
  -> if swarm worker ask-mode:
       permissionSync + mailbox transport carry request/response
  -> if resolved:
       permission decision returns to tool execution path
```

也就是说：

```text
permissionSync.ts 回答“权限请求/响应在 transport 与持久化层怎样表示和传递”
useCanUseTool.tsx 回答“这些权限机制在一次真实工具调用发生时，系统到底怎样选择要走哪条审批路径”
```

---

### 下一站建议

下一站最顺的是：

```text
src/hooks/toolPermission/handlers/swarmWorkerHandler.ts
```

因为现在你已经看懂了：

- `useCanUseTool.tsx` 里 ask 路径会优先尝试 `handleSwarmWorkerPermission(...)`
- `permissionSync.ts` 提供了 swarm permission 的 transport helper
- `useSwarmPermissionPoller.ts` 负责 worker 侧等待与恢复
- 但还没看“这个 handler 自己怎样把一次 ask-mode tool use 转成 request 发送、callback 注册、leader 响应等待、以及最终 resolve”

下一步最自然就是把这条 worker 分支补齐：

**swarm worker ask-mode 权限请求在 `handleSwarmWorkerPermission(...)` 中到底怎样创建 request、注册 continuation、发给 leader，并把最终审批结果重新返回给 `useCanUseTool`。**
