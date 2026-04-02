## 第 45 站：`src/utils/swarm/backends/PaneBackendExecutor.ts`

### 这是什么文件

`src/utils/swarm/backends/PaneBackendExecutor.ts` 是 swarm backend 体系里的 **pane backend → teammate lifecycle adapter**。

上一站 `backends/types.ts` 已经看懂：

```text
PaneBackend 只负责 pane 宿主能力
TeammateExecutor 负责 teammate 生命周期能力
```

这意味着系统里一定需要一层桥：

- 底层只有“开 pane / 发命令 / 改标题 / kill pane”
- 上层却想要“spawn teammate / sendMessage / terminate / kill / isActive”

所以这一站回答的是：

```text
tmux / iTerm2 这类 pane backend，怎样被包装成统一的 TeammateExecutor 接口
```

所以最准确的一句话是：

```text
PaneBackendExecutor.ts = pane 宿主能力到 teammate 生命周期能力的适配器
```

---

### 先看它的整体定位

位置：`src/utils/swarm/backends/PaneBackendExecutor.ts:1`

文件头注释已经写得非常清楚：

这层 adapter 负责把 pane backend 的能力翻译成：

- `spawn()`
- `sendMessage()`
- `terminate()`
- `kill()`
- `isActive()`

并且具体对应关系也说了：

- `spawn()` -> 建 pane + 发 Claude CLI 命令
- `sendMessage()` -> 写 mailbox
- `terminate()` -> 发 shutdown request mailbox
- `kill()` -> 直接 kill pane
- `isActive()` -> 看 pane 是否仍被追踪

这说明这份文件不是：

- 具体 tmux backend 实现
- 具体 iTerm2 backend 实现
- team runtime 本体

而是明确在做：

```text
low-level pane API -> high-level teammate executor API
```

这就是一个很标准的 adapter。

---

### 第一部分：`PaneBackendExecutor implements TeammateExecutor` 直接把“适配”变成正式类型契约

位置：`src/utils/swarm/backends/PaneBackendExecutor.ts:39`

类定义直接就是：

- `export class PaneBackendExecutor implements TeammateExecutor`

这意味着这份文件不是“帮忙调用一下 pane backend”的松散工具函数，
而是要正式承担：

```text
对上层伪装成一个完整的 teammate executor
```

内部则持有：

- `backend: PaneBackend`
- `context: ToolUseContext | null`
- `spawnedTeammates: Map<string, { paneId, insideTmux }>`
- `cleanupRegistered`

这几个字段已经把它的角色暴露得很清楚：

#### `backend`
底层宿主能力提供者

#### `context`
让 executor 能访问当前工具调用时的 AppState / permission context

#### `spawnedTeammates`
维护逻辑身份到 pane 资源的映射

#### `cleanupRegistered`
避免重复注册 leader-exit cleanup

所以这个类的本质可以压缩成：

```text
它一手握住 pane backend，一手握住 team/tool runtime context，
中间再维护 agentId -> paneId 的映射表
```

---

### 第二部分：`setContext()` 揭示了 pane executor 不是纯无状态 backend adapter，而是依赖当前 ToolUseContext 的 runtime object

位置：`src/utils/swarm/backends/PaneBackendExecutor.ts:58`

这个方法非常关键：

- `setContext(context: ToolUseContext)`

而且注释明确写了：

```text
Must be called before spawn()
```

这说明 `PaneBackendExecutor` 不是一个只靠传入 `PaneBackend` 就能工作的纯适配器。

它还需要知道：

- 当前 AppState
- 当前 permission mode
- 当前 query/tool use 环境

因为 spawn 时要继承一堆 leader 上下文：

- inherited CLI flags
- permission mode
- 也可能有别的 runtime config

所以它实际上是：

```text
backend adapter + current tool/session context bridge
```

这也解释了为什么 `spawn()` 里如果没有 context 会直接失败。

---

### 第三部分：`spawnedTeammates` 显示 pane backend 路径仍然需要一份本地运行时映射表

位置：`src/utils/swarm/backends/PaneBackendExecutor.ts:45`

这里有一张 map：

- key: `agentId`
- value: `{ paneId, insideTmux }`

注释写得很明确：

```text
这样 kill / terminate 等操作才能找到对应 pane
```

这说明即便 pane-based teammate 已经有：

- mailbox
- team file
- backendType
- paneId 之类持久信息

`PaneBackendExecutor` 仍然保留一份自己的 session-local tracking map。

原因很明显：

```text
executor 需要在当前进程内快速完成 agentId -> pane control routing
```

不然每次 `kill(agentId)` 都还得回头查 team file 或其他状态源。

所以这里体现的是：

```text
持久化 team metadata 之外，executor 自己还维护一份操作用的本地索引
```

---

### 第四部分：`spawn()` 的第一步是 `formatAgentId(config.name, config.teamName)`，说明 pane backend 路径也完全遵守统一 agent identity 模型

位置：`src/utils/swarm/backends/PaneBackendExecutor.ts:79`

这一步很重要：

- `const agentId = formatAgentId(config.name, config.teamName)`

说明 pane backend teammate 和 in-process teammate 一样，
逻辑身份主键都是：

```text
agentName@teamName
```

也就是说：

```text
backend 不同，不改变 teammate 的逻辑身份模型
```

这对后面统一：

- sendMessage(agentId)
- terminate(agentId)
- kill(agentId)
- team file member agentId

都非常关键。

---

### 第五部分：`spawn()` 先 guard `context`，说明 pane teammate 的创建不仅依赖宿主能力，还依赖 leader 当前 query 状态

位置：`src/utils/swarm/backends/PaneBackendExecutor.ts:82`

如果没有 `context`，这里直接返回失败：

- `success: false`
- `error: PaneBackendExecutor not initialized...`

这进一步证明：

```text
pane teammate 的 spawn 不是一个纯“开终端窗口”动作，
而是一次从 leader 当前运行环境继承配置的控制面动作
```

缺少 `ToolUseContext` 时，它无法正确继承：

- AppState 中的 permission mode
- 可能的上下文 flags
- 当前会话相关状态

所以 `PaneBackendExecutor` 的 spawn 本质是：

```text
pane creation + teammate CLI bootstrap with inherited runtime policy
```

---

### 第六部分：`spawn()` 先分配颜色、再建 pane、再决定 insideTmux 行为，说明 pane teammate 启动是“视觉布局 + 执行引导”一体化流程

位置：`src/utils/swarm/backends/PaneBackendExecutor.ts:95`

前半段流程是：

1. 选颜色：
   - `config.color ?? assignTeammateColor(agentId)`
2. 建 pane：
   - `backend.createTeammatePaneInSwarmView(...)`
3. 看当前是不是 inside tmux：
   - `isInsideTmux()`
4. 如果是第一个 teammate 且 inside tmux：
   - `enablePaneBorderStatus()`

这里很有意思，因为它说明 spawn 并不只是启动 agent 进程，
而是先把 pane 作为 swarm UI 的一部分布置好。

也就是说 pane backend teammate 的出生链里，
视觉宿主初始化是第一等公民：

```text
先让它在 swarm 视图里有位置、有颜色、有标题能力，
再把 Claude 命令注进去
```

这和 in-process teammate 明显不同。

#### in-process
先是 task/runtime state 实体

#### pane-based
先是 pane host 实体，再把 teammate loop 注入进去

---

### 第七部分：`teammateArgs` 暴露了 pane teammate 的身份注入方式：不是 ALS，而是 CLI flags

位置：`src/utils/swarm/backends/PaneBackendExecutor.ts:116`

这里给新 pane 里的 Claude Code 命令拼了：

- `--agent-id`
- `--agent-name`
- `--team-name`
- `--agent-color`
- `--parent-session-id`
- `--plan-mode-required`

这说明 pane teammate 的身份不是像 in-process 那样靠 `AsyncLocalStorage` 注入，
而是：

```text
process bootstrap time identity injection via CLI args
```

这正好和前面 `teammate.ts` / `dynamicTeamContext` 的理解吻合：

- in-process -> ALS context
- process/pane teammate -> 启动参数 + 运行时 context

所以这一段就是 process-based teammate 身份链的入口之一。

---

### 第八部分：`buildInheritedCliFlags(...)` + model override 说明 pane teammate 会继承 leader 的执行策略，但允许在 spawn config 层局部覆盖

位置：`src/utils/swarm/backends/PaneBackendExecutor.ts:128`

这里先从当前 `ToolUseContext` 取：

- `appState.toolPermissionContext.mode`

然后构建：

- inherited CLI flags

同时如果 `config.model` 存在，
会专门把继承来的 `--model` 剔掉，再追加新的 `--model`。

这说明 pane teammate 启动策略有两个层级：

#### 1. Leader runtime inheritance
继承当前：
- permission mode
- plan mode 相关 flag
- 其他基础 CLI 行为

#### 2. Teammate-specific override
比如：
- 单独指定 model

所以这段逻辑真正表达的是：

```text
teammate 不是从零开始启动的全新 Claude CLI，
而是 leader runtime policy 的受控派生实例
```

这和 swarm 的“团队协作体”模型完全一致。

---

### 第九部分：`buildInheritedEnvVars()` 说明 pane teammate 除了 CLI flags 外，还会继承 leader 的部分环境语义

位置：`src/utils/swarm/backends/PaneBackendExecutor.ts:151`

这里还会构造：

- `envStr = buildInheritedEnvVars()`

最后命令长成：

```text
cd <cwd> && env <inherited env> <binary> <teammateArgs> <flags>
```

这说明 pane teammate 的启动继承不只靠 flags，
还有环境变量层面的传播。

所以 process-based teammate 的 bootstrap 其实是：

```text
working directory
+ inherited env
+ inherited CLI flags
+ explicit teammate identity flags
```

共同决定的。

这就是为什么 `PaneBackendExecutor` 并不只是“把一条 prompt 发过去”，
而是在构造一个完整的派生 CLI 进程环境。

---

### 第十部分：`sendCommandToPane(...)` 才是 pane backend executor 最核心的桥接动作——把 teammate spawn 语义转成 pane 内 shell command 注入

位置：`src/utils/swarm/backends/PaneBackendExecutor.ts:156`

前面所有准备，最后都落在：

- `await this.backend.sendCommandToPane(paneId, spawnCommand, !insideTmux)`

也就是说对 pane backend 来说，所谓“spawn teammate”，
底层其实就是：

```text
先创建 pane
再往 pane 里发送一条启动 Claude Code teammate 的 shell 命令
```

这非常关键，因为它揭示了 pane teammate 模型的真实本质：

```text
pane backend 不负责直接创建 agent runtime object，
而是负责创建一个终端宿主，然后通过命令注入把 teammate 进程跑起来
```

这与 in-process backend 完全不同。

---

### 第十一部分：`insideTmux` 被记进 tracking map，说明 external session 与 current tmux session 的差异会影响后续控制路径

位置：`src/utils/swarm/backends/PaneBackendExecutor.ts:160`

spawn 完成后会记录：

- `paneId`
- `insideTmux`

后面 `kill()` 时会再用到：

- `backend.killPane(paneId, !insideTmux)`

这说明“当时是 inside tmux 还是通过 external swarm session 创建的 pane”并不是一次性信息，
而会影响后续对该 pane 的控制方式。

所以 executor 记录的不只是“在哪个 pane”，
还记录：

```text
后续控制这个 pane 时该用哪种 session/socket 语义
```

这是很典型的 runtime control metadata。

---

### 第十二部分：cleanup 注册说明 pane teammate 的 leader-exit 回收由 executor 统一兜底

位置：`src/utils/swarm/backends/PaneBackendExecutor.ts:163`

这里第一次 spawn 成功后，会注册一次 cleanup：

- 遍历 `spawnedTeammates`
- 挨个 `killPane(...)`
- 然后清空 map

这说明 pane teammate 的孤儿回收并不只靠 `teamHelpers.ts` 那层 session cleanup。

`PaneBackendExecutor` 自己也会在当前执行器生命周期里登记一层：

```text
如果 leader 退出，至少把本 executor 亲手 spawn 的 pane 都杀掉
```

这是一种很合理的双保险：

- executor 本地知道自己 spawn 了谁
- team/session cleanup 知道整个 team 还有哪些资源

所以 cleanup 策略是分层的，不是单点依赖。

---

### 第十三部分：spawn 后“初始 prompt”不是通过 CLI stdin 传入，而是通过 mailbox 投递

位置：`src/utils/swarm/backends/PaneBackendExecutor.ts:177`

这里非常关键：

pane 里虽然已经启动了 Claude CLI teammate，
但真正的初始任务 prompt 并不是命令行直接塞进去，
而是随后：

- `writeToMailbox(config.name, { from: 'team-lead', text: config.prompt, ... }, config.teamName)`

这说明 pane-based teammate 的启动链是两段式：

#### 第一段
```text
在 pane 里启动带 team identity 的 Claude Code 进程
```

#### 第二段
```text
再通过 mailbox 投递初始工作任务
```

所以 process-based teammate 的运行模型从一开始就是：

```text
process bootstrap
+
mailbox-delivered work
```

而不是“命令启动时一次性把 prompt 打进去”。

这非常关键，因为它把：

- teammate identity/bootstrap
- teammate conversation/task delivery

明确分开了。

---

### 第十四部分：`sendMessage()` 说明 pane teammate 的普通通信完全走 mailbox，而不尝试直接往 pane 写 stdin

位置：`src/utils/swarm/backends/PaneBackendExecutor.ts:216`

这里做的不是：

- `sendCommandToPane(...)`

而是：

1. `parseAgentId(agentId)`
2. 拆出 `agentName` / `teamName`
3. `writeToMailbox(...)`

文件注释也明确说：

```text
All teammates (pane and in-process) use the same mailbox mechanism.
```

这句话非常关键。

它说明在高层 teammate 通信语义上，系统刻意统一成：

```text
消息层统一 -> mailbox semantics
```

而不是：

- pane teammate 用 stdin/pty 注入
- in-process teammate 用 AppState

至少在 executor 层，这里选择的统一表达是 mailbox。

所以 `PaneBackendExecutor` 对 sendMessage 的适配思路不是“利用 pane 特权直接注输入”，
而是：

```text
遵循 swarm 统一 teammate message transport
```

---

### 第十五部分：`terminate()` 不是 kill pane，而是发结构化 `shutdown_request`，说明 graceful shutdown 被视为协议层语义

位置：`src/utils/swarm/backends/PaneBackendExecutor.ts:252`

这里 terminate 的做法是：

- 先 `parseAgentId(...)`
- 构造：
  - `type: 'shutdown_request'`
  - `requestId`
  - `from: 'team-lead'`
  - `reason`
- 再写入 mailbox

这说明对 pane-based teammate 来说，
优雅结束不是宿主层动作，而是：

```text
protocol-level request sent through mailbox
```

也就是说：

- `terminate()` = 协作式关闭请求
- `kill()` = 宿主级强制终止

这和前面几站整个 shutdown request / approval 模型完全对上。

所以 `PaneBackendExecutor` 在这里做的适配非常干净：

```text
把高层 terminate 语义翻译成 swarm mailbox protocol message
```

---

### 第十六部分：`kill()` 才真正落到底层 pane 宿主控制层

位置：`src/utils/swarm/backends/PaneBackendExecutor.ts:295`

`kill()` 的逻辑就完全不同了：

- 从 `spawnedTeammates` 找到 pane
- 调 `backend.killPane(...)`
- 成功则从 map 删除

这说明这里没有任何 mailbox 协议，
也不等待 teammate 自己配合退出。

所以这一步真正表达的是：

```text
force kill = host resource termination
```

与之对照：

```text
terminate = protocol request
kill = pane resource kill
```

这个分层非常干净，
也正说明了 `PaneBackendExecutor` 确实把“协议层 teammate 生命周期”和“宿主层 pane 控制”区分开了。

---

### 第十七部分：`isActive()` 目前只是 best-effort，本质上依赖本地 tracking map，而不是强一致 runtime 探测

位置：`src/utils/swarm/backends/PaneBackendExecutor.ts:329`

这里的实现很坦率：

- 如果 `spawnedTeammates` 里没有 -> false
- 否则 -> true

注释直接承认：

```text
pane 可能还在，但里面进程已经死了；
更可靠的做法是给 PaneBackend 再加一个“查 pane 是否存在”的方法
```

这说明当前 `isActive()` 的语义不是强一致 liveness check，
而是：

```text
executor-local best-effort activity estimate
```

换句话说：

```text
只要这个 executor 还记得它 spawn 过这个 agent，
并且还没显式 kill/remove，就先当它 active
```

这是一个当前实现上的边界，也是在读代码时很值得记住的限制。

---

### 第十八部分：这份文件把 pane teammate 的真实运行模式完全暴露出来了——“pane 宿主 + CLI bootstrap + mailbox protocol”三段式

如果把整份文件压缩，会发现 pane-based teammate 不是单一路径，
而是三层东西叠起来：

#### 1. pane host layer
- create pane
- style pane
- kill pane

#### 2. process bootstrap layer
- 构造 teammate CLI args / env / flags
- `sendCommandToPane(...)`

#### 3. teammate protocol layer
- 初始 prompt 走 mailbox
- 后续消息走 mailbox
- terminate 走 shutdown request mailbox

所以 pane teammate 的最准确模型其实是：

```text
visual pane host
  +
spawned Claude CLI process with identity flags
  +
mailbox-based control/message protocol
```

这就是 `PaneBackendExecutor` 真正封装起来的东西。

---

### 元

问题：**这一站真正想解决的架构问题是什么？**

回答：这一站把 tmux/iTerm2 这类 pane backend 包装成统一的 `TeammateExecutor`。它把“开 pane、发命令、kill pane”翻译成“spawn teammate、sendMessage、terminate、isActive”等上层生命周期语义。

### 反

问题：**如果把这一站的设计反过来，会发生什么？**

回答：如果 pane host 细节直接暴露给上层，tmux 和 iTerm2 的差异会迅速渗透到业务代码。适配器缺失后，多后端统一只是口号。

### 空

问题：**跳出当前文件名，这一站背后更大的问题是什么？**

回答：更大的问题是，底层宿主能力如何升格为平台级生命周期接口。adapter 模式在这里不是形式，而是必要桥梁。


### 读完这一站后，你应该抓住的 10 个事实

1. `PaneBackendExecutor.ts` 是把 `PaneBackend` 适配成 `TeammateExecutor` 的正式 adapter，不是简单工具函数集合。
2. 它不仅持有底层 pane backend，还依赖 `ToolUseContext`，说明 pane teammate 的 spawn 要继承 leader 当前 runtime policy。
3. `spawnedTeammates` 维护 `agentId -> {paneId, insideTmux}` 的本地索引，用于后续 kill/terminate/control routing。
4. pane teammate 的 `spawn()` 本质上是：创建 pane、构造 teammate CLI 启动命令、把命令注入 pane、再通过 mailbox 投递初始 prompt。
5. pane teammate 的身份注入方式是 CLI flags（`--agent-id` / `--team-name` / `--parent-session-id` 等），而不是 in-process 路径里的 AsyncLocalStorage。
6. `buildInheritedCliFlags()` 和 `buildInheritedEnvVars()` 说明 pane teammate 是 leader 当前运行环境的派生实例，而不是从零配置启动的独立 CLI。
7. 初始任务和后续 `sendMessage()` 都走 mailbox，而不是直接把文本写进 pane stdin，这说明 teammate 协议层统一优先使用 mailbox transport。
8. `terminate()` 被翻译成结构化 `shutdown_request` mailbox 消息，体现 graceful shutdown 是协议语义；`kill()` 则直接调用 `killPane()`，体现 force kill 是宿主资源语义。
9. executor 自己会注册 cleanup，在 leader 退出时 best-effort 杀掉本实例 spawn 的所有 panes，形成一层本地兜底回收。
10. `isActive()` 当前只是基于 tracking map 的 best-effort 判断，而不是强一致的 pane/process 存活探测。

---

### 现在把第 44-45 站串起来

```text
backends/types.ts
  -> 定义 PaneBackend 与 TeammateExecutor 这两层抽象边界
PaneBackendExecutor.ts
  -> 真正把 PaneBackend 适配成 TeammateExecutor，并把 pane host、CLI bootstrap、mailbox protocol 组合起来
```

所以现在 pane backend 这条线可以先压缩成：

```text
low-level host API
  -> PaneBackend

adapter layer
  -> PaneBackendExecutor

high-level lifecycle API
  -> TeammateExecutor
```

也就是说：

```text
types.ts 定义“该如何分层”
PaneBackendExecutor.ts 实现“如何把 pane 层拼成 teammate 生命周期层”
```

---

### 下一站建议

下一站最顺的是：

```text
src/utils/swarm/backends/InProcessBackend.ts
```

因为现在你已经看懂了：

- pane backend 怎样被 adapter 成 `TeammateExecutor`
- process-based teammate 的 bootstrap 依赖 pane + CLI flags + mailbox
- `TeammateExecutor` 这层统一接口已经建立起来了

下一步最自然就是看另一半对照实现：

**in-process backend 到底怎样直接实现同一套 `TeammateExecutor` 接口、怎样对接 `spawnInProcess.ts` / AppState / teammate task，而不再经过 pane host 与 CLI bootstrap 这条外部进程路径。**
