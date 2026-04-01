## 第 43 站：`src/utils/swarm/backends/registry.ts`

### 这是什么文件

`src/utils/swarm/backends/registry.ts` 是 swarm 体系里的 **teammate backend 选择器 + 执行器统一门面**。

上一站 `teamHelpers.ts` 已经看到：

```text
team file 里会记录每个成员的 backendType
session cleanup 时会通过 backend registry 找到对应 backend 去 killPane
```

这已经说明 swarm 不是只支持一种 teammate 执行后端。

但系统还需要解决更关键的问题：

- 当前环境到底该用 tmux、iTerm2，还是 in-process
- `auto` 模式怎样解析成实际执行方式
- backend 类如何注册，避免循环依赖
- 上层如何在“不知道底层后端细节”的情况下统一 spawn teammate
- 一旦某次 fallback 到 in-process，UI 和后续行为怎样保持一致

所以这一站回答的是：

```text
swarm 怎样把多种 teammate backend 的检测、选择、缓存、注册和统一执行接口收口到一个 registry 里
```

所以最准确的一句话是：

```text
registry.ts = swarm teammate backend 的统一解析与分发中枢
```

---

### 先看它的整体定位

位置：`src/utils/swarm/backends/registry.ts:1`

这个文件主要做五件事：

1. 注册 backend class
2. 检测当前环境适合哪种 pane backend
3. 决定本 session 是否启用 in-process
4. 把 pane backend / in-process backend 统一包装成 `TeammateExecutor`
5. 对上述结果做 session 级缓存

也就是说，它不是具体 backend 实现：

- 不是 `TmuxBackend`
- 不是 `ITermBackend`
- 不是 `InProcessBackend`

它做的是：

```text
在“多种 backend 实现”之上，提供统一的选择和拿取入口
```

所以从分层上看：

```text
backend implementation
  -> TmuxBackend / ITermBackend / InProcessBackend
registry facade
  -> registry.ts
caller-facing executor abstraction
  -> TeammateExecutor
```

---

### 第一部分：顶部一堆 cache 变量说明 backend 选择在进程生命周期内基本被视为 session-level 常量

位置：`src/utils/swarm/backends/registry.ts:22`

这里缓存了很多东西：

- `cachedBackend`
- `cachedDetectionResult`
- `backendsRegistered`
- `cachedInProcessBackend`
- `cachedPaneBackendExecutor`
- `inProcessFallbackActive`

这说明作者非常明确地把 backend 决策建模成：

```text
once resolved, mostly stable for the lifetime of the process
```

尤其注释直接写了：

```text
Once detected, the backend selection is fixed for the lifetime of the process.
```

这很合理，因为：

- 你是不是在 tmux 里
- 你是不是在 iTerm2 里
- it2 CLI 是否可用
- 某次 spawn 是否已经因为缺 pane backend 而 fallback

这些信息在同一 session 里通常不会频繁变化，
而反复探测会带来额外开销和语义漂移。

所以这里最重要的理解是：

```text
backend 选择不是每次 spawn 都重新“民主投票”，
而是 session 级一次决策、后续复用
```

---

### 第二部分：`ensureBackendsRegistered()` + `registerTmuxBackend()` + `registerITermBackend()` 暴露了 registry 为避免循环依赖采用了“延迟导入 + 反向注册”模式

位置：

- `ensureBackendsRegistered()` `src/utils/swarm/backends/registry.ts:74`
- `registerTmuxBackend()` `src/utils/swarm/backends/registry.ts:85`
- `registerITermBackend()` `src/utils/swarm/backends/registry.ts:93`

这组设计很关键。

顶部先放了两个占位：

- `TmuxBackendClass`
- `ITermBackendClass`

然后：

- backend 文件自己调用 `registerTmuxBackend(...)` / `registerITermBackend(...)`
- registry 需要时再通过 `await import('./TmuxBackend.js')` / `await import('./ITermBackend.js')` 触发注册

这说明作者要解决的是经典问题：

```text
registry 需要 backend class
backend 实现又可能依赖 registry 或相关类型
直接静态互相 import 容易形成循环依赖
```

所以这里的策略是：

```text
类定义在 backend 文件里
registry 只持有构造器插槽
真正需要时再动态 import，让 backend 自己回调注册
```

这是一种很典型的插件式注册模式。

也说明这个 registry 不是硬编码单文件 switch 的简陋实现，
而是已经在模块边界上做了专门处理。

---

### 第三部分：`createTmuxBackend()` / `createITermBackend()` 强调 registry 只负责构造，不负责探测

位置：

- `createTmuxBackend()` `src/utils/swarm/backends/registry.ts:106`
- `createITermBackend()` `src/utils/swarm/backends/registry.ts:119`

这两个函数都很简单：

- 如果 class 还没注册 -> throw
- 否则 `new BackendClass()`

它们没有做环境探测，也没有做模式选择。

这说明 registry 内部其实也把两层语义分开了：

#### 1. backend construction
```text
给定类型，创建实例
```

#### 2. backend detection / selection
```text
当前环境应该选哪个 backend
```

这样 `getBackendByType('tmux')` 这种显式选择场景，
就可以绕过探测流程直接构造。

这也正好服务上一站看到的 cleanup 场景：

```text
team file 里已经知道 backendType 了
只需要按类型拿 backend 去 killPane
不需要再次探测“环境最适合谁”
```

---

### 第四部分：`detectAndGetBackend()` 是 pane backend 选择的核心决策树

位置：`src/utils/swarm/backends/registry.ts:136`

文件注释已经把优先级说得很清楚：

1. inside tmux -> tmux
2. in iTerm2 且 it2 可用 -> iTerm2
3. in iTerm2 但 it2 不可用 -> 看 tmux fallback
4. 非 iTerm2/tmux，但 tmux 可用 -> tmux external session
5. 否则报错

它解决的不是“所有 teammate backend”，而是：

```text
pane-based backend 在当前环境里该选谁
```

因为 in-process 走的是另一套判断口 `isInProcessEnabled()`。

所以这个函数的定位更准确地说是：

```text
pane backend resolver
```

而不是整个 swarm backend 总决策器。

---

### 第五部分：inside tmux 优先级最高，说明“已经处在 pane-native 宿主里”会压过其他终端能力

位置：`src/utils/swarm/backends/registry.ts:158`

逻辑是：

- 只要 `insideTmux` -> 直接 tmux
- 即使也在 iTerm2 里，也不再考虑 iTerm2 backend

注释明确说：

```text
If inside tmux, always use tmux (even in iTerm2)
```

这说明系统非常重视当前已存在的 pane 宿主边界：

```text
如果用户已经在 tmux session 内，最自然的一致性就是继续在这套 tmux pane 体系里扩展 teammate
```

而不是再跨出去开 iTerm2 原生 pane。

这体现的是：

```text
优先保持当前 pane universe 的一致性
```

而不是盲目选择“更原生”的宿主。

---

### 第六部分：iTerm2 分支并不是“只要是 iTerm2 就强制用 iTerm2”，而是一个带用户偏好和 setup 状态的分支

位置：`src/utils/swarm/backends/registry.ts:173`

这里的逻辑比表面复杂得多。

先看：

- `getPreferTmuxOverIterm2()`

如果用户已设置 prefer tmux，就直接跳过 iTerm2 探测。

否则：

- `isIt2CliAvailable()`
- 有 it2 CLI -> 用 iTerm2 backend
- 没有 -> 再看 tmux 是否可用

如果最后只能 fallback 到 tmux，返回值里还会记录：

- `isNative: false`
- `needsIt2Setup: !preferTmux`

这说明 iTerm2 分支实际上同时编码了三件事：

1. 终端宿主能力
2. 用户显式偏好
3. setup 提示是否还应该展示

尤其 `needsIt2Setup` 很关键，它说明 detection 的输出不只是“选谁”，还包括：

```text
UI 是否应该告诉用户：你本可以得到更原生的 iTerm2 体验，只是还没装 it2 CLI
```

所以 `BackendDetectionResult` 不是简单 backend 指针，
而是：

```text
backend selection + UX guidance metadata
```

---

### 第七部分：没有任何 pane backend 可用时直接 throw 安装指引，说明 pane mode 的失败是显式用户环境问题，而不是静默降级

位置：

- `detectAndGetBackend()` `src/utils/swarm/backends/registry.ts:251`
- `getTmuxInstallInstructions()` `src/utils/swarm/backends/registry.ts:259`

如果不在 tmux、不在可用的 iTerm2、tmux 也没装，
这里不会悄悄兜底成某种 pane 假实现，
而是直接抛错，并带平台相关安装说明。

平台分支包括：

- macOS -> `brew install tmux`
- Linux/WSL -> `apt` / `dnf`
- Windows -> 先装 WSL 再装 tmux

这说明对于 pane backend 路径，作者的设计是：

```text
要么选出真实可用的 pane backend
要么明确告诉用户缺了什么环境能力
```

而不是在这里悄悄改用其他模式。

这和后面 `inProcessFallbackActive` 形成有意思的分工：

- pane detection 本身失败 -> 明确报错
- 某些更高层策略决定改走 in-process -> 由 registry 记录 fallback 事实

---

### 第八部分：`getBackendByType()` 说明 registry 还承担“按已知 backendType 反查实现”的角色

位置：`src/utils/swarm/backends/registry.ts:295`

这个函数非常简单：

- `'tmux'` -> `createTmuxBackend()`
- `'iterm2'` -> `createITermBackend()`

但它的架构价值很大。

因为这说明 swarm 里存在一类场景：

```text
当前不需要做环境探测，
而是已经从 team file / runtime state 拿到了明确 backendType，
只需要按类型恢复对应 backend 行为
```

比如上一站看到的：

- cleanup 时按 `backendType` kill pane

所以 registry 并不只是 spawn-time selector，
还是：

```text
persisted backend identity -> runtime backend implementation
```

的恢复器。

---

### 第九部分：`markInProcessFallback()` 很关键，它说明 in-process 不只是配置选项，也可能是 runtime 决策结果

位置：`src/utils/swarm/backends/registry.ts:326`

这里的注释写得特别清楚：

```text
spawn 因为没有可用 pane backend 而 fallback 到 in-process 后，
isInProcessEnabled() 以后都应该返回 true，
这样 UI 和后续 spawn 才反映真实情况
```

这说明系统区分了两种 in-process：

#### 1. 配置层显式 in-process
- teammate mode 就是 `in-process`

#### 2. 运行时被迫 fallback 到 in-process
- 原本是 `auto`
- 但 pane backend 不可用
- 某次 spawn 之后被“锁定”为 in-process reality

这非常关键，因为它表明：

```text
resolved teammate mode 不完全等于 settings snapshot，
还会受本 session 实际执行结果影响
```

所以 registry 记录的不只是“用户想要什么”，
还记录：

```text
本 session 最终实际能做到什么
```

---

### 第十部分：`getTeammateMode()` 从 snapshot 读值，说明 registry 故意忽略运行时配置波动，保持 session 内稳定解释

位置：`src/utils/swarm/backends/registry.ts:335`

这里不是实时读 settings 文件，
而是：

- `getTeammateModeFromSnapshot()`

注释明确写：

```text
Returns the session snapshot captured at startup, ignoring runtime config changes.
```

这说明 teammate backend 模式的解释是：

```text
session-start policy snapshot
```

而不是热更新配置。

原因很明显：

- 中途把 teammate mode 从 tmux 改成 in-process
- 或从 auto 改成 tmux

如果直接热切，可能会造成：

- 已有 team / executor 语义混乱
- UI 解释与已创建 teammate 不一致
- 半途中 backend universe 切换

因此 registry 选择的是：

```text
启动时拍一张 teammate mode 快照，整个 session 按这张快照解释
```

这是一种稳定性优先的设计。

---

### 第十一部分：`isInProcessEnabled()` 才是“auto 模式最终解析口”

位置：`src/utils/swarm/backends/registry.ts:351`

这个函数非常关键，因为它把：

- non-interactive session
- explicit mode (`in-process` / `tmux`)
- `auto`
- `inProcessFallbackActive`
- 当前环境 (`insideTmux` / `inITerm2`)

统一收口成一个布尔值。

逻辑可以压缩成：

#### 1. 非交互 session
```text
直接 true
```

注释说明：

```text
-p 这类非交互模式下，tmux teammate 没有意义
```

#### 2. 显式 `in-process`
```text
true
```

#### 3. 显式 `tmux`
```text
false
```

#### 4. `auto`
- 如果之前已经 fallback -> true
- 否则如果 insideTmux 或 inITerm2 -> false
- 否则 -> true

这意味着 `auto` 在这里真正表达的是：

```text
如果当前已有 pane 宿主环境，就偏 pane backend；
否则默认 in-process
```

所以系统并不是“没有明确设置就默认 tmux”，
而是：

```text
auto = context-sensitive mode resolution
```

---

### 第十二部分：`getResolvedTeammateMode()` 把内部布尔决策重新提升为外部可解释的模式标签

位置：`src/utils/swarm/backends/registry.ts:396`

这个函数只是：

- `isInProcessEnabled() ? 'in-process' : 'tmux'`

看起来简单，但很有用。

因为内部判断最终是布尔值，
而对 UI / 上层逻辑来说，更需要的是：

```text
当前 session 实际 resolved 成哪种 teammate mode
```

所以它扮演的是：

```text
internal boolean policy -> user-facing resolved mode label
```

这也是一个典型 facade 设计。

---

### 第十三部分：`getInProcessBackend()` / `getTeammateExecutor()` / `getPaneBackendExecutor()` 才是真正把多 backend 统一成同一调用接口的关键

位置：

- `getInProcessBackend()` `src/utils/swarm/backends/registry.ts:404`
- `getTeammateExecutor()` `src/utils/swarm/backends/registry.ts:425`
- `getPaneBackendExecutor()` `src/utils/swarm/backends/registry.ts:442`

这组函数是全文件最核心的“统一执行接口”部分。

#### `getInProcessBackend()`
- 懒创建
- 缓存 `InProcessBackend`

#### `getPaneBackendExecutor()`
- 先 `detectAndGetBackend()`
- 再把选中的 pane backend 包成 `PaneBackendExecutor`

这说明原始 pane backend 本身并不直接等于 `TeammateExecutor`，
而是需要一层 adapter：

```text
PaneBackend -> PaneBackendExecutor -> TeammateExecutor
```

#### `getTeammateExecutor(preferInProcess = false)`
逻辑是：

- 如果上层偏好 in-process，且当前允许 in-process -> 返回 in-process backend
- 否则 -> 返回 pane backend executor

这说明 registry 最终对外提供的抽象不是“你该用 tmux 还是 iterm2”，
而是：

```text
给你一个统一的 teammate executor，
你直接 spawn/manage teammate 即可
```

这正是整个 registry 的最高层价值：

```text
把 backend 多态性压缩到一个统一执行接口下面
```

---

### 第十四部分：`PaneBackendExecutor` 的存在说明 pane backend 与 in-process backend 在系统接口层被主动对齐

位置：`src/utils/swarm/backends/registry.ts:445`

虽然这一站没展开 `PaneBackendExecutor` 实现，
但仅从这里已经能看出一个关键架构点：

- in-process backend 直接实现 `TeammateExecutor`
- pane backend 先是 `PaneBackend`
- 然后包一层 `createPaneBackendExecutor(...)`
- 最终也暴露成 `TeammateExecutor`

所以整个 swarm backend 统一的关键不是“所有 backend 原生就长一样”，
而是：

```text
必要时通过 adapter 把不同 backend 统一到同一 executor interface
```

这正是经典的 adapter/facade 组合。

也解释了为什么上层 team runtime 可以在不关心 backend 细节的情况下统一工作。

---

### 第十五部分：`resetBackendDetection()` 暴露了这个 registry 明显考虑了测试重置需求

位置：`src/utils/swarm/backends/registry.ts:457`

这里把所有缓存都清掉：

- `cachedBackend`
- `cachedDetectionResult`
- `cachedInProcessBackend`
- `cachedPaneBackendExecutor`
- `backendsRegistered`
- `inProcessFallbackActive`

这说明作者非常清楚：

```text
这个模块有很多 process-global cache，
测试如果不显式 reset，很容易串状态
```

所以这一点再次证明：

```text
registry.ts 管的是 session/global 级别决策
```

正因为它是全局缓存型模块，
才必须提供测试重置口。

---

### 第十六部分：这份文件真正体现的是“环境探测、用户偏好、执行抽象”三者的折中

如果把整份文件压缩，会发现它一直在平衡三类因素：

#### 1. 环境现实
- inside tmux?
- in iTerm2?
- it2 CLI available?
- tmux available?
- non-interactive?

#### 2. 用户策略
- teammate mode snapshot: `auto` / `tmux` / `in-process`
- prefer tmux over iTerm2

#### 3. 上层调用体验
- 统一拿 `TeammateExecutor`
- 不必关心具体 backend
- 用缓存维持 session 内一致性

所以它真正做���不是“选个后端”这么简单，
而是：

```text
把环境能力、用户策略与统一执行接口之间的复杂映射收敛到一个模块里
```

这就是为什么它对 swarm 架构很关键。

---

### 读完这一站后，你应该抓住的 10 个事实

1. `registry.ts` 是 swarm teammate backend 的统一解析与分发中枢，不是具体 backend 实现文件。
2. backend 选择、executor 实例、fallback 状态等都被按 session/process 级缓存，说明该决策被视为进程生命周期内基本稳定。
3. 为避免循环依赖，registry 通过动态 import + `registerTmuxBackend()` / `registerITermBackend()` 的反向注册方式拿到 backend class。
4. `detectAndGetBackend()` 只负责 pane backend 的环境探测与选择，优先级是 inside tmux -> native iTerm2 -> tmux fallback -> 报错。
5. iTerm2 分支不仅看环境能力，还会考虑用户是否偏好 tmux，以及是否需要继续提示安装 it2 CLI。
6. `getBackendByType()` 说明 registry 还负责把持久化的 `backendType` 恢复成运行时 backend 实现，这对 cleanup 等路径很重要。
7. `markInProcessFallback()` 表明 in-process 不只是显式配置结果，也可能是 auto 模式下的 runtime fallback 结果，而且一旦发生会影响本 session 后续行为与 UI 解释。
8. `isInProcessEnabled()` 才是 teammate mode 的最终解析口：它统一处理 non-interactive、显式 mode、auto 模式、fallback 状态和当前宿主环境。
9. `getTeammateExecutor()` 是对上层最重要的 facade：它把 in-process backend 与 pane backend executor 统一成同一个 `TeammateExecutor` 接口。
10. `resetBackendDetection()` 的存在说明这个模块明显是全局缓存型 registry，因此测试必须有显式 reset 机制避免状态串联。

---

### 现在把第 42-43 站串起来

```text
teamHelpers.ts
  -> 维护 team file、成员 backendType / mode / active、以及 pane/worktree/team/tasks 清理
registry.ts
  -> 负责实际解析 backendType/环境能力，并把不同 teammate backend 统一包装成可供上层调用的 executor
```

所以现在 backend 这一层可以先压缩成：

```text
persisted team metadata
  -> TeamFile.backendType

runtime backend resolution
  -> detectAndGetBackend / isInProcessEnabled / getBackendByType

unified execution surface
  -> getTeammateExecutor -> TeammateExecutor
```

也就是说：

```text
teamHelpers.ts 记录“这个成员属于什么 backend”
registry.ts 决定“当前 session 实际用哪个 backend，以及怎样把它变成统一 executor”
```

---

### 下一站建议

下一站最顺的是：

```text
src/utils/swarm/backends/types.ts
```

因为现在你已经看懂了：

- registry 怎样缓存和选择 backend
- pane backend 与 in-process backend 怎样统一到 `TeammateExecutor`
- team file 里的 `backendType` 怎样回到 runtime implementation

下一步最自然就是把抽象层本体补齐：

**`PaneBackend`、`PaneBackendType`、`BackendDetectionResult`、`TeammateExecutor` 这些核心接口到底长什么样，以及 swarm 是怎样把“pane 操作能力”和“teammate 生命周期能力”区分并建模成不同层级类型的。**
