## 第 34 站：`src/utils/swarm/permissionSync.ts`

### 这是什么文件

`src/utils/swarm/permissionSync.ts` 是 swarm 权限系统里的 **协议层 + 存储层 + 传输层总线**。

前两站已经看懂：

```text
swarmWorkerHandler.ts
  -> worker 侧发起权限请求并等待 leader
useSwarmPermissionPoller.ts
  -> worker 侧接收 leader 回包并恢复原始 ask promise
```

而这一站补的是它们共同依赖的底层基础设施：

```text
permission request / response 到底长什么样
它们存在哪里
怎样发送
怎样轮询
怎样清理
以及 mailbox 方案是如何替代旧文件方案的
```

所以最准确的一句话是：

```text
permissionSync.ts = swarm 权限请求/响应的协议与同步基础设施
```

---

### 先看它解决的系统问题

位置：`src/utils/swarm/permissionSync.ts:1`

文件头注释已经把整个闭环说透了：

1. worker 遇到 permission prompt
2. worker 发 `permission_request` 到 leader mailbox
3. leader 检测并展示请求
4. 用户在 leader UI 上批准/拒绝
5. leader 发 `permission_response` 回 worker mailbox
6. worker 收到响应后继续执行

这说明这个文件解决的不是“权限规则本身”，而是：

```text
在多 agent / 分布式执行环境下，
一次 ask 如何跨执行体传递，并最终回到原发起 worker 继续跑
```

所以它在架构中的定位不是业务 handler，而是：

```text
跨 agent permission transport layer
```

---

### 第一部分：`SwarmPermissionRequestSchema` 定义了跨 agent ask 的标准载荷

位置：`src/utils/swarm/permissionSync.ts:46`

这里定义的请求结构包含：

- `id`
- `workerId`
- `workerName`
- `workerColor`
- `teamName`
- `toolName`
- `toolUseId`
- `description`
- `input`
- `permissionSuggestions`
- `status`
- `resolvedBy`
- `resolvedAt`
- `feedback`
- `updatedInput`
- `permissionUpdates`
- `createdAt`

这说明 swarm 权限请求并不只是“给 leader 发一句请审批”，而是一个完整的、可落盘、可恢复、可审计的事务对象。

最重要的是它同时包含三类信息：

#### 1. 路由信息
- 谁发的：`workerId / workerName / teamName`
- 对应哪次 tool_use：`toolUseId`

#### 2. 审批内容
- `toolName`
- `description`
- `input`
- `permissionSuggestions`

#### 3. 解决结果
- `status`
- `resolvedBy`
- `feedback`
- `updatedInput`
- `permissionUpdates`

所以正确理解是：

```text
SwarmPermissionRequest 既是“请求信封”，也是“请求生命周期记录”
```

---

### 第二部分：它保留了旧的文件系统同步模型，而且结构非常清晰

位置：`src/utils/swarm/permissionSync.ts:108`

这里定义了几层目录：

- `getPermissionDir(teamName)` -> `~/.claude/teams/{team}/permissions`
- `getPendingDir(teamName)` -> `permissions/pending`
- `getResolvedDir(teamName)` -> `permissions/resolved`

并通过 `ensurePermissionDirsAsync(...)` 保证目录存在。

这说明 swarm 权限系统最初/底层的模型是：

```text
pending/ 里放待审批请求
resolved/ 里放已解决请求
worker/leader 通过共享 team 目录做异步同步
```

所以它不是单纯内存队列，而是显式持久化的跨进程 IPC 结构。

这也解释了为什么后面会有：

- 文件锁
- 目录扫描
- old resolution cleanup

这些明显偏“文件协议”的机制。

---

### 第三部分：`createPermissionRequest(...)` 把当前 agent 身份自动注入协议对象

位置：`src/utils/swarm/permissionSync.ts:164`

这里创建请求时会自动补：

- `teamName || getTeamName()`
- `workerId || getAgentId()`
- `workerName || getAgentName()`
- `workerColor || getTeammateColor()`

如果关键字段不存在会直接抛错。

这说明请求对象不是随便拼个 JSON，而是强依赖当前 swarm 身份上下文。

也就是说：

```text
permission request 在协议层就绑定了“哪个 team 的哪个 worker 发起了这次 ask”
```

所以这里是 swarm 身份与 permission runtime 发生正式耦合的入口之一。

---

### 第四部分：旧文件路径里的 `writePermissionRequest(...)` / `resolvePermission(...)` 是一个小型状态机

位置：

- `writePermissionRequest(...)` `src/utils/swarm/permissionSync.ts:209`
- `resolvePermission(...)` `src/utils/swarm/permissionSync.ts:354`

这两段合起来其实就是一个文件态状态机：

```text
create request
  -> write to pending/
  -> leader resolves
  -> write resolved version to resolved/
  -> remove from pending/
```

其中 `resolvePermission(...)` 会：

1. 读 pending request
2. 校验 schema
3. 合并 resolution data
4. 写到 resolved/
5. 删掉 pending/

所以它实现的本质是：

```text
pending -> approved/rejected
```

的显式迁移，而不是“原地改个状态字段”。

这很重要，因为它把：

- 待处理请求
- 已处理结果

拆成了两个物理区域，读取语义也更简单：

- leader 只看 pending/
- worker 只查 resolved/

---

### 第五部分：文件锁说明它明确在处理多进程竞争写入

位置：

- `writePermissionRequest(...)` `src/utils/swarm/permissionSync.ts:223`
- `resolvePermission(...)` `src/utils/swarm/permissionSync.ts:375`

这两处都会：

- 在 pending 目录下创建 `.lock`
- `lockfile.lock(lockFilePath)`
- 在锁保护下执行写入/迁移
- finally release

这说明作者非常清楚这里是跨进程共享目录，不能假设只有一个 writer。

所以这里防的是：

- 多 worker 同时提交 request
- leader 与其他流程同时 resolve
- 文件读写时的中间态可见问题

因此这个模块不只是“把 JSON 写盘”，而是在做：

```text
基于共享目录的并发安全权限同步
```

---

### 第六部分：读取路径普遍走 schema 校验，说明磁盘和 mailbox 都被当成不可信边界

位置：

- `readPendingPermissions(...)` `src/utils/swarm/permissionSync.ts:256`
- `readResolvedPermission(...)` `src/utils/swarm/permissionSync.ts:320`

这里读文件时都不是直接 `JSON.parse` 后信任，而是：

- `SwarmPermissionRequestSchema().safeParse(...)`
- 失败则 debug log 并丢弃

这和上一站 `parsePermissionUpdates(...)` 的思路完全一致：

```text
只要数据跨了进程/磁盘边界，就必须重新校验
```

所以 swarm 权限同步虽然是“团队内部通信”，但在代码语义上仍被当成外部输入源对待。

这是一种很稳的 defensive design。

---

### 第七部分：`readPendingPermissions(...)` / `readResolvedPermission(...)` 透露了 leader 与 worker 的视角差异

位置：

- `readPendingPermissions(...)` `src/utils/swarm/permissionSync.ts:252`
- `readResolvedPermission(...)` `src/utils/swarm/permissionSync.ts:314`

这两个读取函数实际上对应两种角色：

#### leader 视角
- 读所有 pending requests
- 按 `createdAt` 排序（oldest first）
- 用于决定当前有哪些 worker ask 需要处理

#### worker 视角
- 只按 `requestId` 查自己的 resolved request
- 查不到就说明还没批完

这说明协议层在设计上就按职责切开了：

```text
leader = request inbox consumer
worker = specific response waiter
```

这和上一站的 callback registry 正好拼起来。

---

### 第八部分：`PermissionResponse` / `pollForResponse(...)` 明确暴露了新旧两套模型的兼容层

位置：`src/utils/swarm/permissionSync.ts:519`

这里有一个非常关键的注释：

```text
Legacy response type for worker polling
Used for backward compatibility with worker integration code
```

然后：

- `pollForResponse(...)` 读取 resolved request
- 再把它转换成简化版 `PermissionResponse`
- `removeWorkerResponse(...)` 只是 `deleteResolvedPermission(...)` 的别名
- `submitPermissionRequest` 只是 `writePermissionRequest` 的别名

这说明当前模块处在一个明显的演进中间态：

```text
新模型：更完整的 SwarmPermissionRequest + mailbox system
旧接口：worker polling 仍然期待简化 response API
```

所以这里承担的是适配器角色：

```text
把新的 resolved request 数据模型适配成旧 worker integration 还能理解的格式
```

这也是为什么上一站 `useSwarmPermissionPoller.ts` 同时支持 mailbox response dispatcher 和 `pollForResponse(...)` 两条路径。

---

### 第九部分：`isTeamLeader()` / `isSwarmWorker()` 给权限路由提供了角色判定原语

位置：`src/utils/swarm/permissionSync.ts:578`

这里的角色判断逻辑很直接：

- 没有 `agentId`，或者 `agentId === 'team-lead'` -> leader
- 有 `teamName` + 有 `agentId` + 不是 leader -> swarm worker

这说明 swarm 权限链路并不是从 React/handler 层随便猜角色，而是协议层就提供了统一判定原语。

所以：

- `swarmWorkerHandler.ts` 判断要不要走 leader forwarding
- `useSwarmPermissionPoller.ts` 判断要不要轮询

背后都依赖这里的身份定义。

这保证了不同层对“谁是 worker、谁是 leader”使用同一标准。

---

### 第十部分：`getLeaderName(...)` 揭示了 mailbox 路由依赖 team file，而不是硬编码收件人

位置：`src/utils/swarm/permissionSync.ts:647`

这里会：

- `readTeamFileAsync(team)`
- 找到 `leadAgentId`
- 再在 members 里找到对应名字
- 找不到则回退 `'team-lead'`

这说明 leader 路由不是写死的，而是：

```text
通过 team metadata 动态解析当前 leader 的 mailbox 名字
```

这很关键，因为 worker 发权限请求时真正能写 mailbox 的通常是“名字/收件地址”，而不是抽象 agentId。

所以这个函数本质上是在做：

```text
team topology -> mailbox recipient resolution
```

---

### 第十一部分：`sendPermissionRequestViaMailbox(...)` / `sendPermissionResponseViaMailbox(...)` 标志着 mailbox 已成为主传输通道

位置：

- `sendPermissionRequestViaMailbox(...)` `src/utils/swarm/permissionSync.ts:669`
- `sendPermissionResponseViaMailbox(...)` `src/utils/swarm/permissionSync.ts:724`

两段注释都写得很明白：

```text
This is the new mailbox-based approach that replaces the file-based ... directory.
```

也就是说，文件系统 pending/resolved 模型仍然保留，但架构方向已经明确迁移到 mailbox。

这里的关键动作是：

#### worker -> leader
- `createPermissionRequestMessage(...)`
- `writeToMailbox(leaderName, ...)`

#### leader -> worker
- `createPermissionResponseMessage(...)`
- `writeToMailbox(workerName, ...)`

而 `writeToMailbox(...)` 的注释提示更关键：

```text
routes to in-process or file-based based on recipient
```

这说明 mailbox 本身又是一个更高层抽象：

```text
调用方只说“发给谁”，
底层决定是进程内传递还是文件型 mailbox
```

因此 `permissionSync.ts` 在新架构里的位置更像：

```text
permission-specific message adapter on top of generic teammate mailbox
```

---

### 第十二部分：sandbox permission 在这里复用了同一条 leader-mailbox 审批总线

位置：`src/utils/swarm/permissionSync.ts:785`

后半段还有一整套 sandbox API：

- `generateSandboxRequestId()`
- `sendSandboxPermissionRequestViaMailbox(...)`
- `sendSandboxPermissionResponseViaMailbox(...)`

它的模式和 tool permission 几乎完全平行：

```text
worker 需要网络 host 权限
  -> 发 sandbox permission request 给 leader
leader 决定 allow / deny
  -> 发 sandbox permission response 回 worker
```

这说明 leader 审批总线不是只为 tool_use ask 设计的，而是已经被抽象成：

```text
swarm 场景下的通用远端审批运输层
```

tool permission 和 sandbox network permission 只是两个具体协议实例。

---

### 第十三部分：这个文件真正体现的是“从共享目录同步到 mailbox 消息总线”的架构迁移

如果把整份文件纵向看，会发现一个很明显的演进轨迹：

#### 旧模型
```text
permissions/pending/*.json
permissions/resolved/*.json
worker/leader 通过共享 team 目录同步
```

#### 兼容层
```text
pollForResponse()
removeWorkerResponse()
submitPermissionRequest()
legacy response type
```

#### 新模型
```text
createPermissionRequestMessage()
createPermissionResponseMessage()
writeToMailbox(...)
leaderName resolution
in-process or file-based mailbox routing
```

所以这份文件最值得记住的不只是“它做了什么”，而是：

```text
它正站在 swarm 权限系统从 file-based sync 迁移到 mailbox-based messaging 的过渡中心
```

这也解释了为什么前面的 handler/poller 看起来会同时带一点旧接口和新接口痕迹。

---

### 读完这一站后，你应该抓住的 9 个事实

1. `permissionSync.ts` 是 swarm 权限请求/响应的协议与同步基础设施，而不是具体 UI handler。
2. `SwarmPermissionRequestSchema` 定义了完整的跨 agent 权限事务对象，既描述请求，也能承载解决结果。
3. 旧模型基于 `permissions/pending` 与 `permissions/resolved` 两级目录，形成显式的 pending -> resolved 状态迁移。
4. 文件锁说明这个同步层明确在处理多进程共享目录下的并发问题。
5. 无论读 pending 还是 resolved，都会重新跑 schema 校验，说明跨进程/磁盘数据被视为不可信边界。
6. `PermissionResponse`、`pollForResponse()`、`removeWorkerResponse()` 等接口是新旧模型之间的兼容适配层。
7. `isTeamLeader()` / `isSwarmWorker()` 为更上层 handler 和 poller 提供统一角色判定原语。
8. mailbox API 已被注释明确标为“new approach”，说明 swarm 权限传输主方向已迁移到 teammate mailbox。
9. sandbox permission 在同一文件内复用了这条 leader 审批总线，说明该模块已经是通用远端审批传输层，而不只是 tool ask 专用代码。

---

### 现在把第 32-34 站串起来

```text
swarmWorkerHandler.ts
  -> worker 侧创建 ask、注册 callback、上送 leader
useSwarmPermissionPoller.ts
  -> worker 侧接收 leader 回包、按 requestId 找回 continuation
permissionSync.ts
  -> 定义 request/response 协议、角色判定、文件兼容层、mailbox 传输层
```

所以 swarm 权限链路现在已经可以完整压成：

```text
worker ask
  -> createPermissionRequest
  -> sendPermissionRequestViaMailbox
  -> leader resolves
  -> sendPermissionResponseViaMailbox
  -> worker poller/dispatcher
  -> callback resume
  -> PermissionContext 收口
```

---

### 下一站建议

下一站最顺的是：

```text
src/utils/teammateMailbox.ts
```

因为现在你已经看懂了：

- permissionSync 怎样把 swarm 权限请求适配成 mailbox message
- worker / leader 怎样通过 mailbox 完成 ask 往返

下一步最自然就是继续下钻到更底层的通用基础设施：

**`writeToMailbox(...)`、消息格式、收件路由、进程内/文件型投递分流，到底是怎样在 `teammateMailbox.ts` 里实现的。**
