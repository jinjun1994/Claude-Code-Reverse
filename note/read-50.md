## 第 50 站：`src/utils/swarm/backends/it2Setup.ts`

### 这是什么文件

`src/utils/swarm/backends/it2Setup.ts` 是 swarm backend 体系里的 **iTerm2 Python API / it2 CLI 安装与用户偏好状态管理器**。

上一站 `detection.ts` 已经看懂：

```text
系统可以判断：
- 当前是不是在 iTerm2
- it2 CLI 能不能真正打通 Python API
- tmux 是否可用
```

但 registry 只靠这些“事实”还不够。

它还要解决更偏 UX / setup 的问题：

- 用户没装 `it2` CLI 时，怎么探测可安装手段
- 安装时用哪个 Python package manager
- 怎么验证不是“命令存在但 API 还没开”
- 用户如果更想继续用 tmux，怎样记住这个偏好
- 一旦 setup 成功，怎样避免后续反复提示

所以这一站回答的是：

```text
当 swarm 运行在 iTerm2 场景下时，
系统怎样把“可检测的 backend 能力”进一步转成“可落地的 setup / verify / remember preference”流程
```

所以最准确的一句话是：

```text
it2Setup.ts = iTerm2 backend 的安装、验证、记忆偏好辅助层
```

---

### 先看它的整体定位

位置：`src/utils/swarm/backends/it2Setup.ts:1`

这份文件不负责：

- backend 选择
- pane 创建
- iTerm2 split 操作
- CLI teammate 启动

它只负责三类事情：

1. 检测 / 安装 `it2` CLI
2. 验证 iTerm2 Python API 是否真的可用
3. 记录用户对 iTerm2 vs tmux 的持久偏好与 setup 完成状态

也就是说它是：

```text
iTerm2 readiness + preference persistence layer
```

这正好补在：

- `detection.ts` 的环境事实层
- `registry.ts` 的 backend 决策层

之间偏“setup UX”的空白地带。

---

### 第一部分：`PythonPackageManager` 的顺序是 `uvx` → `pipx` → `pip`，说明它在安装策略上优先追求隔离环境，其次才是传统 pip

位置：`src/utils/swarm/backends/it2Setup.ts:14`

这里定义：

- `uvx`
- `pipx`
- `pip`

而注释明确说顺序就是 preference order。

这说明作者不是单纯在问：

```text
系统里有没有任何办法安装 it2
```

而是在编码一个更细的安装偏好：

#### 第一优先级：`uvx`
更现代、隔离性更强

#### 第二优先级：`pipx`
也是隔离安装

#### 第三优先级：`pip`
兼容兜底

所以这段真正表达的是：

```text
it2 setup 的默认策略偏向最小污染的 Python tool installation
```

---

### 第二部分：`detectPythonPackageManager()` 本质上是在找“可用于安装 it2 的控制面入口”，不是在抽象 Python 运行时本身

位置：`src/utils/swarm/backends/it2Setup.ts:40`

这里依次做：

- `which uv`
- `which pipx`
- `which pip`
- `which pip3`

然后返回：

- `uvx`
- `pipx`
- `pip`
- 或 `null`

这里有个很值得注意的小语义：

- 明明检查的是 `uv`
- 返回类型却是 `uvx`

注释解释得很清楚：

```text
真正安装命令是 `uv tool install`
但类型名保留 `uvx` 是为了兼容
```

这说明这个枚举不是严格对应某个 binary 名称，
而是在表达一类：

```text
package-manager strategy label
```

也就是说这个函数服务的不是一般 Python 子系统，
而是后面 `installIt2(...)` 的安装分支选择。

---

### 第三部分：单独提供 `isIt2CliAvailable()`，说明“命令是否装了”与“命令是否真正可操作”被继续严格区分

位置：`src/utils/swarm/backends/it2Setup.ts:79`

这里的实现非常轻：

- `which it2`

只要找到就算 available。

这和上一站 `detection.ts` 里的：

- `it2 session list`

形成很清晰的语义分工：

#### `it2Setup.ts:isIt2CliAvailable()`
只回答 binary presence

#### `detection.ts:isIt2CliAvailable()`
回答 operational availability / Python API 可通信性

也就是说同名概念在两个文件里的关注点并不一样：

```text
setup 层先问“装了没”
backend detection 层再问“真能不能控制 iTerm2”
```

这说明作者对安装态与运行态的边界是有意识拆开的。

---

### 第四部分：`installIt2()` 最大的安全点不是安装命令本身，而是它刻意从 `homedir()` 执行，避免读项目目录下潜在恶意的 `pip.conf` / `uv.toml`

位置：`src/utils/swarm/backends/it2Setup.ts:90`

这里最关键的注释是：

```text
Run from home directory to avoid reading project-level pip.conf/uv.toml
which could be maliciously crafted to redirect to an attacker's PyPI server
```

这段非常重要。

说明作者清楚意识到：

```text
在当前仓库目录里跑 package install，
项目本地配置文件本身就是一个 supply-chain attack surface
```

所以这里不是简单执行：

- `uv tool install it2`
- `pipx install it2`
- `pip install --user it2`

而是统一通过：

- `execFileNoThrowWithCwd(..., { cwd: homedir() })`

把工作目录切到用户 home。

这实际上是在做：

```text
installation-context sanitization
```

这也是这一站最有价值的设计点之一。

---

### 第五部分：`installIt2()` 的三条安装路径说明它建模的是“全局可复用 CLI 工具安装”，而不是项目虚拟环境依赖

位置：`src/utils/swarm/backends/it2Setup.ts:98`

三条路径分别是：

#### `uvx`
- `uv tool install it2`

#### `pipx`
- `pipx install it2`

#### `pip`
- `pip install --user it2`
- 失败再试 `pip3 install --user it2`

这里没有：

- venv 激活
- poetry
- project requirements
- repo local dependency 管理

说明作者想装的是：

```text
用户 shell PATH 下可直接调用的 it2 CLI 工具
```

而不是某个项目私有 Python 环境中的包。

这与它作为 iTerm2 backend 前置能力完全一致。

---

### 第六部分：`pip` 分支里先试 `pip` 再试 `pip3`，说明这里优先考虑“尽量成功”的兼容性，而不是过度追求安装路径纯洁性

位置：`src/utils/swarm/backends/it2Setup.ts:111`

这里逻辑是：

1. `pip install --user it2`
2. 如果失败，再 `pip3 install --user it2`

这说明作者知道现实机器环境会分裂成：

- `pip` 指向正确 Python
- `pip` 不可用但 `pip3` 可用
- 两者都存在但只有一个通

所以 setup 层在这里的目标不是最优雅，
而是：

```text
用最保守、最兼容的方式把 it2 CLI 装起来
```

这和前面 `detectPythonPackageManager()` 的“有层次的优先级”形成互补：

- 优先选更干净的安装器
- 进入低优先级 pip 路径后，再尽量兼容不同系统命名

---

### 第七部分：安装失败时返回结构化 `It2InstallResult`，说明 setup 失败被当成可呈现给上层 UX 的正常状态，而不是异常流

位置：`src/utils/swarm/backends/it2Setup.ts:129`

失败时它不会直接 throw，
而是返回：

- `success: false`
- `error`
- `packageManager`

成功时返回：

- `success: true`
- `packageManager`

这说明它不是一个“内部 helper，失败就崩”的设计，
而是：

```text
面向 UI / prompt / setup flow 的结果对象 API
```

上层可以拿到：

- 用的是哪种安装方式
- 为什么失败
- 后面该提示什么

所以 `it2Setup.ts` 的接口风格明显是给交互式 setup 流程准备的。

---

### 第八部分：`verifyIt2Setup()` 的核心不是验证 binary presence，而是验证 `it2 session list` 能否真正连通 iTerm2 Python API

位置：`src/utils/swarm/backends/it2Setup.ts:152`

这部分和上一站其实正好呼应。

流程是：

1. 先 `isIt2CliAvailable()`
2. 再执行 `it2 session list`
3. 如果失败，分析 stderr
4. 返回更细的失败原因

这说明 verify 阶段真正关心的是：

```text
CLI present
!=
backend operationally ready
```

而这个 verify 函数就是把“命令存在”推进到：

```text
现在真的可以驱动 iTerm2 Python API 了没有
```

这正是 iTerm2 backend setup 流程的关键闭环。

---

### 第九部分：`needsPythonApiEnabled` 是一个很重要的细粒度结果位，说明 setup 流程明确区分“没装 it2”和“装了但 iTerm2 端没开 Python API”

位置：`src/utils/swarm/backends/it2Setup.ts:167`

这里会根据 stderr 是否包含：

- `api`
- `python`
- `connection refused`
- `not enabled`

来判断是否返回：

- `needsPythonApiEnabled: true`

这说明作者在 UX 上显式区分两类完全不同的问题：

#### 问题 A：CLI 侧没准备好
- 没装 `it2`
- PATH 找不到

#### 问题 B：宿主 iTerm2 侧没准备好
- Python API 没打开
- CLI 无法连通 iTerm2

如果不做这个拆分，用户只会得到一个模糊错误：

```text
it2 不可用
```

但这里实际上是在把 setup 失败原因结构化成：

```text
install problem
vs
host-side enablement problem
```

这对后面的提示文案非常重要。

---

### 第十部分：`getPythonApiInstructions()` 说明这个文件不只提供状态与操作，还直接提供 setup UX 文案素材

位置：`src/utils/swarm/backends/it2Setup.ts:200`

这里直接返回一组字符串：

- `Almost done! Enable the Python API in iTerm2:`
- `iTerm2 → Settings → General → Magic → Enable Python API`
- `After enabling, you may need to restart iTerm2.`

这说明 `it2Setup.ts` 不只是偏底层的 backend helper，
它还承担了一点：

```text
user-facing remediation guidance provider
```

也就是说上层如果发现：

- `needsPythonApiEnabled: true`

就可以直接拿这里的指引去提示用户，
而不必在别处重复硬编码一份 setup 文案。

---

### 第十一部分：`markIt2SetupComplete()` 记录的是“不要再提示 setup 了”的全局持久状态，而不是一次性进程缓存

位置：`src/utils/swarm/backends/it2Setup.ts:214`

这里通过：

- `getGlobalConfig()`
- `saveGlobalConfig(...)`

把：

- `iterm2It2SetupComplete: true`

写进全局配置。

这说明系统关心的不是：

```text
这次进程里 verify 成功过一次
```

而是：

```text
用户这台机器上的 it2 setup 已经完成，
以后可以减少提示噪音
```

所以这是一种：

```text
cross-session setup memory
```

也说明 iTerm2 setup 在产品设计里被视为一个会话外持久决策。

---

### 第十二部分：`setPreferTmuxOverIterm2()` / `getPreferTmuxOverIterm2()` 才是这份文件和 registry 真正打通的关键桥

位置：`src/utils/swarm/backends/it2Setup.ts:229`

这里把：

- `preferTmuxOverIterm2`

也持久化进 global config。

这和上一站 `registry.ts` 直接连上了：

- 当系统在 iTerm2 中运行时
- registry 会先看用户是否 prefer tmux
- 再决定是否继续走 iTerm2 setup / fallback / prompt 逻辑

所以这段代码真正编码的是：

```text
backend selection 不只是环境事实驱动，
也受用户历史偏好驱动
```

这一步非常重要，
因为它避免了系统陷入一种烦人的循环：

```text
我明明在 iTerm2 里，
但我就是更想用 tmux fallback，
系统却每次都继续催我配 it2
```

而这个偏好位正是为了消除这种重复打扰。

---

### 第十三部分：这份文件真正补上的，是 iTerm2 backend 决策链里“能力之外的人机协商层”

如果把整份文件压缩，会发现它处理的不是 backend 执行本身，
而是三类更靠近 setup/UX 的状态：

#### 1. 安装准备度
- 有没有可用 Python package manager
- `it2` CLI 能不能装上

#### 2. 运行准备度
- `it2 session list` 能不能打通 Python API
- 是否需要用户去开 iTerm2 设置

#### 3. 用户偏好记忆
- setup 是否已经完成
- 用户是否更偏好 tmux over iTerm2

也就是说：

```text
detection.ts 提供环境事实
registry.ts 做 backend 策略决策
it2Setup.ts 处理 iTerm2 路线里的 setup readiness 与用户偏好持久化
```

这三层合起来，iTerm2 这条路径才真正闭环。

---

### 第十四部分：从安全视角看，这份文件最值得记住的是它对 package-manager 执行上下文的防污染意识

位置：`src/utils/swarm/backends/it2Setup.ts:95`

这份文件本身不复杂，
但它有一个非常成熟的安全设计：

```text
安装第三方 CLI 时，不在当前项目目录执行，
避免仓库内恶意配置文件劫持包源
```

这点尤其重要，因为这类 setup helper 很容易被写成：

- 直接在当前 cwd 跑安装命令
- 认为“只是装个依赖，没什么风险”

而这里明确防住了一个真实的 supply-chain / config injection 面。

这说明作者对“开发工具安装流程也有攻击面”是有清醒认识的。

---

### 元

问题：**这一站真正想解决的架构问题是什么？**

回答：这一站专管 iTerm2 backend 的安装、验证和偏好记忆，包括 `it2` CLI 安装路径、Python API 连通性校验，以及用户更偏向 tmux 还是 iTerm2 的持久化偏好。它填补的是 setup UX，不是执行逻辑。

### 反

问题：**如果把这一站的设计反过来，会发生什么？**

回答：如果只有“能不能检测到 iTerm2”而没有 setup/verify 层，很多用户会卡在半配置状态。系统看起来支持 iTerm2，实际上却无法稳定落地。

### 空

问题：**跳出当前文件名，这一站背后更大的问题是什么？**

回答：更大的问题是，环境能力被检测到之后，怎样被转化成可用产品能力。setup 层的价值就在于把“理论可用”推进到“真实可用”。


### 读完这一站后，你应该抓住的 9 个事实

1. `it2Setup.ts` 不是 backend 执行器，而是 iTerm2 路线的安装、验证、偏好持久化辅助层。
2. 它把 Python 包管理器按 `uvx -> pipx -> pip` 排优先级，体现出对隔离安装环境的偏好。
3. `detectPythonPackageManager()` 关心的是“有什么可用来安装 it2 的入口”，而不是一般意义上的 Python 运行时抽象。
4. `isIt2CliAvailable()` 在本文件里只回答 `it2` binary 是否安装，与 `detection.ts` 里偏运行态的可操作性检查语义不同。
5. `installIt2()` 最大的安全点是统一从 `homedir()` 执行安装，避免项目目录里的恶意 `pip.conf` / `uv.toml` 劫持包源。
6. `verifyIt2Setup()` 的核心是验证 `it2 session list` 能否打通 iTerm2 Python API，而不只是确认命令存在。
7. `needsPythonApiEnabled` 让上层能区分“CLI 没装”与“iTerm2 宿主端没开 Python API”这两类不同问题。
8. `markIt2SetupComplete()` 与 `preferTmuxOverIterm2` 都写入 global config，说明 setup 完成状态和用户偏好都是跨会话持久记忆。
9. 这份文件真正补的是 iTerm2 backend 决策链中的 setup readiness + user preference 层，让 detection 与 registry 能落到更友好的用户体验上。

---

### 现在把第 49-50 站串起来

```text
detection.ts
  -> 提供 tmux / iTerm2 宿主归属、系统能力、控制链路可用性这些底层事实
it2Setup.ts
  -> 处理 iTerm2 路线的安装入口、setup 验证、Python API 启用指引，以及“prefer tmux”这类用户持久偏好
```

所以现在 iTerm2 / tmux 相关链路可以先压缩成：

```text
environment facts
  -> detection.ts

setup readiness + user preference
  -> it2Setup.ts

backend policy decision
  -> registry.ts
```

也就是说：

```text
detection.ts 回答“当前环境客观上是什么”
it2Setup.ts 回答“如果想走 iTerm2，这条路是否已准备好，以及用户到底愿不愿意走它”
```

---

### 下一站建议

下一站最顺的是：

```text
src/utils/swarm/backends/TmuxBackend.ts
```

因为现在你已经看懂了：

- registry 怎样根据环境事实与偏好决定选哪条 backend 路
- detection 怎样判断 tmux / iTerm2 宿主事实
- it2 setup 怎样处理 iTerm2 路线的安装与偏好持久化

下一步最自然就是回到具体 backend 实现本身，先看最核心也最常用的一条：

**tmux backend 自己到底怎样创建 swarm pane 布局、怎样发命令、怎样 kill pane、以及 native tmux 与 external session 这两种控制路径在实现层如何分叉。**
