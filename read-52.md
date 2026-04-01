## 第 52 站：`src/utils/swarm/backends/ITermBackend.ts`

### 这是什么文件

`src/utils/swarm/backends/ITermBackend.ts` 是 swarm backend 体系里的 **iTerm2 native split pane 宿主实现**。

上一站 `TmuxBackend.ts` 已经看懂：

- tmux backend 能在 leader 所在 tmux window 里扩展 pane
- 也能在 leader 不在 tmux 时起 external swarm session
- tmux 的强项是 pane 拓扑控制、hide/show、layout rebalance 都比较完整

而这一站看的就是对照实现：

- 如果 backend 选的是 iTerm2
- 它如何用 `it2` CLI 创建 split pane
- 它能做哪些 pane host 操作
- 又有哪些能力天然弱于 tmux

所以这一站回答的是：

```text
iTerm2 backend 作为 PaneBackend，
怎样通过 it2 CLI 驱动 iTerm2 native split panes，
以及它和 tmux backend 的能力边界差异在哪里
```

所以最准确的一句话是：

```text
ITermBackend.ts = swarm 在 iTerm2 上的 native pane host 实现
```

---

### 先看它的整体定位

位置：`src/utils/swarm/backends/ITermBackend.ts:1`

这份文件和 `TmuxBackend.ts` 一样，都是 `PaneBackend` 实现。

它不负责：

- backend 选择
- teammate lifecycle
- CLI bootstrap 参数继承
- mailbox 协议

它只负责：

- split pane
- 向指定 pane 发命令
- close pane
- 回答哪些 pane host 能力支持，哪些不支持

所以它也是一个：

```text
pane host implementation layer
```

但和 tmux 版不同，它的宿主不是 tmux server，
而是：

```text
iTerm2 + it2 CLI + Python API
```

---

### 第一部分：顶部全局状态 `teammateSessionIds` / `firstPaneUsed` / `paneCreationLock`，说明 iTerm2 backend 和 tmux 一样也需要维护“布局历史 + 并发正确性”，只是状态模型更弱、更依赖 session ID 链

位置：`src/utils/swarm/backends/ITermBackend.ts:8`

这里有三块状态：

- `teammateSessionIds: string[]`
- `firstPaneUsed = false`
- `paneCreationLock`

它们分别在解决：

#### 1. `teammateSessionIds`
后续 split 时要知道从哪个 teammate pane 继续 split

#### 2. `firstPaneUsed`
判断现在要不要走“第一位 teammate 的特殊 split 逻辑”

#### 3. `paneCreationLock`
多 teammate 并发 spawn 时避免布局竞争

这说明 iTerm2 backend 虽然能力不如 tmux 丰富，
但也一样要解决：

```text
pane topology mutation needs serialized stateful control
```

只是它维护的 anchor 不是 tmux pane/window target，
而更偏：

```text
iTerm2 session UUID chain
```

---

### 第二部分：`runIt2()` 很薄，但它暴露了 iTerm2 backend 的最低层控制面完全依赖 it2 CLI，而不是本地 API 对象

位置：`src/utils/swarm/backends/ITermBackend.ts:33`

这里就是：

- `execFileNoThrow(IT2_COMMAND, args)`

说明 `ITermBackend` 并不是直接连某个 JS SDK 或内嵌 Python API，
而是和之前 detection / it2Setup 一样，
都通过外部 `it2` 命令行工具与 iTerm2 交互。

也就是说在 Claude Code 进程看来：

```text
iTerm2 pane control = shell out to it2 CLI
```

这也是后面很多设计取舍的根源，
比如：

- 每次调用都比较慢
- 很多 cosmetic 操作被跳过
- color/title/rebalance 被弱化甚至 no-op

---

### 第三部分：`parseSplitOutput()` 和注释说明 iTerm2 pane 身份的稳定性很依赖“你是从哪个 session 去 split 的”，不像 tmux 那样 pane_id 全局更稳

位置：`src/utils/swarm/backends/ITermBackend.ts:42`

这里解析：

- `Created new pane: <session-id>`

但注释非常关键：

```text
这个 UUID 只有在使用 -s 从特定 session split 时才可靠；
如果从 active session split，且 split 发生在别的 window，UUID 可能不可访问
```

这说明 iTerm2 的 pane/session 身份模型没有 tmux 那么强。

tmux 里通常可以稳定地说：

- `%3`
- `session:window`
- `pane_id`

而这里作者实际上在说：

```text
iTerm2 session identity is context-sensitive;
if you don't target the source session explicitly, control becomes unreliable
```

这也是为什么后面代码会非常执着地使用 `-s <session-id>`。

---

### 第四部分：`getLeaderSessionId()` 说明 iTerm2 路线也有自己的“leader original host anchor”，只是来源从 tmux 的 `TMUX_PANE` 换成了 `ITERM_SESSION_ID`

位置：`src/utils/swarm/backends/ITermBackend.ts:58`

这里从：

- `process.env.ITERM_SESSION_ID`

解析出：

- 冒号后的 UUID

也就是说 iTerm2 backend 和 tmux backend 一样，
都不想依赖“当前活动 pane”。

它也想尽量围绕 leader 最初所在 pane/session 做布局锚定。

所以这里对应上一站 tmux 的：

- `getLeaderPaneId()`

在 iTerm2 里变成：

- `getLeaderSessionId()`

这说明两条 backend 路线虽然宿主不同，
但在设计哲学上很一致：

```text
layout should anchor to the leader's original host identity,
not to whatever pane/window is active later
```

---

### 第五部分：`isAvailable()` 说明 iTerm2 backend 的 availability 语义是“双条件成立”——既要在 iTerm2 里，也要 it2 CLI 真能连通

位置：`src/utils/swarm/backends/ITermBackend.ts:84`

这里调用：

- `isInITerm2()`
- `await isIt2CliAvailable()`

注意这里引的是 `detection.ts` 里的：

- `isIt2CliAvailable()`

而不是 `it2Setup.ts` 那个仅检查 binary presence 的版本。

所以这里的 availability 实际语义是：

```text
host membership: in iTerm2
+
operational availability: it2 CLI can actually work
```

这和前面 detection.ts 的事实分层完全一致。

---

### 第六部分：`supportsHideShow = false` 是这份文件最重要的能力声明之一——iTerm2 backend 从一开始就承认自己不具备 tmux 那种 pane relocate 语义

位置：`src/utils/swarm/backends/ITermBackend.ts:79`

这里直接声明：

- `readonly supportsHideShow = false`

后面也确实：

- `hidePane()` 返回 `false`
- `showPane()` 返回 `false`

这很关键。

说明 `PaneBackend` 这层抽象不是假装所有宿主能力一样强，
而是允许具体 backend 明确暴露：

```text
我实现了 PaneBackend 合同，
但其中某些可选宿主能力在这个平台上天然做不到
```

也就是说 iTerm2 backend 和 tmux backend 的差距不是临时遗漏，
而是被正式建模进接口能力矩阵里了。

---

### 第七部分：`createTeammatePaneInSwarmView()` 的布局策略非常明确——第一位 teammate 从 leader vertical split，后续都从最后一个 teammate 继续 split，形成 leader 左 + teammates 右侧纵向堆叠的链式布局

位置：`src/utils/swarm/backends/ITermBackend.ts:110`

注释已经把意图写出来了：

- 第一位 teammate：`-v` from leader session
- 后续 teammates：从最后一个 teammate session split

也就是说它试图实现的视觉效果是：

```text
leader | teammate1
          └ teammate2
             └ teammate3
```

更抽象地说：

```text
leader-fixed-left
+
teammate-stack-on-right
```

这和 tmux backend 的“leader 30% + 右侧 teammate 区域”很接近，
但实现手段更弱：

- tmux 能全局 rebalance
- iTerm2 只能通过“从谁 split”来间接塑形

---

### 第八部分：这段 `while (true)` + prune/retry 逻辑非常关键——作者明确在为“用户手动关掉某个 iTerm2 pane��这种 at-fault 情况做恢复，而不是假设内部状态永远可信

位置：`src/utils/swarm/backends/ITermBackend.ts:131`

这里的设计很成熟。

场景是：

- 内部 `teammateSessionIds` 还记着某个 session ID
- 但用户可能已经在 iTerm2 里手动 `Cmd+W` / 点 X 关掉 pane
- 或该 pane 进程崩了

如果还拿这个 stale session 去 split，就会失败。

所以这里做了：

1. 先按记录的 `targetedTeammateId` 去 split
2. 如果失败
3. 再 `it2 session list`
4. 如果确认该 session 真死了
5. 从数组里 prune 掉
6. 再 retry

这本质上是在做：

```text
lazy stale-target recovery
```

而不是每次 spawn 前都先全量探测一遍活跃 sessions。

注释还明确说明这是有意为之：

- proactive check 每次都要 `session list`
- 成本高
- 不如失败后再确认并修复

这是很典型的性能/正确性折中设计。

---

### 第九部分：这里的 prune 非常谨慎——只有在 `session list` 明确证实 target 不存在时才删除，避免把系统性故障误判成某个 pane 死亡

位置：`src/utils/swarm/backends/ITermBackend.ts:179`

注释点得很透：

```text
如果是 Python API 关了、it2 被移除了、socket 临时故障，
不能误删 live session IDs，
否则会把内部状态越修越坏
```

所以这里的逻辑不是“split 失败就 prune”，
而是：

```text
split 失败
-> session list succeeds
-> target absent
-> 才 prune
```

这说明作者对故障分类很在意：

#### 目标 pane 已死
可以修复内部状态并重试

#### 系统性故障
不能动内部状态，应该把错误上抛

这是这个 backend 最成熟的一点之一。

---

### 第十部分：`firstPaneUsed` 在 iTerm2 backend 里不是复用现有 pane，而只是决定“第一位 teammate 是否从 leader split”，和 tmux external 首 pane复用完全不同

位置：`src/utils/swarm/backends/ITermBackend.ts:138`

要特别和上一站 tmux external 模式区分：

#### tmux external
- 第一位 teammate 可能直接复用 session 初始 pane

#### iTerm2 backend
- 没有 external swarm session
- `firstPaneUsed` 只是布局状态位
- 决定是否走 first split from leader

也就是说这里的 `firstPaneUsed` 语义是：

```text
have we already created the first teammate branch off the leader?
```

而不是：

```text
is there an initial empty host pane to reuse?
```

这是两种 backend 宿主模型的根本区别。

---

### 第十一部分：`sendCommandToPane()` 强制优先使用 `session run -s <paneId>`，再次说明 iTerm2 backend 最重要的正确性原则就是“永远显式 targeting，别依赖 active pane/window”

位置：`src/utils/swarm/backends/ITermBackend.ts:242`

这里注释写得非常明确：

```text
Always use -s flag to target specific session - this ensures the command
 goes to the right pane even if user switches windows
```

也就是说 iTerm2 backend 最害怕的是：

```text
用户在 UI 里点到别的 pane/window，
导致“当前活动 session”不再是我们以为的那一个
```

所以它在：

- split 阶段
- run command 阶段

都尽可能显式绑定具体 session ID。

这与 tmux backend 里用 leader pane ID / pane target 的思路高度一致，
只是 iTerm2 这边对“active session 不可靠”更加敏感。

---

### 第十二部分：颜色、标题、border status、rebalance 在 iTerm2 backend 里几乎都被降成 no-op，核心原因不是不会做，而是每个 it2 CLI 调用都太慢

位置：

- `setPaneBorderColor()` `src/utils/swarm/backends/ITermBackend.ts:266`
- `setPaneTitle()` `src/utils/swarm/backends/ITermBackend.ts:278`
- `enablePaneBorderStatus()` `src/utils/swarm/backends/ITermBackend.ts:291`
- `rebalancePanes()` `src/utils/swarm/backends/ITermBackend.ts:302`

注释反复强调：

- each it2 call is slow
- each call spawns a Python process
- cosmetic features skipped for performance

这说明 iTerm2 backend 的能力边界并不只是“平台做不到”，
还包括：

```text
平台 technically 可以做一些事，
但代价太高，不值得放进热路径
```

所以这里的 no-op 是一种非常明确的工程取舍：

- 保核心功能：create / send / close
- 去次要美化：color / title / rebalance

这和 tmux backend 形成了鲜明对比：

- tmux 调整布局和 pane option 成本低
- iTerm2 每多一次 it2 CLI round trip 都显著更贵

---

### 第十三部分：`killPane()` 使用 `session close -f`，说明 iTerm2 backend 必须显式绕过宿主自己的“关闭确认”UX，才能让 force kill 语义和 tmux 对齐

位置：`src/utils/swarm/backends/ITermBackend.ts:315`

这里的注释非常重要：

```text
without -f, iTerm2 respects the "Confirm before closing" preference
and may show a dialog or refuse when process is still running
```

这说明 iTerm2 的默认宿主语义其实更偏人类交互式终端：

- 关闭 pane 可能要确认
- pane 里 shell 还活着就不一定让你关

但 swarm 的 `killPane()` 语义需要的是：

```text
backend-controlled force termination
```

所以这里必须加 `-f`，
把 iTerm2 的“用户友好确认逻辑”压过去，
让它表现得更像 tmux 的 `kill-pane`。

这也是一个很典型的“把 GUI 终端宿主适配成程序控制 backend”的动作。

---

### 第十四部分：`killPane()` 不管 close 成不成功都清理本地 `teammateSessionIds`，说明这里遵循的是“内部状态宁可乐观修复，也不要继续持有 stale target”

位置：`src/utils/swarm/backends/ITermBackend.ts:328`

这里逻辑是：

- 先尝试 close
- 然后无论结果如何，都把 paneId 从 `teammateSessionIds` 移除
- 如果删空，`firstPaneUsed = false`

注释解释了理由：

```text
即使 pane 实际上早就被用户手关了，
把 stale ID 移出内部状态也是正确的
```

这说明在 kill 路径上，作者采用的是：

```text
best-effort close
+
definitely clean local state
```

因为从 swarm 自己的角度，
一旦决定 kill 某 pane，
之后再拿它当 split target 基本不可能是对的。

---

### 第十五部分：`hidePane()` / `showPane()` 直接返回 false，说明 iTerm2 backend 与 tmux backend 的最大结构性差距，就是缺少 pane relocation primitives

位置：`src/utils/swarm/backends/ITermBackend.ts:341`

这里不是暂时没实现，
而是注释直接说：

- iTerm2 doesn't have direct equivalent to `break-pane`
- iTerm2 doesn't have direct equivalent to `join-pane`

这说明 tmux backend 的 hide/show 强项来自 tmux 的 pane 可迁移拓扑模型，
而 iTerm2 并没有对应 primitive。

所以两者真正的差异可以压缩成：

```text
tmux = programmable pane topology manager
iterm2 = native split-pane host with weaker topology surgery primitives
```

这句话非常关键。

---

### 第十六部分：整份文件真正体现的，是 iTerm2 backend 以“最小可用 pane host”姿态接入 PaneBackend 抽象，而不是试图和 tmux 做能力完全对齐

如果把整份文件压缩，会发现 iTerm2 backend 保留的核心能力只有：

#### 1. create
- `it2 session split`

#### 2. send
- `it2 session run`

#### 3. kill
- `it2 session close -f`

而下列能力不是 no-op 就是不支持：

- border color
- title
- pane border status
- rebalance
- hide/show

这说明它的定位不是：

```text
full-featured tmux-equivalent pane backend
```

而更像：

```text
a pragmatic native iTerm2 backend that preserves the core spawn/send/close path
```

只要这三件事可靠，swarm 就能在 iTerm2 里工作；
其余视觉/拓扑增强则能省则省。

---

### 第十七部分：`registerITermBackend(ITermBackend)` 与 tmux 一样，继续沿用 backend 自注册模式接入 registry

位置：`src/utils/swarm/backends/ITermBackend.ts:367`

文件底部同样：

- `registerITermBackend(ITermBackend)`

这再次说明 registry 的插件式 backend 装载模式是统一的：

```text
registry 动态 import
-> backend module top-level self-register
```

所以 `ITermBackend.ts` 不只是 class 文件，
也是 backend discovery/bootstrap 的 side-effect module。

---

### 第十八部分：把 `TmuxBackend.ts` 和 `ITermBackend.ts` 并排看，最核心的差异不是 API 名字，而是宿主可编程性强弱完全不同

这两份文件接口长得都像 `PaneBackend`，
但宿主本质差很多：

#### tmux backend
- 强 pane identity
- 强 layout control
- 强 rebalance
- 支持 hide/show
- 支持 external standalone session

#### iTerm2 backend
- 依赖 it2 CLI round trips
- session identity 更脆弱
- 大量视觉/拓扑能力因性能或平台限制而降级
- 不支持 hide/show
- 只做 native current-window split

所以这里最值得记住的一句话是：

```text
PaneBackend abstraction 统一了接口，
但并没有抹平宿主能力的真实不对称性
```

而这正是 registry / detection / it2Setup 要存在的根本原因之一。

---

### 读完这一站后，你应该抓住的 10 个事实

1. `ITermBackend.ts` 是 swarm 在 iTerm2 上的正式 `PaneBackend` 实现，通过 `it2` CLI 驱动 native split panes。
2. 它同样维护 `teammateSessionIds`、`firstPaneUsed` 和 pane creation lock，说明 iTerm2 路线也需要状态化布局控制和并发保护。
3. `parseSplitOutput()` 与相关注释说明 iTerm2 pane/session UUID 的可控性很依赖显式 `-s <session-id>` targeting。
4. `getLeaderSessionId()` 让 iTerm2 路线也拥有自己的 leader 原始宿主锚点，对应 tmux 路线里的 `TMUX_PANE` 锚定思路。
5. `createTeammatePaneInSwarmView()` 通过“第一位从 leader split、后续从最后一个 teammate split”的策略，近似实现 leader 左、teammates 右侧纵向堆叠的布局。
6. 针对用户手动关闭 pane 或 pane 崩溃的场景，backend 实现了基于 `session list` 的 lazy stale-target prune/retry 机制，避免每次 spawn 都做高成本全量探测。
7. `sendCommandToPane()` 强制优先用 `session run -s <paneId>`，核心目标是避免 active pane/window 漂移造成命令发错 pane。
8. 颜色、标题、border status、rebalance 在 iTerm2 backend 里基本都降为 no-op，主要原因是每次 it2 CLI 调用都要额外起 Python 进程，热路径代价太高。
9. `killPane()` 用 `session close -f` 绕过 iTerm2 的关闭确认交互，并且无论 close 结果如何都清理本地 session ID 状态，避免 stale target 继续污染后续布局。
10. iTerm2 backend 最大的结构性短板是缺少 tmux 那样的 `break-pane` / `join-pane` 等 pane relocation primitives，因此 `supportsHideShow = false` 不是偶然，而是宿主能力差异的正式建模。

---

### 现在把第 51-52 站串起来

```text
TmuxBackend.ts
  -> 用 tmux server / pane primitives 实现较强的 pane host 能力，包含 rebalance、hide/show、external session 等
ITermBackend.ts
  -> 用 it2 CLI / Python API 实现较轻量的 native split pane host，保住 create/send/close 主链，但明显弱化视觉与拓扑控制能力
```

所以现在 pane backend 对照关系可以先压缩成：

```text
strong programmable pane host
  -> TmuxBackend

lightweight native GUI split host
  -> ITermBackend

shared abstraction
  -> PaneBackend
```

也就是说：

```text
TmuxBackend 回答“当宿主可编程性很强时，swarm 能把 pane 拓扑玩到什么程度”
ITermBackend 回答“当宿主是 GUI terminal + CLI bridge 时，swarm 至少保住哪些核心能力，又必须在哪些地方降级”
```

---

### 下一站建议

下一站最顺的是：

```text
src/utils/swarm/constants.ts
```

因为现在你已经看懂了：

- detection / registry / it2Setup 怎样决定 backend 路线
- tmux 与 iTerm2 两个 concrete backend 各自怎样实现 pane host 能力
- 它们共享哪些抽象，又在哪些能力上明显分化

下一步最自然就是把这条 backend 子系统里的共享命名与协议常量补齐：

**swarm backend / mailbox / session / hidden panes / teammate message 这些关键常量到底定义在哪，怎样把前面几站看到的多个模块串成同一套命名空间。**
