## 第 44 站：`src/utils/swarm/backends/types.ts`

### 这是什么文件

`src/utils/swarm/backends/types.ts` 是 swarm backend 体系里的 **抽象接口定义层**。

上一站 `registry.ts` 已经看懂：

```text
registry 会在 tmux / iTerm2 / in-process 之间做选择
并把不同实现统一收口到 TeammateExecutor
```

但如果只看 registry，仍然还有一个核心问题没完全落地：

- `PaneBackend` 和 `TeammateExecutor` 到底分别负责什么
- pane backend 与 in-process backend 的接口边界怎么划
- spawn result / message / identity 这些跨后端共享对象长什么样
- `BackendType` / `PaneBackendType` 为什么要分开

所以这一站回答的是：

```text
swarm 是怎样把“底层 pane 操作能力”和“高层 teammate 生命周期能力”抽象成一组统一类型合同的
```

所以最准确的一句话是：

```text
types.ts = swarm backend 抽象层的类型契约文件
```

---

### 先看它的整体定位

位置：`src/utils/swarm/backends/types.ts:1`

这份文件没有实现逻辑，
但它定义了整个 backend 子系统最重要的几个公共抽象：

- `BackendType`
- `PaneBackendType`
- `PaneBackend`
- `BackendDetectionResult`
- `TeammateIdentity`
- `TeammateSpawnConfig`
- `TeammateSpawnResult`
- `TeammateMessage`
- `TeammateExecutor`
- `isPaneBackend(...)`

这说明它的作用不是“某个具体 backend 的说明书”，
而是：

```text
所有 backend 实现、registry、team runtime、cleanup 路径共享的一套协议层类型定义
```

也就是说，这里是：

```text
implementation 之前的抽象边界
```

---

### 第一部分：`BackendType` / `PaneBackendType` 的拆分说明系统先区分“所有 teammate backend”与“terminal pane 子集”

位置：

- `BackendType` `src/utils/swarm/backends/types.ts:9`
- `PaneBackendType` `src/utils/swarm/backends/types.ts:15`

这里先定义：

- `BackendType = 'tmux' | 'iterm2' | 'in-process'`
- `PaneBackendType = 'tmux' | 'iterm2'`

这很关键。

因为它说明系统一开始就明确承认：

```text
不是所有 teammate backend 都有 pane
```

所以如果只用一个 `BackendType` 到处乱传，
很多只适用于 pane 的 API 就会被类型上污染。

这也是为什么要单独有：

```text
PaneBackendType = pane-capable backend 的严格子集
```

也就是说：

- `BackendType` 用来描述 teammate 的总执行后端空间
- `PaneBackendType` 用来描述 terminal-pane 技术栈空间

这是整个抽象层最基本的第一刀切分。

---

### 第二部分：`PaneId` 和 `CreatePaneResult` 说明 pane 资源被抽象成跨 backend 统一的“opaque handle”

位置：

- `PaneId` `src/utils/swarm/backends/types.ts:22`
- `CreatePaneResult` `src/utils/swarm/backends/types.ts:27`

`PaneId` 被定义成 `string`，但注释明确说：

- tmux 下是 `%1` 这类 pane ID
- iTerm2 下是 it2 返回的 session ID

这说明系统并不试图在类型层统一它们的内部格式，
而是把它抽象成：

```text
backend-specific identifier, but cross-backend opaque handle
```

也就是：

```text
上层只知道“这是一个 pane 标识”
不需要知道它到底是 tmux pane id 还是 iTerm2 session id
```

`CreatePaneResult` 也很有意思，除了 `paneId` 外还带：

- `isFirstTeammate`

这说明创建 pane 这件事在 swarm 里不仅是“给你一个 ID”，
还会影响布局策略：

```text
第一个 teammate pane 和后续 pane 的布局含义不一样
```

所以 pane create 的抽象里天然包含：

- 资源句柄
- 布局上下文信息

---

### 第三部分：`PaneBackend` 只负责 pane 宿主能力，不负责 teammate 生命周期

位置：`src/utils/swarm/backends/types.ts:39`

`PaneBackend` 这个接口定义得非常清楚，所有方法几乎都围绕 pane 操作：

- `isAvailable()`
- `isRunningInside()`
- `createTeammatePaneInSwarmView(...)`
- `sendCommandToPane(...)`
- `setPaneBorderColor(...)`
- `setPaneTitle(...)`
- `enablePaneBorderStatus(...)`
- `rebalancePanes(...)`
- `killPane(...)`
- `hidePane(...)`
- `showPane(...)`

你会发现它根本没有：

- `spawn(config)`
- `sendMessage(agentId, ...)`
- `terminate(agentId)`

这说明作者非常刻意地把：

```text
pane management
```

和：

```text
teammate lifecycle management
```

拆开了。

所以 `PaneBackend` 的准确定位不是“teammate backend”，而是：

```text
pane-host capability provider
```

这点特别重要，
因为它解释了上一站为什么还需要 `PaneBackendExecutor` 这层 adapter。

---

### 第四部分：`PaneBackend` 的方法集合揭示了 pane backend 的职责是“可视宿主控制”而不是“agent 语义”

位置：`src/utils/swarm/backends/types.ts:49`

细看这些方法，会发现它们几乎全是 terminal/pane 层能力：

#### 可用性 / 环境识别
- `isAvailable()`
- `isRunningInside()`

#### pane 创建与命令注入
- `createTeammatePaneInSwarmView(...)`
- `sendCommandToPane(...)`

#### pane 展示控制
- `setPaneBorderColor(...)`
- `setPaneTitle(...)`
- `enablePaneBorderStatus(...)`
- `rebalancePanes(...)`
- `hidePane(...)`
- `showPane(...)`

#### pane 终止
- `killPane(...)`

也就是说它的主要职责是：

```text
创建/布局/标记/隐藏/显示/关闭 这些“宿主可视资源”
```

而不是：

- 队友的 prompt 是什么
- 队友 agentId 是谁
- 消息协议怎样发
- 什么时候 graceful shutdown

这些 higher-level 语义都不在这里。

所以这个接口本质上是在表达：

```text
terminal-pane backend 是一种 UI/runtime host backend，不是完整 swarm agent backend
```

---

### 第五部分：`BackendDetectionResult` 说明 backend detection 的输出不是单一 backend，而是 backend + 解释元信息

位置：`src/utils/swarm/backends/types.ts:173`

这个类型有三个字段：

- `backend`
- `isNative`
- `needsIt2Setup?`

这和上一站 `registry.ts` 完全对上。

关键点在于它没有被设计成：

```ts
type BackendDetectionResult = PaneBackend
```

而是额外携带：

#### `isNative`
表示：

```text
当前是不是运行在这个 backend 的天然宿主环境里
```

比如：

- 在 tmux session 中使用 tmux -> native
- 在普通终端里启外部 tmux session -> non-native
- 在 iTerm2 原生 pane 中使用 iTerm2 -> native

#### `needsIt2Setup`
表示：

```text
虽然最终选了别的 backend，但 iTerm2 用户还可能需要一个 it2 安装提示
```

所以 detection 结果是：

```text
runtime backend choice + UX/context metadata
```

而不是单纯实现对象。

---

### 第六部分：`TeammateIdentity` 是 backend 层共享的最小身份子集，而不是完整运行时上下文

位置：`src/utils/swarm/backends/types.ts:191`

这里的 `TeammateIdentity` 只有：

- `name`
- `teamName`
- `color`
- `planModeRequired`

注释明确说：

```text
这是和 TeammateContext 共享的一小部分字段，用来避免 circular deps
完整上下文在别处定义
```

这说明 backend 抽象层需要身份信息，
但只需要：

```text
最小可移植 identity slice
```

而不需要完整的：

- `abortController`
- `parentSessionId` 的全部运行时语义
- ALS context
- AppState task state

所以这里表达的是：

```text
backend 层依赖 teammate identity，但只依赖最低限度的身份字段
```

这是一种很典型的去耦做法。

---

### 第七部分：`TeammateSpawnConfig` 把“任意执行模式都需要的 spawn 语义”统一起来了

位置：`src/utils/swarm/backends/types.ts:205`

这个类型非常重要，因为它是跨后端共用的 spawn 输入：

在 `TeammateIdentity` 基础上再加：

- `prompt`
- `cwd`
- `model`
- `systemPrompt`
- `systemPromptMode`
- `worktreePath`
- `parentSessionId`
- `permissions`
- `allowPermissionPrompts`

这说明对 swarm 来说，spawn 一个 teammate 时真正重要的不是它最终落在：

- pane backend
- 还是 in-process backend

而是先固定这名 teammate 的语义配置：

```text
它是谁
它初始要做什么
在哪个 cwd/worktree 里做
用什么 model/system prompt
有什么权限边界
属于哪个父 session
```

所以 `TeammateSpawnConfig` 的意义是：

```text
把 backend-specific spawn 差异压到实现层，
把 backend-agnostic teammate creation intent 固定为统一输入模型
```

这就是跨后端抽象真正成立的关键。

---

### 第八部分：`permissions` 和 `allowPermissionPrompts` 被放进 spawn config，说明权限策略是 teammate 出生配置的一部分

位置：`src/utils/swarm/backends/types.ts:220`

这里的字段很关键：

- `permissions?: string[]`
- `allowPermissionPrompts?: boolean`

注释还明确说：

```text
当 false（默认）时，未列出的工具自动拒绝
```

这说明 teammate 的权限语义不是后面临时拼接的，
而是在 spawn 时就已经被看作：

```text
teammate execution contract 的一部分
```

也就是说后端不只是接收“一个 prompt + cwd”，
还要接收：

```text
这个 teammate 从出生开始有哪些工具权限边界
```

这也和前面看到的 swarm permission 路径呼应起来了。

---

### 第九部分：`TeammateSpawnResult` 揭示了 spawn 返回值同时兼容逻辑身份、UI 管理、backend 资源句柄三层语义

位置：`src/utils/swarm/backends/types.ts:230`

这个返回值包括：

- `success`
- `agentId`
- `error?`
- `abortController?`
- `taskId?`
- `paneId?`

这说明 spawn 返回值不是只关心“创建成功没”，
而是要同时覆盖三层信息：

#### 1. 逻辑身份层
- `agentId`

#### 2. in-process 管理层
- `abortController`
- `taskId`

#### 3. pane backend 资源层
- `paneId`

所以这个结果类型真正表达的是：

```text
spawn 成功后，上层继续管理这个 teammate 所需要的最小跨后端句柄集合
```

也因此它必须是一个 union-like supertype，
允许不同 backend 只填自己需要的那部分字段。

---

### 第十部分：`TeammateMessage` 说明 backend 抽象层并不直接暴露 mailbox 协议，而是统一成更高层的 teammate message 语义

位置：`src/utils/swarm/backends/types.ts:259`

这个类型包括：

- `text`
- `from`
- `color`
- `timestamp`
- `summary`

很明显，这里的 message 不是某个特定 transport 的 wire format，
而是一种更高层的 teammate 通信载荷。

比如：

- pane backend 可能会把它转成命令注入 / mailbox 写入
- in-process backend 可能会转成 task state / pendingUserMessages / 本地调用

所以这个类型的意义在于：

```text
上层只表达“我要给 agentId 发这样一条 teammate message”
具体怎样送达由 backend executor 决定
```

这再次体现了 backend 抽象层的核心目标：

```text
统一语义，屏蔽 transport 差异
```

---

### 第十一部分：`TeammateExecutor` 才是 swarm backend 对上层最重要的统一接口

位置：`src/utils/swarm/backends/types.ts:279`

这个接口只有几个高层操作：

- `isAvailable()`
- `spawn(config)`
- `sendMessage(agentId, message)`
- `terminate(agentId, reason?)`
- `kill(agentId)`
- `isActive(agentId)`

这和 `PaneBackend` 的低层操作形成鲜明对比。

#### `PaneBackend`
关心：
- pane
- layout
- border
- hide/show
- command injection

#### `TeammateExecutor`
关心：
- spawn teammate
- 给 teammate 发消息
- graceful terminate
- force kill
- teammate 是否活着

所以这两个接口的关系可以压缩成：

```text
PaneBackend = host resource API
TeammateExecutor = teammate lifecycle API
```

这就是 swarm backend 抽象层最重要的设计。

也是上一站 `registry.ts` 的统一 facade 能成立的根基。

---

### 第十二部分：`terminate()` 和 `kill()` 同时存在，说明 backend 抽象层原生承认协作式关闭与强制终止的区别

位置：`src/utils/swarm/backends/types.ts:292`

这里不是只给一个 `stop()`，
而是明确区分：

- `terminate(agentId, reason?)`
- `kill(agentId)`

这和之前 `InProcessTeammateTask` 看到的：

- request shutdown
- 强制 kill

是对齐的。

这说明“优雅退出”和“立即终止”的区别，
不是某个具体 backend 的临时实现细节，
而是已经上升为 backend 抽象层的正式 contract。

也就是说：

```text
所有 teammate backend 都应该同时理解：
请求结束
vs
强制结束
```

这是很强的架构信号。

---

### 第十三部分：`isActive(agentId)` 说明“teammate 是否仍在运行”被建模成跨 backend 的公共查询能力

位置：`src/utils/swarm/backends/types.ts:298`

最后一个方法是：

- `isActive(agentId)`

这很值得注意，因为不同 backend 判断活跃状态的方式肯定不同：

- pane backend 可能看 pane / process / mailbox / session
- in-process backend 可能看 task state

但上层并不想知道这些差异。

所以这里把它统一成：

```text
given an agentId, tell me whether this teammate is still active
```

这说明 backend 抽象层不仅统一了“命令式操作”，
还统一了最基本的“状态查询”。

---

### 第十四部分：`isPaneBackend(...)` 这个 type guard 是整个抽象层里很关键的一条分流桥

位置：`src/utils/swarm/backends/types.ts:309`

这个函数看起来很小：

- `type === 'tmux' || type === 'iterm2'`

但它非常关键，
因为很多地方虽然只拿到 `BackendType`，
却需要进一步分流：

```text
如果是 pane backend，允许 killPane / hidePane / showPane
如果是 in-process，就走另一套逻辑
```

所以它实际上是在类型层把：

```text
broad backend space
```

收窄成：

```text
pane-capable backend subset
```

这正是上一站 `cleanupSessionTeams()` / `killOrphanedTeammatePanes()` 之类逻辑能写得干净的重要基础。

---

### 第十五部分：这份文件真正把 swarm backend 抽象切成了三层

如果把全文件压缩，你会发现它不是随便堆类型，
而是在非常清楚地切三层：

#### 第一层：backend identity / classification
- `BackendType`
- `PaneBackendType`
- `isPaneBackend(...)`

#### 第二层：low-level host capability
- `PaneBackend`
- `PaneId`
- `CreatePaneResult`
- `BackendDetectionResult`

#### 第三层：high-level teammate lifecycle
- `TeammateIdentity`
- `TeammateSpawnConfig`
- `TeammateSpawnResult`
- `TeammateMessage`
- `TeammateExecutor`

所以它真正表达的不是“后端有几种”，
而是：

```text
backend 抽象应该先分清：
类型归类
-> 宿主能力
-> teammate 生命周期能力
```

这正是整个 swarm backend 设计不混乱的原因。

---

### 元

问题：**这一站真正想解决的架构问题是什么？**

回答：这一站定义 backend 子系统的类型契约，区分 `BackendType`、`PaneBackendType`、`PaneBackend` 与 `TeammateExecutor`。它清晰划开“底层 pane 宿主能力”和“高层 teammate 生命周期能力”。

### 反

问题：**如果把这一站的设计反过来，会发生什么？**

回答：如果抽象边界不清，pane backend 和 in-process backend 的接口会互相污染。类型层一旦混乱，registry 和各实现也会逐渐失去可替换性。

### 空

问题：**跳出当前文件名，这一站背后更大的问题是什么？**

回答：更大的问题是，系统何时需要正式抽象合同。答案往往是：当你已经确定会长期维护多种后端实现时。


### 读完这一站后，你应该抓住的 9 个事实

1. `types.ts` 是 swarm backend 抽象层的类型契约文件，不承载具体 backend 实现。
2. `BackendType` 与 `PaneBackendType` 被明确分开，说明系统从类型层就区分“所有 teammate backend”与“pane-capable backend 子集”。
3. `PaneId` 被建模成跨 backend 的 opaque handle，屏蔽 tmux pane ID 与 iTerm2 session ID 的具体差异。
4. `PaneBackend` 只负责 pane 宿主能力，如创建/布局/隐藏/显示/关闭 pane，而不直接负责 teammate 生命周期语义。
5. `BackendDetectionResult` 不只是选中的 backend，还包含 `isNative`、`needsIt2Setup` 这类解释与 UX 元信息。
6. `TeammateSpawnConfig` 把 prompt、cwd、model、system prompt、worktree、permissions、parentSessionId 等跨后端共享的 teammate 创建语义统一了起来。
7. `TeammateSpawnResult` 同时兼容逻辑身份（`agentId`）、in-process 管理句柄（`abortController`/`taskId`）和 pane 资源句柄（`paneId`）。
8. `TeammateExecutor` 是对上层最重要的统一接口：它抽象的是 teammate 生命周期，而不是 pane 细节。
9. `terminate()`/`kill()` 与 `isActive()` 的存在说明 graceful shutdown、force kill、活跃状态查询都已经被建模成跨 backend 的正式 contract。

---

### 现在把第 43-44 站串起来

```text
registry.ts
  -> 选择并缓存 backend，把不同实现统一包装成 TeammateExecutor
backends/types.ts
  -> 定义 PaneBackend / TeammateExecutor / SpawnConfig / SpawnResult 等抽象合同，支撑这种统一包装成为可能
```

所以现在 backend 体系可以先压缩成：

```text
types.ts
  -> 定义抽象边界
registry.ts
  -> 负责选择实现并返回统一 executor
```

也就是说：

```text
types.ts 回答“不同 backend 在接口层应该长什么样”
registry.ts 回答“当前 session 实际该用哪一个 backend 实现”
```

---

### 下一站建议

下一站最顺的是：

```text
src/utils/swarm/backends/PaneBackendExecutor.ts
```

因为现在你已经看懂了：

- `PaneBackend` 和 `TeammateExecutor` 的边界不同
- registry 会把 pane backend 包一层再暴露为 `TeammateExecutor`
- 这层 adapter 是整个统一执行接口成立的关键

下一步最自然就是继续看这条适配链：

**`PaneBackendExecutor` 到底怎样把 pane 级操作拼装成 spawn / sendMessage / terminate / kill / isActive 这种 teammate 生命周期接口，以及它在 pane backend 和 mailbox/team runtime 之间具体承担了哪些桥接职责。**
