## 第 49 站：`src/utils/swarm/backends/detection.ts`

### 这是什么文件

`src/utils/swarm/backends/detection.ts` 是 swarm backend 体系里的 **终端宿主环境探测器**。

上一站 `spawnUtils.ts` 已经看懂：

```text
process-based teammate 启动时，
会继承 leader 的 command / flags / env
```

但在真正走到 spawn 之前，registry 还要先回答更底层的问题：

- 当前 Claude 是不是本来就跑在 tmux 里
- leader 原始 pane 是哪个
- 系统有没有 tmux
- 当前是不是 iTerm2
- it2 CLI 是否真的能连到 iTerm2 Python API

这些信息会直接决定：

- 选 tmux 还是 iTerm2
- 是 native pane 还是 external session
- 是否要提示用户做 it2 setup
- auto mode 最终该偏 pane 还是 in-process

所以这一站回答的是：

```text
swarm 怎样判断当前终端宿主环境与可用 pane 能力，
从而为 backend registry 的决策树提供底层事实
```

所以最准确的一句话是：

```text
detection.ts = swarm pane backend 决策的环境事实来源
```

---

### 先看它的整体定位

位置：`src/utils/swarm/backends/detection.ts:1`

这份文件只做环境探测，不做决策，不做 spawn。

它提供的能力主要是：

- `isInsideTmuxSync()`
- `isInsideTmux()`
- `getLeaderPaneId()`
- `isTmuxAvailable()`
- `isInITerm2()`
- `isIt2CliAvailable()`
- `resetDetectionCache()`

也就是说它不负责：

- 选 backend
- 创建 pane
- 跑 teammate

它只负责：

```text
告诉 registry：当前有哪些宿主环境事实成立
```

所以它是一个很纯的：

```text
environment detection layer
```

---

### 第一部分：顶部捕获 `ORIGINAL_USER_TMUX` / `ORIGINAL_TMUX_PANE`，说明作者明确意识到运行过程中环境变量可能被 Claude 自己修改，不能事后再读 `process.env`

位置：`src/utils/swarm/backends/detection.ts:5`

这里在模块加载时就捕获了：

- `ORIGINAL_USER_TMUX = process.env.TMUX`
- `ORIGINAL_TMUX_PANE = process.env.TMUX_PANE`

注释特别强调：

```text
Shell.ts 之后可能会覆盖 TMUX env var，
所以必须在模块 load 时就把用户原始启动环境记下来
```

这说明系统并不把当前 `process.env` 当成永远可信的“原始宿主事实”。

而是非常清楚地知道：

```text
Claude 运行过程中可能会因为 socket / shell 初始化改写环境变量，
导致“当前 env”不再等于“用户启动 Claude 时所在环境”
```

所以这里真正做的是：

```text
capture original host context before runtime mutates it
```

这是一个很成熟的环境探测设计。

---

### 第二部分：`isInsideTmuxSync()` / `isInsideTmux()` 只看原始 `TMUX`，而且明确拒绝用 `tmux display-message` 之类命令做兜底

位置：

- `isInsideTmuxSync()` `src/utils/swarm/backends/detection.ts:36`
- `isInsideTmux()` `src/utils/swarm/backends/detection.ts:50`

这里最关键的点不是返回值本身，
而是它**刻意不做什么**。

注释反复强调：

```text
只检查原始 TMUX env var
绝不 fallback 到 `tmux display-message`
```

原因也写得很清楚：

```text
只要系统上任何 tmux server 在跑，`tmux display-message` 就可能成功；
但那不代表“当前这个 Claude 进程是在 tmux 里面启动的”
```

这说明作者非常清楚要区分两个完全不同的问题：

#### 问题 A：系统上有没有 tmux 可用
- 用 `isTmuxAvailable()` 回答

#### 问题 B：当前 Claude 是不是从 tmux pane 里启动的
- 只能看原始 `TMUX`

如果把这两者混起来，就会误判：

```text
明明 Claude 不在 tmux 里，
却因为系统上有 tmux server 而被误认为 inside tmux
```

所以这段设计的本质是：

```text
host membership detection 必须严格，不能用 capability probing 冒充 membership probing
```

这是很关键的架构判断。

---

### 第三部分：同步版和异步版的 `isInsideTmux` 并存，说明“当前是否 inside tmux”是一个高频条件，且很多路径需要在 sync 上下文里立即判断

位置：`src/utils/swarm/backends/detection.ts:28`

这里既有：

- `isInsideTmuxSync()`
- `isInsideTmux()`

而异步版其实也只是读缓存/原始值，没有真正异步 IO。

这说明系统里有两类使用场景：

#### sync path
例如某些即时条件判断、状态计算、render path

#### async path
例如 backend detection / executor 逻辑里统一使用 async API 习惯

所以这个双接口的意义不在功能差异，
而在：

```text
让“inside tmux”这条环境事实能在不同调用风格中都低成本复用
```

这再次说明它是个非常基础的底层信号。

---

### 第四部分：`getLeaderPaneId()` 说明“leader 原始 pane 是哪个”被视为一条独立且重要的宿主事实

位置：`src/utils/swarm/backends/detection.ts:62`

这个函数只是返回：

- `ORIGINAL_TMUX_PANE || null`

看起来很简单，
但它的架构意义很大。

因为这说明系统不只关心：

- 在不在 tmux

还关心：

- leader 最初是在哪个具体 pane 启动的

也就是说 swarm pane 布局/回收/定位里，
“leader 原始 pane 身份”是个需要长期保真的 anchor。

这也和注释里的这句话一致：

```text
即使用户之后切换到别的 pane，
我们仍然需要知道 leader 原始 pane
```

所以这个模块存的不是抽象环境事实，
而是：

```text
带有后续控制/布局用途的宿主定位信息
```

---

### 第五部分：`isTmuxAvailable()` 明确只回答“tmux 装没装 / PATH 能不能找到”，与“是否 inside tmux”严格分工

位置：`src/utils/swarm/backends/detection.ts:70`

这里做的是：

- `execFileNoThrow(TMUX_COMMAND, ['-V'])`
- `result.code === 0`

这就是最典型的 capability probing：

```text
系统有没有 tmux 这个命令
```

它和 `isInsideTmux()` 刚好形成互补：

#### `isInsideTmux()`
当前会话成员关系 / 宿主归属

#### `isTmuxAvailable()`
系统能力可用性

这两个维度在 registry 里会被组合使用：

- inside tmux -> 直接选 tmux native
- 不 inside tmux 但 tmux available -> 还能选 tmux external session

所以这里可以把它记成：

```text
membership fact
vs
capability fact
```

的清晰分离。

---

### 第六部分：`isInITerm2()` 不是只看一个 env，而是三路信号联合判断，说明 iTerm2 探测比 tmux 更依赖启发式组合

位置：`src/utils/swarm/backends/detection.ts:78`

这里会同时看：

1. `TERM_PROGRAM === 'iTerm.app'`
2. `ITERM_SESSION_ID` 是否存在
3. `env.terminal === 'iTerm.app'`

然后把三者 OR 起来。

这和 tmux 的“只看原始 TMUX”形成了鲜明对比。

原因很明显：

- tmux 对“inside tmux”这件事有非常强、专门的环境变量约定
- iTerm2 则更像是多个弱信号拼起来判断“当前是不是 iTerm2”

所以这一段本质是在做：

```text
heuristic host detection for iTerm2
```

而不是像 tmux 那样的强确定性 membership 判断。

---

### 第七部分：`env.terminal` 作为第三路信号，说明这个模块会复用更高层终端识别结果，而不是完全自己从零判断

位置：`src/utils/swarm/backends/detection.ts:98`

这里引入了：

- `env` from `utils/env.js`
- `env.terminal === 'iTerm.app'`

这说明 detection 模块并不是孤立存在的。

它会适度复用代码库里更通用的终端环境识别能力，
把它作为 iTerm2 探测的一条补充信号。

也就是说：

```text
backend detection 不一定自己拥有所有环境认知，
必要时会消费通用 env subsystem 的判断结果
```

这是一个健康的模块边界：

- 不重复造轮子
- 但也不完全依赖单一来源

---

### 第八部分：`isIt2CliAvailable()` 明确不是查 `--version`，而是查 `session list`，说明它想验证的不是“命令存在”，而是“命令能真正打通 iTerm2 Python API”

位置：`src/utils/swarm/backends/detection.ts:112`

注释写得非常关键：

```text
不要用 --version
因为就算 iTerm2 preferences 里没启 Python API，--version 也会成功
真正到 `session split` 时才失败
```

因此这里改用：

- `it2 session list`

只有它成功，才认为 it2 CLI 可用。

这说明作者在这里验证的是：

```text
effective operational availability
```

而不是：

```text
binary presence only
```

这和 `isTmuxAvailable()` 的语义其实不太一样：

- tmux -V 足以判断 tmux command 是否可运行
- it2 则必须进一步验证 API 联通性

这说明不同 backend 的“available”语义其实并不完全相同，
而 detection 层对此是有意识建模的。

---

### 第九部分：tmux 和 iTerm2 两种探测路径的差异，反映了作者对“宿主判定”和“控制面可用性”边界的精细处理

把它们并排看会很清楚：

#### tmux
- inside tmux: 只看原始 `TMUX`
- tmux available: `tmux -V`

#### iTerm2
- in iTerm2: 多信号启发式
- it2 available: `it2 session list`，验证 API 可操作性

这说明 detection.ts 不是在强行用一套统一模板探测所有后端，
而是按每个宿主的真实特性分别设计：

```text
tmux 强在 membership 信号清晰
iterm2 强在宿主环境可识别，但控制 API 可用性必须额外验证
```

这也是 registry 决策树能可靠工作的基础。

---

### 第十部分：缓存 `isInsideTmuxCached` / `isInITerm2Cached` 说明这些环境事实被视为进程生命周期内稳定，不值得反复探测

位置：`src/utils/swarm/backends/detection.ts:21`

这里缓存了：

- `isInsideTmuxCached`
- `isInITerm2Cached`

注释也说：

```text
this won't change during the process lifetime
```

这和前面 registry 的缓存思路完全一致。

因为对当前 Claude 进程来说：

- 是不是从 tmux 启动的
- 是不是运行在 iTerm2

这些事实在启动后就基本固定了。

所以 detection 层的设计是：

```text
把宿主环境识别当作 session-invariant fact
```

而不是需要实时刷新的一类状态。

---

### 第十一部分：`resetDetectionCache()` 的存在再次说明这是一个全局环境事实模块，测试必须能显式重置

位置：`src/utils/swarm/backends/detection.ts:122`

这里会清掉：

- `isInsideTmuxCached`
- `isInITerm2Cached`

虽然 `ORIGINAL_USER_TMUX` / `ORIGINAL_TMUX_PANE` 是模块 load 时就固定的，
但至少缓存层仍然需要可测试重置。

这说明 detection 模块和 registry 一样，
都有典型的：

```text
process-global singleton-ish behavior
```

所以测试必须能控制它的缓存状态，
不然不同测试场景会互相污染。

---

### 第十二部分：这份文件真正做的不是“检测终端类型”，而是区分三类不同事实：宿主归属、系统能力、控制链路可用性

如果把全文件压缩，可以发现它在区分三种不同层级的事实：

#### 1. 宿主归属（membership）
- `isInsideTmux()`
- `getLeaderPaneId()`
- `isInITerm2()`

#### 2. 系统能力（capability）
- `isTmuxAvailable()`

#### 3. 控制链路是否真正可操作（operational availability）
- `isIt2CliAvailable()`

这三者不能混。

例如：

- 系统有 tmux，不等于当前 Claude 在 tmux 里
- 在 iTerm2 里，不等于 it2 CLI 真能控制 Python API

所以 detection.ts 的真正价值在于：

```text
把 backend 选择所需的底层事实拆细，而不是用一个粗暴“能不能用”布尔值糊过去
```

这也是 registry 那条优先级决策树能够写得准确的原因。

---

### 第十三部分：这份文件和 `registry.ts` 的关系，是“事实提供者”与“策略决策者”的分工

这点非常值得串一下。

#### `detection.ts`
回答：
- 现在是不是在 tmux / iTerm2
- leader pane 是哪个
- tmux/it2 CLI 是否可用

#### `registry.ts`
回答：
- 基于这些事实，应该选 tmux / iterm2 / in-process 的哪条路
- 是否 native
- 是否需要提示 it2 setup
- 是否 fallback 到 in-process

也就是说：

```text
detection.ts 提供 raw facts
registry.ts 提供 interpreted policy decision
```

这是一个非常健康的分层：

- 探测不带策略
- 策略不重复探测

---

### 元

问题：**这一站真正想解决的架构问题是什么？**

回答：这一站提供 pane backend 决策所需的环境事实，如是否在 tmux 中、leader pane 是谁、tmux 是否可用、当前是否是 iTerm2，以及 it2 CLI 是否真能打通 Python API。它只探测，不决策。

### 反

问题：**如果把这一站的设计反过来，会发生什么？**

回答：如果 detection 和 registry 混在一起，环境事实与策略选择就会纠缠。更麻烦的是，运行期环境变量可能被改写，不提前缓存原始值就会误判。

### 空

问题：**跳出当前文件名，这一站背后更大的问题是什么？**

回答：更大的问题是，平台决策应该建立在“事实层”和“策略层”的分离上。检测层越纯，后续选择逻辑越稳。


### 读完这一站后，你应该抓住的 9 个事实

1. `detection.ts` 是 swarm pane backend 决策所依赖的环境事实来源，不负责 backend 选择或 teammate spawn。
2. 它在模块加载时就捕获 `TMUX` 与 `TMUX_PANE` 的原始值，避免运行过程中被 `Shell.ts` 等逻辑改写后的 `process.env` 污染宿主判断。
3. `isInsideTmux()`/`isInsideTmuxSync()` 只根据原始 `TMUX` 判断当前 Claude 是否从 tmux 内启动，明确拒绝使用 `tmux display-message` 之类会导致误判的命令式 fallback。
4. `getLeaderPaneId()` 说明 leader 原始 pane ID 被视为一条独立且重要的宿主定位信息，而不只是 tmux 附带细节。
5. `isTmuxAvailable()` 只回答系统是否装有 tmux，这与“当前是否 inside tmux”是两个严格分离的维度。
6. `isInITerm2()` 通过 `TERM_PROGRAM`、`ITERM_SESSION_ID`、`env.terminal` 三路信号联合判断，说明 iTerm2 检测更依赖启发式组合。
7. `isIt2CliAvailable()` 用 `it2 session list` 而不是 `--version`，说明它验证的是 iTerm2 Python API 的实际可操作性，而不仅仅是命令是否存在。
8. 本文件实际上区分了三类事实：宿主归属、系统能力、控制链路可用性；这些概念如果混在一起，就会让 backend 选择逻辑失真。
9. 缓存与 `resetDetectionCache()` 的存在说明这些环境事实被视为进程生命周期内稳定的全局状态，同时测试必须能显式重置缓存以避免污染。

---

### 现在把第 48-49 站串起来

```text
spawnUtils.ts
  -> 决定 process-based teammate 启动时要继承哪些 command / flags / env
backends/detection.ts
  -> 决定当前环境里有哪些 pane backend 事实成立，为 registry 的 backend 选择提供底层输入
```

所以现在 process/pane teammate 这条链可以先压缩成：

```text
environment facts
  -> detection.ts

backend policy decision
  -> registry.ts

spawn inheritance strategy
  -> spawnUtils.ts

pane/process bootstrap execution
  -> PaneBackendExecutor.ts
```

也就是说：

```text
detection.ts 回答“当前环境到底是什么”
spawnUtils.ts 回答“决定好要起进程后，该怎样继承 leader 启动策略”
```

---

### 下一站建议

下一站最顺的是：

```text
src/utils/swarm/backends/it2Setup.ts
```

因为现在你已经看懂了：

- detection 层怎样判断 iTerm2 / tmux 宿主事实与可用性
- registry 会利用这些事实决定是否走 iTerm2、tmux fallback，或提示 `needsIt2Setup`
- iTerm2 还存在一个“用户是否宁愿继续用 tmux、不要反复被提示 it2 setup”的偏好问题

下一步最自然就是把这条 UX/偏好链补齐：

**`getPreferTmuxOverIterm2()` 这类 it2 setup 偏好状态到底存在哪里、怎样影响 backend 选择，以及系统如何避免在 iTerm2 + tmux fallback 场景下反复骚扰用户。**
