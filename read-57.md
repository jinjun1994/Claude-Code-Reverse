## 第 57 站：`src/utils/swarm/permissionSync.ts`

### 这是什么文件

`src/utils/swarm/permissionSync.ts` 是 swarm 权限体系里的 **permission request/response transport 与状态持久化桥接层**。

上一站 `useSwarmPermissionPoller.ts` 已经看懂：

- worker 发出权限请求后会注册 pending callback
- poller 会按 `requestId` 去找 response
- 收到 response 后恢复挂起的工具调用 continuation
- 但“response 到底存在哪、历史上怎么落盘、现在 mailbox 路径又怎么接管”还没补上

这一站回答的正是：

```text
swarm permission request / response 在 transport 层到底怎样表示、
怎样落盘与读取、怎样从旧的文件目录方案迁移到 mailbox 方案，
以及 worker/leader 身份判断怎样和这些路径绑定
```

所以最准确的一句话是：

```text
permissionSync.ts = swarm 权限同步层：把权限请求/响应从运行时 continuation 对接到文件持久化与 mailbox transport
```

---

### 先看它的整体定位

位置：`src/utils/swarm/permissionSync.ts:1`

这个文件不是：

- leader 审批 UI 本体
- worker callback registry
- mailbox unread router
- tool permission policy 决策器

它的职责是：

1. 定义 swarm permission request/resolution 的持久化数据结构
2. 管理基于 team 目录的 pending/resolved 文件布局
3. 提供 worker / leader 读写这些状态的 helper
4. 提供新的 mailbox-based permission request/response 发送入口
5. 统一 sandbox permission 的 mailbox 发送逻辑

所以它其实是一个双栈桥接层：

```text
legacy file-based permission sync
+
new mailbox-based permission sync
```

---

### 第一部分：文件头注释已经把这层定义成“跨 agent 协调 permission prompt 的基础设施”，而不是某个具体 UI 的私有实现

位置：`src/utils/swarm/permissionSync.ts:1`

头部注释强调：

- worker 遇到 permission prompt
- 把 request 转发给 leader
- leader 批准/拒绝
- worker 再继续执行

而且注释里直接写了 message passing：

- worker -> leader mailbox
- leader -> worker mailbox

这说明作者从概念上已经把它定义成：

```text
cross-agent permission coordination infrastructure
```

而不是“给某个 hook 用的几个工具函数”。

它真正关心的是 swarm 里两端如何同步权限决策。

---

### 第二部分：`SwarmPermissionRequestSchema` 很关键，因为它把权限请求建模成一个完整生命周期对象，而不只是瞬时 prompt

位置：`src/utils/swarm/permissionSync.ts:49`

这个 schema 里既有：

#### 请求侧字段
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
- `createdAt`

又有：

#### 解决侧字段
- `status: pending | approved | rejected`
- `resolvedBy`
- `resolvedAt`
- `feedback`
- `updatedInput`
- `permissionUpdates`

这说明作者不是把 request 和 response 分成两个完全无关的对象，
而是把它们合在一个生命周期记录里：

```text
pending request record
  -> later enriched into resolved request record
```

也就是说这层的核心模型是：

```text
one permission request evolves through states
```

而不是消息收发日志。

---

### 第三部分：`PermissionResolution` 单独拆出来，说明“如何解决请求”被当成一个可复用补丁对象，而不是必须重新构造整条 request 记录

位置：`src/utils/swarm/permissionSync.ts:95`

`PermissionResolution` 只包含：

- `decision`
- `resolvedBy`
- `feedback?`
- `updatedInput?`
- `permissionUpdates?`

这说明 resolve 动作的语义是：

```text
take existing pending request
+ apply resolution patch
= resolved request
```

这在 `resolvePermission(...)` 里也完全体现出来了：

- 先读 pending request
- 再把 resolution 字段 merge 回去
- 得到 resolved request

所以 resolution 在架构上更像：

```text
state transition payload
```

而不是完整实体。

---

### 第四部分：`permissions/pending` 与 `permissions/resolved` 双目录结构说明旧方案把权限同步建模成文件状态迁移，而不是单文件状态位更新

位置：

- `getPermissionDir()` `src/utils/swarm/permissionSync.ts:112`
- `getPendingDir()` `src/utils/swarm/permissionSync.ts:119`
- `getResolvedDir()` `src/utils/swarm/permissionSync.ts:126`

目录结构是：

```text
~/.claude/teams/{team}/permissions/
  pending/
  resolved/
```

这说明旧的文件同步方案采用的是：

```text
directory-based lifecycle staging
```

而不是：

- 一个 JSON 文件里改 `status`
- 一个 append-only log
- 数据库式索引

它的状态机物理化成了目录迁移：

```text
pending dir -> resolved dir
```

这样做的好处是：

- worker 轮询 resolved 很直接
- leader 看 pending 很直接
- 生命周期边界清楚

也和前面的 inbox/mailbox 文件式设计风格一致。

---

### 第五部分：`ensurePermissionDirsAsync()` 表明权限同步目录是 lazy materialization 的协作资源，不是启动时强制预建的基础设施

位置：`src/utils/swarm/permissionSync.ts:133`

这里在真正读写前才：

- 建 `permissions/`
- 建 `pending/`
- 建 `resolved/`

这说明权限同步目录和 mailbox 文件一样，
被当作：

```text
on-demand collaboration state
```

而不是系统启动硬依赖。

也就是说：

- 没有 swarm permission activity 时，这些目录可以不存在
- 第一次真正发 request 时再 materialize

这种设计和 swarm 整体的文件型协作模型非常一致。

---

### 第六部分：`generateRequestId()` / `createPermissionRequest()` 说明权限请求的 sender identity、team 路由与 tool context 在这里被统一收口成标准化请求对象

位置：

- `generateRequestId()` `src/utils/swarm/permissionSync.ts:160`
- `createPermissionRequest()` `src/utils/swarm/permissionSync.ts:167`

`createPermissionRequest(...)` 会优先从上下文补齐：

- `teamName || getTeamName()`
- `workerId || getAgentId()`
- `workerName || getAgentName()`
- `workerColor || getTeammateColor()`

缺任何关键身份字段就直接抛错。

这说明 permission request 不是随便一段 payload，
而是一个必须绑定在稳定 worker 身份上的协议对象：

```text
tool permission request = tool context + worker identity + team routing
```

而不是简单“我想用个工具”。

这也解释了为什么后续 leader 侧 UI 能展示：

- 谁发起的
- 什么颜色
- 对应哪个 tool use

---

### 第七部分：`writePermissionRequest()` 用目录级 lock，而不是对单个 request 文件加锁，说明作者主要在保护 pending 目录这一协作命名空间的一致性

位置：`src/utils/swarm/permissionSync.ts:215`

这里流程是：

1. `ensurePermissionDirsAsync(...)`
2. 计算 `pendingPath`
3. 在 pending 目录下写 `.lock`
4. 对 `.lock` 加锁
5. 再写 `${request.id}.json`

这说明锁的核心语义不是“某一条 request 的后续更新”，
而是：

```text
serialize concurrent writes into the pending request namespace
```

因为这里的真实并发风险是：

- 多 worker 同时请求权限
- 多个 request 文件要在同一目录下 materialize

作者选择对目录级命名空间串行化，
而不是给每个 request 单独上更细粒度锁。

---

### 第八部分：`readPendingPermissions()` 体现的是 leader 视角的“待处理工作队列”，但底层仍是目录扫描 + schema 校验，而不是中央内存队列

位置：`src/utils/swarm/permissionSync.ts:256`

这里会：

- `readdir(pendingDir)`
- 过滤 `.json`
- 每个文件 `readFile + safeParse`
- 无效文件记 debug 然后丢弃
- 最后按 `createdAt` 排序

这说明旧方案中 leader 看到的 pending permissions，
本质上不是实时内存队列，
而是：

```text
filesystem-backed work queue reconstructed by directory scan
```

而且 `createdAt` oldest-first 排序说明这层在语义上是把它当作：

```text
attention backlog ordered by age
```

这也很符合“人工审批”的用户感知。

---

### 第九部分：`readResolvedPermission()` 与 `pollForResponse()` 组合说明 worker 侧旧恢复路径的核心语义不是“订阅消息”，而是“按 requestId 查 resolved artifact 是否出现”

位置：

- `readResolvedPermission()` `src/utils/swarm/permissionSync.ts:320`
- `pollForResponse()` `src/utils/swarm/permissionSync.ts:544`

这两个函数连起来其实非常清楚：

#### resolved 层
在 `resolved/` 目录里按 requestId 找完整 resolved request 文件

#### worker poll convenience 层
把 resolved request 再压缩成旧的 `PermissionResponse`

所以旧的 worker 恢复模型是：

```text
for this requestId,
did a resolved file appear yet?
```

而不是：

- 扫所有 response 消息
- 订阅一个事件流
- 等某个 centralized queue push

这和上一站 `useSwarmPermissionPoller.ts` 的 request-centric polling 完全对上。

---

### 第十部分：`resolvePermission()` 不是就地改 pending 文件，而是“读 pending -> 写 resolved -> 删 pending”，说明它把状态迁移建模成显式阶段转换

位置：`src/utils/swarm/permissionSync.ts:360`

流程非常标准：

1. 读 `pending/{requestId}.json`
2. schema 校验
3. merge resolution 字段
4. 写 `resolved/{requestId}.json`
5. 删除 `pending/{requestId}.json`

这不是：

- 在原文件里改 `status`
- 用 rename 直接搬文件
- 或者把 response 单独放另一文件

而是明确把它实现成：

```text
pending state snapshot
  -> resolved state snapshot
  -> remove old stage artifact
```

也就是说作者要的不是最省操作，
而是最清晰的生命周期边界。

---

### 第十一部分：`cleanupOldResolutions()` 说明 resolved 目录被视为短期恢复缓冲，而不是长期审计日志

位置：`src/utils/swarm/permissionSync.ts:452`

这里默认：

- `maxAgeMs = 3600000`（1 小时）

并按 `resolvedAt || createdAt` 判断清理时机。

这说明 resolved request 文件的定位不是：

```text
permanent permission history archive
```

而是：

```text
temporary handoff artifact so workers can notice and consume a resolution
```

worker 一旦恢复完成，
这些文件就没有长期保留价值了。

这和 `removeWorkerResponse(...)` / `deleteResolvedPermission(...)` 的设计完全一致。

---

### 第十二部分：`PermissionResponse` 被明确标注为 legacy，说明 worker polling API 需要维持兼容，但系统概念模型其实已经升级成 `SwarmPermissionRequest + resolution`

位置：`src/utils/swarm/permissionSync.ts:520`

注释直接写：

```text
Legacy response type for worker polling
Used for backward compatibility with worker integration code
```

这很关键。

说明目前系统内部同时存在两层表示：

#### 新的核心表示
- `SwarmPermissionRequest`
- `PermissionResolution`

#### 旧的 worker 兼容表示
- `PermissionResponse`

而 `pollForResponse()` 的工作就是把新模型压扁回旧模型。

所以这份文件其实是一个迁移层，
它在保护：

```text
new lifecycle model internally
while keeping old worker-side polling contracts alive
```

---

### 第十三部分：`isTeamLeader()` / `isSwarmWorker()` 把权限同步角色判断收口在这里，说明 transport 层也需要知道自己服务的是 leader 还是 worker

位置：

- `isTeamLeader()` `src/utils/swarm/permissionSync.ts:581`
- `isSwarmWorker()` `src/utils/swarm/permissionSync.ts:596`

这里的判断规则是：

#### leader
- 没有 `agentId`
- 或 `agentId === 'team-lead'`

#### worker
- 有 `teamName`
- 有 `agentId`
- 且不是 leader

这说明角色身份不是只给 UI 用的，
transport/helper 层本身也依赖它来决定：

- 谁该发请求
- 谁该等响应
- 谁该轮询

也就是说 permission sync 不是纯文件/消息工具层，
它带有明确的 swarm role awareness。

---

### 第十四部分：`deleteResolvedPermission()` / `removeWorkerResponse()` 说明 worker 消费 response 后的清理语义是显式删除 resolved artifact，而不是标记 consumed

位置：

- `deleteResolvedPermission()` `src/utils/swarm/permissionSync.ts:607`
- `removeWorkerResponse()` `src/utils/swarm/permissionSync.ts:570`

这里的设计很直接：

- `removeWorkerResponse()` 只是旧名 alias
- 真正做的是 `unlink(resolvedPath)`

这和 mailbox 的 `read: boolean` 语义不同。

也就是说旧的 resolved 文件路径不是 mailbox，
而是：

```text
single-use response artifact
```

worker 一旦处理完，
这个 artifact 就应该物理删除。

所以这里和 mailbox 的差别很鲜明：

- mailbox：持久消息 + read state
- resolved permission file：短暂响应工件 + consume then delete

---

### 第十五部分：`sendPermissionRequestViaMailbox()` 说明 mailbox 新方案不是另起一套请求对象，而是把 `SwarmPermissionRequest` 再映射成 teammateMailbox 的 structured protocol message

位置：`src/utils/swarm/permissionSync.ts:676`

这里做了几件关键事：

1. `getLeaderName(request.teamName)`
2. `createPermissionRequestMessage(...)`
3. `writeToMailbox(leaderName, ...)`

也就是说 mailbox 新方案没有推翻现有请求模型，
而是：

```text
SwarmPermissionRequest
  -> createPermissionRequestMessage payload
  -> teammate mailbox transport
```

这非常重要。

说明 `permissionSync.ts` 真正做的是：

```text
domain model stays stable,
transport changes from file directories to mailbox messages
```

这是一个很成熟的迁移分层方式。

---

### 第十六部分：`getLeaderName()` 必须读 team file 才能完成 mailbox 路由，说明新 transport 的关键前提是“leader 不只是协议角色，还必须有真实 member name”

位置：`src/utils/swarm/permissionSync.ts:651`

这里流程是：

- 读 team file
- 根据 `leadAgentId` 找成员
- 返回 leader member name
- 找不到才 fallback `'team-lead'`

这说明 mailbox 路由的关键不是抽象角色，
而是：

```text
which specific recipient inbox should receive the message?
```

所以相较于旧文件方案“直接用 teamName + requestId 目录路径”，
新 mailbox 方案多了一步：

```text
resolve leader identity to inbox recipient name
```

这也是 mailbox recipient-centric 模型带来的要求。

---

### 第十七部分：`sendPermissionResponseViaMailbox()` 把 leader 决策重新编码成 `permission_response` structured message，说明 mailbox 替代的不只是 request 投递，也包括 response handoff

位置：`src/utils/swarm/permissionSync.ts:734`

这里会：

- 把 `PermissionResolution` 转成 `createPermissionResponseMessage(...)`
- `approved` -> `subtype: 'success'`
- `rejected` -> `subtype: 'error'`
- 带上 `updated_input` / `permission_updates`
- 再写到 worker inbox

这说明 mailbox 化不是单边的。

它替换的是整条旧链：

```text
pending dir + resolved dir
```

变成：

```text
request mailbox message + response mailbox message
```

也就是 request 和 response 两边都 transport-swapped 了。

---

### 第十八部分：`sendSandboxPermissionRequest/ResponseViaMailbox()` 说明 sandbox permission 从一开始就只走 mailbox 路径，没有再复用旧的 pending/resolved 文件状态机

位置：

- request `src/utils/swarm/permissionSync.ts:805`
- response `src/utils/swarm/permissionSync.ts:882`

这里 sandbox permission 只有 mailbox 发送 helper：

- `createSandboxPermissionRequestMessage(...)`
- `createSandboxPermissionResponseMessage(...)`
- `writeToMailbox(...)`

没有看到：

- `writeSandboxPermissionRequest()` 到 pending 目录
- `resolveSandboxPermission()` 到 resolved 目录

这说明 sandbox permission 的 transport 架构更“新”：

```text
sandbox permission was designed directly around mailbox transport,
not migrated from the old file-based lifecycle directories
```

这也解释了上一站 sandbox callback registry 为什么更窄、更像 promise gate。

---

### 第十九部分：这份文件最重要的架构意义，不是某个 helper，而是它清楚暴露了 swarm permission system 正处在“旧文件同步模型 -> 新 mailbox 模型”的过渡层

如果把整份文件压缩，会发现它同时保留了两套世界：

#### 旧世界
- `permissions/pending`
- `permissions/resolved`
- `writePermissionRequest`
- `readPendingPermissions`
- `resolvePermission`
- `pollForResponse`
- `deleteResolvedPermission`
- legacy `PermissionResponse`

#### 新世界
- `sendPermissionRequestViaMailbox`
- `sendPermissionResponseViaMailbox`
- `sendSandboxPermissionRequestViaMailbox`
- `sendSandboxPermissionResponseViaMailbox`
- leader/worker inbox routing

所以最准确的压缩表达应该是：

```text
permissionSync.ts = swarm 权限同步的迁移桥：旧的文件阶段目录模型仍被保留给 worker polling 兼容层，而新的 leader<->worker 协调主路径已转向 teammate mailbox structured messages
```

---

### 读完这一站后，你应该抓住的 10 个事实

1. `permissionSync.ts` 是 swarm 权限同步层，负责在 worker continuation、文件持久化状态、mailbox transport 之间做桥接，而不是直接做审批 UI 或 permission policy。
2. `SwarmPermissionRequestSchema` 把权限请求建模成一个完整生命周期对象，既包含发起时的 worker/tool 上下文，也包含后续 resolution 字段。
3. 旧的文件同步方案使用 `permissions/pending/` 与 `permissions/resolved/` 双目录，体现的是目录级生命周期阶段迁移，而不是单文件状态位更新。
4. `writePermissionRequest()` 使用 pending 目录级 `.lock` 来串行化 request 文件写入，保护的是权限请求命名空间的一致性。
5. `readPendingPermissions()` 通过目录扫描 + schema 校验 + oldest-first 排序，把文件系统重建成 leader 侧的待处理审批队列。
6. `resolvePermission()` 的语义是“读 pending -> merge resolution -> 写 resolved -> 删除 pending”，说明 resolved artifact 只是 worker 恢复执行前的临时交付物。
7. `PermissionResponse` 与 `pollForResponse()` 被标为 legacy/compatibility，表明旧 worker polling 合同还在，但系统概念模型其实已经提升到 `SwarmPermissionRequest + PermissionResolution`。
8. `sendPermissionRequestViaMailbox()` / `sendPermissionResponseViaMailbox()` 显示新的主路径已经转向 teammate mailbox structured protocol message，而不是继续依赖 `pending/` 与 `resolved/` 目录。
9. `getLeaderName()` 通过 team file 把 `leadAgentId` 解析成真实 leader member name，说明 mailbox 路由的前提是 recipient-centric inbox identity。
10. sandbox permission 没有再复用旧的文件阶段目录模型，而是直接提供 mailbox request/response helper，说明这条权限流从设计上更偏新 transport 路线。

---

### 现在把第 56-57 站串起来

```text
useSwarmPermissionPoller.ts
  -> 维护 worker 侧 pending callbacks
  -> 收到响应后恢复被挂起的工具调用 continuation
permissionSync.ts
  -> 定义 permission request/resolution 数据模型
  -> 提供旧文件同步路径与新 mailbox transport 路径
```

所以现在 swarm 权限握手链可以先压缩成：

```text
worker tool call hits ask-mode
  -> create/register pending continuation
  -> permissionSync builds request model
  -> request sent via mailbox (or legacy file path)
  -> leader resolves
  -> response appears via mailbox / resolved artifact
  -> useSwarmPermissionPoller resumes continuation
```

也就是说：

```text
useSwarmPermissionPoller.ts 回答“worker 本地怎样挂起并恢复权限等待中的执行”
permissionSync.ts 回答“这些权限请求/响应在 transport 与持久化层到底怎样表示、怎样传递、以及旧新两套同步方案怎样并存”
```

---

### 下一站建议

下一站最顺的是：

```text
src/hooks/useCanUseTool.ts
```

因为现在你已经看懂了：

- permission request / response 的 transport 与状态同步层
- worker 侧 callback registry 与 response continuation 恢复层
- leader mailbox 如何回发审批结果
- 但还没看“工具真正发起 ask-mode 权限请求时，是谁决定走本地 allow / reject，还是注册 callback 并发给 leader”的入口

下一步最自然就是把这条链补到工具调用前线：

**一次 tool use 在 worker 侧到底怎样进入 permission 检查、怎样决定要不要发 swarm permission request、怎样注册 pending callback，并把审批结果重新接回工具调用主路径。**
