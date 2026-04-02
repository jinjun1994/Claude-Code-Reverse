## 第 53 站：`src/utils/swarm/constants.ts`

### 这是什么文件

`src/utils/swarm/constants.ts` 是 swarm 子系统里的 **共享命名约定与宿主协议常量定义文件**。

上一站 `ITermBackend.ts` 已经看懂：

- tmux / iTerm2 两类 backend 都在操纵 pane host
- 它们需要共用一些固定名字
- 例如 session 名、window 名、leader 名、spawn env var 名

这些东西如果散落在各文件里，就会导致：

- backend 命名不一致
- mailbox / session / hidden pane 语义飘散
- spawn 入口和 runtime 读取的 env var 对不上

所以这一站回答的是：

```text
swarm 子系统跨 backend、跨 spawn、跨 runtime 共用的命名与协议常量，到底集中定义在哪里
```

所以最准确的一句话是：

```text
constants.ts = swarm 命名空间与宿主协议的共享字典
```

---

### 先看它的整体定位

位置：`src/utils/swarm/constants.ts:1`

这个文件非常短，
但它的职责很明确：

- 定义 team lead 的保留名字
- 定义 tmux swarm session / window / hidden session 命名
- 定义 teammate spawn command override env var
- 定义 teammate 运行时身份相关 env var
- 定义 external swarm socket 的命名规则

也就是说它不做：

- backend 逻辑
- mailbox 读写
- spawn 参数组装
- team metadata 持久化

它只做：

```text
shared protocol naming
```

这是一个典型的小文件，但它对整条 swarm 链路的一致性很关键。

---

### 第一部分：`TEAM_LEAD_NAME = 'team-lead'` 说明 team lead 在 swarm 里不是临时字符串，而是一个正式保留身份名

位置：`src/utils/swarm/constants.ts:1`

这里定义：

- `TEAM_LEAD_NAME = 'team-lead'`

这看起来简单，
但架构意义很大。

因为前面几站已经反复看到：

- mailbox 消息会从 `team-lead` 发出
- shutdown request 由 `team-lead` 发起
- 初始 prompt 也是 `team-lead` 投递
- teammate 在协议层会区分 lead 与 peer message

所以这里并不是随便写了个展示文案，
而是在固定：

```text
team control plane sender identity
```

也就是说 `team-lead` 在 swarm 里更像是协议角色名，
而不是普通 teammate 名称之一。

---

### 第二部分：`SWARM_SESSION_NAME` / `SWARM_VIEW_WINDOW_NAME` / `HIDDEN_SESSION_NAME` 三个常量，把 tmux backend 的空间拓扑命名正式化了

位置：`src/utils/swarm/constants.ts:2`

这里定义：

- `SWARM_SESSION_NAME = 'claude-swarm'`
- `SWARM_VIEW_WINDOW_NAME = 'swarm-view'`
- `HIDDEN_SESSION_NAME = 'claude-hidden'`

这三个名字正好对应前面在 `TmuxBackend.ts` 里看到的三类空间：

#### `claude-swarm`
外部 swarm session

#### `swarm-view`
teammates 展示用 window

#### `claude-hidden`
hidePane 时 pane 被迁移到的 detached hidden session

这说明 tmux backend 的拓扑不是隐式约定，
而是由这一层正式命名：

```text
external visible swarm space
+
hidden parked pane space
```

这让整个 tmux 路线的资源命名变得可预测且可共享。

---

### 第三部分：`TMUX_COMMAND = 'tmux'` 看似 trivial，但它说明 swarm 把“tmux 这个宿主命令名”也纳入集中协议常量，而不是让各 backend 自己硬编码

位置：`src/utils/swarm/constants.ts:4`

这里直接导出：

- `TMUX_COMMAND = 'tmux'`

虽然只是一个字符串，
但这说明作者希望连“宿主 CLI 的 canonical command name”都统一收口。

这带来的价值是：

- detection.ts 用它做 capability probing
- TmuxBackend.ts 用它做 pane 操作
- 任何未来 helper 也都不会各自硬编码 `tmux`

所以它的意义不是减少几个字符，
而是：

```text
host command naming should have a single source of truth
```

---

### 第四部分：`getSwarmSocketName()` 最重要的不是字符串格式，而是它把 external tmux session 与用户自己的 tmux 世界彻底隔离，并用 PID 避免多 Claude 实例互撞

位置：`src/utils/swarm/constants.ts:7`

这里返回：

- ``claude-swarm-${process.pid}``

注释点出了两个关键意图：

#### 1. isolate swarm operations from user's tmux sessions
即 external swarm session 不要污染用户自己的 tmux socket

#### 2. includes PID to ensure multiple Claude instances don't conflict
即同一台机器上多个 Claude 进程也不要抢同一个 swarm socket

所以这不是简单取个名字，
而是在定义：

```text
external swarm tmux namespace = per-process isolated socket namespace
```

这也是前面 `TmuxBackend.ts` 能安全起 external session 的基础之一。

---

### 第五部分：这里用 PID 而不是固定名字，说明 swarm 的 external tmux backend 被建模成“每个 Claude 进程自带一套独立 tmux 控制域”

位置：`src/utils/swarm/constants.ts:12`

如果这里只返回：

- `claude-swarm`

那多个 Claude 进程同时跑时，
会共享同一个 tmux socket，
从而造成：

- pane 误混
- session 冲突
- cleanup 互相误杀

而 PID 命名意味着：

```text
每个 Claude 进程都有自己私有的 tmux swarm socket
```

这和前面看到的：

- session-local cleanup
- executor-local teammate tracking

是同一套隔离哲学。

---

### 第六部分：`TEAMMATE_COMMAND_ENV_VAR` 说明 process-based teammate 的启动入口覆盖能力，被明确建模成一个稳定的协议级环境变量，而不是临时调试技巧

位置：`src/utils/swarm/constants.ts:16`

这里定义：

- `CLAUDE_CODE_TEAMMATE_COMMAND`

而上一站 `spawnUtils.ts` 已经看到：

- `getTeammateCommand()` 会先读这个 env var
- 否则才回退到 `process.execPath` / `process.argv[1]`

这说明：

```text
teammate command override
```

不是某个文件里的私有实现细节，
而是 swarm 子系统公开承认的一条控制通道。

也就是说这个常量其实定义了一个：

```text
spawn-time override protocol
```

对部署、调试、特殊分发环境都很关键。

---

### 第七部分：`TEAMMATE_COLOR_ENV_VAR` 说明 teammate 的颜色不仅存在于 pane UI / identity 对象里，也被设计成可跨进程传递的环境语义

位置：`src/utils/swarm/constants.ts:23`

这里定义：

- `CLAUDE_CODE_AGENT_COLOR`

注释说明它用于：

- colored output
- pane identification

这说明颜色在 swarm 里不只是前端展示色值，
而是 teammate 身份的一部分，
并且需要在 process-based teammate 场景中跨进程传播。

所以它表达的是：

```text
agent color is part of runtime identity/context, not just local UI decoration
```

虽然前面 `PaneBackendExecutor` 主要通过 CLI flags 传 agent color，
但这里仍保留 env var 常量，
说明系统对“颜色语义跨边界传播”是有意识建模的。

---

### 第八部分：`PLAN_MODE_REQUIRED_ENV_VAR` 把“这名 teammate 必须先 plan”也提升成了运行时环境信号，说明 plan requirement 被视为派生进程的启动约束之一

位置：`src/utils/swarm/constants.ts:29`

这里定义：

- `CLAUDE_CODE_PLAN_MODE_REQUIRED`

注释说明：

- 设为 `'true'`
- teammate 必须先进入 plan mode 并获批再写代码

这点很关键。

因为我们前面在 `spawnUtils.ts` 看到的是：

- `planModeRequired` 会影响 CLI flags 继承
- 它还能压过 bypass permissions

而这里再给出 env 常量，说明 `plan mode required` 不只是一个当前内存态布尔值，
还是：

```text
spawned teammate contract signal
```

也就是说对 process-based teammate 来说，
它可以被编码进派生执行环境本身。

---

### 第九部分：这个文件最有价值的点，不是某个常量本身，而是它把“swarm 命名”和“swarm 协议信号”放在同一个地方统一收口

如果把全文件压缩，可以发现它定义的其实是两大类常量：

#### 1. 宿主命名 / 资源命名
- `SWARM_SESSION_NAME`
- `SWARM_VIEW_WINDOW_NAME`
- `HIDDEN_SESSION_NAME`
- `TMUX_COMMAND`
- `getSwarmSocketName()`

#### 2. 运行时协议 / 派生进程信号
- `TEAM_LEAD_NAME`
- `TEAMMATE_COMMAND_ENV_VAR`
- `TEAMMATE_COLOR_ENV_VAR`
- `PLAN_MODE_REQUIRED_ENV_VAR`

也就是说它不是单纯的“magic strings file”，
而更准确地说是：

```text
swarm namespace definition
```

这会让 backend、spawn、runtime、mailbox 在引用这些概念时都保持一致。

---

### 第十部分：这份文件和前面多站的关系，是“统一词汇表”对“具体实现层”的支撑

这一点值得串起来看。

#### `TmuxBackend.ts`
依赖：
- `SWARM_SESSION_NAME`
- `SWARM_VIEW_WINDOW_NAME`
- `HIDDEN_SESSION_NAME`
- `TMUX_COMMAND`
- `getSwarmSocketName()`

#### `spawnUtils.ts`
依赖：
- `TEAMMATE_COMMAND_ENV_VAR`

#### mailbox / control-plane 路径
依赖：
- `TEAM_LEAD_NAME`

#### process teammate 运行语义
依赖：
- `TEAMMATE_COLOR_ENV_VAR`
- `PLAN_MODE_REQUIRED_ENV_VAR`

所以可以非常准确地说：

```text
constants.ts 自己不做执行，
但它给 swarm backend / spawn / control plane 提供了统一词汇表
```

这类文件通常很小，
却能显著降低系统各处“同一概念多种写法”的漂移风险。

---

### 第十一部分：从架构成熟度看，这份文件说明 swarm 不是零散功能拼接，而是已经形成了稳定的内部协议表面

如果一个系统里这些名字都散落在各个文件里，
通常说明它还停留在：

- 局部实现先跑起来
- 再慢慢补收口

但这里已经能看到作者明确收口了：

- 保留控制角色名
- backend 资源名
- 宿主 socket 名
- spawn override env var 名
- runtime contract env var 名

这说明 swarm 子系统已经有一套比较稳定的内部协议面。

也就是说：

```text
这些常量不是偶然复用，
而是系统内部对“哪些名字值得被视为协议的一部分”已经有清楚判断
```

---

### 元

问题：**这一站真正想解决的架构问题是什么？**

回答：这一站的核心，不是几个字符串常量，而是 swarm 命名协议为什么必须集中定义。team lead 名称、session/window 名称、socket 名称和 env var 一旦分散，整个系统就会失去同一语言。

### 反

问题：**如果把这一站的设计反过来，会发生什么？**

回答：如果这些命名留给各处硬编码，最先坏掉的不是功能，而是协议一致性。backend、spawn、mailbox 和 runtime 会开始各说各话，问题也会变得很难定位。

### 空

问题：**跳出当前文件名，这一站背后更大的问题是什么？**

回答：更大的问题是，多模块系统怎样维护共享词汇表。`constants.ts` 的意义，是把“名字”提升成正式协议，而不是实现细节。

### 读完这一站后，你应该抓住的 8 个事实

1. `constants.ts` 是 swarm 子系统的共享命名与协议常量定义文件，不负责 backend 逻辑或 runtime 行为，只负责提供统一词汇表。
2. `TEAM_LEAD_NAME = 'team-lead'` 说明 team lead 在 swarm 中是一个正式保留的控制面身份名，而不是普通 teammate 名称。
3. `SWARM_SESSION_NAME`、`SWARM_VIEW_WINDOW_NAME`、`HIDDEN_SESSION_NAME` 把 tmux backend 的 external session / visible window / hidden pane parking space 正式命名化了。
4. `TMUX_COMMAND` 说明连宿主 CLI 命令名也被集中收口，避免 detection/backend/helpers 各自硬编码。
5. `getSwarmSocketName()` 用 `process.pid` 派生 socket 名，体现了 external tmux swarm socket 的 per-process 隔离模型，既不污染用户 tmux，也避免多 Claude 实例互撞。
6. `TEAMMATE_COMMAND_ENV_VAR` 把 process-based teammate 启动入口覆盖能力正式建模成一个稳定的环境变量协议。
7. `TEAMMATE_COLOR_ENV_VAR` 与 `PLAN_MODE_REQUIRED_ENV_VAR` 说明 teammate 的颜色与 plan-mode 约束都被视为可跨进程传播的运行时信号。
8. 这份文件真正的价值在于把“宿主资源命名”和“派生进程协议信号”统一收口，让 backend、spawn、mailbox、runtime 使用同一套内部命名空间。

---

### 现在把第 52-53 站串起来

```text
ITermBackend.ts
  -> 实现 iTerm2 native pane host 的 create/send/close 主路径，并暴露其能力短板
constants.ts
  -> 提供 swarm backend / spawn / control plane 共用的命名约定与协议常量
```

所以现在 backend 相关子系统可以先压缩成：

```text
shared protocol names
  -> constants.ts

tmux pane host implementation
  -> TmuxBackend.ts

iterm2 pane host implementation
  -> ITermBackend.ts

backend detection / policy
  -> detection.ts + registry.ts
```

也就是说：

```text
ITermBackend.ts 回答“iTerm2 这条 pane host 能做什么、做不到什么”
constants.ts 回答“这些 backend / spawn / control 面里的关键名字究竟统一定义在哪”
```

---

### 下一站建议

下一站最顺的是：

```text
src/utils/swarm/inbox.ts
```

因为现在你已经看懂了：

- backend 如何建 pane / 发命令
- `team-lead` 这类控制面身份名如何被统一定义
- process / pane teammate 的初始 prompt、shutdown request、普通消息都依赖 mailbox/inbox 语义

下一步最自然就是把这条控制面传输层补齐：

**swarm 的 inbox / mailbox 文件到底怎么组织、消息怎样落盘、读取/标记已读/轮询又是怎样支撑 leader 和 teammates 之间的通信协议的。**
