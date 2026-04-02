## 第 35 站：`src/utils/teammateMailbox.ts`

### 这是什么文件

`src/utils/teammateMailbox.ts` 是 agent swarm 系统里的 **通用 teammate 消息总线适配层**。

上一站 `permissionSync.ts` 已经看到：

```text
permissionSync.ts
  -> 把 swarm permission request/response 适配成 mailbox message
  -> 再通过 writeToMailbox(...) 发给 leader 或 worker
```

而这一站真正往下钻到更底层：

```text
mailbox 到底存在哪
消息长什么样
怎样读写
怎样避免并发写乱
以及哪些消息应该进入协议处理器，哪些消息应该变成普通 LLM 附件上下文
```

所以最准确的一句话是：

```text
teammateMailbox.ts = swarm 多 agent 通信的通用邮箱协议与文件投递层
```

---

### 先看它解决的系统问题

位置：`src/utils/teammateMailbox.ts:1`

文件头注释已经很直白：

- 每个 teammate 都有一个 inbox 文件
- 其他 teammate 往里面写消息
- 收件方会把这些消息看成 attachments
- inbox 以 agent name 为 key，而不是 UUID

这说明它不是“消息对象定义文件”这么简单，而是把 swarm 里的 teammate 通信落实成了：

```text
每个 agent 一个 inbox 文件，
跨 agent 通信就是向对方 inbox 追加一条消息
```

所以它是一个非常具体的通信基础设施，而不是抽象事件总线。

---

### 第一部分：mailbox 的物理模型是“每个 agent 一个 inbox JSON 文件”

位置：`src/utils/teammateMailbox.ts:52`

`getInboxPath(agentName, teamName)` 会生成路径：

```text
~/.claude/teams/{team_name}/inboxes/{agent_name}.json
```

并且会先做：

- `sanitizePathComponent(team)`
- `sanitizePathComponent(agentName)`

这说明 mailbox 的存储模型非常清晰：

```text
team 是一级命名空间
agentName 是二级收件箱标识
每个 inbox 文件里存放这个 agent 收到的所有消息数组
```

同时路径组件清洗说明它明确把：

- team name
- agent name

视为潜在外部输入，需要先防止路径污染。

所以这里既是路由计算点，也是基本安全边界。

---

### 第二部分：`TeammateMessage` 是通用信封，不关心具体协议类型

位置：`src/utils/teammateMailbox.ts:43`

基础消息类型只有这些字段：

- `from`
- `text`
- `timestamp`
- `read`
- `color?`
- `summary?`

这说明 mailbox 最底层并不认识 permission、shutdown、task assignment 这些高阶概念。

它只保存一个通用信封：

```text
发件人是谁
消息正文文本是什么
有没有读过
UI 预览信息是什么
```

换句话说：

```text
具体协议都被编码进 text 里；
mailbox 层本身只是 message envelope store
```

这也是后面会有那么多 `isPermissionRequest(...)` / `isShutdownRequest(...)` 之类解析器的原因。

---

### 第三部分：`readMailbox(...)` / `readUnreadMessages(...)` 表明 inbox 是“整箱读取”，不是流式 cursor

位置：

- `readMailbox(...)` `src/utils/teammateMailbox.ts:79`
- `readUnreadMessages(...)` `src/utils/teammateMailbox.ts:110`

这里的做法是：

- 直接读整个 inbox JSON
- 解析成 `TeammateMessage[]`
- unread 则在内存里 filter `!m.read`

这说明 mailbox 模型不是 Kafka 那种偏移量流，也不是 append-only log + cursor，而是：

```text
一个当前 inbox 快照文件
```

收件方每次读到的是“当前箱子里所有消息”，再自行做：

- 未读筛选
- 协议路由
- 标记已读

所以这是一种很简单但很直接的 mailbox 设计。

---

### 第四部分：`writeToMailbox(...)` 才是整份文件最核心的基础设施

位置：`src/utils/teammateMailbox.ts:127`

这里的主流程是：

1. `ensureInboxDir(...)`
2. 算 inbox path
3. 若 inbox 文件不存在，先创建 `[]`
4. 对 inbox 文件加锁
5. 加锁后重新 `readMailbox(...)`
6. `messages.push(newMessage)`
7. 回写整个数组

这说明 mailbox 写入不是 append syscall，而是：

```text
读整个 inbox 数组
  -> 追加一条
  -> 重写整个 inbox 文件
```

因此锁就变成了必需品，因为这里天然存在 read-modify-write 竞态。

所以它的真正语义是：

```text
原子化追加一条 teammate message 到某个 inbox 快照
```

---

### 第五部分：锁策略体现的是“多 Claude 并发写同一个 inbox”的现实场景

位置：`src/utils/teammateMailbox.ts:31`

文件顶部 `LOCK_OPTIONS` 注释写得很关键：

- async lock API 需要显式 retries
- 多个 Claudes in a swarm 会并发竞争同一把锁
- 目标是等待锁而不是立即失败

这说明作者非常明确这个场景：

```text
多个 agent 可能同时给同一个收件人发消息
```

所以 mailbox 不是单线程玩具，而是真按 swarm 并发环境设计的。

这里的重点不是“用了 lock”，而是：

```text
mailbox 的数据模型要求序列化写入，
否则任何并发追加都会丢消息
```

---

### 第六部分：所有“标记已读”操作也都走锁，说明 read-state 也是共享状态

位置：

- `markMessageAsReadByIndex(...)` `src/utils/teammateMailbox.ts:194`
- `markMessagesAsRead(...)` `src/utils/teammateMailbox.ts:273`
- `markMessagesAsReadByPredicate(...)` `src/utils/teammateMailbox.ts:1097`

这几组 API 都遵循同样的模式：

- 先锁 inbox 文件
- 再重新读取最新状态
- 更新 `read` 字段
- 回写整个 inbox

这说明在这个模型里，“已读/未读”并不是 UI 本地态，而是 mailbox 文件的一部分共享状态。

因此它不仅保存消息内容，还保存：

```text
这个 inbox 里的哪些消息已经被处理/消费过
```

所以 mailbox 同时承担了：

- message storage
- read receipt state

两类职责。

---

### 第七部分：`clearMailbox(...)` 和 `formatTeammateMessages(...)` 说明 mailbox 同时服务于协议层与 LLM 上下文层

位置：

- `clearMailbox(...)` `src/utils/teammateMailbox.ts:344`
- `formatTeammateMessages(...)` `src/utils/teammateMailbox.ts:370`

`clearMailbox(...)` 很简单：把 inbox 重置成 `[]`。

而 `formatTeammateMessages(...)` 则把普通 teammate 消息渲染成：

```xml
<teammate_message teammate_id="..." color="..." summary="...">
...
</teammate_message>
```

这一步很关键，因为它说明 mailbox 消息并不总是只给程序逻辑消费，
还有一条直接进入模型上下文的路径：

```text
收件消息 -> formatTeammateMessages -> attachment/XML 注入给 LLM
```

所以 mailbox 既是协议总线，也是 prompt 输入源的一部分。

---

### 第八部分：permission request/response 在这里被定义成 mailbox 协议消息，而不是 inbox 特例

位置：

- `PermissionRequestMessage` `src/utils/teammateMailbox.ts:449`
- `PermissionResponseMessage` `src/utils/teammateMailbox.ts:464`
- `createPermissionRequestMessage(...)` `src/utils/teammateMailbox.ts:485`
- `createPermissionResponseMessage(...)` `src/utils/teammateMailbox.ts:509`

这里的设计非常清楚：

- worker -> leader：`type: 'permission_request'`
- leader -> worker：`type: 'permission_response'`

而且注释明确说字段名要与 SDK `can_use_tool` 对齐，尤其使用 snake_case。

这说明 permissionSync 上一站里所谓“mailbox 化”，在这一层真实落地成了：

```text
把权限审批包装成标准结构化 teammate message
```

所以 `teammateMailbox.ts` 不是权限模块的下游实现细节，
而是所有高层协议真正共享的格式定义处。

---

### 第九部分：sandbox / shutdown / plan approval / mode set / task assignment 都复用同一邮箱总线

位置：整份文件中段到后段

这份文件里你会看到一长串协议类型：

- `sandbox_permission_request`
- `sandbox_permission_response`
- `plan_approval_request`
- `plan_approval_response`
- `shutdown_request`
- `shutdown_approved`
- `shutdown_rejected`
- `task_assignment`
- `team_permission_update`
- `mode_set_request`
- `idle_notification`

这说明 teammate mailbox 绝不是“给 SendMessage 用的 DM 收件箱”这么窄。

它真正的定位是：

```text
swarm runtime 的统一跨 agent 控制平面消息通道
```

而普通自然语言消息，只是这条总线承载的一类 payload。

所以你应该把它记成：

```text
mailbox = shared transport
message type = protocol layer on top
```

---

### 第十部分：解析函数分成两类，反映了协议成熟度不同

位置：整份文件后半段

这里有两种明显不同的解析风格：

#### A. 直接 `jsonParse` + `type` 判断
例如：
- `isPermissionRequest(...)`
- `isPermissionResponse(...)`
- `isTaskAssignment(...)`
- `isTeamPermissionUpdate(...)`

#### B. `zod` schema `safeParse(...)`
例如：
- `isPlanApprovalRequest(...)`
- `isPlanApprovalResponse(...)`
- `isShutdownRequest(...)`
- `isShutdownApproved(...)`
- `isShutdownRejected(...)`
- `isModeSetRequest(...)`

这说明不同协议的严谨程度并不完全一致：

```text
有些协议只做轻量类型判别；
有些协议已经升级到 schema 级验证
```

从架构角度，这通常意味着：

- 一部分是较早或较轻的协议
- 一部分是后续更重、更正式的控制面协议

所以这份文件还能看出协议系统正在逐步向更严格校验迁移。

---

### 第十一部分：`isStructuredProtocolMessage(...)` 是 mailbox 与 LLM 上下文之间的分流闸门

位置：`src/utils/teammateMailbox.ts:1064`

这段注释极其关键：

```text
这些 structured protocol messages 应该由 useInboxPoller 路由，
而不是先被 getTeammateMailboxAttachments 当作原始文本塞进 LLM context。
```

它的意义非常大，因为 mailbox 里本来同时混着两种东西：

#### 1. 给程序处理的协议消息
- permission request/response
- sandbox permission
- shutdown
- mode set
- plan approval
- team permission update

#### 2. 给模型看的普通 teammate 文本
- 对话式 DM
- 协作说明
- 任务交流

如果不分流，结构化协议消息会被：

- 当成普通附件文本
- 注入到 LLM 上下文
- 从而错过真正的程序 handler

所以 `isStructuredProtocolMessage(...)` 实际上是：

```text
mailbox control-plane message 与 mailbox conversational message 的分拣阀门
```

这是整份文件里最关键的架构点之一。

---

### 第十二部分：`getLastPeerDmSummary(...)` 说明 mailbox 还承担了协作 UI 的摘要来源

位置：`src/utils/teammateMailbox.ts:1144`

这个函数会从最近的 assistant messages 倒着找：

- `SendMessage` tool_use
- 目标是 peer，不是 `*`，也不是 team lead
- 提取 `summary` 或 message 前 80 字
- 生成 `[to name] summary`

这说明 mailbox 除了实际递送文本，还在为上层 UI / idle notification / 协作状态展示提供“最近一次 peer DM 摘要”。

所以它影响的不只是投递本身，还影响：

```text
系统如何向用户/leader 展示 teammate 最近做了什么沟通
```

这属于非常明显的 control-plane 可观测性设计。

---

### 第十三部分：`sendShutdownRequestToMailbox(...)` 体现了“高层操作复用通用邮箱写入原语”的模式

位置：`src/utils/teammateMailbox.ts:822`

这个函数做的事情是：

- 生成 shutdown request ID
- 构造 `shutdown_request` 协议消息
- 再统一走 `writeToMailbox(...)`

它把这整个文件的架构模式暴露得很清楚：

```text
高层功能 = create<Protocol>Message(...)
        + writeToMailbox(...)
        + 对端 is<Protocol>Message(...) 解析
```

也就是说 teammate mailbox 体系实际上是一种：

```text
message schema + common transport + per-type router
```

模式。

这和 permission request/response 的实现方式是完全同构的。

---

### 第十四部分：把它和第 34 站连起来看，才能真正理解 mailbox 在 swarm 里的地位

上一站 `permissionSync.ts` 里其实已经出现：

- `createPermissionRequestMessage(...)`
- `createPermissionResponseMessage(...)`
- `writeToMailbox(...)`

但如果不读这一站，很容易误以为 mailbox 只是“一个方便发消息的 helper”。

实际上这里告诉你：

```text
mailbox 不是 helper，
而是 swarm 整个控制平面共享的底层 transport substrate
```

permission 只是其上的一个协议；
shutdown、plan approval、sandbox、mode set、task assignment 也都在同一条总线上跑。

所以 mailbox 在 swarm 体系里的层级，远高于单一功能模块。

---

### 读完这一站后，你应该抓住的 9 个事实

1. `teammateMailbox.ts` 定义的是每个 agent 一个 inbox 文件的通用 mailbox 通信模型，而不是某个单一功能模块的私有实现。
2. `TeammateMessage` 是通用信封，具体协议内容都编码在 `text` 字段里。
3. `writeToMailbox(...)` 的本质是“加锁后读-改-写整个 inbox 数组”，因此锁是数据正确性的核心条件。
4. 已读/未读状态也是 inbox 文件里的共享状态，所以标记已读同样必须加锁。
5. `formatTeammateMessages(...)` 说明 mailbox 消息不只给程序消费，也会被格式化后注入 LLM 上下文。
6. permission request/response、sandbox、shutdown、plan approval、mode set、task assignment 等都复用同一条邮箱总线，说明 mailbox 是 swarm 的统一控制平面传输层。
7. 不同协议的解析严格度不同，部分走轻量 `type` 判断，部分已升级为 zod schema 校验。
8. `isStructuredProtocolMessage(...)` 是控制平面消息与普通对话消息的关键分流闸门，避免协议消息误入 LLM 原始上下文。
9. 高层功能普遍遵循“create message -> writeToMailbox -> peer parser/router”这一统一实现模式。

---

### 现在把第 34-35 站串起来

```text
permissionSync.ts
  -> 定义 swarm permission 的请求/响应协议，并把它适配到 mailbox 传输
teammateMailbox.ts
  -> 提供通用 mailbox 存储、并发写入、协议消息格式、解析与分流基础设施
```

所以现在你应该把两层关系记成：

```text
permissionSync = permission-specific adapter
teammateMailbox = generic swarm transport substrate
```

---

### 下一站建议

下一站最顺的是：

```text
src/hooks/useInboxPoller.ts
```

因为现在你已经看懂了：

- mailbox 消息存在哪、怎样写
- 结构化协议消息怎样与普通 teammate 文本分流
- permission/shutdown 等协议怎样编码成 mailbox message

下一步最自然就是看运行时消费端：

**`useInboxPoller.ts` 是怎样轮询 inbox、识别 structured protocol messages、把它们路由到 worker permissions / sandbox permissions / shutdown / plan approval 等具体处理链上的。**
