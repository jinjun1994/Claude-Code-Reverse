## 第 51 站：`src/utils/swarm/backends/TmuxBackend.ts`

### 这是什么文件

`src/utils/swarm/backends/TmuxBackend.ts` 是 swarm backend 体系里的 **tmux pane host 具体实现**。

上一站 `it2Setup.ts` 已经把 iTerm2 路线的 setup / preference 层补上了，
而更早前也已经看过：

- `types.ts`：定义 `PaneBackend`
- `registry.ts`：决定什么时候选 tmux
- `PaneBackendExecutor.ts`：把 pane backend 适配成 `TeammateExecutor`

但那时还没真正落到：

- tmux backend 自己怎样建 pane
- 在 leader 已经身处 tmux 时怎样沿 leader 窗口扩展
- 在 leader 不在 tmux 时怎样起 external swarm session
- pane hide/show / rebalance / send command 到底怎么做

所以这一站回答的是：

```text
tmux backend 作为 PaneBackend，
到底怎样把 tmux 宿主能力实现成 swarm 所需的 pane 管理与控制操作
```

所以最准确的一句话是：

```text
TmuxBackend.ts = swarm 在 tmux 上的 pane 宿主实现层
```

---

### 先看它的整体定位

位置：`src/utils/swarm/backends/TmuxBackend.ts:1`

这个文件负责的是 `PaneBackend` 这一层，不是：

- backend 选择
- teammate lifecycle 适配
- CLI flags/env 注入
- mailbox 协议

它只负责 tmux 宿主能力本身，例如：

- 建 pane
- 给 pane 发命令
- 调整 pane 布局
- 设置 pane 标题和颜色
- kill / hide / show pane

所以它是一个很纯的：

```text
tmux host operations layer
```

也就是说 `PaneBackendExecutor` 负责“把 pane host 拼成 teammate lifecycle”，
而 `TmuxBackend` 负责“tmux 这类 pane host 自己能做什么”。

---

### 第一部分：文件顶部三个全局变量直接暴露了 tmux backend 的三个核心运行时问题——external 首 pane复用、leader window 定位缓存、并发建 pane 竞态控制

位置：`src/utils/swarm/backends/TmuxBackend.ts:22`

顶部有三个全局状态：

- `firstPaneUsedForExternal`
- `cachedLeaderWindowTarget`
- `paneCreationLock`

它们分别对应三件非常具体的事：

#### 1. `firstPaneUsedForExternal`
外部 swarm session 第一个 pane 是否已被占用

#### 2. `cachedLeaderWindowTarget`
leader 所在 window target 是否已解析过

#### 3. `paneCreationLock`
多个 teammate 并发 spawn 时，tmux split 是否需要串行化

这说明 tmux backend 最真实的复杂度不在“发一条 tmux 命令”，
而在：

```text
同一会话的布局状态管理 + 定位缓存 + 并发正确性
```

---

### 第二部分：`paneCreationLock` 说明作者明确把“并发创建 pane”视为有竞态风险的操作，而不是可以随便并行跑的 tmux 命令序列

位置：`src/utils/swarm/backends/TmuxBackend.ts:28`

这里用了一个 promise lock：

- `paneCreationLock: Promise<void> = Promise.resolve()`
- `acquirePaneCreationLock()` 返回 release 函数

然后在 `createTeammatePaneInSwarmView()` 开头：

- `const releaseLock = await acquirePaneCreationLock()`
- `finally { releaseLock() }`

这说明作者知道 tmux pane 创建不是幂等的纯查询操作。

因为建 pane 会依赖当下瞬时状态：

- 当前 pane 数
- 哪个 pane 应该被 split
- 第一个 external pane 是否已经被复用
- rebalance 应该按哪种布局来

如果并发调用同时读这些状态，就会产生：

```text
多个 spawn 都以为自己在操作同一份旧布局
```

所以这里本质上是在做：

```text
pane topology mutation serialization
```

这是这个文件非常关键的正确性设计。

---

### 第三部分：`runTmuxInUserSession()` 与 `runTmuxInSwarm()` 的并列，揭示 tmux backend 的真正核心分叉不是“同一个 tmux”，而是“用户原始 session”与“外部 swarm socket”两套控制面

位置：`src/utils/swarm/backends/TmuxBackend.ts:73`

这里定义了两类命令入口：

#### `runTmuxInUserSession(args)`
- 直接 `tmux ...`
- 面向用户原始 tmux session

#### `runTmuxInSwarm(args)`
- `tmux -L <swarm-socket> ...`
- 面向 standalone swarm session

这非常重要。

说明 tmux backend 并不是简单“总是在同一个 tmux server 上 split pane”，
而是明确支持两种完全不同的运行模式：

#### native tmux mode
leader 本来就在 tmux 里

#### external tmux mode
leader 不在 tmux 里，但系统另外起一个 swarm 专用 tmux socket/session

也就是说，这个文件的主线不是：

```text
tmux backend
```

而更准确地说是：

```text
tmux native-host mode
+
external swarm-session mode
```

的统一实现。

---

### 第四部分：类头注释把两种布局策略说得很清楚——inside tmux 保留 leader 30% / teammates 70%，outside tmux 则纯 teammate 平铺

位置：`src/utils/swarm/backends/TmuxBackend.ts:94`

注释写得非常关键：

#### 当 leader 在 tmux 里
- 在 leader 当前 window 里继续 split
- leader 在左侧 30%
- teammates 在右侧 70%

#### 当 leader 不在 tmux 里
- 建 `claude-swarm` session
- 建 `swarm-view` window
- 所有 teammates 等权分布

这说明 tmux backend 不只是技术上支持两种执行路径，
还为两种场景设计了不同的视觉组织原则：

```text
有 leader pane 时：leader-centered layout
无 leader pane 时：peer-only tiled layout
```

这点很关键，
因为它说明 layout 不是附带效果，
而是 backend 语义的一部分。

---

### 第五部分：`isAvailable()` / `isRunningInside()` 都委托给 detection 层，说明具体 backend 实现并不重复定义宿主事实，而是消费 detection.ts 的结论

位置：`src/utils/swarm/backends/TmuxBackend.ts:109`

这里没有自己再写：

- `tmux -V`
- `process.env.TMUX`
- `TMUX_PANE`

而是直接调用：

- `isTmuxAvailable()`
- `isInsideTmuxFromDetection()`

这说明模块边界很健康：

```text
detection.ts 负责环境事实
TmuxBackend.ts 负责基于这些事实执行 tmux host 操作
```

也就是说 backend 实现不是“自带探测器”，
而是“消费探测结果的执行器”。

---

### 第六部分：`createTeammatePaneInSwarmView()` 说明 tmux backend 对外只暴露一个统一入口，但内部会按 inside/outside tmux 分流到两条完全不同的 pane 创建链

位置：`src/utils/swarm/backends/TmuxBackend.ts:125`

对外 API 很简单：

- `createTeammatePaneInSwarmView(name, color)`

内部却做了：

1. 获取 lock
2. `insideTmux = await this.isRunningInside()`
3. `insideTmux ? createTeammatePaneWithLeader(...) : createTeammatePaneExternal(...)`

这说明 `PaneBackend` 抽象层看到的是统一的“创建 teammate pane”，
但 tmux backend 自身清楚知道：

```text
同一个动作在 native tmux 与 external swarm session 中，
底层拓扑与控制路径完全不同
```

所以这份文件的重要设计不是只实现一个 create，
而是：

```text
把两条宿主拓扑路径压平到同一 PaneBackend 合同里
```

---

### 第七部分：`sendCommandToPane()` 的实现很直白，但它说明 tmux backend 的“运行 teammate”本质仍然只是往 pane 里 `send-keys`

位置：`src/utils/swarm/backends/TmuxBackend.ts:148`

这里做的是：

- `tmux send-keys -t <paneId> <command> Enter`

这再次坐实了之前 `PaneBackendExecutor` 的理解：

```text
pane backend 并不直接创建 agent runtime object，
只是创建一个终端宿主并向其中注入 shell command
```

而 `TmuxBackend` 就是这件事在 tmux 上的最低层实现。

所以它的职责非常清晰：

- 它不懂 teammate 协议
- 不懂 mailbox
- 不懂 agent loop
- 只懂“往哪个 pane 发什么命令”

---

### 第八部分：`setPaneBorderColor()` / `setPaneTitle()` / `enablePaneBorderStatus()` 说明 tmux backend 把 pane 的视觉识别能力当成正式接口能力，而不是纯 UI 装饰

位置：

- `setPaneBorderColor()` `src/utils/swarm/backends/TmuxBackend.ts:166`
- `setPaneTitle()` `src/utils/swarm/backends/TmuxBackend.ts:205`
- `enablePaneBorderStatus()` `src/utils/swarm/backends/TmuxBackend.ts:231`

这里连续做了：

- 设置 pane-specific border style
- 设置 pane title
- 设置 pane border format
- 打开 `pane-border-status top`

这说明 swarm 对 tmux pane 的使用不是匿名终端格子，
而是明确希望每个 pane 成为：

```text
可识别、可区分、带 teammate 身份标签的视觉单元
```

因此颜色和标题在这里不是 cosmetic extra，
而是 swarm UI 的组成部分。

---

### 第九部分：`getCurrentPaneId()` / `getCurrentWindowTarget()` 明确围绕“leader 原始 pane/window 锚点”设计，而不是跟随用户后来切换的 pane/window

位置：

- `getCurrentPaneId()` `src/utils/swarm/backends/TmuxBackend.ts:365`
- `getCurrentWindowTarget()` `src/utils/swarm/backends/TmuxBackend.ts:394`

这两段最关键的点是：

- 优先使用 `getLeaderPaneId()`
- `getCurrentWindowTarget()` 还会缓存 leader window target

注释明确说：

```text
即使用户之后切到别的 pane/window，
仍要锚定 leader 原始 pane/window
```

这和上一站 detection.ts 的理解完全连上了：

- `ORIGINAL_TMUX_PANE` 被保存下来
- 这里继续把它用作 pane layout 的 anchor

所以真正的设计语义是：

```text
swarm tmux layout 围绕 leader original pane/window 建模，
而不是围绕“用户此刻光标在哪”建模
```

这对布局稳定性至关重要。

---

### 第十部分：`createExternalSwarmSession()` 揭示了 external tmux 模式的真实资源模型——独立 socket、独立 session、固定 `swarm-view` window

位置：`src/utils/swarm/backends/TmuxBackend.ts:465`

这里的流程是：

1. 检查 swarm socket 下是否已有 `SWARM_SESSION_NAME`
2. 没有就：
   - `new-session -d -s <session> -n <window>`
3. 有 session 但没有 `swarm-view` window：
   - `new-window`
4. 如果 window 已有，直接返回它的首 pane

这说明 outside-tmux 模式不是临时起几个 pane 就完事，
而是有自己完整的宿主结构：

```text
external swarm socket
  -> swarm session
    -> swarm-view window
      -> panes
```

也就是说 external tmux 不是用户当前 tmux 的一个附属 window，
而是 swarm 自己独立维护的一套 tmux 世界。

---

### 第十一部分：`createTeammatePaneWithLeader()` 的第一位 teammate 和后续 teammate 走不同 split 策略，说明“leader + teammate 区域”的布局是有显式结构意图的

位置：`src/utils/swarm/backends/TmuxBackend.ts:548`

inside tmux 时：

#### 第一位 teammate
- 直接从 leader pane 水平 split
- 大小 `70%`
- 形成 leader 左 30%、teammate 区域右 70%

#### 后续 teammate
- 不再继续从 leader split
- 而是从已有 teammate panes 中选一个 split
- `teammatePanes = panes.slice(1)`

这说明作者明确不想让多个 teammates 不断侵蚀 leader pane。

真正的拓扑意图是：

```text
leader pane 是固定主 pane
teammates 都生长在右侧 teammate 区域内部
```

这是一个很重要的布局语义。

---

### 第十二部分：后续 teammate 的 split 方向是交替的，而且目标 pane 取中间索引，说明它在追求一种近似平衡树式的 teammate 子布局

位置：`src/utils/swarm/backends/TmuxBackend.ts:582`

这里算法很有意思：

- `teammateCount = teammatePanes.length`
- `splitVertically = teammateCount % 2 === 1`
- `targetPaneIndex = Math.floor((teammateCount - 1) / 2)`
- 从该 pane 去 split

这说明作者不是简单总从最后一个 pane split，
而是在努力让 teammate 区域保持较均衡分裂。

也就是说这里做的是一种很轻量的：

```text
balanced-ish pane subdivision heuristic
```

然后再配合 `rebalancePanesWithLeader()` 做整体整理。

这说明布局生成是：

- 局部启发式 split
- 之后再全局 rebalance

两层结合的。

---

### 第十三部分：`createTeammatePaneExternal()` 与 inside-tmux 版的最大差异，是第一位 teammate 直接复用 session 初始 pane，而不是立即再 split 一个新 pane

位置：`src/utils/swarm/backends/TmuxBackend.ts:632`

这里有个关键状态：

- `firstPaneUsedForExternal`

逻辑是：

#### 第一个 external teammate
- 直接使用 `createExternalSwarmSession()` 创建出来的首 pane
- 不额外 split
- 然后标记 `firstPaneUsedForExternal = true`

#### 后续 external teammates
- 才开始在 swarm-view window 里 split

这说明 external 模式里，首 pane 不是“空壳宿主”，
而是会被真实 teammate 复用。

所以 external swarm session 的第一个 pane 同时承担：

- session/window 初始化产物
- 第一名 teammate 宿主

这就是为什么需要额外的 `firstPaneUsedForExternal` 状态位。

---

### 第十四部分：inside/outside tmux 两条路径的 rebalance 策略完全不同——一个是 `main-vertical + leader 30%`，一个是纯 `tiled`

位置：

- `rebalancePanesWithLeader()` `src/utils/swarm/backends/TmuxBackend.ts:704`
- `rebalancePanesTiled()` `src/utils/swarm/backends/TmuxBackend.ts:736`

#### 有 leader 时
- `select-layout main-vertical`
- `resize-pane leader -x 30%`

#### 无 leader 时
- `select-layout tiled`

这再次说明 swarm 把两种模式的空间语义区分得非常清楚：

```text
leader present -> asymmetric layout
leader absent -> symmetric peer layout
```

而且 `showPane()` 恢复 hidden pane 之后也会重新套用有 leader 的那套布局，
这说明 layout 不是一次性的初始化动作，
而是这个 backend 长期维持的拓扑不变量。

---

### 第十五部分：`hidePane()` / `showPane()` 说明 tmux backend 的 hide/show 不是视觉开关，而是真正把 pane 在 session 之间迁移

位置：

- `hidePane()` `src/utils/swarm/backends/TmuxBackend.ts:277`
- `showPane()` `src/utils/swarm/backends/TmuxBackend.ts:308`

`hidePane()` 做的是：

- `new-session -d -s <hidden-session>`
- `break-pane -d -s <pane> -t <hidden-session>:`

`showPane()` 做的是：

- `join-pane -h -s <pane> -t <target>`
- 然后重新 `select-layout`
- 再把 leader 调回 30%

这说明 tmux backend 的 hide/show 语义其实是：

```text
从当前可见布局中移走 pane
<->
再把该 pane 接回目标窗口
```

而不是简单切换某个 display flag。

所以这里是���准的：

```text
pane relocation semantics
```

这也解释了为什么 `supportsHideShow = true` 在 tmux 上是成立的。

---

### 第十六部分：`waitForPaneShellReady()` 那个 200ms delay 说明作者明确承认“pane 刚创建好 ≠ shell 已经准备好接命令”

位置：`src/utils/swarm/backends/TmuxBackend.ts:31`

这里有：

- `PANE_SHELL_INIT_DELAY_MS = 200`
- `sleep(200)`

而且注释明确提到：

- shell init
- rc files
- prompts
- starship / oh-my-zsh

这说明 pane backend 这里解决的是一个很实际的问题：

```text
tmux split 成功返回 pane_id 后，
新 pane 里的 shell 可能还没初始化完，立即 send-keys 可能不稳定
```

所以这里用一个很轻量但明确的 host-readiness delay，
保证后面 `PaneBackendExecutor` 可以“创建完 pane 就马上发启动命令”。

这是典型的终端宿主现实兼容层。

---

### 第十七部分：`registerTmuxBackend(TmuxBackend)` 的顶层副作用再次说明 backend registry 采用“具体 backend 自注册”模型来规避循环依赖

位置：`src/utils/swarm/backends/TmuxBackend.ts:761`

文件底部直接：

- `registerTmuxBackend(TmuxBackend)`

这和之前 `registry.ts` 里：

- `ensureBackendsRegistered()` 动态 import backend 模块

正好闭环。

也就是说这里的架构模式是：

```text
registry 负责触发模块加载
backend 模块在加载时自注册 constructor
```

这能避免 registry 直接静态 import 具体 backend 引起的循环依赖。

所以 `TmuxBackend.ts` 不只是 class 定义文件，
还是 registry bootstrap 过程中的一个 side-effect module。

---

### 第十八部分：整份文件真正封装的不是“tmux 命令集合”，而是一套稳定的 swarm pane topology 管理模型

如果把整份文件压缩，会发现它真正处理的是四层问题：

#### 1. tmux 命令执行上下文
- user session
- swarm socket

#### 2. pane 视觉与身份表示
- border color
- pane title
- border status

#### 3. pane topology 变更
- split
- rebalance
- hide/show
- kill

#### 4. runtime correctness
- pane creation lock
- leader pane/window anchoring
- shell ready delay
- first external pane reuse

所以最准确的压缩表达应该是：

```text
TmuxBackend.ts = 把 tmux 从“能执行命令的终端复用器”提升成“可承载 swarm teammate 拓扑与视觉布局的 pane host”
```

---

### 元

问题：**这一站真正想解决的架构问题是什么？**

回答：这一站的元问题，是为什么 swarm 需要一层专门的 tmux 宿主实现，而不是把 pane 操作散在 executor 或 registry 里。读完会看到，它真正承接的是建 pane、发命令、调布局、做 hide/show 这些宿主能力本身。

### 反

问题：**如果把这一站的设计反过来，会发生什么？**

回答：如果反过来做，把 inside-tmux、outside-tmux、pane 竞态与 leader 锚点逻辑拆散到别处，tmux 路线会立刻失去统一边界。结果不是更灵活，而是更难保证 pane 创建、复用和重平衡的一致性。

### 空

问题：**跳出当前文件名，这一站背后更大的问题是什么？**

回答：更大的问题不是 tmux 命令怎么拼，而是多 agent 协作系统怎样把“终端宿主能力”抽象成可替换后端。TmuxBackend 的价值，就在于它把 swarm 对窗格宿主的要求显式化了。

### 读完这一站后，你应该抓住的 10 个事实

1. `TmuxBackend.ts` 是 swarm 在 tmux 上的正式 `PaneBackend` 实现，负责 pane 宿主能力而不是 teammate lifecycle 或 backend 选择。
2. 它内部明确区分两套控制面：用户原始 tmux session 与 external swarm socket/session，这才是 tmux backend 的真正主分叉。
3. `paneCreationLock` 说明 pane 创建被视为有竞态风险的拓扑变更操作，必须串行化处理。
4. inside-tmux 模式下，布局目标是 leader 固定在左侧 30%，所有 teammates 在右侧 70% 区域继续分裂。
5. outside-tmux 模式下，backend 会在独立 swarm socket 中创建专用 session/window，所有 teammates 平铺分布，不存在 leader pane。
6. `getCurrentPaneId()` 和 `getCurrentWindowTarget()` 都围绕 leader 原始 pane/window 锚点设计，避免用户后来切 pane/window 破坏 swarm 布局定位。
7. `sendCommandToPane()` 的本质仍然只是 `tmux send-keys`，说明 pane backend 负责的是宿主命令注入，而不是直接创建 agent runtime。
8. `hidePane()` / `showPane()` 不是简单显示开关，而是通过 `break-pane` / `join-pane` 在 session/window 间真实迁移 pane。
9. `waitForPaneShellReady()` 的 200ms delay 反映了一个关键现实：pane 创建成功并不代表 shell 已经完成初始化、可以立刻稳定接收命令。
10. 文件底部的 `registerTmuxBackend(TmuxBackend)` 说明 tmux backend 通过模块导入时自注册接入 registry，避免了静态循环依赖。

---

### 现在把第 50-51 站串起来

```text
it2Setup.ts
  -> 处理 iTerm2 路线的安装、验证、Python API 启用提示与用户偏好持久化
TmuxBackend.ts
  -> 实现 tmux 路线本身的 pane host 能力，包括 pane 创建、命令注入、布局重排、hide/show 与 kill
```

所以现在 backend 相关链路可以先压缩成：

```text
environment facts
  -> detection.ts

setup readiness / user preference
  -> it2Setup.ts

concrete pane host implementation
  -> TmuxBackend.ts

executor adaptation
  -> PaneBackendExecutor.ts

backend selection
  -> registry.ts
```

也就是说：

```text
it2Setup.ts 回答“iTerm2 这条路准没准备好、用户愿不愿意走”
TmuxBackend.ts 回答“如果决定走 tmux，这个 backend 具体怎样操纵 pane 宿主”
```

---

### 下一站建议

下一站最顺的是：

```text
src/utils/swarm/backends/ITermBackend.ts
```

因为现在你已经看懂了：

- registry 怎样选 backend
- detection / it2Setup 怎样为 iTerm2 路线提供事实与 setup 状态
- tmux backend 作为 pane host 是怎样具体实现的

下一步最自然就是看对照实现：

**iTerm2 backend 自己怎样创建 split pane、怎样与 `it2` CLI / Python API 对接、以及它与 tmux backend 在 pane hide/show、命令注入、布局控制上的能力边界到底有哪些不同。**
