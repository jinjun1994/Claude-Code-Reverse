# Claude Code 源码阅读笔记

## 先读这一页：从“逐站阅读”升级为“按模块理解”

这套笔记原本主要按站点推进：从 `read.md` 的早期总站笔记，到 `read-28.md` 之后的逐站拆分，再到 `read-143.md`、`read-146.md` 这样的架构总览文档。

现在更适合把它理解成 4 个顶层模块：

### 模块 1：启动与入口

先回答一个最上游的问题：**Claude Code 怎样从“进程启动”进入“可运行会话”？**

这一模块建议优先读：

- `read.md` 第 1-4 站：`main.tsx`、`entrypoints/init.ts`、`commands.ts`、`query.ts`
- `read-28.md` 到 `read-35.md`：CLI 入口、初始化、主链路继续展开
- `read-142.md`：第 195-198 站，补齐 Entrypoints / Commands / Constants / UI 完整图

### 模块 2：控制面与运行时

这一模块关注：**配置、权限、工具执行、Query 主循环如何拼成一次真实调用。**

建议重点读：

- `read-98.md`：Bash 安全与校验链
- `read-126.md`：工具执行管线、权限运行时、Hook、Turn Loop 综合视角
- `read-130.md`：Compact 系统
- `read-137.md`、`read-139.md`、`read-141.md`：Query、Scheduler、Shutdown、Session State 等运行时骨架

### 模块 3：扩展面与协作

这一模块关注：**Claude Code 怎样接入外部系统、扩展能力，并支持多智能体协作。**

建议重点读：

- `read-71.md` 到 `read-80.md`：MCP 相关
- `read-81.md` 到 `read-85.md`：Plugin / Marketplace
- `read-86.md` 到 `read-100.md`：Memory / Agent Team
- `read-135.md`、`read-136.md`：Bridge、Skill、Buddy、Coordinator、Remote
- `read-146.md`：Transport 层完整架构

### 模块 4：架构总览与阅读路径

这一模块回答：**如果不再按文件逐个读，而是要建立整体架构脑图，应该从哪里收束？**

建议重点读：

- `read-143.md`：架构导读、覆盖矩阵、关键设计决策
- `read-138.md`：中后段系统综合与总结
- `read-142.md`：入口与 UI 全景补图
- `read-146.md`：Transport 作为收尾的系统级补完

## 推荐阅读顺序

如果你是第一次读，建议按下面顺序：

1. 先读本文件开头 + 第 1-4 站，抓住启动编排、初始化、命令树、主循环。
2. 再跳到 `read-143.md`，建立全局模块地图。
3. 接着按“控制面与运行时”阅读核心执行链。
4. 最后读“扩展面与协作”，把 MCP、Plugin、Memory、Agent Team、Transport 拼回系统全景。

## 这份笔记现在怎么用

- 如果你想**按站点顺推**：从本文件继续往下读。
- 如果你想**按模块建立脑图**：先读本页，再配合 `read-143.md` 的矩阵跳读。
- 如果你想**抓住系统级架构**：优先看 `read-126.md`、`read-135.md`、`read-142.md`、`read-143.md`、`read-146.md`。

---

## 第 1 站：`src/main.tsx`

### 这是什么文件

`src/main.tsx` 是整个 Claude Code 的**真实主入口**。如果把整个项目类比成一台机器，这个文件不是某个普通模块，而是“总开关 + 总调度台”。

它负责的不是单一功能，而是把下面这些事情串起来：

- 进程启动时的早期初始化
- CLI 参数解析与 Commander 根命令注册
- interactive / print / special mode 的分流
- settings / auth / telemetry / plugin / MCP / team 等启动装配
- 最终把请求送到 REPL 或非交互执行路径

可以把它理解为：

```text
main.tsx = 启动编排层（startup orchestrator）
```

---

### 先看最重要的两个函数

#### 1. `main()`
位置：`src/main.tsx:585`

这是进程级入口，负责：

- 设置安全相关环境变量
- 注册退出与中断处理
- 提前改写某些特殊 argv（如 direct connect、deep link、assistant、ssh）
- 提前判断当前是不是 non-interactive
- 初始化 entrypoint / client type / session 基础状态
- 最后进入 `run()` 构建 Commander CLI

你可以把 `main()` 理解成：

```text
进程启动后的第一层总控
```

#### 2. `run()`
位置：`src/main.tsx:884`

这个函数负责真正构建 Commander 命令树：

- 创建 root command
- 注册 `preAction`
- 定义大量全局选项
- 定义默认 action
- 后续再挂 subcommands

你可以把 `run()` 理解成：

```text
CLI 路由装配层
```

所以：

```text
main() 负责“进程启动和模式预处理”
run() 负责“Commander 路由和命令执行入口”
```

---

### 读这个文件时，先抓 5 条主线

#### 主线 1：它不是普通 CLI 入口，而是“多宿主入口”
在 `main()` 前半段可以看到，它会提前处理很多特殊入口：

- `cc://` / `cc+unix://` 直连 URL
- `--handle-uri` deep link
- `assistant`
- `ssh`
- `-p/--print`
- `--init-only`
- `--sdk-url`

对应位置：
- `src/main.tsx:609`
- `src/main.tsx:644`
- `src/main.tsx:679`
- `src/main.tsx:702`
- `src/main.tsx:797`

这说明 Claude Code 不是“只有一个命令行入口”的工具，而是**多个宿主 / 多种启动姿态共用一个总入口**。

---

#### 主线 2：启动阶段非常重视“抢跑初始化”
文件最顶部在 import 期间就执行了几个 side effects：

- `profileCheckpoint('main_tsx_entry')`
- `startMdmRawRead()`
- `startKeychainPrefetch()`

对应位置：`src/main.tsx:1-20`

意思是：

- 性能 profiling 要尽早开始
- MDM 读取要和后续 import 并行
- keychain 预取要提前启动，避免后面同步阻塞

这说明 `main.tsx` 不只是“调模块”，它还承担**启动性能优化**职责。

---

#### 主线 3：真正的 CLI 初始化发生在 `run().hook('preAction')`
位置：`src/main.tsx:907`

这里非常关键。`preAction` 里做了很多“执行命令前必须完成”的初始化：

- 等待 MDM / keychain 预取完成
- `init()`
- 初始化 logging sinks
- 处理 `--plugin-dir`
- 跑 migrations
- 非阻塞加载 remote managed settings
- 非阻塞 settings sync

所以不要把 `main.tsx` 理解成：

```text
main() -> 直接业务执行
```

更准确是：

```text
main() -> run() -> Commander preAction -> 再进入真正 action
```

这就是 Claude Code CLI 控制面的关键结构。

---

#### 主线 4：默认 action 很大，承担 interactive / print 模式的主分流
默认 `.action(async (prompt, options) => { ... })` 从 `src/main.tsx:1006` 开始���

在这段里会做很多高层决策，例如：

- bare mode 开关
- prompt 预处理
- assistant / kairos 模式判定
- permission 相关选项提取
- output/input format 判定
- slash commands 开关
- tasks / team / agent 相关模式参数准备

所以默认 action 不是“简单接 prompt 然后调模型”，而是**真正的会话装配中心**。

---

#### 主线 5：这个文件本质上是“把所有子系统接起来”
只看 import 就能发现它把这些系统全接进来了：

- context：`src/context.ts`
- init：`src/entrypoints/init.ts`
- commands：`src/commands.ts`
- repl：`src/replLauncher.tsx`
- tools：`src/tools.ts`
- settings：`src/utils/settings/settings.ts`
- permissions：`src/utils/permissions/permissionSetup.ts`
- mcp：`src/services/mcp/client.ts`
- plugins：`src/plugins/bundled/index.ts`
- skills：`src/skills/bundled/index.ts`
- app state：`src/state/AppStateStore.ts`

这正说明：

```text
main.tsx 不是业务实现细节文件，
而是系统总装配文件。
```

---

### 第一轮阅读，应该先看懂什么

第一轮不要试图看懂这 4000+ 行所有分支。

只需要先看懂 4 件事：

#### A. `main()` 在做什么
记住它是：
- 进程入口
- 模式预处理层
- 早期安全 / session / signal 初始化层

#### B. `run()` 在做什么
记住它是：
- Commander 根命令构建层
- preAction 初始化层
- 默认 action / subcommands 注册层

#### C. `preAction` 在做什么
记住它是：
- 命令真正执行前的统一初始化关卡

#### D. 默认 `.action()` 在做什么
记住它是：
- interactive / print / assistant / ssh 等模式进入运行时前的总分流层

---

### 元

问题：**这一站真正想解决的架构问题是什么？**

回答：这一站的核心不是“启动一个 CLI”，而是把多种入口姿态统一收口到同一个启动编排层。它同时处理 early init、argv 改写、模式分流和 Commander 路由装配，是整机的总开关。

### 反

问题：**如果把这一站的设计反过来，会发生什么？**

回答：如果把入口逻辑拆散到各子命令或各模式里，启动路径会迅速分叉。deep link、assistant、ssh、print mode 这类特殊入口会越来越难保持一致。

### 空

问题：**跳出当前文件名，这一站背后更大的问题是什么？**

回答：更大的问题是，一个产品如何在“多宿主、多模式、多入口”下仍保持单一启动真相源。`main.tsx` 只是入口文件，背后其实是整套启动控制面的统一性问题。


### 读完 `main.tsx` 后，你应该建立的脑图

```text
进程启动
  -> main()
    -> 早期安全/信号/argv 特殊处理
    -> 判断 interactive vs non-interactive
    -> initializeEntrypoint()
    -> run()
      -> 构建 Commander root command
      -> preAction: init/settings/migrations/plugins/remote settings
      -> 默认 action: 模式分流与会话装配
      -> REPL 或 print 路径
```

---

### 这一站先不要深挖的东西

先跳过这些细节，不然会陷进去：

- 所有 CLI flags 的细枝末节
- assistant / kairos 的全部逻辑
- ssh / deep link / direct connect 的全部分支
- 每个 migration 做了什么
- 每个 telemetry event 做了什么

当前只要抓住：

```text
main.tsx = 入口编排 + CLI 路由 + 初始化装配
```

就够了。

---

### 下一站建议

读完 `src/main.tsx` 后，下一站最合适的是：

1. `src/entrypoints/init.ts`  —— 看“执行前初始化到底做了什么”
2. `src/commands.ts` —— 看“命令树和 slash command 怎么挂进来”
3. `src/query.ts` —— 看“真正的运行时主循环”

如果按最顺的主线，我建议下一站读：

```text
src/entrypoints/init.ts
```

---

## 第 2 站：`src/entrypoints/init.ts`

### 这是什么文件

`src/entrypoints/init.ts` 是 Claude Code 的**统一初始化入口**。

如果说 `src/main.tsx` 负责“把进程带到命令执行入口”，那 `src/entrypoints/init.ts` 负责的就是：

```text
在真正执行命令前，把运行环境准备好
```

它不是用来处理具体用户请求的，也不是主循环；它更像是一个“起飞前检查 + 基础设施装配”模块。

---

### 这个文件最重要的函数

#### 1. `init()`
位置：`src/entrypoints/init.ts:57`

这是整个文件最核心的函数，而且被 `memoize(...)` 包起来了。

这意味着：

- 初始化逻辑是**幂等**的
- 同一进程里即使多次调用，也只会真正执行一次

这一点非常重要，因为 Claude Code 里有很多命令路径、模式路径、异步分支，如果没有 memoize，很容易重复初始化。

所以第一结论就是：

```text
init() = 全局只执行一次的初始化总入口
```

#### 2. `initializeTelemetryAfterTrust()`
位置：`src/entrypoints/init.ts:247`

这个函数说明了一件非常关键的事情：

**Telemetry 初始化被刻意拆成了“信任前”和“信任后”两个阶段。**

也就是说，Claude Code 不是一启动就把所有东西全配好，而是会根据 trust / remote managed settings 的状态，延后某些初始化动作。

这反映了它的一个重要架构原则：

```text
初始化不是一次性平铺完成，而是分阶段完成。
```

---

### `init()` 到底做了什么

下面按执行顺序理解。

#### 1. 启用配置系统
位置：`src/entrypoints/init.ts:62-69`

这里先 `enableConfigs()`。

意思是：Claude Code 的很多能力不是直接读死配置，而是先把配置系统整体拉起来，再让后续模块通过统一接口读取配置。

这一步是后续所有 settings / env / policy / telemetry 的前提。

---

#### 2. 只应用“安全的环境变量”
位置：`src/entrypoints/init.ts:71-84`

这里非常关键：

- `applySafeConfigEnvironmentVariables()`
- `applyExtraCACertsFromConfig()`

这里体现的是一个很重要的安全设计：

```text
trust 建立前，只允许应用 safe env vars
trust 建立后，才应用完整 env vars
```

也就是说，Claude Code 不会在最早期把所有配置都无脑注入到进程环境里。

这是为了避免：
- 不可信目录影响启动行为
- TLS / 证书 / 网络配置过早被污染

这也是你读整个项目时要记住的一条主线：

**Claude Code 很多地方都在区分“可以在 trust 前做的事”和“必须在 trust 后做的事”。**

---

#### 3. 建立优雅退出机制
位置：`src/entrypoints/init.ts:86-88`

`setupGracefulShutdown()` 在初始化早期就执行。

说明这个项目不是把 shutdown 当作边角料，而是把它当成基础设施的一部分。

这和后面很多能力有关：
- transcript flush
- telemetry flush
- background tasks 清理
- LSP manager 清理
- team cleanup

---

#### 4. 启动一批“后台预热任务”
位置：`src/entrypoints/init.ts:90-132`

这一段本质上是在做异步预热：

- first-party event logging
- GrowthBook refresh hook
- OAuth account info 补齐
- JetBrains 检测缓存
- 当前仓库检测缓存
- remote managed settings loading promise
- policy limits loading promise
- first start time 记录

这些都说明：

```text
init() 不只是同步初始化，
还负责为后续路径启动大量后台预取 / 预热 / promise 协调。
```

特别要注意这两项：

- `initializeRemoteManagedSettingsLoadingPromise()`
- `initializePolicyLimitsLoadingPromise()`

这不是“立刻把设置全加载完”，而是**先把 promise 和等待机制建好**，方便后续别的模块去 await。

这是一种很典型的“初始化先搭协调结构，再逐步补数据”的设计。

---

#### 5. 配置网络底座
位置：`src/entrypoints/init.ts:134-159`

这一段是初始化里的另一条主线：

- `configureGlobalMTLS()`
- `configureGlobalAgents()`
- `preconnectAnthropicApi()`

这三步连起来看，意思非常清楚：

1. 先配置 mTLS
2. 再配置全局代理 / HTTP agents
3. 再预连 Anthropic API

所以这里不是“网络功能模块”，而是：

```text
Claude Code 在正式发请求前，先把网络传输层底座铺好
```

尤其 `preconnectAnthropicApi()` 很值得注意，它说明这个项目会主动优化首个 API 请求延迟，而不是被动等待首次调用时才建连接。

---

#### 6. 处理 remote 特有代理链路
位置：`src/entrypoints/init.ts:161-183`

如果处于 remote 环境，还会额外初始化 upstream proxy。

关键点不是“它用了什么代理”，而是：

```text
init() 会根据宿主环境分叉初始化路径
```

也就是说，初始化不是一条静态固定链，而是带环境判定的动态装配链。

---

#### 7. 注册清理逻辑
位置：`src/entrypoints/init.ts:188-200`

这里注册了至少两类 cleanup：

- `shutdownLspServerManager`
- `cleanupSessionTeams()`

这一点很重要，因为说明初始化阶段不仅管“启动”，也提前把“退出时怎么收尾”登记好了。

所以更准确地说：

```text
init() = 启动装配 + 生命周期清理注册
```

---

#### 8. 初始化 scratchpad
位置：`src/entrypoints/init.ts:202-209`

如果 scratchpad 开启，会确保目录存在。

这属于权限 / 文件系统辅助基础设施的一部分，说明 `init()` 也承担少量本地运行环境准备职责。

---

### 这个文件体现出的 4 个架构原则

#### 原则 1：初始化要幂等
靠 `memoize(init)` 保证。

#### 原则 2：初始化要分阶段
尤其是：
- trust 前 vs trust 后
- 立即执行 vs 后台预热
- 本地模式 vs remote 模式

#### 原则 3：初始化不仅做启动，也登记清理
`registerCleanup(...)` 是关键证据。

#### 原则 4：初始化优先准备“基础设施”，不是直接碰业务
这里主要准备的是：
- config
- env
- network
- telemetry
- oauth metadata
- policy / remote settings promise
- cleanup hooks

而不是直接进入 query loop。

---

### 你应该如何把它和 `main.tsx` 串起来

把两站连起来看，主链路现在应该更新为：

```text
main.tsx
  -> run()
    -> preAction
      -> init()
        -> 配置系统
        -> safe env
        -> graceful shutdown
        -> 预热任务
        -> 网络层准备
        -> cleanup 注册
      -> 再进入具体 command/action
```

这一步非常关键。

因为很多人读 Claude Code 会误以为：

```text
main.tsx -> 直接开始对话
```

但真实情况是中间隔着一个很厚的初始化层，而这个初始化层就在 `src/entrypoints/init.ts`。

---

### 第一轮阅读时，最该记住的点

只记这 5 句就够：

1. `init()` 是全局一次性的初始化入口
2. 它做的是基础设施装配，不是业务执行
3. 它区分 trust 前 / trust 后的初始化边界
4. 它既负责启动，也负责注册退出清理
5. 它为后续 command / REPL / query loop 铺路

---

### 暂时不用深挖的部分

先别陷进去看：

- telemetry 的具体 OpenTelemetry 实现
- GrowthBook 的完整刷新机制
- upstream proxy 的完整 remote 细节
- OAuth account populate 的内部实现
- scratchpad 的完整用途

当前重点是理解：

```text
init.ts = Claude Code 的运行环境准备层
```

---

### 下一站建议

现在最顺的下一站是：

```text
src/commands.ts
```

原因是：
- `main.tsx` 让你知道“从哪进来”
- `init.ts` 让你知道“执行前准备了什么”
- `commands.ts` 会让你知道“命令树到底怎么组织”

等 `commands.ts` 读完，再去 `src/query.ts`，你对整条链会更稳。

---

---

### 元

问题：**这一站真正想解决的架构问题是什么？**

回答：这一站解决的是“正式执行前，环境到底怎样被安全地准备好”。它把配置、环境变量、信任状态、telemetry 等初始化拆成分阶段流程，而不是一次平铺完成。

### 反

问题：**如果把这一站的设计反过来，会发生什么？**

回答：如果初始化不做幂等控制，也不区分 trust 前后，重复调用和过早注入配置都会变成隐患。启动过程会更脆弱，安全边界也会被模糊。

### 空

问题：**跳出当前文件名，这一站背后更大的问题是什么？**

回答：更大的问题不是初始化多少步骤，而是初始化应不应该有阶段感和权限感。系统越复杂，越需要把“能先做什么、信任后再做什么”讲清楚。

## 第 3 站：`src/commands.ts`

### 这是什么文件

`src/commands.ts` 是 Claude Code 的**命令注册中心**。

如果说：

- `main.tsx` 负责总入口和模式分流
- `init.ts` 负责执行前初始化

那么 `commands.ts` 负责的就是：

```text
系统里到底有哪些命令，它们怎么被组装成最终可用命令集
```

它不是某个单独命令的实现文件，而是**整个 command surface 的聚合层**。

---

### 读这个文件时，先抓住一句话

```text
commands.ts = 内置命令 + skills + plugin commands + workflow commands + 动态 skills 的总装配器
```

这是这个文件最核心的定位。

因为 Claude Code 的“命令”并不只是 `/help`、`/clear` 这种内置 slash command，它还包括：

- builtin commands
- bundled skills
- skill 目录里的 commands
- plugin commands
- plugin skills
- workflow commands
- MCP skill commands（部分场景）
- dynamic skills

所以它不是一个简单的数组清单，而是一个**多来源命令汇总层**。

---

### 这个文件最重要的几个部分

#### 1. 顶部 import 区
位置：`src/commands.ts:1` 开始

这里一眼能看到 Claude Code 命令面的复杂度。

它导入了大量命令实现，例如：

- `./commands/help/index.js`
- `./commands/mcp/index.js`
- `./commands/plugin/index.js`
- `./commands/tasks/index.js`
- `./commands/permissions/index.js`
- `./commands/plan/index.js`
- `./commands/agents/index.js`
- `./commands/memory/index.js`
- `./commands/review.js`
- `./commands/commit.js`

同时还有 feature gate 控制的条件命令：
- `assistant`
- `bridge`
- `voice`
- `workflows`
- `fork`
- `buddy`
- `torch`
- `peers`

这说明第一件大事：

```text
Claude Code 的命令面是“静态命令 + feature-gated 命令”混合构成的
```

也就是说，最终命令集不是写死的，而是跟构建特性、用户类型、环境有关。

---

#### 2. `INTERNAL_ONLY_COMMANDS`
位置：`src/commands.ts:225`

这组命令是内部专用命令集合。

它告诉你一件很重要的事：

```text
同一套源码里存在“外部用户可见命令”和“内部构建专用命令”两层命令面
```

这意味着 Claude Code 的 command surface 不是对所有发行形态都完全一致。

比如：
- 某些命令只在 ant/internal 环境出现
- 某些命令会被 external build 剪掉

从源码阅读角度，这很重要，因为你不能看到命令实现就默认用户一定能用到。

---

#### 3. `COMMANDS = memoize(() => [...])`
位置：`src/commands.ts:258`

这是最核心的静态命令列表。

它本质上是在说：

```text
先定义一个“基础命令集合”
```

这个集合里包括：
- addDir
- agents
- branch
- clear
- compact
- config
- doctor
- help
- mcp
- memory
- plugin
- resume
- skills
- status
- tasks
- vim
- review
- securityReview
- hooks
- permissions
- plan
- files
- model
- theme
- outputStyle
- 以及大量 gated commands

这里要特别注意两点：

##### A. 它被 `memoize` 了
说明这个基础集合不是每次临时重新拼，而是有缓存的。

##### B. 它刻意写成函数，不在模块加载时立即求值
源码注释已经说明原因：

- 底层函数会读 config
- 而 config 不能在模块初始化阶段就读取

所以这里体现了一个很重要的设计：

```text
命令注册虽然看起来像静态声明，
但它实际上被延迟到了合适的初始化时机。
```

---

#### 4. `getSkills(cwd)`
位置：`src/commands.ts:353`

这是第二个关键点。

它会同时加载 4 类 skill 来源：

- `skillDirCommands`
- `pluginSkills`
- `bundledSkills`
- `builtinPluginSkills`

这说明在 Claude Code 里，skills 不是单一路径来的。

你可以把它理解成：

```text
skills 是一个聚合概念，不是一个单一目录概念
```

也就是说，命令系统不只接 builtin command，还会接入一整套“可被模型调用的 prompt command 生态”。

---

#### 5. `loadAllCommands(cwd)`
位置：`src/commands.ts:449`

这是整个文件真正的“命令总装配器”。

它会并行加载：

- skills
- pluginCommands
- workflowCommands

然后把它们和 `COMMANDS()` 拼起来：

```text
bundledSkills
builtinPluginSkills
skillDirCommands
workflowCommands
pluginCommands
pluginSkills
COMMANDS()
```

这个顺序非常值得记住。

因为它说明 Claude Code 的命令系统不是“内置命令为主，其他是补丁”，而是把多个来源命令统一并入一个总列表，再交给后续逻辑做过滤。

所以第二个核心结论是：

```text
commands.ts 解决的不是“定义命令”，而是“合并命令来源”
```

---

#### 6. `getCommands(cwd)`
位置：`src/commands.ts:476`

这是你阅读这个文件时最重要的导出函数。

它做的事分三层：

##### 第一层：拿到所有来源命令
调用 `loadAllCommands(cwd)`。

##### 第二层：做 availability + isEnabled 过滤
只保留：
- `meetsAvailabilityRequirement(cmd)`
- `isCommandEnabled(cmd)`

##### 第三层：插入 dynamic skills
如果运行时发现了动态 skills，还会把它们去重后插入到 plugin skills 和 builtin commands 之间。

所以你应该把 `getCommands(cwd)` 理解成：

```text
返回“当前用户、当前环境、当前 cwd 下真正可用的命令列表”
```

这不是静态清单，而是**上下文相关的最终命令视图**。

---

### `meetsAvailabilityRequirement(cmd)` 在解决什么问题
位置：`src/commands.ts:417`

这个函数专门处理“命令是否对当前用户可见”。

源码里至少区分了：

- `claude-ai`
- `console`

也就是说，有些命令不是单纯由 feature flag 决定，而是还取决于：

- 你是不是 claude.ai 订阅用户
- 你是不是 console API 用户
- 你是不是 3P provider 用户
- 你当前的 base URL 是什么

这就说明命令可见性其实有两层：

```text
1. feature / build 层
2. account / provider availability 层
```

这一点非常重要，因为很多功能并不是“代码在就能用”。

---

### `getSkillToolCommands(...)` 在暗示什么
位置：`src/commands.ts:563`

这个函数很值得注意。

它说明 SkillTool 看到的，并不是“所有命令”，而是经过筛选的**prompt-based model-invocable commands**。

过滤条件包括：
- `cmd.type === 'prompt'`
- 不能 `disableModelInvocation`
- 不是 builtin
- 必须满足某些来源 / 描述条件

这意味着：

```text
“命令系统” 和 “模型可调用 skill 系统” 是重叠但不相同的两层
```

也就是说：
- 有些命令是给用户手动敲的
- 有些命令也是模型可调用 skill
- 但不是所有命令都能被模型调用

这是后面读 `SkillTool`、tool 系统时非常重要的背景知识。

---

### 这个文件体现出的 5 个架构事实

#### 事实 1：命令是多来源聚合的
不是只有 `./commands/*`。

#### 事实 2：命令集不是静态固定的
它受以下因素影响：
- feature gates
- USER_TYPE
- auth/provider 状态
- cwd
- plugin / skill / workflow 装载结果
- dynamic skills

#### 事实 3：`getCommands(cwd)` 才是最终视图
不要把 `COMMANDS()` 当成最终命令集。

#### 事实 4：skills 在命令系统里是一等公民
不是外挂，不是额外补充，而是命令组装逻辑的一部分。

#### 事实 5：命令系统和 tool 系统是两层不同结构
这里解决的是：
- 用户或模型“可以触发哪些命令”

而不是：
- tool_use 怎么执行

这两者后面要分开理解。

---

### 你现在应该怎么把前三站串起来

到现在为止，你可以把主链路理解为：

```text
main.tsx
  -> run()
    -> preAction
      -> init()
    -> 默认 action / subcommands
      -> getCommands(cwd)
        -> 组装当前环境可用命令集
```

所以前三站分别在回答三个问题：

1. `main.tsx`：系统从哪里进来？
2. `init.ts`：执行前准备了什么？
3. `commands.ts`：当前环境到底有哪些命令？

这三步拼起来后，Claude Code 的 CLI 控制面就已经清楚很多了。

---

### 第一轮阅读时，不要陷进去的点

先别深挖这些：

- 每个具体 command 的实现细节
- 每个 feature flag 的业务意义
- workflows 的完整机制
- plugin skills 和 builtin plugin skills 的所有差异
- MCP skill commands 的更深层用法

当前只要先抓住：

```text
commands.ts = 命令注册与多来源聚合层
```

---

### 下一站建议

现在最应该读的是：

```text
src/query.ts
```

因为到这里为止，你已经知道：
- 怎么进入系统
- 执行前做了什么准备
- 命令怎么被组装

下一步就该看：

**真正的一次对话 / 一次 turn 在运行时内部是怎么跑的。**

而这正是 `src/query.ts` 的核心职责。


---

---

### 元

问题：**这一站真正想解决的架构问题是什么？**

回答：这一站真正关心的不是某个命令，而是命令面如何被汇总出来。内置命令、skills、plugin commands、workflow commands 和动态 skills 都在这里汇合。

### 反

问题：**如果把这一站的设计反过来，会发生什么？**

回答：如果没有统一装配层，每种命令来源都会各自暴露接口。最后用户看到的命令面会变得碎裂，特性开关和来源差异也难以治理。

### 空

问题：**跳出当前文件名，这一站背后更大的问题是什么？**

回答：更大的问题是，一个可扩展 CLI 如何维护统一 command surface。`commands.ts` 只是列表聚合点，背后其实是“命令生态如何收口”的架构题。

## 第 4 站：`src/query.ts`

### 这是什么文件

`src/query.ts` 是 Claude Code 的**运行时主循环**。

如果前面三站回答的是：

- 从哪里进来（`main.tsx`）
- 执行前准备了什么（`init.ts`）
- 当前有哪些命令（`commands.ts`）

那么这一站回答的是：

```text
一次真正的对话 / turn，在运行时里到底怎么推进
```

它不是单纯“发一次模型请求”的包装器，而是把这些事情串成一个循环：

- 构造本轮请求上下文
- 做 compact / microcompact / context collapse
- 发起模型流式采样
- 识别 tool_use
- 执行工具
- 回写 tool_result / attachments
- 处理 stop hooks / token budget / max turns
- 再决定是否进入下一轮

所以最准确的定位是：

```text
query.ts = Claude Code 的 turn loop / agent loop 控制器
```

---

### 先看最重要的两个入口

#### 1. `query(params)`
位置：`src/query.ts:219`

这是对外暴露的主入口。

它本身很薄，核心只做两件事：

1. 创建 `consumedCommandUuids`
2. `yield* queryLoop(params, consumedCommandUuids)`

结束后再统一把已消费命令标记成 `completed`。

所以你可以把它理解成：

```text
query() = 对外 API 包装层
```

真正的复杂度不在这里，而在 `queryLoop(...)`。

#### 2. `queryLoop(...)`
位置：`src/query.ts:241`

这才是整个文件的核心。

它是一个 `while (true)` 的异步生成器循环，不断地产出：

- `stream_request_start`
- assistant messages
- tool results
- attachments
- tombstones
- summaries

直到返回一个 terminal reason，例如：

- `completed`
- `max_turns`
- `prompt_too_long`
- `blocking_limit`
- `aborted_streaming`
- `aborted_tools`
- `hook_stopped`
- `stop_hook_prevented`

所以这一层不是“一次 API call”，而是：

```text
一个会跨多轮 assistant -> tool -> assistant 的状态机循环
```

---

### 这个文件最关键的几个类型

#### 1. `QueryParams`
位置：`src/query.ts:181`

这是 query loop 的输入总配置。

里面最重要的字段有：

- `messages`
- `systemPrompt`
- `userContext`
- `systemContext`
- `canUseTool`
- `toolUseContext`
- `fallbackModel`
- `querySource`
- `maxOutputTokensOverride`
- `maxTurns`
- `taskBudget`
- `deps`

这说明 `query()` 并不是只吃一个 prompt string。

它真正吃的是：

```text
一整套运行时上下文 + 工具能力 + 控制参数 + 依赖注入
```

尤其 `deps` 很重要，它意味着这套 loop 并不把所有实现写死，而是允许把模型调用、compact、microcompact 等能力通过依赖注入接进来。

---

#### 2. `State`
位置：`src/query.ts:204`

这是跨循环迭代传递的可变状态。

里面有：

- `messages`
- `toolUseContext`
- `autoCompactTracking`
- `maxOutputTokensRecoveryCount`
- `hasAttemptedReactiveCompact`
- `maxOutputTokensOverride`
- `pendingToolUseSummary`
- `stopHookActive`
- `turnCount`
- `transition`

这一点非常关键。

Claude Code 在这里没有把状态散落到一堆局部变量，而是显式维护一个 loop state。

这说明：

```text
query.ts 不是线性流程，而是一个显式状态推进器
```

特别是 `transition` 字段很值得记住，它记录“上一轮为什么继续”，方便测试和调试恢复路径。

---

### 先抓整条主链路

把 `queryLoop(...)` 粗略展开，可以先得到这张脑图：

```text
进入 queryLoop
  -> 初始化 state / budget / config / memory prefetch
  -> while(true)
    -> 取当前 messages 和 toolUseContext
    -> 做 skill prefetch
    -> 处理 tool result budget / snip / microcompact / context collapse / autocompact
    -> 组装 system prompt 与 query options
    -> 调用模型流式输出
    -> 收集 assistant messages / tool_use blocks / streaming tool results
    -> 如果没有 tool_use：走 stop hooks / budget / recovery / return
    -> 如果有 tool_use：执行工具
    -> 注入 attachments / memory / queued commands / refreshed tools
    -> 生成下一轮 state
    -> continue
```

先有这张图，再看细节就不会迷路。

---

### 这个文件最重要的 8 条主线

#### 主线 1：它是“消息驱动的递进循环”，不是单请求函数
位置：`src/query.ts:306` 开始

`while (true)` 这一点决定了这个文件的本质。

每一轮都不是“从零开始”，而是基于上一轮的 `state.messages` 继续往前推进。

所以 Claude Code 的 turn loop 更像：

```text
不断扩展消息历史，并让模型基于更新后的历史继续推演
```

而不是传统 RPC 风格的“请求 -> 响应 -> 结束”。

---

#### 主线 2：每轮发请求前，会先做大量“上下文整形”
关键位置：
- `applyToolResultBudget(...)`：`src/query.ts:379`
- `snipCompactIfNeeded(...)`：`src/query.ts:401`
- `deps.microcompact(...)`：`src/query.ts:414`
- `contextCollapse.applyCollapsesIfNeeded(...)`：`src/query.ts:440`
- `deps.autocompact(...)`：`src/query.ts:454`

这部分特别重要。

说明 Claude Code 在真正调模型前，不是直接把完整消息数组丢过去，而是会按层做上下文管理：

1. 先限制大 tool result 的体积
2. 再做 snip
3. 再做 microcompact
4. 再做 context collapse
5. 再做 autocompact

所以你应该记住：

```text
query.ts 的第一大职责不是“调模型”，而是“把可送给模型的上下文整理成可控形态”
```

这也是为什么 Claude Code 能支持长对话、长工具输出、多轮 agent loop。

---

#### 主线 3：模型调用是流式的，而且内建 fallback / withhold / recovery
关键位置：
- `deps.callModel(...)`：`src/query.ts:659`
- fallback 处理：`src/query.ts:893`
- prompt too long recovery：`src/query.ts:1065`
- max_output_tokens recovery：`src/query.ts:1185`

这段是整个运行时的核心中的核心。

这里不是简单 `await model()`，而是：

- 流式接收 assistant message
- 中途可能触发 model fallback
- 某些错误先不立刻 yield，而是 withheld
- 等确认 recovery 失败后再真正暴露错误

这说明 Claude Code 的设计原则是：

```text
优先把一次 turn 修复并继续跑下去，
而不是一出错就立刻把失败暴露给上层
```

举几个典型恢复路径：

- `prompt_too_long`：先尝试 collapse drain，再尝试 reactive compact
- `max_output_tokens`：先提高上限，再插入 meta recovery message 继续
- `FallbackTriggeredError`：切到 fallback model 重试

也就是说，`query.ts` 本身就内建了一个**恢复控制层**。

---

#### 主线 4：tool_use 才是继续循环的真正信号
关键位置：`src/query.ts:551-558`

这里有一句很关键的注释：

```ts
// Note: stop_reason === 'tool_use' is unreliable
```

所以代码并不依赖 API 的 stop reason，而是自己扫描 assistant content 里的 `tool_use` block：

- 如果出现 `tool_use`，`needsFollowUp = true`
- 如果没有，则进入 stop hooks / completion 路径

这说明 Claude Code 对 tool loop 的控制，不是依赖外部 API 的某个单一字段，而是**自己根据消息内容做判定**。

这是很稳的实现思路。

---

#### 主线 5：工具执行有两条路径：streaming / batch
关键位置：
- `StreamingToolExecutor` 初始化：`src/query.ts:561`
- streaming path 收集结果：`src/query.ts:847`
- 统一 toolUpdates 来源：`src/query.ts:1380`

你应该把它理解成：

```text
query.ts 不直接执行具体工具，
它只负责决定“这一轮工具调用由哪种执行器来跑”
```

两种模式：

1. **streamingToolExecutor**
   - 在 assistant streaming 阶段就能逐步接收完成的 tool result
   - 更像边生成边执行

2. **runTools(...)**
   - 在 assistant 输出结束后统一跑工具
   - 更像批处理执行

这说明工具系统和 query loop 是解耦的：

- `query.ts` 负责 orchestration
- 具体执行放在 `services/tools/*`

这也正好为后面读 tool call loop 铺路。

---

#### 主线 6：工具跑完后，不是立刻下一轮，而是先注入“附加上下文”
关键位置：
- `getAttachmentMessages(...)`：`src/query.ts:1580`
- memory prefetch consume：`src/query.ts:1599`
- skill prefetch consume：`src/query.ts:1620`
- refresh tools：`src/query.ts:1659`

这一段非常值得重视。

Claude Code 在工具执行结束后，会把很多“辅助上下文”再塞回消息流：

- 文件改动类 attachment
- 排队中的 command / task notification
- memory attachment
- skill discovery attachment
- 新连上的 MCP tools 刷新结果

所以一轮 loop 的输入并不只是：

```text
上轮 assistant + tool_result
```

而是：

```text
上轮 assistant + tool_result + attachment + memory + queued notifications + refreshed tool set
```

这说明 Claude Code 的运行时是一个**多输入源汇流系统**。

---

#### 主线 7：停止条件不是单一的，而是很多层共同决定
关键位置：
- abort streaming：`src/query.ts:1015`
- stop hooks：`src/query.ts:1267`
- token budget：`src/query.ts:1308`
- max turns：`src/query.ts:1704`
- hook stopped continuation：`src/query.ts:1518`

也就是说，一次 turn 什么时候停，不只是“模型不再请求工具”。

还会受这些因素影响：

- 用户中断
- stop hooks 阻止继续
- token budget 达到阈值
- max turns 达到上限
- hook attachment 要求停止 continuation

所以 `query.ts` 其实是一个：

```text
多重停止条件仲裁器
```

这也是它为什么复杂。

---

#### 主线 8：它高度重视可观测性和恢复安全
通篇可见：
- `queryCheckpoint(...)`
- `logEvent(...)`
- `logError(...)`
- `logAntError(...)`
- `transition` state
- tombstone orphaned messages

这说明 Claude Code 不是只追求“能跑”，还特别在意：

- 哪一步慢
- 哪一步失败
- fallback 是否触发
- compact 是否成功
- orphaned partial streaming output 如何清理

特别是 tombstone orphaned messages 这一段（`src/query.ts:712-740`）很关键：

如果 streaming fallback 发生，旧的半截 assistant message 不能留在 transcript/UI 里，因为其中可能有无效 thinking signatures。

所以它会显式发 tombstone 把这些 orphaned message 清掉。

这体现的是：

```text
query.ts 不只管理“正确路径”，也精细管理失败路径下的状态一致性
```

---

### 你可以把 `queryLoop` 切成 5 个阶段来看

#### 阶段 A：循环准备阶段
做的事：
- 取 state
- 建立 queryTracking
- 启动 skill/memory prefetch
- 整理 messagesForQuery

#### 阶段 B：上下文压缩与整形阶段
做的事：
- tool result budget
- snip
- microcompact
- context collapse
- autocompact

#### 阶段 C：模型流式采样阶段
做的事：
- `deps.callModel(...)`
- 收集 assistant messages
- 收集 tool_use blocks
- 处理中途 fallback / withheld errors / streaming executor

#### 阶段 D：分叉阶段
分成两条：

1. **没有 tool_use**
   - stop hooks
   - token budget
   - recovery / completion / return

2. **有 tool_use**
   - 执行工具
   - 收集 tool_result
   - 注入 attachment / memory / queued commands
   - 更新 tools
   - 进入下一轮

#### 阶段 E：状态回写阶段
做的事：
- 组装下一轮 `State`
- `state = next`
- 继续 while loop

这 5 段脑内分层会让整个文件清楚很多。

---

### 这一站最该记住的 6 句话

1. `query.ts` 是 Claude Code 的 turn loop，不是单次请求函数。
2. 它先做上下文整形，再调模型，再跑工具，再决定是否续轮。
3. `tool_use` 的存在才是进入下一轮的真正信号。
4. 它内建了 fallback、compact、overflow、max token 等恢复路径。
5. 工具执行后还会继续注入 attachment / memory / queued commands / refreshed tools。
6. 它本质上是 Claude Code 运行时的总调度循环。

---

### 现在把前四站串起来

到这里，主链路应该升级成：

```text
main.tsx
  -> run()
    -> preAction
      -> init()
    -> 默认 action / subcommands
      -> getCommands(cwd)
      -> query(params)
        -> queryLoop()
          -> 上下文整形
          -> 流式采样
          -> tool loop
          -> attachment/memory 注入
          -> 下一轮 or 结束
```

这样你就第一次真正看到了 Claude Code 从 CLI 控制面进入运行时内环的闭环。

---

### 第一轮阅读时，先不要深挖的部分

先别陷进去看这些细节：

- 每一种 compact 策略的内部算法
- `deps.callModel` 背后的 API 层细节
- `StreamingToolExecutor` 的完整实现
- token budget 的统计细节
- task budget 与 server-side budget 的全部关系
- skill prefetch / memory prefetch 的完整实现

当前阶段你只要先牢牢记住：

```text
query.ts = Claude Code 的运行时主循环
```

就够了。

---

### 下一站建议

下一站最顺的是：

```text
src/Tool.ts
```

原因很简单：

- `commands.ts` 让你知道“可触发哪些命令”
- `query.ts` 让你知道“一轮运行时怎么推进”
- `Tool.ts` 会让你知道“工具在运行时里到底以什么抽象存在”


---

---

### 元

问题：**这一站真正想解决的架构问题是什么？**

回答：这一站回答的是一次真实对话在运行时里如何推进，而不是单次模型请求怎么发。它把 compact、流式采样、tool_use、tool_result、stop hooks 和多轮循环连成一个 turn loop。

### 反

问题：**如果把这一站的设计反过来，会发生什么？**

回答：如果把 query 理解成一次 API call，就无法解释 assistant 和 tools 的多轮往返。很多中断、预算和终止原因也会失去统一收口位置。

### 空

问题：**跳出当前文件名，这一站背后更大的问题是什么？**

回答：更大的问题是，agent 系统的基本单位到底是“请求”还是“循环”。这一站说明 Claude Code 实际上把一次工作建模成跨多轮的状态机。

## 第 5 站：`src/Tool.ts`

### 这是什么文件

`src/Tool.ts` 是 Claude Code 的**工具抽象定义层**。

如果说：

- `commands.ts` 解决“有哪些命令可用”
- `query.ts` 解决“一轮运行时如何推进”

那么 `Tool.ts` 解决的是：

```text
运行时里的“工具”到底是什么对象、必须实现什么能力、带着什么上下文运行
```

它不是某个具体工具的实现文件，而是所有内置工具、MCP 工具、包装型工具共同依赖的**统一协议定义**。

所以一句话先记住：

```text
Tool.ts = Claude Code 工具系统的接口层 + 上下文层 + 通用类型层
```

---

### 先看最重要的 3 个核心概念

#### 1. `ToolUseContext`
位置：`src/Tool.ts:158`

这是整个文件最重要的类型，没有之一。

它不是一个小参数对象，而是“工具运行时上下文总线”。

里面装着的东西非常多，核心可以分成 6 类：

##### A. 运行配置
- `options.commands`
- `options.tools`
- `options.mainLoopModel`
- `options.thinkingConfig`
- `options.mcpClients`
- `options.agentDefinitions`
- `options.refreshTools`

##### B. 生命周期与中断控制
- `abortController`
- `setInProgressToolUseIDs`
- `setHasInterruptibleToolInProgress`
- `setResponseLength`
- `setStreamMode`

##### C. 应用状态读写
- `getAppState()`
- `setAppState(...)`
- `setAppStateForTasks(...)`

##### D. UI / 交互桥接
- `setToolJSX`
- `appendSystemMessage`
- `sendOSNotification`
- `requestPrompt(...)`
- `openMessageSelector()`
- `setSDKStatus(...)`

##### E. 记忆 / 文件 / 追踪状态
- `readFileState`
- `nestedMemoryAttachmentTriggers`
- `loadedNestedMemoryPaths`
- `dynamicSkillDirTriggers`
- `discoveredSkillNames`
- `queryTracking`
- `contentReplacementState`
- `renderedSystemPrompt`

##### F. 子代理 / 权限 / 特殊运行态
- `agentId`
- `agentType`
- `requireCanUseTool`
- `toolUseId`
- `localDenialTracking`
- `criticalSystemReminder_EXPERIMENTAL`

这说明一个非常重要的架构事实：

```text
在 Claude Code 里，工具不是“拿输入 -> 返回输出”的纯函数，
而是运行在一个很厚的 runtime context 里的有状态执行单元。
```

也就是说，工具系统天然和：

- AppState
- UI
- 权限
- memory
- agent
- MCP
- 中断/恢复

这些子系统强连接。

---

#### 2. `Tool<Input, Output, P>`
位置：`src/Tool.ts:362`

这是所有工具都要实现的统一接口。

最核心的方法有：

- `call(...)`
- `description(...)`
- `inputSchema`
- `checkPermissions(...)`
- `prompt(...)`
- `userFacingName(...)`
- `mapToolResultToToolResultBlockParam(...)`
- `renderToolUseMessage(...)`

你应该把它理解成：

```text
Tool 接口定义了“一个工具从模型可见、到权限判定、到执行、到结果回写、到 UI 渲染”的完整生命周期面
```

也就是说，一个工具不只是“怎么执行”，还要同时回答这些问题：

1. 它给模型看的 schema 是什么？
2. 它给用户看的权限描述是什么？
3. 它能不能执行？
4. 它是不是只读 / destructive / concurrency safe？
5. 它的结果怎么转成 `tool_result` block？
6. 它在 UI 中怎么显示？
7. 它在 transcript/search 中怎么被索引？

所以 `Tool.ts` 本质上定义的是一个**全栈工具协议**，而不只是执行签名。

---

#### 3. `buildTool(def)`
位置：`src/Tool.ts:783`

这是工具定义的统一构造器。

它做的事看起来简单，本质却很关键：

```ts
return {
  ...TOOL_DEFAULTS,
  userFacingName: () => def.name,
  ...def,
}
```

也就是说，Claude Code 没有要求每个工具都手写全量接口，而是允许工具作者只写必要部分，再由 `buildTool` 补默认行为。

默认补的关键项包括：

- `isEnabled` → `true`
- `isConcurrencySafe` → `false`
- `isReadOnly` → `false`
- `isDestructive` → `false`
- `checkPermissions` → allow
- `toAutoClassifierInput` → `''`
- `userFacingName` → `name`

这里非常能体现设计倾向：

```text
Tool.ts 用“统一默认值 + 显式覆盖”来降低工具实现成本，同时保持调用侧拿到的永远是完整 Tool 对象。
```

这能减少大量 `?.()` / fallback 判断。

---

### `ToolPermissionContext` 在这个文件里为什么重要
位置：`src/Tool.ts:123`

`ToolPermissionContext` 也是关键类型。

里面包括：

- `mode`
- `additionalWorkingDirectories`
- `alwaysAllowRules`
- `alwaysDenyRules`
- `alwaysAskRules`
- `isBypassPermissionsModeAvailable`
- `shouldAvoidPermissionPrompts`
- `awaitAutomatedChecksBeforeDialog`
- `prePlanMode`

它说明：

```text
权限系统不是工具外部的附属物，
而是 Tool 抽象本身就显式依赖的一部分。
```

因为工具接口里有：

- `checkPermissions(...)`
- `description(...)` 也需要 permission context
- `prompt(...)` 也依赖 permission context

所以后面读 permission runtime 时，你要把它理解成：

```text
权限系统是挂在 Tool 接口层上的，不是 query.ts 临时拼上去的
```

---

### 这个接口最值得关注的几个能力面

#### 能力面 1：执行面
关键方法：`call(...)`
位置：`src/Tool.ts:379`

这是最直接的一层：工具真正怎么执行。

签名里能看到 5 个输入：

- `args`
- `context`
- `canUseTool`
- `parentMessage`
- `onProgress`

这里有两个特别值得记住：

##### `canUseTool`
说明工具内部在需要时还能再次走权限/可用性判断，不是只能在外层调度器里判断一次。

##### `onProgress`
说明工具接口天生支持进度流，不是执行完才一次性返回。

这也是为什么 Bash、Agent、MCP、TaskOutput 这种长时工具能有细粒度进度显示。

---

#### 能力面 2：模型暴露面
关键成员：
- `inputSchema`
- `inputJSONSchema`
- `description(...)`
- `prompt(...)`
- `strict`
- `shouldDefer`
- `alwaysLoad`

这部分说明 Tool 接口不仅服务于“运行时执行”，还服务于“模型怎么看到工具”。

尤其要注意：

##### `shouldDefer`
表示工具可 deferred，不会在初始 prompt 中完整展开，而要通过 ToolSearch 才能进入模型视野。

##### `alwaysLoad`
表示即使开启 ToolSearch，这个工具也必须在 turn 1 就让模型看到。

所以你要记住：

```text
Tool.ts 同时定义了“执行协议”和“prompt 暴露协议”。
```

这就是为什么后面读 `src/tools.ts` 时会看到“built-in tools / deferred tools / MCP tools”这些区别。

---

#### 能力面 3：安全面
关键成员：
- `validateInput(...)`
- `checkPermissions(...)`
- `isReadOnly(...)`
- `isDestructive(...)`
- `preparePermissionMatcher(...)`
- `toAutoClassifierInput(...)`

这一层非常关键。

Claude Code 并没有把“安全”只放在 query loop 或单独 permissions 模块里，而是让每个工具自己申明其安全属性。

例如：

- 它是不是只读
- 它是不是 destructive
- 它的输入是否合法
- 它如何参与 permission rule 匹配
- 它该给 classifier 暴露什么紧凑表示

所以可以说：

```text
工具的安全边界，一部分由统一权限系统控制，
另一部分由工具自己通过接口元数据声明。
```

---

#### 能力面 4：并发与中断面
关键成员：
- `isConcurrencySafe(...)`
- `interruptBehavior?()`
- `contextModifier?`（在 `ToolResult` 中）

这一层经常被忽略，但很重要。

`Tool.ts` 明确把并发安全性和中断行为做成了工具属性：

- 某工具能不能并发跑
- 用户提交新消息时它是 cancel 还是 block
- 工具结束后是否需要修改 `ToolUseContext`

这说明：

```text
工具不是统一调度策略下的同质节点，
而是每种工具都能声明自己的并发/中断语义。
```

这也是为什么工具编排层不能只看名字，还得读工具元数据。

---

#### 能力面 5：UI / transcript 渲染面
关键成员非常多：
- `renderToolUseMessage(...)`
- `renderToolResultMessage(...)`
- `renderToolUseProgressMessage(...)`
- `renderToolUseRejectedMessage(...)`
- `renderToolUseErrorMessage(...)`
- `renderGroupedToolUse(...)`
- `renderToolUseTag?(...)`
- `extractSearchText?(...)`
- `getToolUseSummary?(...)`
- `getActivityDescription?(...)`

这一层说明一个非常重要的事实：

```text
Claude Code 的 Tool 抽象不是后端纯协议，
而是把 transcript/UI 表示也内建进同一个对象里。
```

也就是说，同一个 Tool 对象既知道：

- 怎么执行
- 怎么做权限判断
- 怎么映射成 tool_result
- 怎么在界面里渲染 tool use / result / progress / error

这是非常“前后端一体”的设计。

---

### `ToolResult<T>` 在解决什么问题
位置：`src/Tool.ts:321`

`ToolResult<T>` 不只是返回 `data`。

它还能带：

- `newMessages?`
- `contextModifier?`
- `mcpMeta?`

这就说明：

```text
工具的输出不一定只是“给模型看的结果文本”，
还可以顺便向运行时注入额外消息、上下文修改、MCP 协议元数据。
```

尤其这三项分别代表：

- `newMessages`：工具执行后可以额外补消息进入消息流
- `contextModifier`：可修改后续执行上下文
- `mcpMeta`：给 SDK / MCP 消费者透传结构化元数据

所以工具返回值本质上是：

```text
数据 + 运行时副作用 + 协议附带信息
```

---

### `findToolByName(...)` / `toolMatchesName(...)` 在暗示什么
位置：
- `src/Tool.ts:348`
- `src/Tool.ts:358`

这里虽然只是小函数，但很有代表性。

它说明工具查找不是简单的 `name === xxx`，还支持 `aliases`。

也就是说：

```text
工具标识层考虑了重命名 / 兼容性，不要求所有调用方只依赖唯一主名。
```

这跟命令系统里的 alias / 向后兼容设计是相呼应的。

---

### 你应该如何理解这个文件的定位

读完后，最好建立下面这个脑图：

```text
Tool.ts
  -> 定义 ToolUseContext（工具运行所处环境）
  -> 定义 ToolPermissionContext（权限上下文）
  -> 定义 Tool 接口（执行 + 权限 + prompt 暴露 + UI 渲染）
  -> 定义 ToolResult（结果 + 副作用）
  -> 定义 Tools 集合类型
  -> 提供 buildTool() 统一补默认实现
```

所以它的真正角色不是“某个工具文件”，而是：

```text
整个工具子系统的基础协议文件
```

---

### 这一站最该记住的 7 句话

1. `Tool.ts` 定义的是工具系统的统一协议，不是具体工具实现。
2. `ToolUseContext` 是工具运行时上下文总线，连接 AppState、UI、memory、agent、MCP、权限等子系统。
3. `Tool` 接口同时覆盖执行、权限、prompt 暴露、结果映射、UI 渲染。
4. 工具不是纯函数，而是运行在厚上下文里的有状态执行单元。
5. `ToolResult` 不只是 data，还可以携带新消息、上下文修改和 MCP 元数据。
6. `buildTool()` 用统一默认值把工具定义补全，降低实现成本。
7. 工具系统从接口层开始就是“执行层 + 安全层 + 展示层”一体化设计。

---

### 现在把前五站串起来

到现在为止，主链路可以升级成：

```text
main.tsx
  -> run()
    -> preAction
      -> init()
    -> getCommands(cwd)
    -> query(params)
      -> queryLoop()
        -> 识别 tool_use
        -> 根据 Tool 接口调度工具
          -> 工具在 ToolUseContext 中执行
          -> 走 validate / permissions / progress / result mapping / rendering
        -> 写回消息流
```

这一步很关键，因为从这里开始，你已经不是只在看“控制流”，而是在看 Claude Code 运行时最核心的抽象边界：

```text
query loop 负责编排
Tool 接口负责定义工具能力边界
```

---

### 第一轮阅读时，先不要深挖的点

先别陷进去看这些：

- 每个 render 方法在 UI 层里的具体组件实现
- `PermissionResult` 的完整分支语义
- `ToolProgressData` 各具体 progress 类型的全部字段
- `requestPrompt` / `setToolJSX` 在不同宿主里的细节
- `contentReplacementState` 的完整生命周期

当前阶段重点只要抓住：

```text
Tool.ts = 工具系统的协议层
```

就够了。

---

### 下一站建议

下一站最顺的是：

```text
src/tools.ts
```

原因：

- `Tool.ts` 让你知道“工具抽象长什么样”
- `src/tools.ts` 会让你知道“系统实际把哪些工具组装成最终工具集”


---

---

### 元

问题：**这一站真正想解决的架构问题是什么？**

回答：这一站定义了“工具到底是什么”以及工具运行时携带哪些上下文。尤其 `ToolUseContext` 很关键，它把配置、生命周期、UI、状态读写、记忆和 agent 身份都绑进同一条上下文总线。

### 反

问题：**如果把这一站的设计反过来，会发生什么？**

回答：如果工具只被看成输入输出函数，很多 UI 桥接、权限控制和运行时状态都只能散落在外部。那样工具会变轻，但系统集成成本会急剧升高。

### 空

问题：**跳出当前文件名，这一站背后更大的问题是什么？**

回答：更大的问题是，工具在 agent 架构里究竟是纯函数，还是带上下文的执行单元。Claude Code 明显选择了后者，因为真实工具调用远不止算一个返回值。

## 第 6 站：`src/tools.ts`

### 这是什么文件

`src/tools.ts` 是 Claude Code 的**工具装配中心**。

如果说：

- `src/Tool.ts` 定义了“一个工具是什么”
- `src/query.ts` 定义了“工具什么时候被调度”

那么 `src/tools.ts` 定义的就是：

```text
系统最终到底把哪些工具组装进运行时，以及这些工具如何按环境/权限/MCP 状态被过滤
```

它不是具体工具实现文件，而是 built-in tools、feature-gated tools、MCP tools 的**总装配层**。

所以一句话先记：

```text
tools.ts = Claude Code 的工具注册与工具池装配层
```

---

### 先抓最重要的 4 个导出函数

#### 1. `getAllBaseTools()`
位置：`src/tools.ts:193`

这是整个文件最关键的基础函数。

它返回的是：

```text
当前环境里“理论上可能可用”的完整内置工具清单
```

注意这里的关键词是：

- **当前环境**
- **理论上可能可用**
- **内置工具**

它会把大量工具统一放进一个数组里，例如：

- `AgentTool`
- `BashTool`
- `FileReadTool`
- `FileEditTool`
- `FileWriteTool`
- `NotebookEditTool`
- `WebFetchTool`
- `WebSearchTool`
- `TodoWriteTool`
- `AskUserQuestionTool`
- `SkillTool`
- `EnterPlanModeTool`
- `Task*` 系列
- `ListMcpResourcesTool`
- `ReadMcpResourceTool`
- `ToolSearchTool`
- 以及大量 feature gate / ant-only / env-only 工具

所以你可以把它理解成：

```text
getAllBaseTools() = built-in tool universe 的来源总表
```

但它还不是最终给模型/运行时使用的工具池。

---

#### 2. `getTools(permissionContext)`
位置：`src/tools.ts:271`

这是 built-in tool 的实际筛选函数。

它做的不是“列出所有工具”，而是：

```text
根据 permission context、模式开关、REPL 状态、环境变量，产出当前可用的 built-in 工具集合
```

它至少做了几层过滤：

1. **simple mode 特判**
   - `CLAUDE_CODE_SIMPLE` 下只保留极少数工具
   - bare + REPL 时甚至直接返回 `REPLTool`

2. **special tools 排除**
   - 先从 base tools 里去掉 `ListMcpResourcesTool` / `ReadMcpResourceTool` / synthetic output 这类特殊项

3. **deny rules 过滤**
   - `filterToolsByDenyRules(...)`

4. **REPL mode 隐藏 primitive tools**
   - 如果 `REPLTool` 已启用，就把 `REPL_ONLY_TOOLS` 从直接工具列表中隐藏

5. **`isEnabled()` 最终过滤**
   - 再把运行时禁用的工具剔除

所以更准确地说：

```text
getTools(...) = 当前上下文下最终可用的 built-in 工具视图
```

---

#### 3. `assembleToolPool(permissionContext, mcpTools)`
位置：`src/tools.ts:345`

这是 built-in tools 和 MCP tools 的真正汇合点。

它的职责非常明确：

1. 先拿 `getTools(permissionContext)`
2. 再过滤 `mcpTools`
3. built-in 和 MCP 各自按名称排序
4. 合并后按 `name` 去重，built-in 优先

也就是说：

```text
assembleToolPool() = 系统最终完整工具池的单一真相来源
```

源码注释已经写得很明确：

- REPL.tsx
- runAgent.ts

都应该依赖这个函数，保证工具池组装一致。

这点非常重要，因为它说明 Claude Code 很清楚“工具装配逻辑不能在多个调用点各写一份”。

---

#### 4. `getMergedTools(permissionContext, mcpTools)`
位置：`src/tools.ts:383`

这个函数比较轻：

```ts
return [...builtInTools, ...mcpTools]
```

它更像一个“宽松合并视图”，适合：

- tool search 阈值计算
- token counting
- 某些需要把 MCP tools 也算进去的上下文

所以你要区分：

- `assembleToolPool(...)`：真正运行时使用的、带过滤和去重的完整工具池
- `getMergedTools(...)`：更宽松的合并视图

这两个名字很像，但职责不一样。

---

### 文件顶部 import 区在说明什么
位置：`src/tools.ts:1-157`

这段本身就很有信息量。

因为你会发现工具来源很多，而且装配方式也不一样：

#### 1. 直接静态导入的常驻工具
比如：
- `AgentTool`
- `SkillTool`
- `BashTool`
- `FileReadTool`
- `FileEditTool`
- `FileWriteTool`
- `GlobTool`
- `WebFetchTool`
- `WebSearchTool`

#### 2. feature gate / USER_TYPE 控制的条件工具
比如：
- `SleepTool`
- cron tools
- `RemoteTriggerTool`
- `MonitorTool`
- `WorkflowTool`
- `WebBrowserTool`
- `CtxInspectTool`

#### 3. lazy require 解决循环依赖的工具
比如：
- `getTeamCreateTool()`
- `getTeamDeleteTool()`
- `getSendMessageTool()`

#### 4. 环境条件工具
比如：
- `PowerShellTool`
- LSPTool
- worktree tools
- testing tools

这一段说明的核心架构事实是：

```text
Claude Code 的工具集不是静态常量，而是一个由 feature、env、宿主类型、循环依赖约束共同决定的动态装配结果。
```

---

### `getAllBaseTools()` 最值得注意的 5 件事

#### 1. 它是“内置工具宇宙”的中心表
位置：`src/tools.ts:193`

以后你想知道“Claude Code 自带哪些工具”，最先看这里。

这就是 built-in tool surface 的主索引。

---

#### 2. 它会根据宿主能力做裁剪
例如：
- `hasEmbeddedSearchTools()` 时不再提供 `GlobTool` / `GrepTool`
- ant-only 才有 `ConfigTool`、`TungstenTool`、`REPLTool`
- feature 开启才有 `ToolSearchTool`、`SnipTool`、`WorkflowTool`

也就是说：

```text
工具清单和发行形态、宿主能力强相关，不是“一份代码一个固定工具表”。
```

---

#### 3. 它同时装配“元工具”
这里的“工具”不只是外部能力工具，还包括很多控制面工具：

- `AgentTool`
- `SkillTool`
- `EnterPlanModeTool`
- `ExitPlanModeV2Tool`
- `TaskCreateTool`
- `TaskUpdateTool`
- `EnterWorktreeTool`
- `ExitWorktreeTool`

这说明 Claude Code 的 tool system 不只是“执行 shell / 读写文件 / 搜索网页”，还包括：

```text
对 Claude 自身运行方式进行控制的管理型工具
```

这点很重要，因为后面你会看到：

- 命令系统里也有 `/plan`、`/tasks`
- 工具系统里也有 `EnterPlanMode`、`TaskCreate`

两层是对应又不完全相同的。

---

#### 4. 它已经提前考虑 ToolSearch
位置：`src/tools.ts:247-249`

即使 ToolSearch 的最终 defer 决策不是在这里做，`getAllBaseTools()` 也会在 optimistic 条件下把 `ToolSearchTool` 放进来。

这说明：

```text
tools.ts 负责提供“可进入 prompt 的工具宇宙”，
而更细的 defer / prompt 裁剪在后续层处理。
```

---

#### 5. 它必须保持 prompt-cache 稳定性
位置：`src/tools.ts:191`

源码里专门有注释：

```text
MUST stay in sync ... in order to cache the system prompt across users
```

这说明工具列表不是随便排的。

它会直接影响 system prompt caching。

也就是说：

```text
tools.ts 不只是“注册工具”，还是 prompt cache 稳定性的一个组成部分。
```

这个点非常关键。

---

### `filterToolsByDenyRules(...)` 在解决什么问题
位置：`src/tools.ts:262`

这个函数很值得记住。

它做的是：

```text
在工具真正暴露给模型之前，先把被 blanket deny 的工具直接从工具池里剔除
```

而不是等模型调用时再拒绝。

特别是它还支持 MCP server 级别规则，比如：

- `mcp__server`

这意味着某个 MCP server 下的整组工具，可以在 prompt 暴露前就整体消失。

所以它反映的是一个重要原则：

```text
权限控制不只发生在 tool call 时，也发生在 tool exposure 阶段。
```

这是安全设计上非常重要的一层。

---

### `getTools(...)` 里最值得记住的两个分支

#### 分支 1：simple mode
位置：`src/tools.ts:271-298`

这里说明 Claude Code 存在一种极简工具模式：

- 正常 simple mode：`BashTool + FileReadTool + FileEditTool`
- bare + REPL mode：直接给 `REPLTool`
- coordinator mode 下再补 `AgentTool` / `TaskStopTool` / `SendMessageTool`

这个分支说明：

```text
工具集不是只有“全量模式”，而是存在按运行模式大幅收缩能力面的情况。
```

---

#### 分支 2：REPL mode 隐藏 primitive tools
位置：`src/tools.ts:312`

这里很关键。

如果 REPL mode 开启且 REPL tool 已在工具池里，那么原始的底层 primitive tools 会被隐藏。

也就是说：

```text
REPLTool 在某些模式下是一个上层包装器，底层原语仍存在，但不直接暴露给模型。
```

这跟 `Tool.ts` 里的 `isTransparentWrapper` 设计是可以对上的。

---

### `assembleToolPool(...)` 为什么特别关键
位置：`src/tools.ts:345`

这段你最好单独记住 3 个点：

#### 1. built-in 和 MCP 分区排序
不是简单拼接，而是先各自排序。

#### 2. built-in 作为连续前缀保留
这是为了 system prompt cache 稳定。

#### 3. built-in 名字冲突时优先于 MCP
因为 `uniqBy` 保留前面的项。

所以这段解决的其实不是“合并数组”，而是：

```text
在保证缓存稳定性、命名冲突可控、权限过滤一致的前提下，生成最终完整工具池
```

这就是 Claude Code tool assembly 的核心。

---

### 这一站最该记住的 6 句话

1. `tools.ts` 是工具注册与工具池装配层，不是具体工具实现层。
2. `getAllBaseTools()` 定义了当前环境下可能存在的内置工具宇宙。
3. `getTools(permissionContext)` 产出当前上下文下最终可用的 built-in 工具集。
4. `filterToolsByDenyRules(...)` 说明权限会在工具暴露前就参与过滤。
5. `assembleToolPool(...)` 是 built-in + MCP 工具汇合的单一真相来源。
6. 工具列表顺序不仅影响功能，还影响 system prompt cache 稳定性。

---

### 现在把前六站串起来

到这里，主链路可以再升级一层：

```text
main.tsx
  -> run()
    -> preAction
      -> init()
    -> getCommands(cwd)
    -> getTools()/assembleToolPool()
    -> query(params)
      -> queryLoop()
        -> 识别 tool_use
        -> 根据 Tool 接口执行工具
        -> 写回 tool_result / attachment
```

这时候你已经能回答两个关键问题了：

1. **工具在抽象上是什么？** —— `src/Tool.ts`
2. **工具在运行时是怎么被组装成最终工具池的？** —— `src/tools.ts`

这两个点一通，后面再去读 tool orchestration 就会顺很多。

---

### 第一轮阅读时，先不要深挖的部分

先别陷进去看：

- 每个具体工具实现细节
- ToolSearch 的完整 defer 策略
- MCP tools 的完整生成过程
- coordinator mode 的全部细节
- REPLTool 内部怎么包底层原语

当前只要牢牢记住：

```text
tools.ts = 工具池装配层
```

就够了。

---

### 下一站建议

下一站最顺的是：

```text
src/services/tools/toolOrchestration.ts
```

因为：

- `Tool.ts` 让你知道工具抽象
- `tools.ts` 让你知道工具池如何组成
- `toolOrchestration.ts` 会让你知道一次 `tool_use` 到底如何被查找、校验、调度、并发执行、回写


---

---

### 元

问题：**这一站真正想解决的架构问题是什么？**

回答：这一站是工具池装配中心，负责把 built-in tools、feature-gated tools 和 MCP tools 组装成当前会话真正可用的集合。重点不在“有哪些工具”，而在“哪些工具此刻能被暴露”。

### 反

问题：**如果把这一站的设计反过来，会发生什么？**

回答：如果没有这一层，每个调用方都得自己判断环境、权限和开关。工具集合会在不同路径下出现隐性分叉，模型看到的能力边界也会混乱。

### 空

问题：**跳出当前文件名，这一站背后更大的问题是什么？**

回答：更大的问题是，工具系统如何从“理论全集”收敛成“当前上下文下的实际能力集”。这其实是产品能力暴露面的动态裁剪问题。

## 第 7 站：`src/services/tools/toolOrchestration.ts`

### 这是什么文件

`src/services/tools/toolOrchestration.ts` 是 Claude Code 的**工具调度层**。

如果说：

- `Tool.ts` 定义“工具是什么”
- `tools.ts` 定义“最终有哪些工具可用”
- `query.ts` 定义“什么时候进入 tool loop”

那么这一站解决的是：

```text
当模型已经产出了多个 tool_use block 之后，这些工具调用到底如何被分组、并发、串行、回写上下文
```

所以它不是具体工具实现，也不是权限判断细节文件，而是：

```text
toolOrchestration.ts = tool_use 批次调度器
```

---

### 先看最重要的入口

#### `runTools(...)`
位置：`src/services/tools/toolOrchestration.ts:19`

这是整个文件的总入口。

它接收：

- `toolUseMessages`
- `assistantMessages`
- `canUseTool`
- `toolUseContext`

然后做的事非常清楚：

1. 先 `partitionToolCalls(...)`
2. 再按 batch 逐批执行
3. concurrency-safe batch 走并发路径
4. 非 concurrency-safe batch 走串行路径
5. 把 message 更新和 context 更新逐步 yield 回上层

所以最准确的理解是：

```text
runTools() = 把一组 tool_use block 按安全性分批调度，然后持续产出执行结果
```

这说明 query loop 并不直接决定“每个工具怎么并发”，而是把这件事下沉给 tool orchestration 层。

---

### 这个文件最关键的 3 个函数

#### 1. `partitionToolCalls(...)`
位置：`src/services/tools/toolOrchestration.ts:91`

这个函数是全文件最关键的决策点。

它会把一串 `tool_use` 分成多个 batch，每个 batch 只有两种可能：

1. **单个非并发安全工具**
2. **多个连续的并发安全工具**

源码注释已经写得很清楚：

```text
1. A single non-read-only tool, or
2. Multiple consecutive read-only tools
```

虽然注释里写的是 read-only，但真正代码判断依赖的是：

```ts
tool?.isConcurrencySafe(parsedInput.data)
```

也就是说，真实规则是：

```text
是否并发，不看“工具名字”，而看工具在当前 input 下声明自己是否 concurrency-safe
```

这点非常重要。

因为它说明 Claude Code 的并发策略不是一个全局硬编码表，而是：

- 先找到 tool
- 解析 input
- 调 `isConcurrencySafe(input)`
- 再决定如何分批

所以并发策略是**工具元数据驱动**的。

---

#### 2. `runToolsSerially(...)`
位置：`src/services/tools/toolOrchestration.ts:118`

这个函数负责串行执行 batch。

关键行为：

- 每次只执行一个 `toolUse`
- 执行前把 toolUse ID 加入 in-progress 集合
- `yield* runToolUse(...)`
- 如果有 `contextModifier`，立即更新 `currentContext`
- 执行完后把该 toolUse ID 从 in-progress 集合里删掉

这说明串行路径的核心语义是：

```text
每个工具调用执行完后，它对上下文的影响会立刻生效，再影响后面的工具调用
```

这非常合理，因为非并发安全工具通常意味着：

- 会写文件
- 会改状态
- 会影响后续工具观察到的世界

所以它们必须按顺序推进。

---

#### 3. `runToolsConcurrently(...)`
位置：`src/services/tools/toolOrchestration.ts:152`

这个函数负责并发执行 batch。

它用的是：

```ts
yield* all(..., getMaxToolUseConcurrency())
```

也就是说：

- 并不是无限并发
- 有最大并发度限制
- 默认来自 `CLAUDE_CODE_MAX_TOOL_USE_CONCURRENCY`，否则默认 10

这里要注意两个点：

##### A. 并发是受限并发，不是全放开
位置：`src/services/tools/toolOrchestration.ts:8`

```ts
parseInt(process.env.CLAUDE_CODE_MAX_TOOL_USE_CONCURRENCY || '', 10) || 10
```

说明系统对并发是谨慎的，不会因为工具“可并发”就无限开。

##### B. 并发路径里的 context modification 不会立刻应用
这一点要结合 `runTools(...)` 主函数看。

并发执行时，`contextModifier` 先被收集到 `queuedContextModifiers`，等整批并发工具都跑完后，再按原 block 顺序统一应用。

这说明它的设计是：

```text
并发执行阶段允许消息先流出来，
但上下文修改要延迟到 batch 完成后再有序提交
```

这是非常关键的设计点。

---

### 为什么并发路径要“延迟提交 contextModifier”
关键位置：
- 收集 modifier：`src/services/tools/toolOrchestration.ts:31-48`
- 批末统一应用：`src/services/tools/toolOrchestration.ts:54-63`

这里体现了一个很漂亮的设计取舍：

#### 如果立即应用会有什么问题？
并发工具的完成顺序是不稳定的。

如果谁先完成谁就先改 context，那么：

- 上下文变化顺序不稳定
- 同一组 tool_use 每次运行可能得到不同上下文结果
- 后续行为难以复现

#### 现在的做法是什么？
- 执行阶段并发
- contextModifier 先缓存
- 最后按原始 tool_use block 顺序提交

所以设计目标是：

```text
执行可以并发，状态提交必须尽量确定性
```

这也是为什么这个文件虽然很短，但非常关键。

---

### `runToolUse(...)` 在这里扮演什么角色
位置：`src/services/tools/toolOrchestration.ts:6`

这个文件本身不做具体的工具执行细节，它只是调用：

```ts
runToolUse(...)
```

也就是说：

```text
toolOrchestration.ts 解决“怎么调度多个工具调用”
toolExecution.ts 解决“单个工具调用怎么真正执行”
```

这两个层次一定要分开。

你现在已经能看到 tool call loop 被拆成了至少两层：

1. **orchestration 层**：批次、并发、串行、上下文传播
2. **execution 层**：单工具查找、校验、权限、调用、消息生成

这正是 Claude Code 工具内环的分层点。

---

### 这个文件体现出的 5 个架构原则

#### 原则 1：并发能力由工具自己声明
不是全局写死，而是由 `tool.isConcurrencySafe(input)` 决定。

#### 原则 2：并发安全是 input-sensitive 的
因为 `isConcurrencySafe` 吃的是解析后的具体 input，不同输入可以有不同判断。

#### 原则 3：消息流和上下文流是分开处理的
消息可以边执行边 yield；上下文修改在并发路径下延后提交。

#### 原则 4：上下文传播必须有序
并发工具的 `contextModifier` 最后按原 tool_use 顺序提交，保持确定性。

#### 原则 5：工具调度与工具执行分层
`runTools(...)` 不实现单工具执行细节，而是委托给 `runToolUse(...)`。

---

### `markToolUseAsComplete(...)` 为什么值得注意
位置：`src/services/tools/toolOrchestration.ts:179`

这只是个小函数，但很说明问题。

它负责把 tool use ID 从 in-progress 集合里移除。

这说明工具调度层除了“跑工具”，还承担一部分运行时可视状态维护：

- 哪些工具正在执行
- 哪些已经完成

这会直接影响 UI、progress、interruptibility 等行为。

所以可以说：

```text
toolOrchestration.ts 不只是调度器，也是 in-progress tool state 的维护点之一
```

---

### 这一站最该记住的 6 句话

1. `toolOrchestration.ts` 负责的是多个 tool_use 的批次调度，不是单工具执行细节。
2. 它先用 `partitionToolCalls(...)` 按 concurrency-safe 属性分批。
3. 并发安全的工具可以批量并发执行，非并发安全工具必须串行执行。
4. 并发路径里消息可以先流出，但 contextModifier 会延迟到 batch 结束后再按原顺序提交。
5. 最大并发度是受限的，默认不是无限并发。
6. 它和 `toolExecution.ts` 是两层：前者管调度，后者管单工具执行。

---

### 现在把前七站串起来

到这里，tool loop 主链路可以进一步具体化：

```text
query.ts
  -> 识别 tool_use blocks
  -> runTools(...)
    -> partitionToolCalls(...)
    -> 并发 batch / 串行 batch
    -> runToolUse(...)
    -> 产出 message updates
    -> 提交 context changes
  -> 把 tool_result 写回消息流
```

这时你已经看清了 Claude Code tool call loop 里的一个关键边界：

```text
query.ts 决定“进入工具阶段”
toolOrchestration.ts 决定“多个工具怎么调度”
toolExecution.ts 决定“单个工具怎么执行”
```

这个分层非常重要。

---

### 第一轮阅读时，先不要深挖的部分

先别陷进去看：

- `all(...)` 这个并发生成器工具的实现
- `runToolUse(...)` 里面的权限/钩子/结果映射细节
- 不同工具具体如何实现 `contextModifier`
- UI 如何消费 in-progress tool IDs

当前阶段只要先抓住：

```text
toolOrchestration.ts = 多个 tool_use 的批次调度器
```

就够了。

---

### 下一站建议

下一站最顺的是：

```text
src/services/tools/toolExecution.ts
```

因为现在你已经知道：

- 工具是什么（`Tool.ts`）
- 工具池怎么组装（`tools.ts`）
- 多个工具调用怎么分批调度（`toolOrchestration.ts`）

下一步就该看：


---

---

### 元

问题：**这一站真正想解决的架构问题是什么？**

回答：这一站关注的是多个 `tool_use` 已经出现之后，该怎么分批、并发和串行执行。核心在 `partitionToolCalls(...)`，它按 concurrency-safe 规则把工具分成可并发批次与必须串行的单项。

### 反

问题：**如果把这一站的设计反过来，会发生什么？**

回答：如果拿到多个工具调用就一股脑并发，读写冲突和上下文污染会非常快出现。反过来全部串行，又会白白丢掉只读工具本可并行的效率。

### 空

问题：**跳出当前文件名，这一站背后更大的问题是什么？**

回答：更大的问题是，agent 工具执行不只是“会不会调用”，还包括“如何编排调用”。这一层本质上是在做安全性与吞吐量之间的调度平衡。

## 第 8 站：`src/services/tools/toolExecution.ts`

### 这是什么文件

`src/services/tools/toolExecution.ts` 是 Claude Code 的**单工具执行内核**。

如果说：

- `query.ts` 决定“什么时候进入工具阶段”
- `toolOrchestration.ts` 决定“多个工具调用怎么分批与并发”

那么这一站解决的是：

```text
单个 tool_use block 从“模型给出调用”到“返回 tool_result 消息”到底怎么走完
```

所以它不是工具池装配层，也不是批次调度层，而是：

```text
toolExecution.ts = 单个 tool_use 的校验、授权、执行、结果回写内核
```

---

### 先看两个最重要的入口

#### 1. `runToolUse(...)`
位置：`src/services/tools/toolExecution.ts:337`

这是单工具执行的总入口。

它做的事可以概括为：

1. 先按 `toolUse.name` 找到 tool
2. 处理 alias fallback
3. 处理未知工具 / 已取消状态
4. 进入 `streamedCheckPermissionsAndCallTool(...)`
5. 捕获最终异常并包装成 `tool_result error`

所以你可以先把它理解成：

```text
runToolUse() = 单个 tool_use 的总包装入口
```

真正的核心逻辑则在后面的 `checkPermissionsAndCallTool(...)`。

---

#### 2. `checkPermissionsAndCallTool(...)`
位置：`src/services/tools/toolExecution.ts:599`

这是整个文件的核心函数。

它把一个 tool_use 的执行流程拆成非常清晰的几步：

1. schema 校验
2. tool-specific input validation
3. pre-tool hooks
4. permission resolution
5. 真正调用 `tool.call(...)`
6. post-tool hooks
7. 结果映射成 `tool_result`
8. 处理异常路径与 failure hooks

所以这一步要记住：

```text
toolExecution.ts 的核心不是“直接调 tool.call”，
而是围绕 tool.call 前后包了一整圈验证、权限、hooks、日志、遥测、错误恢复逻辑。
```

---

### `streamedCheckPermissionsAndCallTool(...)` 为什么存在
位置：`src/services/tools/toolExecution.ts:492`

这个函数非常值得注意。

它的作用不是新增业务逻辑，而是把：

- progress events
- final result messages

统一包装成一个 async iterable。

源码注释也直接承认这是个小 hack：

```text
This is a bit of a hack to get progress events and final results into a single async iterable.
```

这说明：

```text
单工具执行在接口层上被设计成“流式消息源”，
而不只是一个最终 Promise<ToolResult>
```

因此上层 orchestration/query loop 才能一边看到 progress，一边等最终结果。

---

### 单个 tool_use 的完整执行链路

把 `checkPermissionsAndCallTool(...)` 摊开后，可以先记这条主线：

```text
tool_use arrives
  -> zod schema safeParse
  -> validateInput?
  -> speculative checks（bash 特例）
  -> pre-tool hooks
  -> resolve permission decision
  -> if denied: 生成 error tool_result / permission hooks
  -> if allowed: tool.call(...)
  -> map result to ToolResultBlockParam
  -> post-tool hooks / failure hooks
  -> 生成最终 user message(tool_result)
```

这是整个 tool call loop 内环真正最重要的链路之一。

---

### 这个文件最关键的 8 条主线

#### 主线 1：先找工具，再允许 alias fallback
关键位置：`src/services/tools/toolExecution.ts:343-356`

它先在 `toolUseContext.options.tools` 里找工具；
如果找不到，再去 `getAllBaseTools()` 里找 alias fallback。

但 fallback 只在“通过 alias 命中旧名字”时才成立，不是任意名字都能绕过当前工具集。

这说明：

```text
工具查找优先遵循“当前可见工具集”，
alias fallback 只是为旧 transcript / 兼容性兜底。
```

比如注释里提到的旧名：`KillShell` -> `TaskStop`。

---

#### 主线 2：工具输入有两层校验
关键位置：
- schema 校验：`src/services/tools/toolExecution.ts:614`
- tool-specific validation：`src/services/tools/toolExecution.ts:682`

这里非常关键。

第一层：

```ts
tool.inputSchema.safeParse(input)
```

这是结构级校验，保证类型对。

第二层：

```ts
await tool.validateInput?.(...)
```

这是语义级校验，保证值也合理。

所以 Claude Code 的工具输入不是“parse 一次就完”，而是：

```text
schema 负责结构正确
validateInput 负责业务正确
```

这两层设计得很清楚。

---

#### 主线 3：deferred tool 还有“schema 未发送”专门提示
关键位置：`src/services/tools/toolExecution.ts:578`

`buildSchemaNotSentHint(...)` 很值得记住。

当 deferred tool 没有真的进到 discovered-tool set 时，模型可能会把数组/数字/布尔都当字符串生成，导致 Zod 校验失败。

这里不会只报一个冷冰冰的类型错误，而是补一句很重要的引导：

```text
先调 ToolSearch 加载该工具 schema，再重试
```

这说明 Claude Code 已经显式意识到：

```text
deferred tools 带来的失败，不只是“参数错了”，
还可能是“模型根本没看到 schema”。
```

这类失败被单独建模了。

---

#### 主线 4：pre-tool hooks 在权限前就能介入
关键位置：`src/services/tools/toolExecution.ts:800`

`runPreToolUseHooks(...)` 是这条链里的关键切入点。

它可能产生很多种结果：

- progress message
- 普通附加消息
- `hookPermissionResult`
- `hookUpdatedInput`
- `preventContinuation`
- `stopReason`
- `stop`

这说明 pre-tool hook 不只是“观察一下”，而是可能：

```text
修改输入
影响权限决策
提前阻止执行
追加上下文
要求后续停止 continuation
```

也就是说，hook 在这里已经是执行链的一级控制器了，不是简单旁路日志器。

---

#### 主线 5：权限决策是 hook + canUseTool + tool-specific logic 的合流
关键位置：`src/services/tools/toolExecution.ts:921`

这里真正调用的是：

```ts
resolveHookPermissionDecision(...)
```

它会把：

- hook 返回的 permission result
- 工具本身的 `checkPermissions(...)`
- 通用 `canUseTool(...)`

整合成一个最终 `permissionDecision`。

所以更准确地说：

```text
权限不是单一函数拍板，而是多来源决策合流后得到的统一结果
```

而且这个结果还带：

- `behavior`
- `updatedInput`
- `decisionReason`
- `acceptFeedback`
- `contentBlocks`
- `userModified`

这说明“允许/拒绝”不仅是布尔值，而是一整个决策对象。

---

#### 主线 6：被拒绝时也不是简单报错，而是完整生成用户消息
关键位置：`src/services/tools/toolExecution.ts:995-1103`

当权限不允许时，这里会做很多事：

- 生成 `tool_result` 错误块
- 可能附带图片等 content blocks
- 可能附带 hook permission decision attachment
- auto-mode classifier 拒绝时还会跑 `PermissionDenied hooks`
- hook 若表示已批准，可再补一条 meta message 告诉模型“可以重试”

所以拒绝路径不是：

```text
return false
```

而是：

```text
构造一条对模型和 UI 都有意义的拒绝结果消息
```

这点很关键，因为 Claude Code 要让模型知道为什么失败、是否可重试、是否有新上下文。

---

#### 主线 7：真正执行前，还会 carefully 处理 input 形态
关键位置：`src/services/tools/toolExecution.ts:756-793` 与 `1180-1205`

这里有几件很细但很重要的事：

##### A. 去掉 Bash 的内部字段 `_simulatedSedEdit`
防止模型伪造只允许权限系统注入的内部字段。

##### B. `backfillObservableInput(...)`
给 hooks / transcript / SDK stream 看的是 backfilled clone，避免破坏原始 API-bound 输入和 prompt cache。

##### C. 对 file_path 做回写修正
如果 hook/permission 返回的新对象只是沿用了 backfill 后的 expanded path，就恢复成模型原始 path，避免 transcript / VCR hash 漂移。

这一段体现的设计原则很明显：

```text
工具输入同时服务于执行、观察、缓存、转录四个面，
所以必须小心区分“给谁看”的 input 版本。
```

这是高质量 runtime 代码的典型特征。

---

#### 主线 8：成功路径和失败路径都深度接入 telemetry / tracing / hooks
通篇可见：

- `logEvent(...)`
- `logOTelEvent(...)`
- `startToolSpan(...)`
- `startToolExecutionSpan(...)`
- `endToolExecutionSpan(...)`
- `runPostToolUseHooks(...)`
- `runPostToolUseFailureHooks(...)`

这说明 `toolExecution.ts` 不只是“把工具跑通”，还承担：

- 运行可观测性
- tracing span 维护
- tool_decision / tool_result 事件上报
- success/failure hook 生命周期衔接

也就是说：

```text
toolExecution.ts 是单工具执行的生命周期管理器，不只是函数调用器。
```

---

### 成功路径里最该记住的几个点

#### 1. 真正执行是 `tool.call(...)`
位置：`src/services/tools/toolExecution.ts:1207`

但它收到的 context 已经被补上：

- `toolUseId`
- `userModified`

所以工具实现本身也能知道：

- 当前是哪次 tool use
- 权限阶段是否改过输入

---

#### 2. tool result 先映射成标准 block
位置：`src/services/tools/toolExecution.ts:1292`

```ts
const mappedToolResultBlock = tool.mapToolResultToToolResultBlockParam(...)
```

这说明工具执行结果不会直接原样回写，而是先转成统一 API/tool_result 表示。

这一步是 Tool 协议真正落地的地方。

---

#### 3. PostToolUse hooks 可能继续修改输出
位置：`src/services/tools/toolExecution.ts:1483`

特别是 MCP tool，`updatedMCPToolOutput` 会改变最终回写内容。

所以 tool result 并不是 `tool.call()` 返回后就冻结，后置 hook 仍能参与调整。

---

#### 4. `ToolResult` 的副作用会被真正兑现
位置：
- `contextModifier`：`src/services/tools/toolExecution.ts:1467`
- `newMessages`：`src/services/tools/toolExecution.ts:1565`
- `mcpMeta`：`src/services/tools/toolExecution.ts:1464`

这一步把你在 `Tool.ts` 里看到的抽象真正落地了。

也就是说：

```text
ToolResult 里的 data / contextModifier / newMessages / mcpMeta
在这里被正式转换成消息流与上下文更新
```

---

### 失败路径里最该记住的几个点

#### 1. MCP auth error 会回写 AppState
位置：`src/services/tools/toolExecution.ts:1599`

这不是简单报错，而是把对应 MCP client 状态改成 `needs-auth`。

说明失败路径会反向影响系统状态，不只是回一条 error message。

#### 2. 失败后会跑 `PostToolUseFailure hooks`
位置：`src/services/tools/toolExecution.ts:1696`

说明 hook 生命周期区分：

- 成功后 PostToolUse
- 失败后 PostToolUseFailure

#### 3. 最后仍然会生成标准 `tool_result is_error` 消息
位置：`src/services/tools/toolExecution.ts:1715`

所以无论成功还是失败，模型看到的仍是统一消息协议，而不是异常直接冒泡到 query loop。

---

### 这一站最该记住的 8 句话

1. `toolExecution.ts` 是单个 tool_use 的执行内核。
2. 它先做 schema 校验，再做 tool-specific validation，再进入权限与 hook 链。
3. pre-tool hooks 可以改输入、影响权限、提前停止执行。
4. 权限决策是 hook、tool-specific permissions、通用 canUseTool 的合流结果。
5. 真正执行前，代码会 carefully 区分 callInput、observable input、backfilled input。
6. 成功后结果不会直接回写，而要经过 result mapping、post hooks、tool_result 构造。
7. 失败后也会走 failure hooks，并生成统一的 error tool_result。
8. 它本质上是“单工具生命周期管理器”，不只是 `tool.call()` 包装器。

---

### 现在把 tool call loop 串完整一点

到这里，你已经可以把 tool call loop 画成这样：

```text
query.ts
  -> 收到 assistant tool_use blocks
  -> toolOrchestration.runTools(...)
    -> partitionToolCalls(...)
    -> 对每个 tool_use 调 runToolUse(...)
      -> 找 tool
      -> 校验 input
      -> pre-tool hooks
      -> resolve permission
      -> tool.call(...)
      -> post-tool hooks / failure hooks
      -> 生成 tool_result user message
    -> 汇总 message/context updates
  -> 把结果写回下一轮消息流
```

这时 Claude Code 的 tool loop 主体已经非常清楚了。

---

### 第一轮阅读时，先不要深挖的部分

先别陷进去看：

- telemetry / OTel 每个字段的完整含义
- classifier / permission logging 的全部旁支逻辑
- MCP analytics metadata 的所有细节
- `toolHooks.ts` 每个 hook 结果类型的完整实现
- `toolResultStorage.ts` 的完整持久化策略

当前阶段重点只要抓住：

```text
toolExecution.ts = 单个 tool_use 的执行内核
```

就够了。

---

### 下一站建议

下一站最顺的是：

```text
src/utils/permissions/permissions.ts
```

因为现在你已经看到了：

- tool loop 什么时候触发
- 多个 tool_use 怎么调度
- 单个 tool_use 怎么执行

下一步自然就该看：

**通用权限系统本身到底怎样表达 allow / deny / ask / bypass，以及如何参与工具执行决策。**

---

---

### 元

问题：**这一站真正想解决的架构问题是什么？**

回答：这一站是单工具执行内核，把 schema 校验、tool-specific validation、hooks、权限判定、真实调用和结果回写串成完整流水线。它说明 `tool.call(...)` 前后其实包着很厚的一层运行时壳。

### 反

问题：**如果把这一站的设计反过来，会发生什么？**

回答：如果直接从模型输出跳到 `tool.call(...)`，出错、越权和可观测性都会失控。很多前后置逻辑会被迫复制到每个工具实现里。

### 空

问题：**跳出当前文件名，这一站背后更大的问题是什么？**

回答：更大的问题是，单次工具调用如何从“一个函数调用”提升为“可治理的执行事务”。这一站就是那层事务化外壳。

## 第 9 站：`src/utils/permissions/permissions.ts`

### 这是什么文件

`src/utils/permissions/permissions.ts` 是 Claude Code 的**通用权限判定引擎**。

如果说：

- `src/tools.ts` 负责“有哪些工具”
- `src/services/tools/toolOrchestration.ts` 负责“多个 tool_use 怎么排班”
- `src/services/tools/toolExecution.ts` 负责“单个 tool_use 怎么完整执行”

那么这个文件负责回答的是：

```text
某个工具调用，在当前上下文里，到底应该 allow、deny、ask，还是走 auto / headless / hook 特殊路径
```

它不是 UI 层，也不是某个具体工具自己的权限逻辑；它是**被 tool execution 复用的通用决策层**。

所以最准确的定位是：

```text
permissions.ts = 工具权限系统的共享裁决器
```

---

### 先看最重要的入口

#### 1. `hasPermissionsToUseTool(...)`
位置：`src/utils/permissions/permissions.ts:473`

这是对外主入口。

它先调用内部的权限判定逻辑，然后再做两层“全局收口”：

- allow 时重置 auto mode 的 denial tracking
- ask 时根据 mode 再决定是否转成 deny / classifier / headless fallback

也就是说，这个函数不是简单返回“能不能用”，而是：

```text
先拿到基础判定
再根据 mode、classifier、headless 约束做最终收口
```

所以它更像“权限决策总出口”。

#### 2. `hasPermissionsToUseToolInner(...)`
位置：下半文件内部主逻辑

虽然外部入口是 `hasPermissionsToUseTool(...)`，但真正的基础规则判定发生在内部逻辑里。

这一层更接近：

- 当前 tool 是否命中 allow / deny / ask 规则
- 当前 permission mode 应该怎样解释这些规则
- 是否需要构造 permission request
- 是否要生成 suggestions / updates

你可以把两层关系记成：

```text
hasPermissionsToUseToolInner = 基础规则裁决
hasPermissionsToUseTool = 最终模式收口与特殊路径处理
```

---

### 这个文件先做了什么基础工作

#### 1. 把权限规则按 source 汇总
位置：`src/utils/permissions/permissions.ts:109-130`

这里先定义 `PERMISSION_RULE_SOURCES`，然后用：

- `getAllowRules(...)` `src/utils/permissions/permissions.ts:122`
- `getDenyRules(...)` `src/utils/permissions/permissions.ts:213`
- `getAskRules(...)` `src/utils/permissions/permissions.ts:223`

把 `ToolPermissionContext` 里按来源存放的规则，统一转成结构化 `PermissionRule[]`。

关键点是：规则不是单一来源。

它会合并：

- settings sources
- `cliArg`
- `command`
- `session`

这说明权限系统不是“只看一个配置文件”，而是：

```text
多个来源共同叠加出当前会话的权限视图
```

这也是为什么 `ToolPermissionContext` 很重要——它承载的是**当前生效的权限快照**。

---

#### 2. 统一把“为什么要审批”转换成可展示消息
位置：`src/utils/permissions/permissions.ts:137`

`createPermissionRequestMessage(...)` 很值得看。

它并不只是吐一句固定文案，而是会根据 `decisionReason.type` 生成不同提示，例如：

- classifier 触发审批
- hook 阻止或要求审批
- 某条具体 permission rule 触发审批
- Bash 子命令里某些片段需要审批
- 当前 mode 要求审批
- sandbox override / workingDir / safetyCheck 等原因

这说明权限系统不仅负责“判”，还负责：

```text
把判定原因翻译成用户看得懂的审批说明
```

也就是说，权限系统内置了**解释层**，不是纯布尔判断。

---

### 规则匹配是怎么做的

#### 1. `toolMatchesRule(...)`
位置：`src/utils/permissions/permissions.ts:238`

这是规则匹配的基础函数。

这里要特别注意两个点：

##### A. “整工具匹配”与“带内容匹配”是分开的
如果规则带 `ruleContent`，这个函数直接返回 `false`。

也就是说这里判断的是：

- `Bash`
- `Write`
- `mcp__server`

这种**整工具级别**规则，
而不是：

- `Bash(prefix:*)`
- `Agent(Explore)`

这种带内容的细粒度规则。

##### B. MCP 工具按权限检查名匹配，不按展示名硬匹配
这里通过 `getToolNameForPermissionCheck(tool)` 和 `mcpInfoFromString(...)` 做匹配。

这很关键，因为 MCP 工具可能存在：

- `mcp__server__tool`
- server 级 wildcard
- skip-prefix 模式下展示名与 builtin 冲突

所以这里不是简单拿 `tool.name === rule.toolName`，而是专门处理了 **MCP 命名空间语义**。

第一结论：

```text
权限系统不是只会匹配 builtin tool 名，它把 MCP server / tool 级别规则也纳入了统一语义
```

---

#### 2. 整工具规则查询函数
位置：
- `toolAlwaysAllowedRule(...)` `src/utils/permissions/permissions.ts:275`
- `getDenyRuleForTool(...)` `src/utils/permissions/permissions.ts:287`
- `getAskRuleForTool(...)` `src/utils/permissions/permissions.ts:297`

这三组函数很直白，但非常关键。

它们把“allow / deny / ask 三种行为”统一成同一种查询模型：

```text
给一个 tool，看它是否命中了对应行为的整工具规则
```

所以后面更高层的判定逻辑就可以复用这三个入口，而不需要重新展开所有 rule parsing。

---

#### 3. Agent 的 deny 是单独处理的
位置：
- `getDenyRuleForAgent(...)` `src/utils/permissions/permissions.ts:308`
- `filterDeniedAgents(...)` `src/utils/permissions/permissions.ts:325`

这里说明了一个很重要的设计：

`Agent(...)` 这种权限不是普通 tool-name 规则，而是**Agent tool 的内容级规则**。

例如：

```text
Agent(Explore)
```

表示的是：

- 不是禁用整个 `Agent` 工具
- 而是禁用 `Agent` 工具里的某个 agent type

`filterDeniedAgents(...)` 还做了一层优化：

- 先把 deny rules 扫一遍
- 提取出所有被拒绝的 `agentType`
- 用 `Set` 过滤 agent 列表

所以这块回答了一个问题：

```text
权限系统不仅能裁决“能不能用 Agent 工具”，还能裁决“Agent 工具里哪些子 agent 能被看到或调用”
```

---

#### 4. 细粒度规则内容映射
位置：`src/utils/permissions/permissions.ts:349`

`getRuleByContentsForTool(...)` / `getRuleByContentsForToolName(...)` 会把某个工具的内容级规则整理成 `Map<string, PermissionRule>`。

例如 Bash 里的：

```text
Bash(prefix:*)
```

这里真正关心的是 `prefix:*` 这种内容片段。

这说明权限系统不是只有“整工具 allow/deny/ask”，还支持：

```text
同一个工具内部的子语义规则
```

这也是为什么 Bash、Agent 这类工具能做更细权限控制，而不只是全开或全关。

---

### headless / async agent 场景怎么处理

#### `runPermissionRequestHooksForHeadlessAgent(...)`
位置：`src/utils/permissions/permissions.ts:400`

这段非常关键，因为它回答了一个现实问题：

```text
如果当前上下文没法弹权限提示框，那 ask 怎么办？
```

这里的策略不是直接死拒，而是：

1. 先跑 `PermissionRequest` hooks
2. 如果 hook 给出 allow/deny，就采用 hook 决策
3. hook 还可以：
   - 修改 input
   - 持久化 permission updates
   - 更新 appState 里的 `toolPermissionContext`
   - 触发 interrupt 中断
4. 只有 hook 没给结论时，才落回 auto-deny

这说明 Claude Code 的权限系统并不是“没 UI 就没办法”，而是：

```text
headless 场景先交给自动化 hooks 介入，再决定是否拒绝
```

这正好和前面 hooks 体系、async subagent 场景连起来了。

---

### auto / dontAsk / headless 三条特殊路径

#### 1. `dontAsk` 会把 ask 强制收口成 deny
位置：`src/utils/permissions/permissions.ts:503`

这是一个很关键的收口规则：

```text
dontAsk 模式下，不允许留下 ask 这种中间态
```

所以只要基础结果是 `ask`，到了这里就会被转换为：

- `behavior: 'deny'`
- `decisionReason.type: 'mode'`
- 配套拒绝消息 `DONT_ASK_REJECT_MESSAGE(...)`

也就是说：

```text
dontAsk 不是“自动允许”，而是“禁止审批，凡需审批者一律拒绝”
```

---

#### 2. auto mode 不是直接放行，而是走 classifier
位置：`src/utils/permissions/permissions.ts:521-955`

这是整文件最值得精读的一段。

这里说明：

```text
auto mode != bypass permissions
auto mode = 用 classifier 代替人工审批
```

它的大致流程是：

1. 仅在 `ask` 结果上进入 auto mode 流程
2. 非 classifier-approvable 的 safetyCheck 不允许被 auto 绕过
3. 需要明确用户交互的工具直接保留 ask
4. PowerShell 在某些 flag 关闭时不允许自动审批
5. 先尝试 `acceptEdits` fast-path
6. 再尝试 safe-tool allowlist fast-path
7. 最后才调用 `classifyYoloAction(...)`
8. 根据 classifier 结果决定 allow / deny / fallback

所以 auto mode 的核心不是“更宽松”，而是：

```text
把 ask 交给另一套风险评估系统去决定
```

---

#### 3. `acceptEdits` fast-path
位置：`src/utils/permissions/permissions.ts:593`

这一段很妙。

系统会在 auto classifier 前，临时把 `toolPermissionContext.mode` 伪装成 `acceptEdits`，重新调用一次 `tool.checkPermissions(...)`。

如果在 `acceptEdits` 模式下该操作本来就能通过，那就：

- 直接 allow
- 跳过 classifier API 开销
- 记录 telemetry
- 重置 denial tracking

这反映的设计思想是：

```text
先用便宜、确定的本地规则判定；只有本地规则不够时，才调用昂贵的 classifier
```

这是一种很典型的“多层判定、先快后慢”的架构。

同时它还特地排除了：

- `Agent`
- `REPL`

因为这两类工具即使表面安全，也可能在内部拼出更复杂的执行链，不能轻易跳过 classifier。

---

#### 4. safe-tool allowlist fast-path
位置：`src/utils/permissions/permissions.ts:658`

这一步更直接：如果工具名在 allowlist 上，就直接允许，不走 classifier。

这再次说明 auto mode 的整体结构是：

```text
本地明确安全 -> 直接放
不明确 -> 再交给 classifier
```

也就是把 classifier 留给真正有歧义的动作。

---

#### 5. classifier 不是只有 allow / deny，还带 fallback 语义
位置：`src/utils/permissions/permissions.ts:688` 之后

这里不是简单调个模型然后返回布尔值。

它还处理了很多运行时边界：

- classifier transcript 过长
- classifier 服务不可用
- fail closed / fail open feature gate
- denial tracking 与超限回退
- headless 模式下 denial 过多直接 abort
- telemetry / token / cost / latency 统计

这说明 classifier 已经不是“实验性补丁”，而是权限系统里的一个正式子系统。

它不仅做判断，还和这些能力耦合：

- 会话统计
- debug logging
- analytics
- abort / fallback 控制

所以这里你要建立一个更准确的理解：

```text
auto mode classifier = 权限系统里的二级裁决器，而不是简单的辅助函数
```

---

### denial tracking 是干什么的

#### `persistDenialState(...)`
位置：`src/utils/permissions/permissions.ts:963`

#### `handleDenialLimitExceeded(...)`
位置：`src/utils/permissions/permissions.ts:984`

这两段说明 auto mode 不只是“每次单独判”。

系统还会跨轮记录：

- `totalDenials`
- `consecutiveDenials`

然后在超限时做收口：

- CLI 场景：从 classifier deny 回退到人工 prompting
- headless 场景：直接 `AbortError`

这很重要，因为它说明 Claude Code 避免陷入这种坏循环：

```text
模型不断提出高风险动作
-> classifier 不断拒绝
-> 又继续提
-> 无限消耗 token
```

所以 denial tracking 本质上是：

```text
auto 权限流的熔断器
```

---

### 元

问题：**这一站真正想解决的架构问题是什么？**

回答：这一站是通用权限裁决器，回答某次工具调用在当前上下文中到底该 allow、deny 还是 ask。它把规则来源、mode 解释、tool 自带权限检查与最终收口都接在了一起。

### 反

问题：**如果把这一站的设计反过来，会发生什么？**

回答：如果权限逻辑散落在各工具或各 UI 分支里，同一条规则在不同路径下可能给出不同答案。权限系统会看似灵活，实际却不可预测。

### 空

问题：**跳出当前文件名，这一站背后更大的问题是什么？**

回答：更大的问题是，权限到底是“单点规则匹配”还是“多层裁决流水线”。这一站清楚地说明，真实权限系统必须允许全局规则与工具局部语义同时存在。


### 读完这个文件后，你应该抓住的 6 个事实

#### 事实 1：权限系统不是单层 if/else，而是分层裁决
先有基础规则裁决，再有 mode 收口，再有 classifier / headless / hook 特殊路径。

#### 事实 2：规则来源是叠加的
settings、cliArg、command、session 共同构成当前有效权限。

#### 事实 3：整工具规则和内容级规则是两层语义
`Bash` 和 `Bash(prefix:*)` 不是一回事；`Agent` 和 `Agent(Explore)` 也不是一回事。

#### 事实 4：headless 场景不是简单 auto-deny
会先给 `PermissionRequest` hooks 介入机会。

#### 事实 5：auto mode 不是 bypass
它是一条 classifier 驱动的替代审批链路，并带有 fast-path、fallback、熔断机制。

#### 事实 6：权限系统已经深度接入运行时
它不只影响 prompt 展示，还影响：
- tool 执行是否继续
- input 是否被 hook 改写
- appState 是否更新
- classifier telemetry 是否上报
- agent 是否被 abort

---

### 现在把前面几站串起来

到这里可以把 tool 执行内环串成：

```text
query.ts
  -> 识别 tool_use
  -> toolOrchestration.ts 做批次调度
  -> toolExecution.ts 执行单个 tool_use
    -> permissions.ts 做共享权限裁决
      -> allow / deny / ask / auto / headless / hook 收口
    -> 真正调用 tool.call(...)
```

所以这一站其实补上的是：

```text
为什么 toolExecution.ts 能在各种 mode 下做出一致权限决策
```

答案就是：因为底下有 `permissions.ts` 这个共享判定层。

---

### 第一轮阅读时，不要陷进去的点

先不要深挖这些细节：

- 每种 `PermissionDecisionReason` 的所有分支语义
- classifier prompt 的具体构造方式
- `PermissionUpdate` 如何落盘到 settings
- PowerShell 特判背后的全部安全背景
- 各种 feature flag 在不同 build 中的差异

这一轮先抓住：

```text
permissions.ts = 工具权限系统的共享裁决器
```

以及：

```text
allow/deny/ask 只是基础结果
最终行为还会被 mode、hooks、headless、classifier 再收口一次
```

就够了。

---

### 下一站建议

下一站最顺的是：

```text
src/hooks/useCanUseTool.ts
```

因为现在你已经看懂了共享权限引擎本身，下一步最自然的是看：

**这个权限引擎是怎样被 React / hooks 层包装，并接进实际 tool execution 与 UI 状态流里的。**

---

## 第 10 站：`src/hooks/useCanUseTool.tsx`

### 这是什么文件

`src/hooks/useCanUseTool.tsx` 是权限系统的**接线层 / 协调层**。

上一站 `src/utils/permissions/permissions.ts` 解决的是：

```text
权限规则本身怎样裁决 allow / deny / ask / auto / headless
```

而这一站解决的是：

```text
这个裁决结果怎样真正接到运行时、交互式审批队列、swarm worker、classifier UI 状态里
```

所以它不是底层规则引擎，也不是单纯 UI 组件；它位于两者之间，把“权限判定结果”转成“实际运行时动作”。

最准确的定位是：

```text
useCanUseTool.tsx = 权限系统的运行时适配器
```

---

### 先看最重要的类型和入口

#### 1. `CanUseToolFn`
位置：`src/hooks/useCanUseTool.tsx:27`

这里先定义了统一签名：

- 输入：`tool`、`input`、`toolUseContext`、`assistantMessage`、`toolUseID`
- 可选：`forceDecision`
- 输出：`Promise<PermissionDecision<Input>>`

这很关键，因为它说明这层对外暴露的不是 React state，而是一个**异步权限决策函数**。

也就是说，在 query / tool execution 眼里，它像一个能力接口：

```text
给我一个 tool_use，我返回最终权限决定
```

#### 2. `useCanUseTool(...)`
位置：`src/hooks/useCanUseTool.tsx:28`

这个 hook 返回的正是上面的 `CanUseToolFn`。

但它不是简单 `useCallback(hasPermissionsToUseTool)`，而是包了一层完整协调逻辑：

- 创建 permission context
- 检查 abort
- 调底层 `hasPermissionsToUseTool(...)`
- 对 allow / deny / ask 做分流
- 接 interactive dialog / swarm worker / coordinator / classifier 快速路径
- 清理 classifier checking 状态

所以可以把它理解成：

```text
把“纯权限判定”升级成“可在真实运行时落地的权限执行流程”
```

---

### 先抓整个主链路

把这个 hook 返回的函数展开，主链路大概是：

```text
收到 tool_use
  -> createPermissionContext(...)
  -> 如果已 abort，直接返回
  -> forceDecision ? 直接用 : 调 hasPermissionsToUseTool(...)
  -> allow ? 直接放行并记录状态
  -> deny ? 记录拒绝 / 通知 / 返回
  -> ask ? 再分给 coordinator / swarm / classifier / interactive dialog
  -> finally 清理 classifier checking 标记
```

这条链路说明一件事：

```text
permissions.ts 给出的是“应该如何处理”
useCanUseTool.tsx 决定的是“在当前运行时里具体怎么处理”
```

---

### 这个文件做的第一件事：创建权限上下文包装器

#### `createPermissionContext(...)`
位置：调用点 `src/hooks/useCanUseTool.tsx:33`

一进来就先创建 `ctx`，并同时注入：

- tool / input / assistantMessage / toolUseID
- `setToolPermissionContext`
- `createPermissionQueueOps(setToolUseConfirmQueue)`

这说明这个 hook 并不是裸函数，而是会把权限操作所需的“运行时副作用能力”打包进上下文里。

从后面的用法看，这个 `ctx` 至少承担几类职责：

- 判断请求是否已 abort：`ctx.resolveIfAborted(...)`
- 记录权限决定：`ctx.logDecision(...)`
- 构造 allow 结果：`ctx.buildAllow(...)`
- 取消并中止：`ctx.cancelAndAbort(...)`
- 写 permission queue / UI 状态

所以：

```text
PermissionContext 不是规则上下文，而是“权限执行事务上下文”
```

---

### allow 分支怎么处理

位置：`src/hooks/useCanUseTool.tsx:39`

如果底层结果已经是 `allow`，这里不会再弹 UI，而是直接做“运行时收尾”：

1. 再次检查 abort
2. 如果是 auto-mode classifier 给的 allow，调用 `setYoloClassifierApproval(...)`
3. `ctx.logDecision({ decision: 'accept', source: 'config' })`
4. `resolve(ctx.buildAllow(...))`

这里要特别注意第 2 点。

这说明 classifier 的 allow 不只是内部状态，还会同步到上层 UI / 状态系统里，让外面知道：

```text
这次放行不是普通规则 allow，而是 auto mode classifier 批准的
```

所以 allow 分支并不是“直接返回就完事”，而是会把批准来源写回展示层和日志层。

---

### deny 分支怎么处理

位置：`src/hooks/useCanUseTool.tsx:64`

deny 分支先算出 `description`，然后在 `case 'deny'` 里做这些事情：

- `logPermissionDecision(...)`
- 如果是 auto-mode classifier 拒绝：
  - `recordAutoModeDenial(...)`
  - `toolUseContext.addNotification(...)`
  - 通知里明确提示 `/permissions`
- 最后 `resolve(result)`

这里说明 deny 并不是静默失败，而是会把拒绝同步到多个层：

- 审计日志层
- auto mode 拒绝历史
- UI notification 层

也就是说：

```text
deny 结果在这一层被“产品化”了
```

不仅有内部 decision，还有用户可见反馈。

---

### 为什么 `ask` 分支最复杂

位置：`src/hooks/useCanUseTool.tsx:93`

因为 `ask` 才是真正需要把“权限判定”变成“审批流程”的地方。

这里的顺序非常值得记：

#### 1. coordinator worker 优先
如果 `awaitAutomatedChecksBeforeDialog` 为真，就先走：

- `handleCoordinatorPermission(...)`

位置：`src/hooks/useCanUseTool.tsx:95`

注释已经点得很明确：

```text
对 coordinator worker，先等自动化检查，再决定要不要真正打断用户
```

也就是说，这类场景更偏“后台协调者”，所以优先等自动流程，而不是立刻弹窗。

---

#### 2. swarm worker 再尝试 leader 转发路径
位置：`src/hooks/useCanUseTool.tsx:113`

接着会走：

- `handleSwarmWorkerPermission(...)`

这说明在 swarm / teammate 场景下，请求不一定直接由当前进程自己处理，而可能：

```text
转发给 leader / mailbox / 上层协调者
```

这就把权限系统和 agent team 体系接起来了。

换句话说，`ask` 在这里已经不只是“弹一个确认框”，而是：

```text
可能走团队协作审批链路
```

---

#### 3. Bash speculative classifier 的 2 秒 grace period
位置：`src/hooks/useCanUseTool.tsx:126`

这一段很有意思，也是这层 hook 独有的运行时优化。

当满足这些条件时：

- `BASH_CLASSIFIER` 开启
- 当前结果带 `pendingClassifierCheck`
- 当前工具是 `Bash`
- 当前不是 `awaitAutomatedChecksBeforeDialog`

它会：

1. `peekSpeculativeClassifierCheck(command)`
2. 最多 `Promise.race(..., 2秒 timeout)`
3. 如果在 grace period 内拿到高置信 classifier allow：
   - `consumeSpeculativeClassifierCheck(...)`
   - `setClassifierApproval(...)`
   - `ctx.logDecision(...)`
   - 直接 `resolve(ctx.buildAllow(...))`
4. 否则才继续进入 dialog

这说明它在产品体验上做了一个很细的优化：

```text
如果 classifier 很快就能明确放行 Bash，就尽量别把审批框弹给用户
```

所以这层 hook 的价值之一就是：

```text
把底层 classifier 能力转成更顺滑的交互体验
```

---

#### 4. 最后才落到 interactive permission
位置：`src/hooks/useCanUseTool.tsx:160`

前面自动化路径都没解决时，才调用：

- `handleInteractivePermission(...)`

并且会带上：

- `description`
- `result`
- `awaitAutomatedChecksBeforeDialog`
- bridge callbacks
- channel callbacks

这里说明 interactive dialog 并不是 ask 的唯一处理器，而是：

```text
所有自动化 / 协调 / worker 路径都没解决后的最后落点
```

这和上一站 `permissions.ts` 的结论正好一致：

- ask 是中间态
- 真正怎么处理 ask，要看运行时上下文

而 `useCanUseTool.tsx` 正是这个“运行时上下文分发器”。

---

### `forceDecision` 是干什么的

位置：`src/hooks/useCanUseTool.tsx:37`

这里支持：

```ts
forceDecision !== undefined
  ? Promise.resolve(forceDecision)
  : hasPermissionsToUseTool(...)
```

这说明这个 hook 不强制每次都重新跑底层权限引擎。

也就是说，上层在某些场景下可以直接注入一个现成决定，再复用整套后续流程。

这很重要，因为它让这层 hook 变成：

```text
既能做完整权限判定，也能做“已有判定结果的运行时落地”
```

这是一种典型的分层设计：

- 底层负责算结果
- 中层负责消费结果并推进副作用

---

### 错误与清理路径

位置：`src/hooks/useCanUseTool.tsx:171`

这里的错误处理也很典型：

- `AbortError` / `APIUserAbortError`
  - debug log
  - `ctx.logCancelled()`
  - `resolve(ctx.cancelAndAbort(...))`
- 其他错误
  - `logError(error)`
  - 同样 `cancelAndAbort`
- `finally`
  - `clearClassifierChecking(toolUseID)`

这说明权限流程在运行时上被当成一个**可取消事务**来处理。

即使异常了，也不是随便抛出去，而是尽量收口成统一的取消 / 中止结果。

同时 classifier checking 状态不依赖成功失败，都会在 finally 清理，避免 UI 状态挂脏。

---

### 元

问题：**这一站真正想解决的架构问题是什么？**

回答：这一站把底层权限判定变成真实运行时里的最后一道总闸门。它不仅拿到 allow/deny/ask 结果，还负责把 ask 分流到 coordinator、swarm worker、classifier 和 interactive dialog。

### 反

问题：**如果把这一站的设计反过来，会发生什么？**

回答：如果只有 `hasPermissionsToUseTool(...)` 而没有这层接线，权限系统只能停留在“算出一个结果”。真正的 UI 队列、用户反馈和 worker 转发都无法自然落地。

### 空

问题：**跳出当前文件名，这一站背后更大的问题是什么？**

回答：更大的问题是，规则裁决怎样转成可执行流程。`useCanUseTool.tsx` 说明，权限不是静态判断，而是要接入运行时动作链。


### 读完这个文件后，你应该抓住的 6 个事实

#### 事实 1：`useCanUseTool.tsx` 不是规则引擎，而是规则引擎的运行时适配层
它负责把 `PermissionDecision` 变成真正的副作用与交互流程。

#### 事实 2：allow / deny / ask 在这里都被“产品化”了
不仅返回结果，还会更新日志、通知、classifier approval、permission queue 等状态。

#### 事实 3：`ask` 不等于“立即弹窗”
它会先经过 coordinator、swarm worker、classifier grace period 等路径。

#### 事实 4：team / swarm 架构已经接进权限系统
权限请求不一定只发生在本地 UI，也可能通过 worker / leader 协作链路处理。

#### 事实 5：classifier 不只在底层判定层存在
在这一层还被用来优化交互体验，比如 Bash speculative approval 的 2 秒等待窗口。

#### 事实 6：权限流程被建模成可取消事务
abort、异常、classifier checking 清理都有统一收口。

---

### 现在把两站串起来

到这里可以把权限链路理解成：

```text
permissions.ts
  -> 产出基础 PermissionDecision
useCanUseTool.tsx
  -> 把这个 decision 接进真实运行时
  -> 决定是直接放行、记录拒绝、走 worker 协调，还是进入交互式审批
```

所以这两站的分工很清楚：

```text
permissions.ts = 判
useCanUseTool.tsx = 落地执行这个判定
```

---

### 第一轮阅读时，不要陷进去的点

先不要深挖这些细节：

- `PermissionContext` 内部每个 helper 的实现
- `handleCoordinatorPermission` / `handleSwarmWorkerPermission` 的全部分支
- speculative Bash classifier 的完整数据来源
- bridge / channel callbacks 的全部宿主差异

这一轮先抓住：

```text
useCanUseTool.tsx = 权限系统的运行时适配器
```

以及：

```text
ask 只是中间态，真正如何处理 ask，是由这一层根据当前运行时环境继续分发的
```

就够了。

---

### 下一站建议

下一站最顺的是：

```text
src/hooks/toolPermission/PermissionContext.ts
```

因为现在你已经看到了权限判定层和运行时适配层，下一步最自然就是拆开看：

**这个 `ctx` 到底封装了哪些权限事务能力，以及 permission queue / state 更新具体怎么被包装起来。**

---

## 第 11 站：`src/hooks/toolPermission/PermissionContext.ts`

### 这是什么文件

`src/hooks/toolPermission/PermissionContext.ts` 是权限系统里的**事务上下文封装层**。

上一站 `src/hooks/useCanUseTool.tsx` 已经看到，一次权限处理不会只是“算一个 allow / deny / ask”，还会涉及：

- 日志
- queue 操作
- 用户批准后的权限持久化
- hook / classifier 分支
- abort / cancel 收口

这个文件的职责就是把这些副作用能力打包进一个统一 `ctx`，让上层 handler 不用每次手写一遍。

所以最准确的定位是：

```text
PermissionContext.ts = 权限流程的事务工具箱
```

---

### 先看这个文件解决的核心问题

如果没有它，上层代码会变成这样：

- 自己处理 queue push/remove/update
- 自己处理 permission updates 持久化
- 自己处理 allow/deny 结果对象构造
- 自己处理 hook / classifier / abort 收口
- 自己处理日志

那 `useCanUseTool.tsx` 和各个 handler 会很快失控。

而现在它把这些统一成：

```text
createPermissionContext(...) -> 返回一组可复用的权限事务操作
```

也就是：

```text
把一次 permission flow 所需的副作用、结果构造、状态更新全部收口到一个上下文对象里
```

---

### 第一部分：`createResolveOnce(...)`

位置：`src/hooks/toolPermission/PermissionContext.ts:75`

这个小工具很关键，虽然不是文件名主角，但它体现了这里的设计重点：

```text
权限流程是异步竞争场景，必须防止重复 resolve
```

它提供三件事：

- `resolve(value)`
- `isResolved()`
- `claim()`

尤其 `claim()` 的注释很重要：

- 原子地检查并标记已解析
- 用来关闭 async callback 之间 “先检查、后 resolve” 的竞争窗口

这说明作者非常清楚权限流程里会出现这些竞态：

- hook 和 UI 都可能试图结束请求
- classifier 和 user action 可能抢先
- abort 和 normal resolve 可能并发发生

所以第一结论是：

```text
PermissionContext 体系从一开始就按“多异步来源竞争同一结果”来设计
```

---

### 第二部分：`createPermissionContext(...)`

位置：`src/hooks/toolPermission/PermissionContext.ts:96`

这是整个文件的核心工厂函数。

它接收：

- `tool`
- `input`
- `toolUseContext`
- `assistantMessage`
- `toolUseID`
- `setToolPermissionContext`
- `queueOps`

然后返回一个被冻结的 `ctx` 对象。

这里的意义不是“放几个 helper”，而是把**这一次 tool_use 权限事务的所有上下文固定下来**。

这样后面任何 handler 都不需要再单独传：

- 当前 tool 是谁
- 当前 messageId 是谁
- 当前怎么更新 queue
- 当前怎么写回 permission context

这就是典型的 transaction-context 思路。

---

### `ctx` 里面最重要的能力

#### 1. `logDecision(...)`
位置：`src/hooks/toolPermission/PermissionContext.ts:113`

这是统一权限日志入口。

它会把：

- `tool`
- `input`
- `toolUseContext`
- `messageId`
- `toolUseID`

统一补齐，再调 `logPermissionDecision(...)`。

所以这里做的事不是“记录日志”这么简单，而是：

```text
把权限日志需要的公共字段固定注入，避免各处分散拼装
```

---

#### 2. `logCancelled()`
位置：`src/hooks/toolPermission/PermissionContext.ts:132`

取消路径不是随便吞掉，而是专门打一个 analytics event：

- `tengu_tool_use_cancelled`

这说明权限流程中的“取消”被当成一个独立事件对待，而不是普通错误。

也就是说：

```text
取消是权限产品行为的一部分，不只是异常分支
```

---

#### 3. `persistPermissions(...)`
位置：`src/hooks/toolPermission/PermissionContext.ts:139`

这是最关键的 helper 之一。

它把“批准后更新规则”这件事统一处理掉：

1. `persistPermissionUpdates(updates)`
2. 读取当前 appState
3. `applyPermissionUpdates(...)`
4. `setToolPermissionContext(...)`
5. 返回这批更新里是否有可持久化目标

这说明批准一个权限请求，不只是返回 allow，还可能伴随：

```text
把新的 permission rule 落盘
并同步刷新当前运行时里的 ToolPermissionContext
```

这就是一类非常典型的“写配置 + 更新内存快照”的双写动作。

所以这里的核心价值是：

```text
把 permission update 的持久化与内存态刷新绑成一个原子语义步骤
```

---

#### 4. `resolveIfAborted(...)`
位置：`src/hooks/toolPermission/PermissionContext.ts:148`

这个 helper 很实用。

如果 `abortController.signal.aborted` 已经触发，就：

- `logCancelled()`
- `resolve(this.cancelAndAbort(...))`
- 返回 `true`

也就是说，它把“每个异步阶段都要重复写的 abort 检查”收成一个统一动作。

这和上一站看到的 `ctx.resolveIfAborted(resolve)` 完全对上了。

所以这一层实际上承担的是：

```text
权限事务里的中途熔断守卫
```

---

#### 5. `cancelAndAbort(...)`
位置：`src/hooks/toolPermission/PermissionContext.ts:154`

这段非常值得看，因为它定义了“拒绝/取消如何对外表达”。

它会根据场景拼不同消息：

- 普通主 agent
- subagent
- 是否带 feedback
- 是否要附带 memory correction hint

然后在需要时：

- `toolUseContext.abortController.abort()`

最后返回的不是 `deny`，而是：

```ts
{ behavior: 'ask', message, contentBlocks }
```

这个设计很关键。

说明在这里：

```text
“中止当前权限流”不一定等于底层 deny，它更像给上层一个带消息的 ask/reject 收口结果
```

尤其 `withMemoryCorrectionHint(...)` 这一层也说明：

- 这不仅是内部控制流
- 还兼顾给模型 / 用户的反馈语义

---

### classifier 和 hooks 也被封装进 ctx

#### 1. `tryClassifier(...)`
位置：`src/hooks/toolPermission/PermissionContext.ts:176`

这个方法只在 `BASH_CLASSIFIER` 打开时存在。

它做的事是：

- 只处理 Bash + `pendingClassifierCheck`
- `awaitClassifierAutoApproval(...)`
- 如果 classifier 给出批准：
  - 可能 `setClassifierApproval(...)`
  - `logPermissionDecision(... source: classifier)`
  - 返回 `allow`

这说明 classifier 不只是存在于 `permissions.ts` 或 `useCanUseTool.tsx`，它在这里被进一步封装成：

```text
ctx 级别的“自动批准尝试器”
```

也就是说，handler 可以直接调用它，而不需要知道 classifier 的细枝末节。

---

#### 2. `runHooks(...)`
位置：`src/hooks/toolPermission/PermissionContext.ts:216`

这个方法把 `PermissionRequest` hooks 的执行也封进来了。

它会：

1. `executePermissionRequestHooks(...)`
2. 如果 hook 返回 allow：
   - 合并 `updatedInput`
   - 走 `handleHookAllow(...)`
3. 如果 hook 返回 deny：
   - 记 reject 日志
   - 如有 `interrupt` 则 abort
   - 返回 `buildDeny(...)`
4. 否则继续等下一个 hook，全部结束后返回 `null`

这说明这里不是简单“跑 hooks”，而是：

```text
把 hook 结果直接翻译成统一 PermissionDecision
```

所以 hooks 到这里就已经被纳入标准 permission transaction 语义了。

---

### allow / deny 结果对象也是在这里统一构造的

#### `buildAllow(...)`
位置：`src/hooks/toolPermission/PermissionContext.ts:264`

它统一组装：

- `behavior: 'allow'`
- `updatedInput`
- `userModified`
- `decisionReason`
- `acceptFeedback`
- `contentBlocks`

这说明 allow 结果不是只有一个布尔值，而是一个**富结果对象**。

尤其注意：

- 用户是否改了输入
- 批准时是否附带反馈
- 是否附带额外内容块

这些都被纳入标准返回。

#### `buildDeny(...)`
位置：`src/hooks/toolPermission/PermissionContext.ts:285`

相对简单，但也统一收口成：

- `behavior: 'deny'`
- `message`
- `decisionReason`

所以结果对象的构造不再散落在多个 handler 里。

---

### 用户批准与 hook 批准是两套不同 helper

#### `handleUserAllow(...)`
位置：`src/hooks/toolPermission/PermissionContext.ts:291`

这个方法做了几件事：

1. `persistPermissions(permissionUpdates)`
2. 按“用户批准”来源记日志
3. 比较 `tool.inputsEquivalent(...)` 判断用户是否修改了输入
4. 规范化 feedback
5. 返回 `buildAllow(...)`

这里有个很关键的细节：

```text
用户批准不只是 approve，还会记录“用户有没有改动输入”
```

这意味着权限系统关心的不只是 yes/no，还关心：

- 用户有没有在批准时修正参数

这会直接影响后续 tool.call 的输入。

#### `handleHookAllow(...)`
位置：`src/hooks/toolPermission/PermissionContext.ts:319`

它和用户版本很像，但来源变成 hook，且不处理 `userModified`。

这体现的是：

```text
不同批准来源，共享同一骨架，但保留来源特有语义
```

---

### queue 操作为什么也放进 ctx

位置：
- `pushToQueue(...)` `src/hooks/toolPermission/PermissionContext.ts:337`
- `removeFromQueue(...)` `src/hooks/toolPermission/PermissionContext.ts:340`
- `updateQueueItem(...)` `src/hooks/toolPermission/PermissionContext.ts:343`

这三项非常朴素，但它们很重要。

因为这说明 permission queue 在设计上不是 React 组件的私有状态，而是：

```text
权限事务的一部分副作用接口
```

于是上层 handler 可以只和 `ctx` 交互，而不需要直接依赖 `setState`。

这也为后面的非 React 场景留下了余地。

---

### `createPermissionQueueOps(...)` 真正干了什么

位置：`src/hooks/toolPermission/PermissionContext.ts:357`

这个函数就是 React state 到通用 queue 接口的桥：

- `push(item)` -> `setToolUseConfirmQueue(queue => [...queue, item])`
- `remove(toolUseID)` -> filter 掉对应项
- `update(toolUseID, patch)` -> map 定位后 merge patch

关键点不是逻辑复杂，而是抽象边界很清楚：

```text
PermissionContext 不直接依赖 React queue 结构
React 只是 PermissionQueueOps 的一个后端实现
```

这就是典型的 adapter pattern。

---

### 元

问题：**这一站真正想解决的架构问题是什么？**

回答：这一站把一次 permission flow 所需的副作用能力打包成统一 `ctx`，包括 queue、日志、持久化、abort 和结果构造。尤其 `createResolveOnce(...)` 表明作者把权限流程当成多异步来源竞争同一结果的问题来设计。

### 反

问题：**如果把这一站的设计反过来，会发生什么？**

回答：如果每个 handler 都自己管 queue、日志和 resolve 竞态，很快就会出现重复逻辑和时序 bug。权限链会难以维护，也难以确保行为一致。

### 空

问题：**跳出当前文件名，这一站背后更大的问题是什么？**

回答：更大的问题是，复杂审批流程是否需要事务上下文。这里的答案很明确：需要，因为 ask 从来不是单线流程，而是带副作用的异步竞态。


### 读完这个文件后，你应该抓住的 6 个事实

#### 事实 1：`PermissionContext` 是权限流程的事务工具箱
它把日志、持久化、结果构造、abort、hooks、classifier、queue 操作收口成一个对象。

#### 事实 2：权限流被视为多异步来源竞争的场景
`createResolveOnce(...)` 明确就是为了解决重复 resolve 和竞态问题。

#### 事实 3：批准权限可能意味着“修改规则”
`persistPermissions(...)` 会同时落盘 permission updates，并刷新当前 appState 中的权限上下文。

#### 事实 4：取消 / 中止不是普通异常
它有独立消息语义、subagent 语义和 analytics 事件。

#### 事实 5：hooks 和 classifier 在这一层被标准化了
它们都不再是外部特例，而是 `ctx` 可调用的统一事务操作。

#### 事实 6：queue 操作被抽象成通用接口
React 只是一个实现，而不是权限系统本身的核心依赖。

---

### 现在把前三站串起来

到这里可以把权限链路再细化成：

```text
permissions.ts
  -> 给出共享权限裁决
useCanUseTool.tsx
  -> 把裁决接进运行时流程
PermissionContext.ts
  -> 提供日志、持久化、queue、hook、classifier、abort 的事务工具箱
```

所以三层分工是：

```text
permissions.ts = 判定规则
useCanUseTool.tsx = 运行时分发
PermissionContext.ts = 事务副作用封装
```

---

### 第一轮阅读时，不要陷进去的点

先不要深挖这些：

- `ContentBlockParam` 在每个场景里的具体内容
- analytics event 的所有字段用途
- `PermissionUpdate.destination` 的全部持久化目标
- Bash classifier 的更深 prompt 规则

这一轮先抓住：

```text
PermissionContext.ts = 权限流程的事务工具箱
```

以及：

```text
它把 permission flow 需要的副作用与结果构造统一打包，避免上层 handler 到处手搓状态更新
```

就够了。

---

### 下一站建议

下一站最顺的是：

```text
src/hooks/toolPermission/handlers/interactiveHandler.ts
```

因为现在 `ask` 分支里最大的黑箱已经只剩下“真正的交互式审批怎么落地”，下一步最自然就是看：

**permission queue 里的请求最终怎样变成用户可见的审批对话框，以及用户的决定如何回流到 `PermissionContext`。**

---

## 第 12 站：`src/hooks/toolPermission/handlers/interactiveHandler.ts`

### 这是什么文件

`src/hooks/toolPermission/handlers/interactiveHandler.ts` 是权限系统里的**交互式审批驱动器**。

上一站已经知道：

- `permissions.ts` 负责共享权限裁决
- `useCanUseTool.tsx` 负责运行时分发
- `PermissionContext.ts` 提供事务工具箱

而这一站负责的是：

```text
当结果是 ask 时，怎样把一个权限请求真正变成可交互的审批流程，并让本地 UI、bridge、channel、hook、classifier 一起竞争同一个最终结果
```

所以最准确的定位是：

```text
interactiveHandler.ts = ask 分支的多路竞态协调器
```

---

### 先看文件顶部注释

位置：`src/hooks/toolPermission/handlers/interactiveHandler.ts:43`

这段注释其实已经把文件的本质说透了：

- 它把 `ToolUseConfirm` 推进 confirm queue
- 给 queue 项挂上 `onAbort / onAllow / onReject / recheckPermission / onUserInteraction`
- 后台异步跑 hooks 和 bash classifier
- 它们会和用户交互竞争
- 用 `resolve-once` guard 避免重复 resolve

也就是说，这个文件不是“画弹窗”的组件，而是：

```text
围绕一个 ask 请求，组装所有可能的批准/拒绝来源，并保证只有一个赢家生效
```

---

### 整个主链路先抓住

把 `handleInteractivePermission(...)` 粗略展开，可以先得到这张脑图：

```text
收到 ask 结果
  -> createResolveOnce(resolve)
  -> push 一个 ToolUseConfirm 到 queue
  -> 本地用户可操作 onAllow / onReject / onAbort / recheckPermission
  -> 如果有 bridge，则把请求发给远端 CCR 并等回应
  -> 如果有 channel relay，则把请求发往 Telegram/iMessage 等并等回应
  -> 并行执行 hooks
  -> 并行执行 Bash classifier
  -> 谁先 claim() 成功，谁赢
  -> 统一清理 queue / classifier 状态 / bridge / channel 订阅
```

这说明它的核心不是“展示对话框”，而是：

```text
把 ask 变成一个多来源竞争的审批状态机
```

---

### 第一部分：resolve-once 竞态保护

位置：`src/hooks/toolPermission/handlers/interactiveHandler.ts:70`

这里先从 `createResolveOnce(resolve)` 拿到：

- `resolveOnce`
- `isResolved`
- `claim`

然后维护：

- `userInteracted`
- `checkmarkTransitionTimer`
- `checkmarkAbortHandler`
- `channelUnsubscribe`
- `bridgeRequestId`

这说明一开始作者就把它当成**高度并发的竞态场景**处理。

谁会参与竞争？

- 本地用户点允许/拒绝
- 本地用户直接 abort
- recheck 后发现已自动允许
- bridge 远端回应
- channel 远端回应
- hooks 抢先决定
- bash classifier 抢先自动批准

所以这里你要建立一个非常重要的理解：

```text
interactiveHandler.ts 的本质不是 UI，而是“审批竞态仲裁器”
```

---

### 第二部分：先把请求放进 permission queue

位置：`src/hooks/toolPermission/handlers/interactiveHandler.ts:92`

这里调用 `ctx.pushToQueue(...)`，放进去的是一个 `ToolUseConfirm`，里面不仅有展示数据，还有一整套回调：

- `onUserInteraction`
- `onDismissCheckmark`
- `onAbort`
- `onAllow`
- `onReject`
- `recheckPermission`

这说明 permission queue 里存的不是“静态弹窗数据”，而是：

```text
一份可执行的交互事务对象
```

也就是说，UI 不是拿到一个 dead data object 再自己实现逻辑，而是直接拿到已经绑好行为的请求实体。

---

### `onUserInteraction()` 为什么重要

位置：`src/hooks/toolPermission/handlers/interactiveHandler.ts:108`

这个回调在用户开始操作 dialog 时触发，比如：

- 按方向键
- tab
- 输入反馈

它做两件事：

- `userInteracted = true`
- 清掉 classifier checking / UI indicator

但前 200ms 有一个 grace period，不会立刻触发。

这段设计很细：

```text
如果用户已经开始亲自处理这个审批，就不要再让后台 classifier 抢先“替用户决定”
```

同时 200ms 缓冲是为了避免刚弹出来时的误按直接打断 auto-approve。

所以这里体现的不是简单事件处理，而是：

```text
交互体验优先级切换：一旦用户介入，自动审批应退场
```

---

### 本地用户三条主回调

#### 1. `onAbort()`
位置：`src/hooks/toolPermission/handlers/interactiveHandler.ts:137`

这里会：

- `claim()` 抢占唯一决策权
- 如果有 bridge，向远端发 deny + cancel
- 取消 channel 订阅
- `ctx.logCancelled()`
- 记录 `user_abort`
- `resolveOnce(ctx.cancelAndAbort(...))`

也就是说，abort 不是简单“关掉窗口”，而是一次正式的权限拒绝流程收口。

---

#### 2. `onAllow()`
位置：`src/hooks/toolPermission/handlers/interactiveHandler.ts:154`

这里会：

- 先 `claim()`，避免 await 期间被别人抢先
- bridge 回传 allow 结果与 permissionUpdates
- 取消 channel 订阅
- 调 `ctx.handleUserAllow(...)`
- 最终 `resolveOnce(...)`

这里最重要的是：

```text
用户点击 allow 后，不是直接返回 allow，而是复用 PermissionContext 的标准“用户批准事务”
```

也就是把：

- 权限持久化
- 日志
- userModified 判定
- feedback/contentBlocks

全部复用了。

---

#### 3. `onReject()`
位置：`src/hooks/toolPermission/handlers/interactiveHandler.ts:183`

这里会：

- `claim()`
- bridge 回传 deny
- 取消 channel 订阅
- 记录 `user_reject`
- `resolveOnce(ctx.cancelAndAbort(feedback, ...))`

这说明 reject 也不是单纯 UI 行为，而是正式进入 permission transaction 的拒绝分支。

---

### `recheckPermission()` 是干什么的

位置：`src/hooks/toolPermission/handlers/interactiveHandler.ts:204`

这是一个很关键的补偿机制。

它会重新调用：

- `hasPermissionsToUseTool(...)`

如果 fresh result 已经变成 allow，就：

- `claim()`
- 取消 bridge 远端请求
- 取消 channel 订阅
- 移除 queue 项
- 记一次 `source: 'config'` 的 accept
- `resolveOnce(ctx.buildAllow(...))`

这说明审批不是“弹出来以后就固定了”，而是允许在弹窗存在期间**重新评估当前权限状态**。

典型场景就是：

- 期间 mode 变了
- 外部批准已落盘
- bridge 那边触发了模式切换

所以它的核心价值是：

```text
让已显示的审批请求具备“动态重新判定”能力，而不是只能等用户手动点
```

---

### 第三部分：bridge 远端审批竞态

位置：`src/hooks/toolPermission/handlers/interactiveHandler.ts:234`

如果存在 `bridgeCallbacks`，这里会：

1. `sendRequest(...)` 把 permission request 发给远端 CCR
2. `onResponse(...)` 监听 bridge 侧返回
3. 回来后与本地用户 / hook / classifier 竞争 `claim()`
4. 谁赢谁生效

bridge 返回 allow 时：

- 可能持久化 `updatedPermissions`
- 记录 user accept
- `resolveOnce(ctx.buildAllow(...))`

bridge 返回 deny 时：

- 记录 user reject
- `resolveOnce(ctx.cancelAndAbort(...))`

这说明：

```text
本地 CLI 审批框和远端 claude.ai 审批框是并行竞态关系，不是主从关系
```

谁先回应，谁赢。

这是一个非常典型的多终端协同设计。

---

### 第四部分：channel relay 审批竞态

位置：`src/hooks/toolPermission/handlers/interactiveHandler.ts:300`

这一段更有意思，它把 permission request 发往：

- Telegram
- iMessage
- 其他 channel relay 客户端

流程是：

1. 从当前 MCP clients 里筛可用 channel relay
2. 组装 `ChannelPermissionRequestParams`
3. 给每个 client 发 `CHANNEL_PERMISSION_REQUEST_METHOD` 通知
4. 订阅 `channelCallbacks.onResponse(...)`
5. 如果 channel 侧回复 yes/no，就和其他来源继续抢 `claim()`

这里作者还专门强调：

- channel 回复不会变成正常聊天消息进入 Claude
- channel 只有 yes/no，没有 `updatedInput` 路径
- send_message 失败也不致命，本地 dialog 仍然存在

所以 channel relay 的定位非常清楚：

```text
它是交互式审批的一个远端副通道，而不是唯一入口
```

本地 UI 始终是兜底。

---

### 第五部分：后台 hooks 也在和用户竞争

位置：`src/hooks/toolPermission/handlers/interactiveHandler.ts:410`

如果没有 `awaitAutomatedChecksBeforeDialog`，这里会异步执行：

- `ctx.runHooks(...)`

如果 hook 先返回决定，就：

- `claim()`
- 取消 bridge
- 取消 channel
- 移除 queue 项
- `resolveOnce(hookDecision)`

这说明即使交互式 dialog 已经显示出来，后台 hook 仍然可能抢先决定结果。

所以 ask 流程不是“先 UI，后自动化”，而是：

```text
UI、hook、bridge、channel 可以并行竞争
```

这正是这份 handler 最核心的设计特征。

---

### 第六部分：Bash classifier 自动批准竞态

位置：`src/hooks/toolPermission/handlers/interactiveHandler.ts:433`

当满足这些条件时：

- 开启 `BASH_CLASSIFIER`
- 当前 ask 结果带 `pendingClassifierCheck`
- 工具是 `Bash`
- 没有提前 await automated checks

它会：

1. `setClassifierChecking(...)`
2. `executeAsyncClassifierCheck(...)`
3. 传入：
   - `shouldContinue: () => !isResolved() && !userInteracted`
   - `onComplete`
   - `onAllow`

这里最关键的是 `shouldContinue`：

```text
只要已有赢家，或者用户已开始操作，就停止 classifier 抢跑
```

如果 classifier 抢先 allow：

- `claim()`
- 取消 bridge / channel
- 清理 checking 状态
- 更新 queue item 为 auto-approved checkmark
- 同步 classifier approval UI 状态
- 记 classifier accept
- `resolveOnce(ctx.buildAllow(...))`

然后还会出现一个非常产品化的细节：

- 如果终端聚焦，checkmark 留 3 秒
- 否则留 1 秒
- 用户可按 Esc 提前 dismiss
- 如果中途 abort，也会把这个“纯展示态” queue 项清走

这说明 classifier auto-approve 在这里不仅是逻辑行为，还是完整的**交互过渡效果**。

所以这段最应该记住的是：

```text
interactiveHandler.ts 不只协调权限结果，还负责协调“批准结果如何被用户感知”
```

---

### 为什么这个文件是 ask 分支的核心黑箱

因为它把本来可能很散的几类逻辑，全塞进了一个统一竞态模型里：

- 本地 UI 操作
- bridge 远端审批
- channel 远端审批
- hook 自动化
- Bash classifier 自动批准
- 动态 recheck
- abort / cleanup / dismiss 过渡

而且所有这些路径最终都遵守同一个原则：

```text
claim() 成功者赢，其他路径自动失效
```

这就是它最核心的架构价值。

---

### 元

问题：**这一站真正想解决的架构问题是什么？**

回答：这一站不是单纯“弹一个权限框”，而是围绕 ask 组织一场多路竞态。用户、bridge、channel、hooks、classifier 都可能成为最终批准或拒绝来源，但只允许一个赢家生效。

### 反

问题：**如果把这一站的设计反过来，会发生什么？**

回答：如果交互式审批只是普通弹窗，后台自动化来源和远端回应就很难无缝接入。没有 resolve-once 保护，还会频繁出现重复 resolve 和状态打架。

### 空

问题：**跳出当前文件名，这一站背后更大的问题是什么？**

回答：更大的问题是，人机交互审批如何与自动化审批共存。这里展示的不是 UI 组件，而是一种多来源仲裁模型。


### 读完这个文件后，你应该抓住的 6 个事实

#### 事实 1：interactive permission 不是单纯本地弹窗
它其实是本地 UI、bridge、channel、hook、classifier 共同参与的一场竞态。

#### 事实 2：permission queue 里装的是“可执行审批事务”
不只是展示数据，还有一整套回调与重试逻辑。

#### 事实 3：`claim()` 是整个文件最关键的同步原语
它保证多个异步批准/拒绝来源只能有一个最终赢家。

#### 事实 4：用户一旦开始交互，自动 classifier 应该退场
`userInteracted` + grace period 就是在做这个优先级切换。

#### 事实 5：远端 bridge / channel 不是补丁，而是正式审批通道
它们与本地 UI 处于并行竞争关系。

#### 事实 6：这个文件还负责批准后的过渡体验
例如 classifier auto-approve 的 checkmark 展示与延迟移除。

---

### 现在把权限链再串一层

到这里，ask 分支的执行链可以更完整地写成：

```text
permissions.ts
  -> 得到 ask
useCanUseTool.tsx
  -> 把 ask 分给 interactiveHandler
PermissionContext.ts
  -> 提供日志/持久化/queue/abort/hook/classifier 工具箱
interactiveHandler.ts
  -> 让本地 UI、bridge、channel、hooks、classifier 竞争最终决策
```

所以四层分工是：

```text
permissions.ts = 基础裁决
useCanUseTool.tsx = 运行时分发
PermissionContext.ts = 事务工具箱
interactiveHandler.ts = ask 分支的竞态协调器
```

---

### 第一轮阅读时，不要陷进去的点

先不要深挖这些：

- channel relay 的具体 MCP server 实现
- bridgeCallbacks 背后的远端协议细节
- classifier prompt 规则本身
- `ToolUseConfirm` UI 组件的全部渲染细节

这一轮先抓住：

```text
interactiveHandler.ts = ask 分支的多路竞态协调器
```

以及：

```text
交互式审批不是一个弹窗，而是一套让多个批准来源并发竞争、最终只允许一个赢家生效的机制
```

就够了。

---

### 下一站建议

下一站最顺的是：

```text
src/hooks/toolPermission/handlers/coordinatorHandler.ts
```

因为现在你已经看懂了“主 agent 的交互式 ask 流”，下一步最自然就是对照看：

**当 `awaitAutomatedChecksBeforeDialog` 开启时，coordinator worker 是怎样优先跑自动化检查、尽量减少打断用户的。**

---

## 第 13 站：`src/hooks/toolPermission/handlers/coordinatorHandler.ts`

### 这是什么文件

`src/hooks/toolPermission/handlers/coordinatorHandler.ts` 是权限系统里的**coordinator worker 自动化前置处理器**。

上一站 `interactiveHandler.ts` 看到的是：

```text
ask -> 先把 dialog 弹出来，再让本地用户 / bridge / channel / hook / classifier 并发竞争
```

而这个文件处理的是另一种策略：

```text
ask -> 先顺序等待自动化检查（hook、classifier）
   -> 只有它们都没解决时，才回落到 interactive dialog
```

所以最准确的定位是：

```text
coordinatorHandler.ts = “先自动化、后打扰用户”的 ask 预处理器
```

---

### 文件非常短，但位置很关键

这个文件只有一个核心函数：

- `handleCoordinatorPermission(...)` `src/hooks/toolPermission/handlers/coordinatorHandler.ts:26`

别因为它短就低估它。

它的重要性不在“逻辑量大”，而在于它明确表达了一条权限产品策略：

```text
coordinator worker 应尽量先靠自动化解决权限问题，只有自动化解决不了，才升级为真正打断用户
```

这其实是在定义 **ask 的调度顺序**。

---

### 先看输入参数

位置：`src/hooks/toolPermission/handlers/coordinatorHandler.ts:8`

它吃的参数有：

- `ctx`
- `pendingClassifierCheck`
- `updatedInput`
- `suggestions`
- `permissionMode`

这几个参数刚好覆盖了自动化检查最需要的上下文：

- `ctx`：事务工具箱
- `pendingClassifierCheck`：Bash classifier 的挂起检查
- `updatedInput`：可能已被前序逻辑修正过的输入
- `suggestions`：可能的 permission updates 建议
- `permissionMode`：当前权限模式

这说明它不是自己算权限，而是：

```text
消费上游已经准备好的 ask 上下文，然后决定自动化检查是否足以把它消化掉
```

---

### 核心主链路只有三步

#### 第 1 步：先跑 hooks
位置：`src/hooks/toolPermission/handlers/coordinatorHandler.ts:32`

这里先调：

- `ctx.runHooks(permissionMode, suggestions, updatedInput)`

注释里写得很明确：

```text
hooks first (fast, local)
```

这说明在 coordinator 路径里，hook 被当作：

- 本地
- 快速
- 零额外交互成本

的第一优先级处理器。

如果 hook 已经能给出 allow/deny，就直接返回，不再往下走。

所以这里的架构意图非常清楚：

```text
能本地自动解决的，就不要去跑更贵的 classifier，更不要打断用户
```

---

#### 第 2 步：再跑 classifier
位置：`src/hooks/toolPermission/handlers/coordinatorHandler.ts:40`

如果 hooks 没解决，再尝试：

- `ctx.tryClassifier?.(...)`

而且注释也写清楚了：

```text
classifier (slow, inference -- bash only)
```

这和上一站、前几站读到的设计一脉相承：

```text
classifier 是第二层、更贵的自动化裁决器
```

在 coordinator worker 场景里，它不是和用户并发竞争，而是被**顺序等待**。

这点和 `interactiveHandler.ts` 的主 agent 路径形成鲜明对照：

- 主 agent：dialog 已经出现，classifier 在后台抢跑
- coordinator worker：先等 classifier，等它没结果才决定要不要 dialog

这正是两条 ask 路径的核心区别。

---

#### 第 3 步：都没解决，就返回 `null`
位置：`src/hooks/toolPermission/handlers/coordinatorHandler.ts:59`

返回 `null` 的语义不是错误，而是：

```text
自动化检查没有解决，请调用方继续走 interactive dialog
```

这说明它的职责边界非常纯粹：

- 它不负责显示 dialog
- 它不负责继续竞态
- 它只负责“自动化前置检查是否已经足够解决 ask”

所以你可以把它理解成：

```text
interactive dialog 前的一道自动化预检门
```

---

### 为什么这里是“顺序等待”，不是“并行竞争”

这正是本文件最值得记住的设计点。

在主 agent 交互流里，用户已经被打断了，所以：

- classifier
n- hooks
- bridge/channel
- 本地用户

可以一起竞争，追求的是整体响应最快。

但在 coordinator worker 里，目标变成了：

```text
尽量避免打断用户
```

所以它采用的是：

1. hook（快）
2. classifier（慢但仍自动）
3. dialog（最后兜底）

也就是把“降低用户打扰”作为首要目标，而不是“谁先返回都行”。

---

### 错误处理也体现了产品取向

位置：`src/hooks/toolPermission/handlers/coordinatorHandler.ts:47`

这里如果自动化检查出错，不会直接失败，也不会中断整个 ask 流。

而是：

- `logError(...)`
- 然后继续 fall through 到 dialog

也就是说：

```text
自动化检查是优化层，不是单点故障点
```

只要 hooks / classifier 自己坏了，最终仍然可以退回人工审批。

这是很稳妥的产品化设计。

尤其注释里还提到：

- 非 Error throw 也会包上上下文前缀再记录

说明这里在意的是：

- 日志可追踪
- 但不让自动化失败破坏主流程

---

### 元

问题：**这一站真正想解决的架构问题是什么？**

回答：这一站专门定义 coordinator worker 的 ask 策略，即先跑 hooks 和 classifier，只有自动化都无法决策时才回落到交互式审批。重点是“先自动化、后打断用户”的顺序。

### 反

问题：**如果把这一站的设计反过来，会发生什么？**

回答：如果 coordinator 也一上来就交互式询问，自动化 worker 的意义会被削弱，很多本可本地消化的审批都会不必要地打断用户。

### 空

问题：**跳出当前文件名，这一站背后更大的问题是什么？**

回答：更大的问题是，不同 agent 角色是否应该共享同一审批顺序。这个文件给出的答案是否定的，角色不同，ask 的默认处置顺序也应该不同。


### 读完这个文件后，你应该抓住的 5 个事实

#### 事实 1：它不是另一个 interactive handler
它不负责 dialog，只负责 dialog 之前的自动化预检查。

#### 事实 2：coordinator worker 的 ask 策略和主 agent 不一样
主 agent 倾向并发竞态；coordinator worker 倾向顺序等待自动化检查。

#### 事实 3：hook 是第一优先级
因为它更快、更本地、成本更低。

#### 事实 4：classifier 是第二优先级
它更慢、更贵，但仍优先于打断用户。

#### 事实 5：自动化失败不会阻塞人工审批
错误只会被记录，然后回落到 dialog。

---

### 现在把两种 ask 路径对照起来

你可以把当前权限 ask 流分成两种风格：

#### 风格 A：主 agent 交互式 ask
对应：`src/hooks/toolPermission/handlers/interactiveHandler.ts:57`

```text
先展示审批入口
再让本地用户、bridge、channel、hook、classifier 并发竞争
```

#### 风格 B：coordinator worker ask
对应：`src/hooks/toolPermission/handlers/coordinatorHandler.ts:26`

```text
先按顺序等待 hook -> classifier
只有都没解决，才进入 interactive dialog
```

这两者背后的优化目标不一样：

- 主 agent：已经进入交互，追求谁最快解决
- coordinator worker：尽量不打断用户

这是你理解权限系统行为差异的关键。

---

### 现在把权限链再完整串一遍

到这里可以更完整地写成：

```text
permissions.ts
  -> 得到 allow / deny / ask 基础裁决
useCanUseTool.tsx
  -> 决定 ask 进入哪条运行时路径
PermissionContext.ts
  -> 提供日志/持久化/queue/abort/hook/classifier 工具箱
coordinatorHandler.ts
  -> coordinator 场景下先顺序等待自动化检查
interactiveHandler.ts
  -> 主交互场景下让多路审批来源并发竞争
```

所以到这一站，你已经基本把权限系统的主干读通了。

---

### 第一轮阅读时，不要陷进去的点

这个文件很短，反而容易过度解读。

这一轮不要陷进去：

- 为什么一定是 hooks 再 classifier，而不是反过来
- 各种 feature flag 在不同构建里的全部差异
- coordinator worker 的更外层业务上下文

先抓住：

```text
coordinatorHandler.ts = “先自动化、后打扰用户”的 ask 预处理器
```

就够了。

---

### 下一站建议

下一站最顺的是：

```text
src/hooks/toolPermission/handlers/swarmWorkerHandler.ts
```

因为现在还差最后一块对照图：

**swarm / teammate worker 在 ask 场景下，怎样把权限请求转发给 leader 或其他协调链路，而不是直接自己处理。**

---

## 第 14 站：`src/hooks/toolPermission/handlers/swarmWorkerHandler.ts`

### 这是什么文件

`src/hooks/toolPermission/handlers/swarmWorkerHandler.ts` 是权限系统里的**swarm worker 权限转发器**。

如果说：

- `interactiveHandler.ts` 解决的是主 agent 的交互式 ask
- `coordinatorHandler.ts` 解决的是 coordinator worker 的自动化前置 ask

那么这个文件解决的是第三种典型场景：

```text
当前 ask 发生在 swarm / teammate worker 身上，worker 自己不直接做最终审批，而是把请求上送给 leader
```

所以最准确的定位是：

```text
swarmWorkerHandler.ts = swarm worker 的权限上行代理层
```

---

### 文件顶部注释已经给出了主链路

位置：`src/hooks/toolPermission/handlers/swarmWorkerHandler.ts:26`

注释列了 4 件事：

1. Bash classifier auto-approval
2. 把 permission request 发给 leader mailbox
3. 注册 leader 响应回调
4. 等待期间设置 pending indicator

这已经说明它不是“弹窗处理器”，而是：

```text
一个把 worker 侧 ask 升级成 leader 协调审批的桥接器
```

---

### 先看入口判断

位置：`src/hooks/toolPermission/handlers/swarmWorkerHandler.ts:40`

一开始先判断：

- `isAgentSwarmsEnabled()`
- `isSwarmWorker()`

如果不满足，就直接返回 `null`。

这说明它的边界非常清晰：

```text
只有真正处于 swarm worker 场景时，这条权限路径才生效
否则调用方继续落回本地 interactive handling
```

也就是说，它是一个**条件性拦截器**，不是所有 ask 都会走这里。

---

### 第一部分：先尝试 classifier 自动批准

位置：`src/hooks/toolPermission/handlers/swarmWorkerHandler.ts:49`

和 coordinator 路径类似，这里先尝试：

- `ctx.tryClassifier?.(...)`

而且注释专门强调：

```text
agent 会等待 classifier 结果，而不是像 main agent 那样和用户交互竞态
```

这说明 swarm worker 的策略和主 agent 一样不一样？

- 和主 agent 不一样：不会先弹本地 dialog 再竞态
- 和 coordinator 有点像：都会先等自动化检查

因为 worker 不是最终审批终端，所以优先先看看能不能自动解决，减少往 leader 转发。

所以这里的第一结论是：

```text
swarm worker ask 也优先走自动化；只有自动化解决不了，才升级为 leader 审批
```

---

### 第二部分：真正的核心——通过 mailbox 转发给 leader

位置：`src/hooks/toolPermission/handlers/swarmWorkerHandler.ts:59`

这一段才是全文件的主角。

整体思路是：

```text
worker 不自己持有最终审批权
worker 负责创建 permission request，发给 leader，并等待 leader 回信
```

这就是一个很标准的“代理审批”模型。

---

### 转发链路的细分步骤

#### 1. 先定义 `clearPendingRequest()`
位置：`src/hooks/toolPermission/handlers/swarmWorkerHandler.ts:61`

这里先包了一个 helper，把：

- `pendingWorkerRequest: null`

写回 appState。

意思很明确：

```text
worker 在等待 leader 审批期间，UI/状态层会显示“有一个挂起的 leader 审批请求”
```

等审批结束，就统一清掉。

所以这个 handler 不只是消息转发，还负责维护 worker 侧的等待态展示。

---

#### 2. 用 Promise 包住整段“等待 leader 回应”的生命周期
位置：`src/hooks/toolPermission/handlers/swarmWorkerHandler.ts:67`

这里起了一个：

```ts
new Promise<PermissionDecision>(resolve => { ... })
```

里面再配合：

- `createResolveOnce(resolve)`

这说明 leader 审批本质上被建模成：

```text
一次异步 RPC / mailbox round-trip
```

worker 发请求之后，就进入等待状态，直到：

- leader allow
- leader reject
- 本地 abort

其中一个事件先完成。

---

#### 3. 先创建 request
位置：`src/hooks/toolPermission/handlers/swarmWorkerHandler.ts:70`

通过：

- `createPermissionRequest(...)`

把这些信息打包出去：

- `toolName`
- `toolUseId`
- `input`
- `description`
- `permissionSuggestions`

这说明 leader 端不是只看到一个“允许/拒绝”按钮，而是收到了一份比较完整的审批上下文。

也就是说，这不是最小化协议，而是**带语义信息的权限请求对象**。

---

#### 4. 关键细节：先注册 callback，再发请求
位置：`src/hooks/toolPermission/handlers/swarmWorkerHandler.ts:79`

这里有个很重要的竞态保护注释：

```text
先注册回调，再发送请求，避免 leader 回得太快而 callback 还没挂上
```

这说明作者明确处理了一个经典异步 bug：

- 请求发出
- leader 秒回
- 本地还没开始监听
- 响应丢失

所以这里的设计非常扎实：

```text
先挂 listener，再 send
```

这是读这种代码时必须注意的高质量细节。

---

### leader 回复 allow 时怎么处理

位置：`src/hooks/toolPermission/handlers/swarmWorkerHandler.ts:84`

回调里如果 leader 允许：

- `claim()` 抢占唯一决策权
- `clearPendingRequest()`
- 合并 `allowedInput`（没有就退回 `ctx.input`）
- 调 `ctx.handleUserAllow(...)`
- `resolveOnce(...)`

这里非常关键的一点是：

```text
leader 的批准在 worker 侧被当成“用户批准”语义处理
```

所以它会复用：

- permissionUpdates 持久化
- user allow 日志
- feedback/contentBlocks 处理
- 最终 allow 结果构造

换句话说：

```text
leader 就是 worker 的审批 authority
```

在 worker 看来，leader 的批准等价于“上级用户批准”。

---

### leader 回复 reject 时怎么处理

位置：`src/hooks/toolPermission/handlers/swarmWorkerHandler.ts:109`

如果 leader 拒绝：

- `claim()`
- `clearPendingRequest()`
- 记 `user_reject`
- `resolveOnce(ctx.cancelAndAbort(...))`

这说明 leader 的拒绝同样被归入标准 permission transaction 语义，而不是单独定义一个新结果类型。

所以你应该记住：

```text
swarm 权限转发不是另起一套 decision 协议，而是把 leader 响应翻译回现有 allow/reject 语义
```

---

### 第三部分：发送请求并设置等待态

位置：`src/hooks/toolPermission/handlers/swarmWorkerHandler.ts:122`

当 callback 注册好之后，才真正：

- `sendPermissionRequestViaMailbox(request)`

然后马上更新 appState：

- `pendingWorkerRequest = { toolName, toolUseId, description }`

这一步很重要，因为它把“正在等 leader 审批”显式暴露给了运行时状态。

也就是说：

```text
worker 在权限等待期不是隐身挂起，而是有明确的可观察等待态
```

这对 UI、状态提示、调试都很重要。

---

### 第四部分：abort 也要收口，不允许 promise 挂死

位置：`src/hooks/toolPermission/handlers/swarmWorkerHandler.ts:135`

这里对 `abortController.signal` 挂了监听。

如果等待 leader 回应期间本地 abort：

- `claim()`
- `clearPendingRequest()`
- `ctx.logCancelled()`
- `resolveOnce(ctx.cancelAndAbort(undefined, true))`

这段注释写得很清楚：

```text
如果 abort 发生，必须把 promise resolve 掉，不能无限挂等 leader
```

这就是很成熟的异步设计：

```text
所有等待远端响应的流程，都必须有本地终止出口
```

---

### 第五部分：失败时为什么回落到本地 interactive handling

位置：`src/hooks/toolPermission/handlers/swarmWorkerHandler.ts:150`

如果整个 swarm permission submission 过程抛错：

- `logError(toError(error))`
- 返回 `null`

而 `null` 的语义还是一样：

```text
这一条路径没能处理，请调用方继续落回本地 interactive handling
```

这说明 swarm forwarding 也是一层优化 / 专用通道，而不是不可替代的单点。

即使 mailbox 同步坏了，也不会把 ask 流彻底卡死。

---

### 元

问题：**这一站真正想解决的架构问题是什么？**

回答：这一站解决 swarm worker 自己遇到 ask 时怎么办。它会先尝试 classifier 自动批准，不行再把请求上送 leader，由 leader 负责最终裁决。

### 反

问题：**如果把这一站的设计反过来，会发生什么？**

回答：如果 worker 自己直接弹本地审批，team 模式就会失去中心化控制面。权限体验会碎片化，用户也很难知道到底该看哪个会话。

### 空

问题：**跳出当前文件名，这一站背后更大的问题是什么？**

回答：更大的问题是，多 agent 协作里权限应当由谁最终负责。这里体现的是“执行者可以发起请求，但控制权最好集中在 leader”。


### 读完这个文件后，你应该抓住的 6 个事实

#### 事实 1：swarm worker 默认不自己做最终审批
它把 ask 转发给 leader，让 leader 作为审批 authority。

#### 事实 2：自动化检查仍然优先于 leader 转发
先试 classifier，能自动解决就不必上送。

#### 事实 3：转发链路本质上是一次 mailbox RPC
create request -> register callback -> send -> await response。

#### 事实 4：先注册 callback 再发送请求是一个关键竞态修复点
否则 leader 秒回时可能丢响应。

#### 事实 5：worker 侧有明确的 `pendingWorkerRequest` 等待态
这使 leader 审批过程对 UI / 状态系统可见。

#### 事实 6：远端 leader 响应最终仍会被翻译回标准 permission transaction 语义
allow 走 `handleUserAllow(...)`，reject 走 `cancelAndAbort(...)`。

---

### 现在把三种 ask 路径完整对照起来

到这里，权限系统里最关键的三条 ask 路径已经齐了：

#### 1. 主 agent 交互式 ask
文件：`src/hooks/toolPermission/handlers/interactiveHandler.ts:57`

```text
先显示审批入口
本地用户 / bridge / channel / hook / classifier 并发竞态
```

#### 2. coordinator worker ask
文件：`src/hooks/toolPermission/handlers/coordinatorHandler.ts:26`

```text
先顺序等 hook -> classifier
都没解决再回落到 interactive dialog
```

#### 3. swarm worker ask
文件：`src/hooks/toolPermission/handlers/swarmWorkerHandler.ts:40`

```text
先试 classifier
不行就把权限请求发给 leader mailbox
等待 leader 回应
失败时再回落本地 handling
```

这三条路径背后对应的是三种不同运行时角色：

- main agent
- coordinator worker
- teammate / swarm worker

这也是权限系统和 agent team 体系真正接上的地方。

---

### 现在把权限主干读通后的脑图写出来

到当前为止，可以把权限系统主干总结成：

```text
permissions.ts
  -> 产出共享 allow / deny / ask 裁决
useCanUseTool.tsx
  -> 根据当前运行时角色选择后续处理路径
PermissionContext.ts
  -> 提供日志/持久化/queue/abort/hook/classifier 工具箱
coordinatorHandler.ts
  -> coordinator: 先自动化再打断用户
interactiveHandler.ts
  -> main agent: 多来源并发竞态审批
swarmWorkerHandler.ts
  -> worker: 转发给 leader mailbox 审批
```

到这里，这条权限主线已经非常完整了。

---

### 第一轮阅读时，不要陷进去的点

先不要深挖：

- `permissionSync.ts` 的 mailbox 协议细节
- leader 端具体如何轮询和消费 callback
- `pendingWorkerRequest` 最终在 UI 哪个组件里展示

这一轮先抓住：

```text
swarmWorkerHandler.ts = swarm worker 的权限上行代理层
```

以及：

```text
worker ask 的关键不是本地交互，而是把审批 authority 上送给 leader，再把 leader 的回应翻译回标准 permission decision
```

就够了。

---

### 下一站建议

下一站最顺的是：

```text
src/utils/swarm/permissionSync.ts
```

因为现在你已经看到 worker 侧如何“发出并等待”权限请求，下一步最自然就是看：

**leader / mailbox 之间的权限同步协议到底长什么样，请求对象如何构造、发送、消费和回传。**

---

## 第 15 站：`src/utils/swarm/permissionSync.ts`

### 这是什么文件

`src/utils/swarm/permissionSync.ts` 是 swarm 权限系统里的**同步协议层**。

上一站 `swarmWorkerHandler.ts` 看到的是 worker 侧逻辑：

- 什么时候要上送 leader
- 怎样注册 callback
- 怎样等待 leader 回复

而这一站回答的是更底层的问题：

```text
worker 和 leader 之间，权限请求到底用什么数据结构表示、落在哪里、怎样传输、怎样清理
```

所以最准确的定位是：

```text
permissionSync.ts = swarm 权限审批的协议与存储层
```

---

### 文件头注释先给了总图

位置：`src/utils/swarm/permissionSync.ts:1`

这段注释已经明确了核心模型：

1. worker 遇到 permission prompt
2. worker 发 `permission_request` 给 leader mailbox
3. leader 轮询 mailbox 看见请求
4. 用户在 leader UI 上批准/拒绝
5. leader 发 `permission_response` 回 worker mailbox
6. worker 轮询 / 监听 mailbox 后继续执行

所以这一层解决的不是权限判定，而是：

```text
跨 agent 的 permission request / response 通信协议
```

---

### 第一部分：请求对象长什么样

#### `SwarmPermissionRequestSchema`
位置：`src/utils/swarm/permissionSync.ts:49`

这里定义了一份很完整的请求结构，里面包含：

- `id`
- `workerId` / `workerName` / `workerColor`
- `teamName`
- `toolName`
- `toolUseId`
- `description`
- `input`
- `permissionSuggestions`
- `status`
- `resolvedBy`
- `resolvedAt`
- `feedback`
- `updatedInput`
- `permissionUpdates`
- `createdAt`

这说明 swarm 权限请求不是一个临时聊天消息，而是一份**可持久化、可流转、可回写结果**的正式协议对象。

尤其要注意：

```text
一个请求对象里同时容纳了“请求期字段”和“解决后字段”
```

也就是说，同一份对象会经历：

- pending
- approved / rejected

两个阶段，而不是拆成两个完全不同的数据结构。

---

### 第二部分：权限系统先有一套文件系统落地模型

位置：`src/utils/swarm/permissionSync.ts:108`

这里先定义了三层目录概念：

- `getPermissionDir(teamName)`
- `getPendingDir(teamName)`
- `getResolvedDir(teamName)`

路径结构是：

```text
~/.claude/teams/{teamName}/permissions/
  pending/
  resolved/
```

再通过 `ensurePermissionDirsAsync(...)` 保证目录存在。

这说明即使后来引入了 mailbox，这个子系统的根仍然是：

```text
权限请求可以被表示为团队目录下的持久化文件状态机
```

也就是：

- pending 目录 = 待审批
- resolved 目录 = 已审批

这是一种非常直观的磁盘级队列模型。

---

### 第三部分：请求如何创建

#### `generateRequestId()`
位置：`src/utils/swarm/permissionSync.ts:160`

#### `createPermissionRequest(...)`
位置：`src/utils/swarm/permissionSync.ts:167`

`createPermissionRequest(...)` 会自动补齐：

- `teamName`
- `workerId`
- `workerName`
- `workerColor`
- `status: 'pending'`
- `createdAt`

如果关键身份缺失，会直接报错。

这说明 permission request 不是随手拼对象，而是有一个明确的构造入口，用来保证：

```text
每个 worker 发出去的请求都带有足够的路由与身份信息
```

这也是为什么上层 handler 不需要自己关心 team / agent 元数据的完整拼装。

---

### 第四部分：旧文件系统方案是怎样工作的

#### `writePermissionRequest(...)`
位置：`src/utils/swarm/permissionSync.ts:215`

这里会：

1. 确保目录存在
2. 算出 `pending/{requestId}.json`
3. 在目录里建 `.lock`
4. 用 `lockfile.lock(...)` 做目录级锁
5. 写入 JSON 文件

这说明旧方案的重点是：

```text
把权限请求作为一个原子写入的 pending 文件
```

而不是直接依赖内存态。

#### `readPendingPermissions(...)`
位置：`src/utils/swarm/permissionSync.ts:256`

leader 侧可以把 pending 目录下的所有 JSON 读出来，过 schema 校验，再按 `createdAt` 排序。

这就形成了一个很清晰的 leader 审批视图：

```text
读 pending 目录 -> 得到按时间排序的待处理请求列表
```

#### `resolvePermission(...)`
位置：`src/utils/swarm/permissionSync.ts:360`

解决请求时会：

1. 上锁
2. 读取 pending 文件
3. 补齐 resolution 数据
4. 写到 resolved 目录
5. 删除 pending 文件

这是一套标准的文件状态迁移：

```text
pending -> resolved
```

所以旧方案的本质就是：

```text
基于文件系统目录迁移的权限审批状态机
```

---

### 第五部分：为什么还能看到 `pollForResponse()` 等旧接口

位置：`src/utils/swarm/permissionSync.ts:520`

这里还有：

- `PermissionResponse`
- `pollForResponse(...)`
- `removeWorkerResponse(...)`
- `submitPermissionRequest = writePermissionRequest`

这些都带有明显的 backward compatibility 意味道。

说明这个模块并不是从一开始就是 mailbox-only，而是经历过演进：

```text
文件系统 polling 方案 -> mailbox 方案
```

而当前代码为了兼容旧 worker integration，保留了一批旧接口。

这也是读这个文件时要抓住的一个关键事实：

```text
permissionSync.ts 其实是一个“新旧协议并存”的过渡层
```

---

### 第六部分：mailbox 方案才是现在的新主线

位置：`src/utils/swarm/permissionSync.ts:643`

这里直接写了：

```text
Mailbox-Based Permission System
```

这就是当前更重要的部分。

---

### `isTeamLeader()` / `isSwarmWorker()` 在这里定义了角色边界

位置：
- `isTeamLeader()` `src/utils/swarm/permissionSync.ts:581`
- `isSwarmWorker()` `src/utils/swarm/permissionSync.ts:596`

这里的判断逻辑很直接：

- 有 teamName
- 有 agentId
- 且不是 team leader

就算 worker。

所以这个模块不仅存协议，还定义了：

```text
谁负责发请求，谁负责收请求
```

也就是 swarm 权限同步的角色判定基础。

---

### 第七部分：mailbox 请求怎样发给 leader

#### `getLeaderName(...)`
位置：`src/utils/swarm/permissionSync.ts:651`

它会读 team file，找到 `leadAgentId` 对应成员名。

这很关键，因为 mailbox 路由是按名字发的，不只是按 teamName。

#### `sendPermissionRequestViaMailbox(...)`
位置：`src/utils/swarm/permissionSync.ts:676`

这里的流程是：

1. 找 leader name
2. 调 `createPermissionRequestMessage(...)`
3. 通过 `writeToMailbox(...)` 发给 leader
4. message 里带：
   - `request_id`
   - `agent_id`（workerName）
   - `tool_name`
   - `tool_use_id`
   - `description`
   - `input`
   - `permission_suggestions`

这里最值得记住的是：

```text
mailbox 不是直接发原始 request JSON 文件，而是发一条标准化 teammate mailbox message
```

也就是说，这层做了一次协议翻译：

```text
SwarmPermissionRequest 对象
  -> teammateMailbox message
  -> writeToMailbox(...)
```

这解释了为什么上层 handler 只用 `sendPermissionRequestViaMailbox(request)` 就够了。

---

### 第八部分：mailbox 响应怎样发回 worker

#### `sendPermissionResponseViaMailbox(...)`
位置：`src/utils/swarm/permissionSync.ts:734`

流程和请求相反：

1. 构造 `createPermissionResponseMessage(...)`
2. 根据 `resolution.decision` 选 `success` / `error`
3. 填入：
   - `request_id`
   - `error` / `feedback`
   - `updated_input`
   - `permission_updates`
4. 用 `writeToMailbox(...)` 发回 workerName

所以 leader -> worker 的返回也不是 ad hoc 文本，而是一条结构化 mailbox 消息。

这说明 mailbox 方案的本质是：

```text
请求和响应两端都走统一的 teammate mailbox envelope
```

因此可以天然兼容：

- in-process 路由
- file-based mailbox
- 其他底层传递实现

注释里也明确写了：

```text
routes to in-process or file-based based on recipient
```

这就是协议层和传输层分离的体现。

---

### 第九部分：sandbox 权限复用了同一条 mailbox 思路

位置：
- `sendSandboxPermissionRequestViaMailbox(...)` `src/utils/swarm/permissionSync.ts:805`
- `sendSandboxPermissionResponseViaMailbox(...)` `src/utils/swarm/permissionSync.ts:882`

这部分虽然是 sandbox network access 的专门分支，但它很能说明架构思路。

它把“worker 向 leader 请求网络访问批准”也建模成同一类 mailbox request/response。

这说明 mailbox 权限协议不是只服务 tool permission，而是更通用的：

```text
任何需要 worker -> leader 批准往返的权限请求，都可以复用这套 mailbox 模式
```

也就是说，这里已经开始从“工具权限同步”长成“团队权限同步总线”。

---

### 元

问题：**这一站真正想解决的架构问题是什么？**

回答：这一站把 worker 与 leader 之间的权限 ask 变成正式协议，包括请求结构、响应结构、传输路径和清理方式。它说明 swarm 权限不是随手发消息，而是一份可持久化、可回写状态的事务对象。

### 反

问题：**如果把这一站的设计反过来，会发生什么？**

回答：如果跨 agent 权限只是自由文本消息，回包匹配、状态恢复和失败清理都会极不稳固。协议一旦不正式，就无法支撑可靠恢复。

### 空

问题：**跳出当前文件名，这一站背后更大的问题是什么？**

回答：更大的问题是，分布式 agent 之间如何传递一个“尚未完成的审批事务”。权限同步本质上是把人机确认转译成跨执行体协议。


### 读完这个文件后，你应该抓住的 7 个事实

#### 事实 1：`permissionSync.ts` 是协议层，不是 UI 层
它定义请求/响应结构、角色判断、目录布局、mailbox 消息封装。

#### 事实 2：这个模块处在新旧方案过渡期
旧的 pending/resolved 文件方案还在，新的 mailbox 方案已经成为主线。

#### 事实 3：旧方案本质是文件系统状态迁移
`pending/` 到 `resolved/` 构成一个磁盘上的审批状态机。

#### 事实 4：新方案本质是 teammate mailbox request/response
worker 和 leader 之间通过结构化 mailbox 消息传递权限请求与结果。

#### 事实 5：mailbox 方案不是替换语义，而是替换传输方式
请求对象语义还在，只是从目录轮询切到 mailbox envelope + routing。

#### 事实 6：leader / worker 角色边界也在这个模块里定义
`isTeamLeader()` 和 `isSwarmWorker()` 是上层 handler 选择路径的重要基础。

#### 事实 7：sandbox 权限也复用了这套模式
说明这套同步机制已经具备更通用的团队权限协调能力。

---

### 现在把 worker 权限链完整串起来

到这里，swarm worker 侧权限链已经能完整写成：

```text
swarmWorkerHandler.ts
  -> createPermissionRequest(...)
  -> sendPermissionRequestViaMailbox(...)
permissionSync.ts
  -> 把 request 翻译成 mailbox message 发给 leader
leader 侧
  -> 收到 request 后做审批
permissionSync.ts
  -> sendPermissionResponseViaMailbox(...)
worker 侧
  -> callback / poller 消费 response
  -> 翻译回标准 PermissionDecision
```

这样你就不再只知道“worker 会发请求”，而是知道它底下具体依赖的协议层了。

---

### 现在把权限系统的团队协作面总结一下

到这一站为止，权限系统已经不只是本地 allow/deny/ask，而是至少包含这几层：

```text
本地规则裁决
  -> permissions.ts
本地运行时适配
  -> useCanUseTool.tsx
本地 ask 流控制
  -> interactive/coordinator/swarm handlers
团队权限同步协议
  -> permissionSync.ts
底层消息传输
  -> teammateMailbox / writeToMailbox
```

也就是说，Claude Code 的权限系统已经延伸成：

```text
单 agent 审批 + 多 agent 协调审批
```

这正是 agent team 架构和 permission system 真正交叉的地方。

---

### 第一轮阅读时，不要陷进去的点

先不要深挖这些：

- `teammateMailbox.ts` 的具体 envelope 格式实现
- leader 端 UI 如何展示 mailbox request
- 各种 lockfile 边界条件与目录清理细节
- sandbox permission 的更深 runtime 接入点

这一轮先抓住：

```text
permissionSync.ts = swarm 权限审批的协议与存储层
```

以及：

```text
它同时保留了旧的文件状态机方案，并引入了新的 mailbox request/response 方案来完成 worker 与 leader 之间的权限同步
```

就够了。

---

### 下一站建议

下一站最顺的是：

```text
src/utils/teammateMailbox.ts
```

因为现在协议层已经看到了，下一步最自然就是继续下探：

**这些 permission request/response 被包装成什么 mailbox message，mailbox 本身如何路由到 in-process 或文件式收件箱。**

---

## 第 16 站：`src/utils/teammateMailbox.ts`

### 这是什么文件

`src/utils/teammateMailbox.ts` 是 agent swarm 通信里的**邮箱基础设施层**。

上一站 `permissionSync.ts` 已经看到了：

- worker / leader 权限请求会走 mailbox
- `sendPermissionRequestViaMailbox(...)`
- `sendPermissionResponseViaMailbox(...)`

但那一站更偏“权限协议”。

这一站回答的是更底层的问题：

```text
mailbox 自己是什么、消息文件怎么存、结构化消息怎么编码、各种 swarm 事件怎样共享这条基础设施
```

所以最准确的定位是：

```text
teammateMailbox.ts = agent team 的通用消息总线基础层
```

---

### 文件头注释已经说透了它的物理模型

位置：`src/utils/teammateMailbox.ts:1`

这里直接写明：

- 每个 teammate 有一个 inbox 文件
- 路径在 `.claude/teams/{team_name}/inboxes/{agent_name}.json`
- 其他 teammate 往这个文件写消息
- 收件人把这些消息当 attachments 看

所以 mailbox 在最底层并不神秘，本质上就是：

```text
每个 agent 一个 JSON inbox 文件
```

这点很重要，因为它解释了为什么很多上层 swarm 功能看起来像“实时消息”，但其实可以同时兼容：

- 文件式收件箱
- in-process 路由
- 其他后端

mailbox 这里提供的是**统一抽象**，而不是只等于某一个 transport。

---

### 第一部分：最基础的数据结构

#### `TeammateMessage`
位置：`src/utils/teammateMailbox.ts:43`

基础消息结构很简单：

- `from`
- `text`
- `timestamp`
- `read`
- `color?`
- `summary?`

这说明 mailbox 的底层 envelope 很薄。

真正的业务语义（permission request、shutdown、plan approval 等）主要都塞在：

```text
text 里的结构化 JSON
```

也就是说，mailbox 把两层分开了：

- 外层：谁发的、什么时候发、是否已读、预览信息
- 内层：这条消息到底是什么业务类型

这是一种非常典型的 envelope + payload 设计。

---

### 第二部分：邮箱文件路径和目录管理

#### `getInboxPath(...)`
位置：`src/utils/teammateMailbox.ts:56`

这里会：

- 取 `teamName`
- `sanitizePathComponent(...)`
- 路径落到 `getTeamsDir()/team/inboxes/agent.json`

这说明 mailbox 路由在磁盘层面是按：

```text
team + agentName
```

来定位的，不是按 agent UUID。

这和前面 `permissionSync.ts` 里 `getLeaderName()` 的逻辑也正好对上：

- mailbox 的收件人标识是名字
- 所以先要把 leader / worker 的名字解析出来

#### `ensureInboxDir(...)`
位置：`src/utils/teammateMailbox.ts:71`

就是确保 inbox 目录存在。

虽然简单，但这再次说明 mailbox 的根基是**文件系统 inbox 目录**。

---

### 第三部分：最核心的三个文件操作

#### `readMailbox(...)`
位置：`src/utils/teammateMailbox.ts:84`

它会读取整个 inbox JSON 数组。

读不到文件时返回空数组，而不是报错。

这说明 mailbox 在设计上默认支持“延迟创建 / 按需出现”的收件箱。

#### `writeToMailbox(...)`
位置：`src/utils/teammateMailbox.ts:134`

这是全文件最关键的基础函数之一。

它会：

1. 确保 inbox 目录存在
2. 确保 inbox 文件存在（必要时创建 `[]`）
3. 对 inbox 文件加锁
4. 加锁后重新读最新消息
5. push 新消息
6. 覆盖写回 JSON

这里非常关键的设计点是：

```text
先锁，再重新读，再写回
```

这就是典型的并发安全追加模式。

因为 swarm 里可能多个 agent 同时给一个 inbox 发消息，如果只是“先读后锁”就可能丢消息。

所以这一层其实承担的是：

```text
并发安全的 mailbox append 日志
```

#### `markMessageAsReadByIndex(...)` / `markMessagesAsRead(...)`
位置：
- `src/utils/teammateMailbox.ts:201`
- `src/utils/teammateMailbox.ts:279`

它们说明 inbox 不是只写不管，而是有显式的读状态管理。

也就是说，mailbox 不是 event stream 风格，而更像：

```text
可持久保留、可标已读的收件箱模型
```

---

### 第四部分：`formatTeammateMessages(...)` 说明消息最终怎样进入上层

位置：`src/utils/teammateMailbox.ts:373`

这个函数把消息格式化成 XML：

- 使用 `TEAMMATE_MESSAGE_TAG`
- 带 `teammate_id`
- 可带 `color`
- 可带 `summary`

这说明 inbox 消息最终不是直接裸 JSON 暴露给模型 / UI，而是会被转成一套统一附件表示。

所以这里你要记住：

```text
mailbox 的底层存储是 JSON
但进入上层展示/上下文时，会转成 XML-like attachment 片段
```

这也是为什么文件头注释里说“recipient sees them as attachments”。

---

### 第五部分：权限消息在这里被正式定义成 mailbox payload

#### `PermissionRequestMessage`
位置：`src/utils/teammateMailbox.ts:453`

字段是：

- `type: 'permission_request'`
- `request_id`
- `agent_id`
- `tool_name`
- `tool_use_id`
- `description`
- `input`
- `permission_suggestions`

#### `PermissionResponseMessage`
位置：`src/utils/teammateMailbox.ts:468`

分两种：

- `subtype: 'success'`
  - `updated_input`
  - `permission_updates`
- `subtype: 'error'`
  - `error`

这里非常关键，因为它说明上一站 `permissionSync.ts` 所说的“mailbox request/response”并不是抽象概念，而是在这里被落实成：

```text
一套明确的 JSON payload schema
```

而且注释里还说：

- 请求字段和 SDK `can_use_tool` 对齐（snake_case）
- 响应镜像 SDK ControlResponse / ControlErrorResponse 结构

所以这一层其实还承担了一个作用：

```text
把 swarm mailbox 协议尽量对齐到 SDK / control-plane 语义
```

这会让不同子系统之间更容易复用思维模型。

---

### 对应的 create / parse 函数说明了完整消息生命周期

位置：
- `createPermissionRequestMessage(...)` `src/utils/teammateMailbox.ts:488`
- `createPermissionResponseMessage(...)` `src/utils/teammateMailbox.ts:512`
- `isPermissionRequest(...)` `src/utils/teammateMailbox.ts:541`
- `isPermissionResponse(...)` `src/utils/teammateMailbox.ts:558`

这四类函数正好对应完整生命周期：

```text
业务对象 -> 创建消息 -> 写进 mailbox -> 读取后解析消息
```

所以 mailbox 这层不只是“存文件”，而是同时提供：

- message construction
- message detection / parsing

也就是一个完整的消息边界层。

---

### 第六部分：sandbox permission 复用同一套 mailbox 模式

位置：`src/utils/teammateMailbox.ts:573` 之后

这里定义了：

- `SandboxPermissionRequestMessage`
- `SandboxPermissionResponseMessage`
- create / parse 函数

这和上一站 `permissionSync.ts` 的结论一致：

```text
mailbox 不只是承载 tool permission
它已经是更广义的 worker <-> leader 权限同步消息通道
```

也就是说，tool permission 和 sandbox permission 在 mailbox 层共享同一套 message-bus 思路。

---

### 第七部分：你会发现 mailbox 远不止 permission 一种业务

读到中段以后能看到很多别的消息类型：

- `idle_notification` `src/utils/teammateMailbox.ts:394`
- `plan_approval_request` / `plan_approval_response` `src/utils/teammateMailbox.ts:684`
- `shutdown_request` / `shutdown_approved` / `shutdown_rejected` `src/utils/teammateMailbox.ts:720`
- `task_assignment` `src/utils/teammateMailbox.ts:953`
- `team_permission_update` `src/utils/teammateMailbox.ts:983`
- `mode_set_request` `src/utils/teammateMailbox.ts:1019`

这说明 mailbox 绝不是“权限模块的私有工具”，而是：

```text
整个 agent team 协作的通用控制平面消息总线
```

权限同步只是其中一个消息族。

这一点非常重要，因为它会改变你对 swarm 架构的理解：

- swarm 不是若干 ad hoc helper 拼起来
- 而是很多协作能力共享一套 mailbox 通信底座

---

### 第八部分：`PermissionModeSchema` / `BackendType` 这些 import 暗示什么

在这文件里还能看到：

- `PermissionModeSchema`
- `BackendType`
- `SEND_MESSAGE_TOOL_NAME`

即使当前片段还没把所有用法完全展开，它们已经在暗示一个事实：

```text
mailbox 消息不只是“聊天”
它承载的是运行时控制信号、权限模式同步、后端信息交换
```

换句话说，mailbox 更接近：

```text
swarm control plane transport
```

而不仅是一个 inbox 工具函数集合。

---

### 元

问题：**这一站真正想解决的架构问题是什么？**

回答：这一站给出 swarm 消息系统最底层的物理模型：每个 teammate 一个 inbox JSON 文件。外层是通用信封，内层再承载 permission、shutdown、plan approval 等结构化 payload。

### 反

问题：**如果把这一站的设计反过来，会发生什么？**

回答：如果没有统一 mailbox 底座，各类 swarm 事件会各自发明传输方式。上层看似灵活，底层却很难共享读写、加锁和协议识别能力。

### 空

问题：**跳出当前文件名，这一站背后更大的问题是什么？**

回答：更大的问题是，多 agent 通信如何既简单到能落盘，又足够通用去承载多种业务消息。mailbox 的设计就是一种轻量总线答案。


### 读完这个文件后，你应该抓住的 7 个事实

#### 事实 1：mailbox 的物理实现是 team/agent 维度的 inbox JSON 文件
每个 agent 一个 inbox，消息是 JSON 数组项。

#### 事实 2：`writeToMailbox(...)` 是最核心的基础设施函数
它通过锁 + 重读 + 追加写回，保证并发写入安全。

#### 事实 3：mailbox 用 envelope + payload 模式
外层 `TeammateMessage` 只管 from/text/timestamp/read；具体业务语义塞在 `text` JSON 里。

#### 事实 4：mailbox 消息最终会被格式化成 XML-like attachments
所以它既是存储层，也是进入上层上下文的桥。

#### 事实 5：permission request/response 在这里被正式定义成标准 payload schema
这就是上一站 permissionSync mailbox 方案真正依赖的消息边界。

#### 事实 6：sandbox permission、plan approval、shutdown、task assignment 都复用了同一套 mailbox
说明它是 agent team 的通用控制平面总线，而不是单一功能模块。

#### 事实 7：mailbox 是 swarm 架构里的共享底座
很多上层协作特性之所以能成立，是因为它们共享了这条低层消息通道。

---

### 现在把权限同步链和 mailbox 底座串起来

到这里可以把上一站的链路继续往下展开：

```text
swarmWorkerHandler.ts
  -> createPermissionRequest(...)
permissionSync.ts
  -> createPermissionRequestMessage(...)
  -> writeToMailbox(leaderName, ...)
teammateMailbox.ts
  -> 把结构化 permission_request 写进 leader inbox
leader 侧处理后
permissionSync.ts
  -> createPermissionResponseMessage(...)
  -> writeToMailbox(workerName, ...)
teammateMailbox.ts
  -> 把结构化 permission_response 写回 worker inbox
```

这样 worker <-> leader 权限同步这条线就真正打到底了。

---

### 现在你应该怎样重新理解 agent team 架构

到当前为止，至少可以看出这几个分层：

```text
高层协作语义
  -> 权限审批 / plan approval / shutdown / task assignment / idle 通知
协议层
  -> createXMessage / isXMessage
mailbox 存储与收发层
  -> inbox JSON + locking + read/unread 管理
展示/上下文接入层
  -> XML-like teammate attachments
```

这说明 Claude Code 的 swarm 系统已经有点像一个轻量级分布式控制面：

- 有消息总线
- 有结构化协议
- 有角色分工
- 有持久 inbox
- 有上层控制事件

这比“多个 agent 可以互发消息”要深得多。

---

### 第一轮阅读时，不要陷进去的点

先不要深挖这些：

- 所有消息类型的完整业务流
- `ModeSetRequest` / `Shutdown` 在 UI 与后端的全部处理细节
- inbox 消息最终何时被转成 attachment 注入 prompt

这一轮先抓住：

```text
teammateMailbox.ts = agent team 的通用消息总线基础层
```

以及：

```text
permissionSync.ts 依赖它来把权限请求/响应翻译成标准 mailbox payload，再交给 inbox 文件或其他路由后端传输
```

就够了。

---

### 下一站建议

下一站最顺的是：

```text
src/hooks/useInboxPoller.ts
```

因为现在你已经看到了 mailbox 的写入面，下一步最自然就是看：

**leader / teammate 这一侧是如何轮询 inbox、识别不同消息类型，并把 permission response、idle notification、task assignment 等事件重新接回运行时状态的。**


---

## 第 17 站：`src/hooks/useInboxPoller.ts`

### 这是什么文件

`src/hooks/useInboxPoller.ts` 是 agent team mailbox 的**运行时消费器**。

如果前面两站回答的是：

- `src/utils/swarm/permissionSync.ts`：权限请求/响应协议如何生成并通过 mailbox 发送
- `src/utils/teammateMailbox.ts`：消息怎样写入 inbox、读取 unread、标记 read、识别 payload 类型

那么这一站回答的是：

```text
mailbox 里的消息在运行时里由谁轮询、怎么分类、怎样重新接回权限系统、team 状态和 query turn
```

所以一句话先记：

```text
useInboxPoller.ts = team mailbox 的消费与分发层
```

它不是“读一下 inbox”这么简单，而是承担了多条运行时回流链路：

- 权限请求 -> leader 权限 UI
- 权限响应 -> worker callback 恢复
- mode / permission update -> 本地 permission context
- shutdown -> pane / task / teammate 清理
- 普通队友消息 -> 重新包装成新一轮 turn 输入

---

### 先看两个最重要的入口

#### 1. `getAgentNameToPoll(appState)`
位置：`src/hooks/useInboxPoller.ts:81`

这个函数先解决一个很基础但关键的问题：

```text
当前这个运行实例，到底应该替谁去轮询 inbox？
```

源码把场景分得很清楚：

- **in-process teammate**：返回 `undefined`
- **process-based teammate**：轮询自己的 agent name
- **team lead**：轮询 leader 自己的名字
- **standalone session**：返回 `undefined`

这里最重要的是第一条。

源码注释明确说明：in-process teammate 不该使用 `useInboxPoller`，因为它们和 leader 共用 React context / AppState，真正的消息等待是 `waitForNextPromptOrShutdown()` 那条专用路径。

这说明：

```text
mailbox 消费层从一开始就区分 in-process 与 out-of-process teammate
```

这和前面 swarm backend / teammate runner 的分层是一致的。

---

#### 2. `useInboxPoller(...)`
位置：`src/hooks/useInboxPoller.ts:126`

这是主 hook。它本质上做三件事：

1. 每秒轮询 unread mailbox messages
2. 按消息协议类型分发到不同运行时处理器
3. 在 session 空闲时把普通消息重新提交成新一轮 turn

可以把它压缩成一句话：

```text
mailbox -> runtime actions / AppState / new turn
```

所以它不是普通 UI hook，而是 **agent team 控制平面重新接回 runtime 的桥**。

---

### 先抓主链路

把 `poll()` 粗略展开，可以得到这张脑图：

```text
poll()
  -> getAgentNameToPoll(...)
  -> readUnreadMessages(...)
  -> teammate 处于 plan mode 时先处理 plan approval response
  -> 按协议类型拆分 unread messages
  -> permission request: leader 侧进 ToolUseConfirmQueue
  -> permission response: worker 侧触发已注册 callback
  -> sandbox request/response: 更新 sandbox 审批状态
  -> team permission update: 改 permission context
  -> mode set request: 改 mode 并回写 team file
  -> plan approval request: leader 自动批准并回信
  -> shutdown request/approval: 更新 pane / task / teamContext / inbox
  -> regular messages: XML 包装后立即提交或入队
  -> 成功投递/可靠入队后 markMessagesAsRead(...)
```

这条链路很关键，因为它把前面读到的 mailbox、permission callback、team state、message reinjection 真正接成了闭环。

---

### 关键职责 1：轮询 unread mailbox
关键位置：`src/hooks/useInboxPoller.ts:107`, `src/hooks/useInboxPoller.ts:139-152`, `src/hooks/useInboxPoller.ts:952-968`

最核心的读取动作是：

```ts
const unread = await readUnreadMessages(
  agentName,
  currentAppState.teamContext?.teamName,
)
```

配套还有两点：

- `INBOX_POLL_INTERVAL_MS = 1000`
- mount 时还会做一次 initial poll

所以这里的模型是：

```text
固定 1s poll + 启动即补读一次
```

目的很明确：既保证及时性，也避免刚启动时必须等一个完整 polling 周期。

---

### 关键职责 2：统一 transport，按协议类型分流
关键位置：`src/hooks/useInboxPoller.ts:204-248`

这里把 unread messages 分成：

- `permissionRequests`
- `permissionResponses`
- `sandboxPermissionRequests`
- `sandboxPermissionResponses`
- `shutdownRequests`
- `shutdownApprovals`
- `teamPermissionUpdates`
- `modeSetRequests`
- `planApprovalRequests`
- `regularMessages`

这个拆分很重要，因为它说明：

```text
teammateMailbox.ts 提供的是统一 transport，
真正把消息接进具体业务通道的是 useInboxPoller.ts
```

也就是说，mailbox 自身只是总线；消费侧才完成“协议解释”。

---

### 关键职责 3：leader 侧把 worker 权限请求接回标准 ToolUseConfirm UI
关键位置：`src/hooks/useInboxPoller.ts:250-364`

这是本文件最关键的一段之一。

leader 收到 `permissionRequests` 后，不是走一套简化版审批逻辑，而是：

- `getLeaderToolUseConfirmQueue()` 取 leader 侧标准权限队列
- `findToolByName(getAllBaseTools(), parsed.tool_name)` 找真实工具定义
- 构造标准 `ToolUseConfirm`
- `onAllow / onReject / onAbort` 再通过 `sendPermissionResponseViaMailbox(...)` 回给 worker

这里最值得记住的结论是：

```text
out-of-process worker 的权限审批，
最终复用了 leader 本地已有的标准权限 UI 管道
```

也就是说 mailbox 负责跨进程运输，但审批体验本身仍然统一落在已有的 ToolUseConfirm 体系里。

---

### 关键职责 4：worker 侧收到响应后恢复挂起回调
关键位置：`src/hooks/useInboxPoller.ts:366-495`

worker 侧对两类响应做处理：

- `permissionResponses`
- `sandboxPermissionResponses`

逻辑是：

- 先看是否存在已注册 callback：`hasPermissionCallback(...)` / `hasSandboxPermissionCallback(...)`
- 如果存在，再调用：
  - `processMailboxPermissionResponse(...)`
  - `processSandboxPermissionResponse(...)`

这就把前面 `swarmWorkerHandler.ts` 注册 callback 的另一半补齐了。

所以完整模型其实是：

```text
worker ask
  -> register callback
  -> mailbox 发给 leader
  -> leader 审批
  -> mailbox 响应返回
  -> callback 恢复等待中的 promise
```

这就是 swarm worker 权限 ask 的异步闭环。

---

### 关键职责 5：同步 team-level 权限规则和 mode
关键位置：
- `src/hooks/useInboxPoller.ts:497-547`
- `src/hooks/useInboxPoller.ts:549-597`

这里实际上做了两种控制面同步：

#### A. `teamPermissionUpdates`
- 解析 `permissionUpdate.rules` / `behavior`
- 通过 `applyPermissionUpdate(...)` 更新本地 `toolPermissionContext`
- destination 固定写入 `session`

#### B. `modeSetRequests`
- 只接受来自 `team-lead` 的请求
- `permissionModeFromString(...)` 解析 mode
- 更新本地 mode
- `setMemberMode(teamName, agentName, targetMode)` 回写 team file

这里的核心认识是：

```text
leader 不只是审批单次 tool_use，
还可以通过 mailbox 下发持续生效的 team policy
```

所以 inbox poller 也是 team policy 的同步器。

---

### 关键职责 6：plan approval 与 shutdown 是 teammate 生命周期控制信号
关键位置：
- `src/hooks/useInboxPoller.ts:156-196`
- `src/hooks/useInboxPoller.ts:599-660`
- `src/hooks/useInboxPoller.ts:664-800`

这块要分开记：

#### plan approval response（teammate 侧）
如果 teammate 正处于 plan mode，且收到来自 `team-lead` 的批准响应，就退出 plan mode，并切换到 leader 给定的 permission mode。

这说明 plan approval 不是普通文本消息，而是：

```text
teammate 运行模式迁移信号
```

#### plan approval request（leader 侧）
leader 收到计划审批请求后会自动批准，并把 response 写回 teammate inbox；若是 in-process teammate，还会同步更新任务状态。

#### shutdown approval（leader 侧）
这段非常重，不只是显示消息。它还会：

- 如有 `paneId` / `backendType` 就尝试 kill pane
- `removeTeammateFromTeamFile(...)`
- `unassignTeammateTasks(...)`
- 从 `teamContext.teammates` 移除成员
- 将对应 teammate task 标记为 completed
- 往 inbox 追加 `teammate_terminated` system message

所以真正的结论是：

```text
shutdown approval = teammate 生命周期收尾点
```

这说明 useInboxPoller 不只是消息消费器，还是部分 team runtime 清理器。

---

### 关键职责 7：普通队友消息会重新进入模型消息流
关键位置：`src/hooks/useInboxPoller.ts:802-864`

普通消息最终会被包装成 XML：

```ts
<teammate_message teammate_id="..." color="..." summary="...">
...
</teammate_message>
```

然后：

- **session idle**：直接 `onSubmitTeammateMessage(formatted)` 提交为新一轮 turn
- **session busy**：先写入 `AppState.inbox.messages`，稍后再投递

这说明 mailbox 消息最终不是只给 UI 展示，而是会重新喂给模型：

```text
teammate message -> XML wrapper -> new turn input
```

所以 `useInboxPoller.ts` 其实就是 agent team 和 query loop 之间的再注入层。

---

### idle delivery 与可靠性语义
关键位置：`src/hooks/useInboxPoller.ts:875-968`

文件后半段还有一个 `useEffect(...)`，专门在 session 从 busy 变 idle 时：

- 清理 `processed` messages
- 找出 `pending` messages
- 再包装成 XML
- 再次尝试提交
- 只有提交成功才移除对应消息

这里体现出两个非常重要的设计：

#### 1. `AppState.inbox.messages` 是可靠排队区
mailbox unread 一旦成功进入 AppState，就不再依赖文件 unread 状态，而是在内存队列里继续保证投递。

#### 2. `markMessagesAsRead(...)` 被刻意延后
源码明确写明：

```text
只有消息被成功提交或可靠入队后，才标记为已读
```

这就是在防止：

```text
先标 read，后崩溃 -> 消息永久丢失
```

所以这个文件虽然叫 poller，但实际上包含了一层轻量的可靠投递语义。

---

### 这个文件体现出的 8 个架构事实

1. `useInboxPoller.ts` 是 mailbox 的运行时消费与分发层，不是存储层。
2. 它从一开始就区分 in-process teammate 与 out-of-process teammate 的收信方式。
3. mailbox 使用统一 transport，消费时再按权限、sandbox、shutdown、plan、mode 等协议分流。
4. leader 侧 worker 权限审批复用了标准 `ToolUseConfirm` UI 队列。
5. worker 侧靠 callback 注册与响应恢复完成异步 ask 闭环。
6. 普通 teammate 消息最终会重新进入模型消息流，而不是停留在 UI。
7. shutdown / plan approval / mode set 都是运行时控制信号，不只是“通知文本”。
8. 已读标记被延后到成功投递或可靠入队之后，这是它的可靠性关键点。

---

### 现在把第 15-17 站串起来

到这里，权限同步与 mailbox runtime 链路已经闭环了：

```text
swarmWorkerHandler.ts
  -> createPermissionRequest(...)
  -> registerPermissionCallback(...)
  -> sendPermissionRequestViaMailbox(...)

permissionSync.ts
  -> 定义 request/response 协议
  -> 调 mailbox 发送辅助函数

teammateMailbox.ts
  -> inbox JSON 落盘
  -> unread/read 管理
  -> 结构化 payload 边界

useInboxPoller.ts
  -> 轮询 unread messages
  -> 分类后接入 leader UI / worker callback / AppState / query turn
  -> 必要时同步 permission / mode / shutdown / task 状态
```

所以现在应该非常清楚：

```text
mailbox 不是附属功能，
而是 Claude Code agent team 控制平面的核心通信总线
```

---

### 这一站最该记住的 8 句话

1. `useInboxPoller.ts` 是 mailbox 的运行时消费器和分发器。
2. 它先决定当前实例到底代表谁轮询 inbox。
3. 它把统一 mailbox 消息拆成权限、sandbox、mode、plan、shutdown、regular 等多个协议分支。
4. leader 收到 worker 权限请求后，会把它转成标准 `ToolUseConfirm` 进入已有权限 UI。
5. worker 收到 leader 响应后，通过已注册 callback 恢复等待中的 ask 请求。
6. 普通 teammate 消息最终会包装成 XML 并重新提交给模型。
7. shutdown approval 会触发 pane 清理、任务解绑、成员移除等真正的生命周期收尾动作。
8. 已读时机被延后到“成功投递或可靠入队之后”，避免消息丢失。

---

### 下一站建议

下一站最顺的是：

```text
src/utils/swarm/teamHelpers.ts
```

原因：

- 你已经看清了 mailbox 运行时消费层
- `useInboxPoller.ts` 里多次回写 team file、成员 mode、teammate 移除

下一步就该看：

**team 的持久化结构、成员增删改、mode 更新，以及 leader 视角下团队元数据是如何落地维护的。**

---

---

### 元

问题：**这一站真正想解决的架构问题是什么？**

回答：这一站是 mailbox 的运行时消费器，负责轮询 unread messages，并把不同协议消息重新接回权限系统、team 状态和 query turn。它不是“读一下 inbox”，而是消息回流到 runtime 的桥。

### 反

问题：**如果把这一站的设计反过来，会发生什么？**

回答：如果只有 mailbox 写入没有统一消费层，权限响应、shutdown、普通队友消息都会卡在文件里。系统会有传输，却没有真正的运行时闭环。

### 空

问题：**跳出当前文件名，这一站背后更大的问题是什么？**

回答：更大的问题是，离散消息如何重新进入活着的 agent 会话。轮询与分流层在这里承担了协议总线到运行时动作的转换。

## 第 18 站：`src/utils/swarm/teamHelpers.ts`

### 这是什么文件

`src/utils/swarm/teamHelpers.ts` 是 agent team 子系统里的**团队元数据持久化与清理工具箱**。

如果说：

- `permissionSync.ts` 负责权限同步协议
- `teammateMailbox.ts` 负责消息总线
- `useInboxPoller.ts` 负责运行时消费与分发

那么这一站回答的是：

```text
team 自己的配置文件长什么样，成员状态如何落盘，team 结束时怎样做目录/任务/worktree/pane 清理
```

所以一句话先记：

```text
teamHelpers.ts = swarm team 的持久化控制面
```

它不是消息通道，也不是执行后端，而是负责维护 team 的**磁盘真相源**：

- team 目录与 `config.json`
- members 列表
- hidden pane 列表
- teammate mode / active 状态
- session cleanup 注册表
- team / tasks / worktree 的终态清理

---

### 先看这个文件最重要的两个基础类型

#### 1. `TeamFile`
位置：`src/utils/swarm/teamHelpers.ts:64`

这是整个 swarm team 持久化结构的核心。

里面最关键的字段有：

- `name`
- `description`
- `createdAt`
- `leadAgentId`
- `leadSessionId`
- `hiddenPaneIds`
- `teamAllowedPaths`
- `members[]`

而 `members[]` 里又保存了：

- `agentId`
- `name`
- `agentType`
- `model`
- `planModeRequired`
- `joinedAt`
- `tmuxPaneId`
- `cwd`
- `worktreePath`
- `sessionId`
- `subscriptions`
- `backendType`
- `isActive`
- `mode`

这说明 team file 不是只有“有哪些成员”这么简单，而是：

```text
leader 视角下的团队运行时元数据快照
```

也就是说，它既服务于 UI / discovery，也服务于 mode 同步、pane 管理、shutdown 清理等控制面逻辑。

---

#### 2. `TeamAllowedPath`
位置：`src/utils/swarm/teamHelpers.ts:57`

这个类型也很值得记。

它保存：

- `path`
- `toolName`
- `addedBy`
- `addedAt`

也就是说，team file 不只记录“人”，还记录**团队级别共享授权路径**。

这说明 team config 其实还承担了一部分：

```text
team 范围内的权限传播记账
```

和前面 `teamPermissionUpdates` / `modeSetRequests` 的控制面是对得上的。

---

### 基础主线 1：team 文件路径与读写 API
关键位置：
- `src/utils/swarm/teamHelpers.ts:115`
- `src/utils/swarm/teamHelpers.ts:122`
- `src/utils/swarm/teamHelpers.ts:131`
- `src/utils/swarm/teamHelpers.ts:147`
- `src/utils/swarm/teamHelpers.ts:166`
- `src/utils/swarm/teamHelpers.ts:175`

这里先定义了最基础的路径与 I/O：

- `getTeamDir(teamName)`
- `getTeamFilePath(teamName)`
- `readTeamFile(...)`
- `readTeamFileAsync(...)`
- `writeTeamFile(...)`
- `writeTeamFileAsync(...)`

路径模型很直接：

```text
~/.claude/teams/{sanitizeName(teamName)}/config.json
```

再加上同步 / 异步两套读写接口。

这里最值得注意的点是：

```text
teamHelpers.ts 明确把 sync path 和 async path 都保留了
```

原因也写得很清楚：

- sync 版本给 React render / 同步上下文用
- async 版本给 tool handler / 其他异步流程用

所以这个文件不是纯工具函数堆，而是在照顾不同宿主上下文对 I/O 方式的约束。

---

### 基础主线 2：team file 是所有成员状态更新的真相源
关键位置：
- `src/utils/swarm/teamHelpers.ts:188`
- `src/utils/swarm/teamHelpers.ts:235`
- `src/utils/swarm/teamHelpers.ts:259`
- `src/utils/swarm/teamHelpers.ts:285`
- `src/utils/swarm/teamHelpers.ts:326`
- `src/utils/swarm/teamHelpers.ts:357`
- `src/utils/swarm/teamHelpers.ts:415`
- `src/utils/swarm/teamHelpers.ts:454`

这些函数本质上都在做一件事：

```text
修改 teamFile，然后写回 config.json
```

具体包括：

- `removeTeammateFromTeamFile(...)`
- `addHiddenPaneId(...)`
- `removeHiddenPaneId(...)`
- `removeMemberFromTeam(...)`
- `removeMemberByAgentId(...)`
- `setMemberMode(...)`
- `setMultipleMemberModes(...)`
- `setMemberActive(...)`

这说明 team runtime 的很多“当前状态”并不是只存在内存里，而是会落回 team file。

也就是说：

```text
teamContext 是运行时视图
team file 是持久化真相源
```

前一站 `useInboxPoller.ts` 之所以不断回写 team file，就是因为这里定义了那些落盘 API。

---

### 关键职责 1：成员移除有多套入口，分别对应不同 backend 形态
关键位置：
- `src/utils/swarm/teamHelpers.ts:188`
- `src/utils/swarm/teamHelpers.ts:285`
- `src/utils/swarm/teamHelpers.ts:326`

这里有三类“删除成员”：

1. `removeTeammateFromTeamFile(...)`
   - 允许按 `agentId` 或 `name` 删除
   - `useInboxPoller.ts` 处理 shutdown approval 时会用到

2. `removeMemberFromTeam(...)`
   - 按 `tmuxPaneId` 删除
   - 同时从 `hiddenPaneIds` 清掉对应 pane

3. `removeMemberByAgentId(...)`
   - 专门给 in-process teammate 用
   - 因为 in-process teammates 可能共享同一个 `tmuxPaneId`

这非常有代表性。它说明：

```text
team 成员身份并不是只有一种主键，
不同 backend / 生命周期场景会用不同标识来做删除和同步
```

这正是 swarm 同时支持 pane-based teammate 和 in-process teammate 的副产物。

---

### 关键职责 2：mode 与 active 状态会显式落盘
关键位置：
- `src/utils/swarm/teamHelpers.ts:357`
- `src/utils/swarm/teamHelpers.ts:397`
- `src/utils/swarm/teamHelpers.ts:415`
- `src/utils/swarm/teamHelpers.ts:454`

这里有 4 个值得一起记：

- `setMemberMode(...)`
- `syncTeammateMode(...)`
- `setMultipleMemberModes(...)`
- `setMemberActive(...)`

这几段代码背后说明了两件重要事情：

#### A. teammate 当前 permission mode 是 team file 的一部分
leader 修改 teammate mode 后，不只是改内存态；teammate 自己也可以 `syncTeammateMode(...)` 回写到 config.json，让 leader/UI 看到最新状态。

#### B. teammate 当前是否活跃也会持久化
`setMemberActive(...)` 用于把成员标成 active / idle。

所以 team file 本质上记录的不只是静态成员列表，而是：

```text
团队成员的控制面状态面板
```

这就是为什么 leader 能从 team 元数据里看见谁空闲、谁活跃、谁是什么 mode。

---

### 关键职责 3：hiddenPaneIds 说明 UI 可见性也被放进 team file
关键位置：
- `src/utils/swarm/teamHelpers.ts:235`
- `src/utils/swarm/teamHelpers.ts:259`

`addHiddenPaneId(...)` / `removeHiddenPaneId(...)` 看起来很小，但很有意思。

它说明 pane 是否在 UI 中隐藏，也不是纯内存状态，而是被记录进 `teamFile.hiddenPaneIds`。

也就是说：

```text
team file 不只保存协作语义，还保存一部分 pane/UI 可见性控制状态
```

这再次说明这个文件的定位不是“业务模型定义”，而是**leader 视角下的 team 全量控制面快照**。

---

### 关键职责 4：session cleanup 跟踪的是“本 session 创建的 team”
关键位置：
- `src/utils/swarm/teamHelpers.ts:560`
- `src/utils/swarm/teamHelpers.ts:568`
- `src/utils/swarm/teamHelpers.ts:576`

这里有一组很关键的函数：

- `registerTeamForSessionCleanup(teamName)`
- `unregisterTeamForSessionCleanup(teamName)`
- `cleanupSessionTeams()`

它们操作的不是 team file，而是 `bootstrap/state.ts` 里的 session-level Set。

这表示：

```text
系统会额外记住“哪些 team 是本次 Claude session 创建出来的”
```

这样在 leader 非正常退出时，就可以统一清理这些“孤儿 team”。

这个设计很重要，因为它把：

- **持久化 team 元数据**
- **当前 session 的清理责任**

分成了两层，而不是混在同一个 config.json 里。

---

### 关键职责 5：team 清理不只是删目录，还包括 pane / worktree / tasks 清理
关键位置：
- `src/utils/swarm/teamHelpers.ts:492`
- `src/utils/swarm/teamHelpers.ts:598`
- `src/utils/swarm/teamHelpers.ts:641`

这是文件后半段最值得记的部分。

#### A. `destroyWorktree(...)`
它会先尝试：

- 从 worktree 的 `.git` 文件反推出主 repo
- 调 `git worktree remove --force`
- 如果失败，再 fallback 到 `rm -rf`

说明 team cleanup 是知道 git worktree 语义的，不是简单删目录。

#### B. `killOrphanedTeammatePanes(...)`
在 session 非优雅退出时，先把 pane-based teammate 的 pane 干掉，避免 tmux/iTerm2 里留下孤儿进程。

#### C. `cleanupTeamDirectories(teamName)`
它会：

1. 先读 team file 收集所有 `worktreePath`
2. 逐个删 worktree
3. 删除 team directory
4. 删除 tasks directory
5. `notifyTasksUpdated()`

所以真正的结论是：

```text
team cleanup = worktree cleanup + pane cleanup + team dir cleanup + task dir cleanup
```

这远不只是“删除 config.json”。

---

### 为什么 `cleanupSessionTeams()` 很关键
位置：`src/utils/swarm/teamHelpers.ts:576`

这里的注释非常重要：

- TeamDelete 正常路径下，不一定需要额外 kill pane
- 但 leader 如果因为 `SIGINT/SIGTERM` 这种方式退出，teammate 进程可能还活着
- 这时如果只删目录，不 kill pane，会留下孤儿 tmux / iTerm2 pane

这说明：

```text
teamHelpers.ts 处理的不只是“逻辑一致性”，还处理进程与宿主资源泄漏问题
```

也就是说它是 swarm 系统的**善后层**。

---

### 这个文件体现出的 7 个架构事实

1. `teamHelpers.ts` 把 team 的持久化真相源固定为 `~/.claude/teams/{team}/config.json`。
2. `TeamFile` 不是简单成员清单，而是 leader 视角下的团队运行时元数据快照。
3. 成员状态、mode、active、hidden panes 都会显式写回 team file。
4. 不同 backend 形态需要不同的成员删除主键：name / agentId / tmuxPaneId。
5. session cleanup 跟踪的是“本 session 创建的 teams”，这是额外的一层清理责任模型。
6. team 清理不只是删 team 目录，还要清理 worktree、tasks 和可能遗留的 pane 进程。
7. 这个文件是 swarm 系统的持久化控制面和善后层，而不是消息通道层。

---

### 现在把第 17-18 站串起来

到这里，mailbox runtime 和 team 持久化控制面就接上了：

```text
useInboxPoller.ts
  -> 处理 modeSet / shutdown / permission update
  -> 调 setMemberMode(...) / removeTeammateFromTeamFile(...)

teamHelpers.ts
  -> 负责把这些 team-level 状态真正写进 config.json
  -> 在 team 结束时清理 pane / worktree / tasks / team dirs
```

所以现在你可以把 swarm 控制面拆成两层：

```text
mailbox/runtime 层
  -> 消息消费、事件分发、turn reinjection

team persistence 层
  -> team file、成员状态、资源清理、session cleanup
```

这两层合起来，Claude Code 的 agent team 才真正闭环。

---

### 这一站最该记住的 8 句话

1. `teamHelpers.ts` 是 swarm team 的持久化控制面。
2. `TeamFile` 记录的是团队运行时元数据快照，而不只是成员列表。
3. teammate 的 mode、active 状态、hidden pane 等都会写回 team file。
4. 删除成员有多种入口，是因为不同 teammate backend 的身份主键不同。
5. team file 是运行时 teamContext 背后的持久化真相源。
6. session 还会额外跟踪“本次创建的 teams”，用于退出时统一善后。
7. team cleanup 不只是删配置，还会处理 worktree、tasks 和 pane 资源。
8. 这个文件承担的是控制面落盘与资源善后职责，而不是消息通信职责。

---

### 下一站建议

下一站最顺的是：

```text
src/utils/swarm/backends/registry.ts
```

原因：

- 你已经看到了 team 元数据和 pane/worktree 的清理面
- 这里多次出现 `backendType`、pane backend、killPane、backend registry

下一步就该看：

**Claude Code 到底如何抽象不同 teammate 运行后端（如 tmux / iTerm / in-process），以及这些 backend 是怎样被注册和查找的。**

---

---

### 元

问题：**这一站真正想解决的架构问题是什么？**

回答：这一站关心的是 team 作为持久化实体如何存在，包括 `config.json`、成员列表、hidden panes、allowed paths 和 cleanup 注册表。它是 swarm 的磁盘真相源和收尾控制面。

### 反

问题：**如果把这一站的设计反过来，会发生什么？**

回答：如果团队元数据只存在内存里，崩溃恢复、成员清理和异常退出后的回收都会变得非常脆弱。team 生命周期也无法被统一管理。

### 空

问题：**跳出当前文件名，这一站背后更大的问题是什么？**

回答：更大的问题是，多 agent 团队是否应被当成正式系统对象而不是临时会话。这里的答案是肯定的，所以才有完整的元数据与清理机制。

## 第 19 站：`src/utils/swarm/backends/registry.ts`

### 这是什么文件

`src/utils/swarm/backends/registry.ts` 是 swarm teammate 执行后端的**选择器 + 注册表 + 缓存层**。

如果说上一站 `teamHelpers.ts` 解决的是：

```text
team 元数据怎么落盘、team 结束时资源怎么善后
```

那么这一站解决的是：

```text
Claude Code 在运行时到底如何决定 teammate 应该跑在 tmux、iTerm2 还是 in-process，
以及这些 backend 如何被统一暴露给上层调用方
```

所以一句话先记：

```text
registry.ts = swarm backend 的运行时选路中心
```

它不是具体后端实现文件，而是负责：

- backend class 注册
- 环境探测
- 后端优先级选择
- 检测结果缓存
- in-process / pane executor 的统一出口

---

### 先看最关键的几个缓存变量
位置：`src/utils/swarm/backends/registry.ts:22-54`

这个文件一上来就声明了很多 cached state：

- `cachedBackend`
- `cachedDetectionResult`
- `backendsRegistered`
- `cachedInProcessBackend`
- `cachedPaneBackendExecutor`
- `inProcessFallbackActive`

这说明 registry 不是纯函数式探测工具，而是：

```text
一个“本进程生命周期内只解析一次环境”的会话级选路器
```

也就是说，backend 选择不是每次 spawn teammate 都重新探测，而是尽量固定下来，保持整场 session 里的一致性。

---

### 基础主线 1：具体 backend class 通过注册而不是直接静态依赖
关键位置：
- `src/utils/swarm/backends/registry.ts:60-66`
- `src/utils/swarm/backends/registry.ts:74-100`
- `src/utils/swarm/backends/registry.ts:106-126`

这里先定义了两个占位符：

- `TmuxBackendClass`
- `ITermBackendClass`

然后通过：

- `registerTmuxBackend(...)`
- `registerITermBackend(...)`

把真实类注入进来。

再由：

- `createTmuxBackend()`
- `createITermBackend()`

负责实例化。

这里的设计点很重要：

```text
registry 不直接 import 具体 backend class 作为强依赖，
而是用“动态导入 + 注册回填”来避免循环依赖
```

源码注释也明确说了这一点。

所以这个文件可以看作：

```text
backend implementation 与 backend selection 之间的解耦层
```

---

### 基础主线 2：`ensureBackendsRegistered()` 只做类注册，不做环境探测
位置：`src/utils/swarm/backends/registry.ts:74`

这个函数非常值得单独记住。

它会动态 import：

- `./TmuxBackend.js`
- `./ITermBackend.js`

但源码注释明确说：

- **不会 spawn subprocess**
- **不会抛出环境探测错误**
- 只是为了让 `getBackendByType()` 有类可构造

也就是说：

```text
“让类可用” 和 “真的决定当前该用哪个 backend” 是两个分开的步骤
```

这在前面 `useInboxPoller.ts` / `teamHelpers.ts` 的清理路径里就很重要：

- 有时只是想按 `backendType` 调 `killPane(...)`
- 并不想重新跑完整环境探测

所以 `ensureBackendsRegistered()` 是一个**轻量注册入口**，不是完整选路入口。

---

### 核心主线：`detectAndGetBackend()` 是真正的后端选择器
位置：`src/utils/swarm/backends/registry.ts:136`

这是整个文件的核心。

源码自己已经写出了优先级：

1. inside tmux -> 用 tmux
2. in iTerm2 且 it2 CLI 可用 -> 用 iTerm2 backend
3. in iTerm2 但没 it2 -> 看 tmux 能否 fallback
4. 其他环境如果 tmux 可用 -> 用 tmux external session
5. 否则抛错给安装说明

这五步其实就是 swarm backend 的**运行时选路规则**。

---

### 优先级 1：只要在 tmux 里，就总是 tmux
关键位置：`src/utils/swarm/backends/registry.ts:158-171`

这里的规则很硬：

```text
inside tmux => always tmux
```

即使当前也在 iTerm2 里，只要你已经处在 tmux session 内，系统仍然优先选择 tmux backend。

这说明 Claude Code 更看重的是：

```text
当前实际 pane 宿主是谁
```

而不是“外层终端应用长什么样”。

这个优先级很合理，因为一旦已经在 tmux 里，原生 tmux pane 管理就是最直接、最稳定的控制面。

---

### 优先级 2：iTerm2 native pane 是次优先级，但有用户偏好开关
关键位置：`src/utils/swarm/backends/registry.ts:173-220`

在 iTerm2 环境里，逻辑不是直接无脑选 iTerm2，而是先看：

- `getPreferTmuxOverIterm2()`

如果用户明确偏好 tmux，就跳过 iTerm2 检测。

否则再看：

- `isIt2CliAvailable()`

如果 it2 CLI 可用，就选 `iterm2` backend，并标记：

- `isNative: true`
- `needsIt2Setup: false`

如果 it2 不可用，但 tmux 可用，则 fallback 到 tmux，同时可能返回：

- `needsIt2Setup: !preferTmux`

这说明：

```text
iTerm2 不是简单能力探测，
还叠加了“用户是否更愿意退回 tmux”的偏好层
```

以及：

```text
选到 tmux fallback 时，registry 还能顺便告诉上层“是否该提示用户去装 it2”
```

所以 `BackendDetectionResult` 不只是 backend 本身，还带了 UI/设置层决策元数据。

---

### 优先级 3：脱离 tmux / iTerm2 时，tmux external session 是通用兜底
关键位置：`src/utils/swarm/backends/registry.ts:233-254`

如果既不在 tmux，也不在 iTerm2 native pane 条件里，就看：

- `isTmuxAvailable()`

若可用，就选 tmux，并标记：

- `isNative: false`
- `needsIt2Setup: false`

这里的语义很重要：

```text
tmux 不只是“当前就在 tmux 时可用”，
也是默认的跨宿主通用 pane backend
```

也就是即使当前在普通终端里，也可以通过外部 tmux session 来承载 teammate panes。

这就是 swarm teammate 在多数环境里的标准兜底方案。

---

### 错误路径：没有 pane backend 时会返回平台相关安装说明
关键位置：`src/utils/swarm/backends/registry.ts:251-285`

如果 tmux 也不可用，registry 不只是抛通用错误，而是调用：

- `getTmuxInstallInstructions()`

并按平台给出不同安装建议：

- macOS -> `brew install tmux`
- Linux / WSL -> `apt` / `dnf`
- Windows -> 先装 WSL 再装 tmux

这说明 registry 不只承担技术选路，还承担了一部分：

```text
用户可恢复错误提示生成
```

也就是说，它已经非常靠近产品层体验，而不是纯底层调度器。

---

### `getBackendByType()` 为什么重要
位置：`src/utils/swarm/backends/registry.ts:295`

这个函数不做环境探测，只做：

```text
显式类型 -> backend 实例
```

支持：

- `'tmux'`
- `'iterm2'`

这在前面已经见过用途：

- `useInboxPoller.ts` 处理 shutdown approval 时，按存下来的 `backendType` 找回 backend 并 kill pane
- `teamHelpers.ts` 善后时，也可能按成员记录的 backendType 做资源清理

所以它的价值在于：

```text
运行时恢复路径不必重新探测环境，
只要按已记录的 backendType 找回正确实现即可
```

---

### `isInProcessEnabled()` 才是 in-process 模式的最终裁决器
位置：`src/utils/swarm/backends/registry.ts:351`

这一段也非常关键。

它综合考虑：

- 是否 non-interactive session
- teammateMode snapshot 是 `in-process` / `tmux` / `auto`
- `inProcessFallbackActive` 是否已触发
- 当前是否在 tmux / iTerm2

规则可以压缩成：

#### 明确配置优先
- `in-process` -> 一定启用 in-process
- `tmux` -> 一定不用 in-process

#### `auto` 模式下再看环境
- 如果之前因为 pane backend 不可用而 fallback 到 in-process，则本 session 后续一直沿用
- 否则：
  - 在 tmux / iTerm2 中 => 不启用 in-process
  - 不在 pane 宿主中 => 启用 in-process

#### 非交互会话强制 in-process
这个特判尤其重要：

```text
-p / non-interactive session 下，tmux teammate 没意义，所以直接强制 in-process
```

因此准确理解应该是：

```text
isInProcessEnabled() = teammate 执行模式的最终运行时收口器
```

不是简单看配置，而是把配置、环境、fallback 现实一起折叠后的最终答案。

---

### `markInProcessFallback()` 说明 backend 选择会被现实失败反向修正
位置：`src/utils/swarm/backends/registry.ts:326`

这个函数很有意思。

如果某次 spawn 发现 pane backend 根本不可用，就调用它把：

- `inProcessFallbackActive = true`

之后 `isInProcessEnabled()` 就会改口，认为当前 session 应该继续走 in-process。

这说明：

```text
backend 选择不是一次性只看“理论配置”，
还会被实际执行失败后的现实状态反向修正
```

也就是说，registry 在维护的不是纯配置快照，而是：

```text
本 session 下已经验证过的可行执行模式
```

这点非常产品化，也很务实。

---

### 上层统一出口：`getTeammateExecutor()`
位置：`src/utils/swarm/backends/registry.ts:425`

这是给上层最重要的统一接口。

它返回的不是具体 pane backend，而是统一的 `TeammateExecutor`：

- 若 `preferInProcess` 且当前允许 in-process -> `getInProcessBackend()`
- 否则 -> `getPaneBackendExecutor()`

而 `getPaneBackendExecutor()` 又会：

- `detectAndGetBackend()`
- `createPaneBackendExecutor(detection.backend)`

所以最终上层看到的是：

```text
不管底下是 tmux / iTerm2 / in-process，
我都拿到一个统一的 executor 接口
```

这就是 backend registry 对上层最大的价值：

```text
把“后端选路”封装掉，让调用方只关心执行器，不关心宿主细节
```

---

### `getResolvedTeammateMode()` 很值得记
位置：`src/utils/swarm/backends/registry.ts:396`

这个函数虽然很小，但概念上很重要。

它解决的是：

```text
配置里可能还是 auto，
但当前环境下 auto 实际到底解析成 in-process 还是 tmux？
```

也就是说：

- `getTeammateModeFromSnapshot()` 给你配置视角
- `getResolvedTeammateMode()` 给你运行时实际视角

这两个层次分得很清楚。

---

### 这个文件体现出的 8 个架构事实

1. `registry.ts` 是 backend 选择、注册、缓存、统一出口的中心，而不是具体 backend 实现。
2. 具体 backend class 通过动态导入 + register 回填，避免与实现文件形成硬循环依赖。
3. backend 环境探测结果会在进程内缓存，保持整场 session 的稳定性。
4. 选路优先级是：tmux inside > iTerm2 native > tmux fallback > 报安装错误。
5. iTerm2 路径还叠加了用户偏好和 `needsIt2Setup` 这类产品层元数据。
6. `isInProcessEnabled()` 是配置、环境、fallback 现实三者合并后的最终裁决器。
7. 某次真实 spawn 的失败可以通过 `markInProcessFallback()` 反向修正后续模式选择。
8. 上层最终通过统一的 `TeammateExecutor` 接口工作，不必关心底层宿主是 tmux、iTerm2 还是 in-process。

---

### 现在把第 18-19 站串起来

到这里，team persistence 层和 backend 选路层接上了：

```text
teamHelpers.ts
  -> member.backendType / paneId / worktreePath 会写进 team file
  -> cleanup 时按 backendType / pane 信息回收资源

registry.ts
  -> 决定当前 session 该用哪种 teammate backend
  -> 提供 getBackendByType(...) / getTeammateExecutor(...)
  -> 让清理路径和 spawn 路径都能找到统一 backend 抽象
```

所以现在 swarm 运行底座已经可以拆成三层：

```text
team persistence
  -> config.json / task dir / cleanup
backend routing
  -> tmux / iTerm2 / in-process 选路与统一执行器
mailbox runtime
  -> 消息消费、控制信号分发、turn reinjection
```

这三层合起来，Claude Code 的 agent team 运行框架已经非常清晰了。

---

### 这一站最该记住的 8 句话

1. `registry.ts` 是 swarm backend 的运行时选路中心。
2. 它把 backend 注册、环境探测、结果缓存和统一 executor 出口放在一起。
3. tmux inside 的优先级最高，因为这代表当前真实 pane 宿主已经确定。
4. iTerm2 只有在 it2 CLI 可用且用户不偏好 tmux 时才会走 native backend。
5. tmux 是最重要的跨宿主 fallback pane backend。
6. `isInProcessEnabled()` 不是简单读配置，而是最终运行时裁决器。
7. `markInProcessFallback()` 说明 registry 会根据现实失败修正后续策略。
8. 上层最终拿到的是统一 `TeammateExecutor`，而不是直接操作具体 backend 细节。

---

### 下一站建议

下一站最顺的是：

```text
src/tasks/InProcessTeammateTask/InProcessTeammateTask.ts
```

原因：

- 你已经看到了 backend registry 如何决定何时走 in-process
- 也已经知道 in-process teammate 不走 `useInboxPoller.ts` 那条收信路径

下一步就该看：

**当 backend 选成 in-process 时，teammate task 在主进程里是怎样被表示、调度、接收消息与结束的。**

---

---

### 元

问题：**这一站真正想解决的架构问题是什么？**

回答：这一站决定 teammate 到底跑在 tmux、iTerm2 还是 in-process，并对探测结果做 session 级缓存。它是 backend 注册、环境选路与统一出口的中枢。

### 反

问题：**如果把这一站的设计反过来，会发生什么？**

回答：如果每次 spawn 都临时决定后端，整场 session 会出现行为漂移。没有统一 registry，不同调用方也会各自实现一套 backend 选择逻辑。

### 空

问题：**跳出当前文件名，这一站背后更大的问题是什么？**

回答：更大的问题是，多种执行承载方式如何对上层表现成同一类 teammate。registry 代表的正是“差异后端，统一执行语义”。

## 第 20 站：`src/tasks/InProcessTeammateTask/InProcessTeammateTask.tsx`

### 这是什么文件

`src/tasks/InProcessTeammateTask/InProcessTeammateTask.tsx` 是 in-process teammate 的**Task 适配层**。

前一站 `registry.ts` 解决的是：

```text
什么时候选择 in-process backend
```

而这一站解决的是：

```text
一旦 teammate 真的跑在主进程里，AppState 里如何表示它、如何给它注入消息、如何请求关闭、如何查找这些 task
```

所以一句话先记：

```text
InProcessTeammateTask.tsx = in-process teammate 的任务模型与状态操作入口
```

它不是实际 runner，也不是 backend 选路器，而是把 in-process teammate 接进统一 Task 框架的那一层。

---

### 先看这个文件最重要的对象

#### `InProcessTeammateTask`
位置：`src/tasks/InProcessTeammateTask/InProcessTeammateTask.tsx:24`

它实现了一个很薄的 `Task` 对象：

- `name: 'InProcessTeammateTask'`
- `type: 'in_process_teammate'`
- `kill(...)` -> `killInProcessTeammate(...)`

这说明：

```text
在 Task 框架眼里，in-process teammate 首先是一个特殊 task type
```

也就是说，上层 task 系统并不需要知道它是不是 tmux pane、是不是本进程共享 AsyncLocalStorage；对上层来说，先统一表现成一种 task。

这就是这一层存在的最大意义：

```text
把 teammate lifecycle 翻译成 Task 接口
```

---

### 这个文件的核心，不在“运行逻辑”，而在“状态操作”

读完整个文件会发现，它几乎没有复杂执行代码。

它主要暴露的是几类 helper：

- `requestTeammateShutdown(...)`
- `appendTeammateMessage(...)`
- `injectUserMessageToTeammate(...)`
- `findTeammateTaskByAgentId(...)`
- `getAllInProcessTeammateTasks(...)`
- `getRunningTeammatesSorted(...)`

所以这文件更准确的定位是：

```text
in-process teammate task state 的读写工具箱
```

而不是真正的 runner 主循环。

---

### 关键职责 1：shutdown 在这里先变成 task 状态位
关键位置：`src/tasks/InProcessTeammateTask/InProcessTeammateTask.tsx:35`

`requestTeammateShutdown(...)` 做的事情很简单：

- 只在 `task.status === 'running'` 且还没请求过 shutdown 时生效
- 把 `shutdownRequested` 置为 `true`

这说明对 in-process teammate 来说，“请求关闭”在这一层并不是立刻 kill，而是：

```text
先把 shutdown 意图写进 task state
```

也就是说，真正的执行体会在后续某个检查点读取这个标志，再决定如何收尾。

所以 shutdown 在这里体现的是：

```text
控制信号先落到 task state，再由 runner 消费
```

---

### 关键职责 2：teammate transcript 会被附着在 task 上
关键位置：`src/tasks/InProcessTeammateTask/InProcessTeammateTask.tsx:51`

`appendTeammateMessage(...)` 会把一条 `Message` 追加到：

- `task.messages`

并且通过：

- `appendCappedMessage(...)`

说明这个消息历史不是无限增长，而是有 cap 的。

这段的意义很重要：

```text
in-process teammate 自己的会话历史，是 task state 的一部分
```

这也解释了为什么 UI 可以做 teammate transcript zoomed view —— 因为 task 上本来就挂着一份专用消息历史。

---

### 关键职责 3：给 teammate 发消息，本质上是往 pending queue 里塞用户输入
关键位置：`src/tasks/InProcessTeammateTask/InProcessTeammateTask.tsx:68`

`injectUserMessageToTeammate(...)` 是这一站最值得记住的函数之一。

它会：

1. 检查 task 是否处于 terminal state
2. 如果已终态，丢弃消息并打 debug log
3. 否则把消息追加到：
   - `pendingUserMessages`
   - 同时也把一条 user message 立刻追加到 `messages`

这说明：

```text
给 in-process teammate 发消息，并不是直接“调用一个函数让它立刻处理”
而是先进入 task 的 pendingUserMessages 队列
```

同时为了让 UI 立刻可见，又会同步往 transcript 里插一条 user message。

所以这一层同时解决了两件事：

- **执行语义**：消息进入待消费队列
- **展示语义**：消息立刻出现在 teammate transcript

这是很典型的“运行时状态 + UI 状态”双写设计。

---

### 为什么它允许 idle 状态也能注入消息
关键位置：`src/tasks/InProcessTeammateTask/InProcessTeammateTask.tsx:70-72`

源码注释明确写了：

- running 可以注入
- idle（waiting for input）也可以注入
- 只有 terminal state 才拒绝

这很关键，因为它说明 in-process teammate 的生命周期里，“idle”不是结束，而是：

```text
仍然可恢复、可继续接活的等待态
```

这和普通 background task 的很多语义不完全一样。

也就是说 in-process teammate 更像：

```text
可暂停、可再唤醒的协作者
```

而不是“一次性跑完就结束”的任务。

---

### 关键职责 4：按 agentId 查找 task 时，优先 running 实例
关键位置：`src/tasks/InProcessTeammateTask/InProcessTeammateTask.tsx:92`

`findTeammateTaskByAgentId(...)` 的逻辑很有代表性：

- 遍历所有 tasks
- 找到 `identity.agentId` 命中的 in-process teammate task
- **优先返回 `status === 'running'` 的那个**
- 如果没有 running，才用第一个匹配项作为 fallback

这说明系统已经考虑到了这种现实情况：

```text
同一个 agentId 可能在 AppState 里同时存在旧的 killed/completed task 和新的 running task
```

所以这里不是简单 map lookup，而是带有生命周期语义的查找。

这也说明 AppState.tasks 更像历史累积集合，而不是“每个 agent 只有一个槽位”。

---

### 关键职责 5：运行中的 in-process teammates 需要稳定排序
关键位置：`src/tasks/InProcessTeammateTask/InProcessTeammateTask.tsx:113`

后两个 helper：

- `getAllInProcessTeammateTasks(...)`
- `getRunningTeammatesSorted(...)`

其中 `getRunningTeammatesSorted(...)` 的注释尤其关键：

它被多个 UI / 导航逻辑共享：

- `TeammateSpinnerTree`
- `PromptInput` footer selector
- `useBackgroundTaskNavigation`

并且这些地方都依赖同一套排序结果，否则 `selectedIPAgentIndex` 会错位。

所以这里的意义不只是“排序好看”，而是：

```text
in-process teammate 列表顺序本身是跨组件共享的导航协议
```

这类细节很容易忽略，但它说明 task helper 层还在承担 UI 一致性约束。

---

### 这个文件和真正 runner 的边界

读完这个文件后要特别注意，不要把它理解得太重。

它没有告诉你：

- AsyncLocalStorage 怎么隔离 teammate 上下文
- pendingUserMessages 在哪里被真正消费
- shutdownRequested 在哪里被真正检查
- teammate idle / active 如何在主循环里切换

它只告诉你：

```text
这些信息在 Task/AppState 里是怎么表示、怎么更新、怎么被查找的
```

所以它和真正 runner 的关系可以先记成：

```text
InProcessTeammateTask.tsx = 任务状态层
spawn / runner 代码 = 执行层
```

这个边界很重要。

---

### 这个文件体现出的 7 个架构事实

1. in-process teammate 在上层 task 系统里被建模成一种专门的 `Task` 类型。
2. shutdown 请求在这一层先变成 task state 标志，而不是立刻强杀。
3. teammate transcript 被挂在 task.messages 上，说明 task 同时承担执行状态与展示状态。
4. 用户发给 teammate 的消息会先进入 `pendingUserMessages` 队列，再由执行层消费。
5. idle 的 teammate 仍然可接收消息，说明 idle 是等待态而不是终态。
6. 同一个 agentId 可能对应多个历史 task，因此查找时要优先 running 实例。
7. running teammate 的排序结果是多个 UI/导航模块共享的稳定协议。

---

### 现在把第 19-20 站串起来

到这里，in-process 这条线已经接上了：

```text
registry.ts
  -> 决定当前 session 是否启用 in-process backend
  -> 提供统一 executor 出口

InProcessTeammateTask.tsx
  -> 定义 in-process teammate 在 AppState/Task 框架中的表示
  -> 管理 shutdownRequested / pendingUserMessages / messages
  -> 提供按 agentId 查找与按名称稳定排序的 helper
```

也就是说：

```text
registry.ts 决定“走不走 in-process”
InProcessTeammateTask.tsx 决定“走了以后 task state 长什么样”
```

---

### 这一站最该记住的 8 句话

1. `InProcessTeammateTask.tsx` 是 in-process teammate 的 Task 适配层。
2. 它把 teammate lifecycle 翻译成统一 task 类型 `in_process_teammate`。
3. shutdown 在这里先表现为 `shutdownRequested` 状态位。
4. teammate transcript 被挂在 `task.messages` 上，并且有容量上限。
5. 发给 teammate 的用户消息会先进入 `pendingUserMessages` 队列。
6. idle teammate 仍可继续收消息，说明它是等待态而不是终态。
7. 按 agentId 查找 task 时会优先 running 实例，避免命中旧历史 task。
8. running teammates 的排序结果是多个 UI/导航模块共享的稳定约定。

---

### 下一站建议

下一站最顺的是：

```text
src/utils/swarm/spawnInProcess.ts
```

原因：

- 这一站只看到了 in-process teammate 的任务状态层
- 还没看到 pending messages、shutdown flag、AsyncLocalStorage 隔离到底在哪里被真正消费

下一步就该看：

**in-process teammate 是怎样被真正拉起、进入自己的执行上下文、等待新消息、响应 shutdown 并驱动 task 状态变化的。**

---

---

### 元

问题：**这一站真正想解决的架构问题是什么？**

回答：这一站把 in-process teammate 接进统一 Task 框架，重点不在执行逻辑，而在任务形态和状态操作。它提供 shutdown 请求、消息追加、用户输入注入和任务查找这些入口。

### 反

问题：**如果把这一站的设计反过来，会发生什么？**

回答：如果 in-process teammate 不被包装成 task，上层 task 系统就无法统一展示和管理它。它会成为一条难以观察、难以控制的特殊旁路。

### 空

问题：**跳出当前文件名，这一站背后更大的问题是什么？**

回答：更大的问题是，共享进程里的后台协作者如何被纳入同一生命周期模型。Task 化是把“特殊运行单元”收口回平台框架的典型做法。

## 第 21 站：`src/utils/swarm/spawnInProcess.ts`

### 这是什么文件

`src/utils/swarm/spawnInProcess.ts` 是 in-process teammate 的**出生与终止管理器**。

上一站 `InProcessTeammateTask.tsx` 解决的是：

```text
in-process teammate 在 Task/AppState 里长什么样
```

而这一站解决的是：

```text
这个 teammate 一开始是怎么被创建出来的，初始 task state 怎么注册，
AsyncLocalStorage 用的 teammateContext 从哪来，被 kill 时又怎样收尾
```

所以一句话先记：

```text
spawnInProcess.ts = in-process teammate 的 spawn / kill 控制层
```

它仍然不是完整 runner 主循环，但它已经从“状态定义层”进入到了“生命周期编排层”。

---

### 这个文件最重要的两个入口

#### 1. `spawnInProcessTeammate(...)`
位置：`src/utils/swarm/spawnInProcess.ts:104`

这是 in-process teammate 的创建入口。

它负责：

1. 生成 agentId / taskId
2. 创建独立 `AbortController`
3. 创建 `TeammateIdentity`
4. 创建 `teammateContext`
5. 生成初始 `InProcessTeammateTaskState`
6. 注册 cleanup handler
7. `registerTask(...)` 写入 AppState
8. 返回 `taskId + abortController + teammateContext`

所以它的本质是：

```text
把“配置”翻译成“可运行 teammate 的初始运行时对象”
```

---

#### 2. `killInProcessTeammate(...)`
位置：`src/utils/swarm/spawnInProcess.ts:227`

这是终止入口。

它负责：

- 中止 `abortController`
- 触发 cleanup handler
- 改写 task 状态为 `killed`
- 从 `teamContext.teammates` 移除成员
- 清空等待中的 idle callbacks / pending messages / in-progress tool state
- 从 team file 删除该成员
- 清理 task output / sdk bookend / terminal task / perfetto registry

所以它不是一个简单 `abort()` 包装器，而是：

```text
in-process teammate 的集中式收尸器
```

---

### 关键职责 1：agentId / taskId / session 关联在 spawn 时一次性定好
关键位置：`src/utils/swarm/spawnInProcess.ts:111-147`

创建时会先得到：

- `agentId = formatAgentId(name, teamName)`
- `taskId = generateTaskId('in_process_teammate')`
- `parentSessionId = getSessionId()`

然后组装：

- `TeammateIdentity`
- `createTeammateContext(...)`

这里最重要的理解是：

```text
in-process teammate 的身份不是运行时临时拼出来的，
而是在 spawn 时就固化为一组 identity + context
```

其中：

- `identity` 是可持久留在 AppState 的纯数据
- `teammateContext` 是给 AsyncLocalStorage / 执行层使用的运行上下文

这说明这个文件在显式区分：

```text
状态可序列化表示
vs
执行期上下文对象
```

---

### 关键职责 2：in-process teammate 有自己的独立 AbortController
关键位置：`src/utils/swarm/spawnInProcess.ts:120-123`

源码注释直接写明：

```text
Teammates should not be aborted when the leader's query is interrupted
```

所以这里不会复用 leader 的 abort controller，而是新建一个独立 controller。

这点非常关键。

它说明 in-process teammate 虽然和 leader 共进程，但在中断语义上不是 leader 的子调用栈，而是：

```text
同进程、独立可中止的协作者
```

这也是为什么“主 query 被打断”并不等于“所有 in-process teammate 都一起死掉”。

---

### 关键职责 3：`createTeammateContext(...)` 才是 AsyncLocalStorage 隔离的桥
关键位置：`src/utils/swarm/spawnInProcess.ts:137`

这里并没有直接跑 agent loop，而是先创建：

- `teammateContext`

然后把它返回给后续执行层使用。

这说明 `spawnInProcessTeammate(...)` 的职责边界很清楚：

```text
它不亲自执行 teammate，
而是负责准备好“稍后能在 runWithTeammateContext(...) 里执行”的上下文载体
```

所以这一层和真正 runner 的边界可以记成：

```text
spawnInProcess.ts = 准备执行环境
runner = 消费执行环境并进入 agent loop
```

---

### 关键职责 4：初始 task state 在这里一次性补齐了很多 runtime 字段
关键位置：`src/utils/swarm/spawnInProcess.ts:154-180`

这一段非常值得记，因为它告诉你 in-process teammate 的“出生状态”是什么。

初始化时会写入：

- `status: 'running'`
- `identity`
- `prompt`
- `model`
- `abortController`
- `awaitingPlanApproval: false`
- `spinnerVerb`
- `pastTenseVerb`
- `permissionMode: planModeRequired ? 'plan' : 'default'`
- `isIdle: false`
- `shutdownRequested: false`
- `lastReportedToolCount: 0`
- `lastReportedTokenCount: 0`
- `pendingUserMessages: []`
- `messages: []`

这里最关键的结论有两个：

#### A. planModeRequired 会直接决定初始 permissionMode
也就是说，是否要先过 plan approval，不是后面再临时判断，而是在 task 出生时就决定了初始模式。

#### B. teammate 一出生就是 running，但不一定是 busy working
`isIdle` 单独存在，说明：

```text
running / idle 是两套不同维度
```

- `running` 代表任务生命周期还活着
- `isIdle` 代表当前是否在等待工作

这对理解 in-process teammate 很重要。

---

### 关键职责 5：cleanup handler 把“进程退出时 abort teammate”接进全局清理体系
关键位置：`src/utils/swarm/spawnInProcess.ts:182-189`

这里会调用：

- `registerCleanup(async () => abortController.abort())`

并把返回的 `unregisterCleanup` 挂到 taskState 上。

这说明：

```text
in-process teammate 不是孤立任务，
它被接进了全局 graceful shutdown 清理注册表
```

也就是说，当整个 Claude session 做 cleanup 时，它们会被统一通知 abort。

同时把 `unregisterCleanup` 挂回 taskState，也意味着：

```text
后续 kill 路径可以主动把这份 cleanup 注册撤掉，避免重复清理
```

这是一个很典型的生命周期对称设计。

---

### 关键职责 6：spawn 成功的返回值本质上是“交给 runner 的启动包”
关键位置：`src/utils/swarm/spawnInProcess.ts:197-203`

成功返回时并不是只回一个布尔值，而是：

- `success`
- `agentId`
- `taskId`
- `abortController`
- `teammateContext`

这说明 spawn API 的真实定位是：

```text
先把 teammate 注册进系统，
再把执行层真正需要的句柄交给上层继续启动 runner
```

所以它更像：

```text
spawn = 注册 + 返回启动句柄
```

而不是“调用后就已经完整跑起来”。

---

### 关键职责 7：kill 路径同时修改内存态、持久化态和外围观测态
关键位置：`src/utils/swarm/spawnInProcess.ts:237-327`

`killInProcessTeammate(...)` 非常值得细读，因为它一次处理了多层后果。

#### A. 先改 AppState
它会：

- 找到 task
- 确认类型是 `in_process_teammate`
- 只处理 `status === 'running'`
- `abortController.abort()`
- `unregisterCleanup?.()`
- 调用 `onIdleCallbacks`
- 从 `teamContext.teammates` 删掉该 agentId
- 把 task 标记成 `killed`
- 清空：
  - `onIdleCallbacks`
  - `pendingUserMessages`
  - `inProgressToolUseIDs`
  - `abortController`
  - `unregisterCleanup`
  - `currentWorkAbortController`

并且为了减轻残留状态，还把：

- `messages` 收缩成只保留最后一条

这说明 kill 路径非常注意**释放状态引用**，避免 stale references 残留在 AppState。

#### B. 再改 team file
状态回调外再调用：

- `removeMemberByAgentId(teamName, agentId)`

说明 team file 作为持久化真相源，也必须同步去掉这个成员。

#### C. 再处理外围观测与展示
成功 kill 后还会：

- `evictTaskOutput(taskId)`
- `emitTaskTerminatedSdk(...)`
- 延迟 `evictTerminalTask(...)`
- `unregisterPerfettoAgent(agentId)`

这说明 kill 不只是“停执行”，还要把：

```text
磁盘输出
SDK 事件
终端展示
trace registry
```

全部一起收尾。

所以最准确的理解是：

```text
killInProcessTeammate() = 内存状态清理 + team 持久化清理 + 可观测性收尾
```

---

### `onIdleCallbacks` 为什么很值得注意
关键位置：`src/utils/swarm/spawnInProcess.ts:264-265`

kill 路径里会主动调用：

- `teammateTask.onIdleCallbacks?.forEach(cb => cb())`

注释还提到：

- `engine.waitForIdle`

这说明系统里存在某些等待者，是在“等 teammate 进入 idle”这个事件的。

而 kill 时如果不主动触发这些回调，就可能把等待者永久卡住。

所以这里体现的是：

```text
终止不仅要停 worker，还要唤醒所有等待这个 worker 状态变化的观察者
```

这是很成熟的并发生命周期处理方式。

---

### 为什么 kill 路径会保留最后一条消息
关键位置：`src/utils/swarm/spawnInProcess.ts:288-290`

这里把 `messages` 收缩成：

- 只保留最后一条

这说明它在两件事情间取平衡：

- 不想在 killed task 上保留大量历史，避免状态膨胀
- 但又想保留最后一个可见上下文，给 UI / 用户一点收尾信息

也就是说：

```text
killed task 不是完全清空，而是保留最小可见痕迹
```

这很像一种轻量 tombstone 设计。

---

### 这个文件体现出的 8 个架构事实

1. `spawnInProcess.ts` 负责 in-process teammate 的出生和终止管理，而不是完整执行 loop。
2. teammate 的 identity 与 teammateContext 在 spawn 时就被拆成“可持久数据”和“执行期上下文”两层。
3. in-process teammate 有自己的独立 AbortController，不跟随 leader query interruption 一起终止。
4. `planModeRequired` 会直接决定初始 `permissionMode`，这是出生配置的一部分。
5. in-process teammate 会被注册进全局 cleanup registry，参与整个 session 的 graceful shutdown。
6. spawn 成功返回的不只是 success，而是一组供 runner 继续启动的句柄。
7. kill 路径会同时清理 AppState、team file、SDK 事件、磁盘输出、terminal 展示和 tracing 注册。
8. kill 时还会唤醒 idle waiters，避免等待中的协作逻辑被永久卡死。

---

### 现在把第 20-21 站串起来

到这里，in-process 这条链又推进了一层：

```text
InProcessTeammateTask.tsx
  -> 定义 task state 结构与状态操作 helper

spawnInProcess.ts
  -> 创建 agentId / teammateContext / abortController / 初始 task state
  -> registerTask(...) 进 AppState
  -> kill 时做内存态、持久化态、观测态的统一收尾
```

所以现在可以更准确地区分：

```text
InProcessTeammateTask.tsx = 状态操作层
spawnInProcess.ts = 生命周期编排层
```

---

### 这一站最该记住的 8 句话

1. `spawnInProcess.ts` 是 in-process teammate 的 spawn / kill 控制层。
2. 它在 spawn 时就同时准备好 identity、teammateContext、abortController 和初始 task state。
3. teammateContext 是给 AsyncLocalStorage 执行层用的，而 identity 是给 AppState 持久表示用的。
4. in-process teammate 拥有独立 abort 语义，不会因为 leader query 被打断而必然终止。
5. `planModeRequired` 会直接决定初始 `permissionMode`。
6. spawn 成功返回的是一组“runner 启动包”，而不只是布尔值。
7. kill 路径是集中式收尾器，会清理状态、团队元数据和可观测性记录。
8. kill 时会主动唤醒 idle waiters，避免上层协作逻辑卡死。

---

### 下一站建议

下一站最顺的是：

```text
src/utils/inProcessTeammateHelpers.ts
```

原因：

- 你已经看到了 in-process teammate 的 task 状态层和 spawn/kill 生命周期层
- 前面在 `useInboxPoller.ts` 里也见过 `findInProcessTeammateTaskId(...)`、`handlePlanApprovalResponse(...)`

下一步就该看：

**leader 与 in-process teammate 之间的计划审批响应、task 查找、状态桥接 helper 到底是如何补上 mailbox 之外那条协作路径的。**

---

---

### 元

问题：**这一站真正想解决的架构问题是什么？**

回答：这一站负责 in-process teammate 的出生与终止，生成 agentId、taskId、AbortController、identity 和 teammateContext，并在 kill 时做集中清理。它是生死边界，而不是完整 runner。

### 反

问题：**如果把这一站的设计反过来，会发生什么？**

回答：如果 spawn 和 kill 没有集中管理，身份注册、cleanup handler、team file 更新会散落各处。最后最容易出问题的是“死不干净”。

### 空

问题：**跳出当前文件名，这一站背后更大的问题是什么？**

回答：更大的问题是，一个共享进程 worker 的生命周期边界应该在哪里被定义。这里明确把“创建对象”和“持续运行”分成了两个层次。

## 第 22 站：`src/utils/inProcessTeammateHelpers.ts`

### 这是什么文件

`src/utils/inProcessTeammateHelpers.ts` 是 in-process teammate 协作链路里的**轻量桥接 helper 层**。

上一站 `spawnInProcess.ts` 已经让你看到：

```text
in-process teammate 如何出生、如何有 task state、如何被 kill
```

而这一站解决的是：

```text
leader 或其他运行时模块在处理 in-process teammate 相关事件时，
怎样通过一组很小的 helper 把 plan approval、task 查找、permission response 识别这些动作接起来
```

所以一句话先记：

```text
inProcessTeammateHelpers.ts = in-process teammate 的状态桥接小工具层
```

它不负责执行、不负责消息总线、不负责 backend 选路，而是专门负责把一些跨模块小动作标准化。

---

### 这个文件整体上很“小”，但位置很关键

读完会发现，这里没有复杂逻辑，只有 4 个导出 helper：

- `findInProcessTeammateTaskId(...)`
- `setAwaitingPlanApproval(...)`
- `handlePlanApprovalResponse(...)`
- `isPermissionRelatedResponse(...)`

所以它的价值不在复杂度，而在：

```text
把 scattered 的 in-process teammate 小动作收口成统一 helper，
避免别的模块直接手搓 task 遍历和 message 识别细节
```

这正是桥接层常见的职责。

---

### 关键职责 1：按 agentName 找 taskId
关键位置：`src/utils/inProcessTeammateHelpers.ts:33`

`findInProcessTeammateTaskId(...)` 做的事很直接：

- 遍历 `appState.tasks`
- 找出 `isInProcessTeammateTask(task)`
- 再匹配 `task.identity.agentName === agentName`
- 返回 `task.id`

这说明 in-process teammate 的某些运行时协作路径，并不是先拿到 taskId，而是先拿到：

```text
agentName
```

例如前面 `useInboxPoller.ts` 里 leader 处理 plan approval request 后，就会按 teammate 名字去找它在 AppState 里的 task。

所以这个 helper 解决的是：

```text
从“队友名字”跳到“任务槽位”
```

这是一条非常典型的 UI / runtime 桥接动作。

---

### 为什么这里按 `agentName` 而不是 `agentId`

这个细节很值得注意。

前面很多底层逻辑更偏向 `agentId`，但这里偏偏用 `agentName`。

这说明在某些上层事件里，消息来源方携带的是：

- teammate 可见名字

而不是底层格式化后的完整 `agentId`。

所以 helper 的意义之一就是：

```text
把上层协作语义里的“人类可见身份”映射回 task 内部表示
```

这和 `useInboxPoller.ts` 里的 `m.from` / mailbox sender name 语义是对上的。

---

### 关键职责 2：`awaitingPlanApproval` 有专门 setter
关键位置：`src/utils/inProcessTeammateHelpers.ts:55`

`setAwaitingPlanApproval(...)` 很简单：

- 调 `updateTaskState(...)`
- 把 `awaitingPlanApproval` 改成传入布尔值

但它的重要性在于，它把：

```text
“teammate 当前是否在等 leader 批 plan”
```

做成了一个明确的 task state 标志，并且给了专门 helper。

这说明 plan approval 在 in-process teammate 体系里不是临时消息状态，而是：

```text
一个显式可观察的任务状态位
```

这让 UI、runner、leader-side 协作逻辑都能围绕同一个状态字段工作。

---

### 关键职责 3：plan approval response 在这里先只负责清掉等待态
关键位置：`src/utils/inProcessTeammateHelpers.ts:77`

`handlePlanApprovalResponse(...)` 的实现极薄：

- 直接调用 `setAwaitingPlanApproval(taskId, setAppState, false)`

而注释非常关键：

```text
The permissionMode from the response is handled separately by the agent loop
```

这说明 plan approval 响应在系统里被**拆成了两部分处理**：

#### 这一层负责：
- task UI / 状态层面的等待标志收口

#### 另一层负责：
- 真正把 `permissionMode` 应用到 agent loop / 执行上下文

这点非常重要，因为它说明系统没有把所有“收到 plan approval response 后要做的事”都堆在一个 helper 里，而是显式分层：

```text
状态桥接层 -> 清 waiting flag
执行层 -> 吃 permissionMode 并继续跑
```

这种分层是很健康的。

---

### 这也解释了为什么 `useInboxPoller.ts` 要单独 special-case in-process teammate

前面在 `useInboxPoller.ts` 里读到：

- mailbox consumer 会在某些 plan approval 流程里找到 in-process teammate task
- 然后调用 `handlePlanApprovalResponse(...)`

现在就更清楚了：

```text
mailbox/runtime 层只负责把“response 已到达”这件事桥接进 task 状态层
真正的 mode 变更和继续执行，不在这个 helper 里做
```

所以这个文件正是那条 bridge 的一段薄薄接缝。

---

### 关键职责 4：识别 permission-related response
关键位置：`src/utils/inProcessTeammateHelpers.ts:97`

`isPermissionRelatedResponse(messageText)` 会检查：

- `isPermissionResponse(messageText)`
- `isSandboxPermissionResponse(messageText)`

只要任一命中，就返回 true。

这说明 in-process teammate 的消息处理链里，并不只关心 plan approval，也关心：

```text
普通 tool permission response
sandbox/network permission response
```

也就是说，in-process teammate 虽然不走 `useInboxPoller.ts` 的常规收信路径，但它依然需要识别和处理一部分 leader 回发的权限类消息。

所以这个 helper 的角色是：

```text
把多种“权限响应消息”折叠成一个上层可复用布尔判断
```

让调用方不必自己重复写两种 parser 的 OR 逻辑。

---

### 这个文件真正的边界

读完后要非常清楚，它不是：

- mailbox 层
- message parser 定义层
- in-process runner 主循环
- task state 结构定义层

它只是在这些层之间补小桥。

所以更精确地说：

```text
teammateMailbox.ts = 消息协议与解析边界
InProcessTeammateTask.tsx = task 状态层
inProcessTeammateHelpers.ts = 两者之间的胶水层
```

这正是为什么它代码不多，但依然值得单独读。

---

### 这个文件体现出的 6 个架构事实

1. in-process teammate 也需要一层专门的 helper 来做状态桥接，而不是让调用方到处直接改 task。
2. 某些协作事件天然拿到的是 `agentName`，所以需要 helper 把名字映射回 taskId。
3. `awaitingPlanApproval` 是一个显式 task 状态位，而不是隐含在消息流里的临时状态。
4. plan approval response 的处理被拆层：helper 只清等待态，permissionMode 由执行层另行处理。
5. in-process teammate 仍然需要识别 tool permission 和 sandbox permission 两类 leader 响应。
6. 这个文件的职责是“胶水层标准化”，而不是执行逻辑本身。

---

### 现在把第 20-22 站串起来

到这里，in-process teammate 这条支线可以拆成三层：

```text
InProcessTeammateTask.tsx
  -> task state 结构与基本状态操作

spawnInProcess.ts
  -> spawn / kill 生命周期编排

inProcessTeammateHelpers.ts
  -> 按名字找 task、plan approval waiting 状态桥接、permission response 识别
```

所以可以更清楚地理解为：

```text
Task 层
  -> 存状态
Lifecycle 层
  -> 生与死
Glue 层
  -> 把 leader/runtime 事件接进 in-process teammate 状态
```

这三层一起，in-process teammate 的控制面已经很清楚了。

---

### 这一站最该记住的 7 句话

1. `inProcessTeammateHelpers.ts` 是 in-process teammate 的轻量桥接 helper 层。
2. 它解决的不是执行，而是“其他模块如何更方便地接入 in-process teammate 状态”。
3. `findInProcessTeammateTaskId(...)` 把 teammate 名字映射回 taskId。
4. `awaitingPlanApproval` 是专门暴露出来的显式任务状态位。
5. `handlePlanApprovalResponse(...)` 只负责把等待态清掉，不负责真正应用 permissionMode。
6. `isPermissionRelatedResponse(...)` 把普通权限响应和 sandbox 权限响应统一成一个上层判断。
7. 这个文件的价值在于把 mailbox/runtime 事件与 task 状态层之间的接缝标准化。

---

### 下一站建议

下一站最顺的是：

```text
src/tasks/InProcessTeammateTask/types.ts
```

原因：

- 你已经看到了 task 操作层、lifecycle 层、glue helper 层
- 现在最自然就是把它们共同操作的 `InProcessTeammateTaskState` 类型本体补齐

下一步就该看：

**in-process teammate task 到底有哪些字段、哪些字段是展示用、哪些字段是执行用、哪些字段是协作用。**

---

---

### 元

问题：**这一站真正想解决的架构问题是什么？**

回答：这一站很小，但专门解决 in-process teammate 的几处关键桥接动作，比如按 agentName 找 taskId、处理 plan approval response 和识别 permission-related response。它是在消息世界与 task 世界之间做细小转换。

### 反

问题：**如果把这一站的设计反过来，会发生什么？**

回答：如果这些小动作散落在各上层模块里，每个地方都要自己遍历 tasks、识别消息类型。逻辑虽小，却会制造很多重复和偏差。

### 空

问题：**跳出当前文件名，这一站背后更大的问题是什么？**

回答：更大的问题是，系统里那些看似不大的“跨模块胶水动作”应该放哪。把它们收成 helper，往往比继续塞进上层流程更稳。

## 第 23 站：`src/tasks/InProcessTeammateTask/types.ts`

### 这是什么文件

`src/tasks/InProcessTeammateTask/types.ts` 是 in-process teammate 的**状态模型定义文件**。

前面几站已经反复操作这些字段：

- `InProcessTeammateTask.tsx` 在改 task 状态
- `spawnInProcess.ts` 在初始化 task state
- `inProcessTeammateHelpers.ts` 在桥接 plan approval / permission response

而这一站终于把问题收口成一句话：

```text
in-process teammate 在 AppState 里到底长什么样
```

所以一句话先记：

```text
types.ts = in-process teammate 的状态 schema / 语义分层图
```

它代码很短，但因为前面很多文件都在围着它转，所以这一站反而是补全理解闭环的关键。

---

### 最核心的两个类型

#### 1. `TeammateIdentity`
位置：`src/tasks/InProcessTeammateTask/types.ts:13`

这个类型保存：

- `agentId`
- `agentName`
- `teamName`
- `color`
- `planModeRequired`
- `parentSessionId`

注释已经说得很明确：

- 它和 `TeammateContext` 形状相近
- 但这里存的是**plain data**
- 用于 AppState persistence
- 不是 AsyncLocalStorage 里的运行时对象引用

这和前面 `spawnInProcess.ts` 的结论完全对上：

```text
identity = 持久化数据视图
teammateContext = 执行期上下文视图
```

所以 `TeammateIdentity` 的真正作用是：

```text
给 UI / AppState / task system 一个稳定、可序列化的 teammate 身份表示
```

---

#### 2. `InProcessTeammateTaskState`
位置：`src/tasks/InProcessTeammateTask/types.ts:22`

这是整条 in-process 支线最核心的状态类型。

它是：

- `TaskStateBase`
- 再加上一整组 teammate 专属字段

所以你可以先把它理解成：

```text
通用 task 骨架 + in-process teammate 扩展状态
```

这说明 in-process teammate 并没有绕开统一 task 框架，而是标准 task 的一种专门变体。

---

### 这个类型可以明显分成 6 组字段

读这个文件最好的方式，不是按声明顺序逐个背字段，而是按语义分组。

---

### 第 1 组：身份层
关键位置：`src/tasks/InProcessTeammateTask/types.ts:25-27`

最核心的是：

- `identity: TeammateIdentity`

注释也再次强调：

- shape 跟 `TeammateContext` 对齐
- 但这里只存 plain data

这说明设计者很刻意地把：

```text
“这个 teammate 是谁”
```

单独抽成一个子对象，而不是把 agentId / teamName / color 散落在顶层。

好处很明显：

- identity 相关字段聚合
- 和 runtime context 结构保持心智一致
- 未来跨模块传递更容易

所以这个子对象其实就是 in-process teammate task 的**身份头部**。

---

### 第 2 组：执行配置层
关键位置：`src/tasks/InProcessTeammateTask/types.ts:29-45`

这里包括：

- `prompt`
- `model?`
- `selectedAgent?`
- `abortController?`
- `currentWorkAbortController?`
- `unregisterCleanup?`
- `awaitingPlanApproval`
- `permissionMode`

这一组最关键，因为它把“怎么跑”和“当前执行控制状态”都放在一起了。

其中尤其要记 4 个点：

#### A. `selectedAgent?`
说明不是所有 in-process teammate 都绑定某个预定义 agent definition。

注释写得很清楚：

- 很多 teammate 只是 general-purpose agent
- 没有预定义 definition

这说明 Claude Code 的 in-process teammate 模型兼容两种来源：

```text
预定义 agent
vs
临时 general-purpose teammate
```

#### B. `abortController?`
注释写得非常明确：

- aborts WHOLE teammate

也就是整个 teammate 生命周期级别的终止器。

#### C. `currentWorkAbortController?`
这又是一层更细的控制：

- aborts current turn without killing teammate

这非常关键，说明 in-process teammate 至少有两级中断语义：

```text
whole teammate abort
vs
current work / current turn abort
```

这正是“可持续协作者”与“一次性任务”最大的区别之一。

#### D. `awaitingPlanApproval` + `permissionMode`
这两个字段一起说明：

- plan approval 状态是显式的
- permission mode 也是 teammate 独立维护的

也就是说，in-process teammate 不只是共享 leader 的权限模型，而是拥有自己的 mode 状态面。

---

### 第 3 组：结果与进度层
关键位置：`src/tasks/InProcessTeammateTask/types.ts:46-49`

这里包括：

- `error?`
- `result?`
- `progress?`

很有意思的是，`result` 直接复用了：

- `AgentToolResult`

注释写明：

- teammates run via `runAgent()`

这说明从结果模型角度看，in-process teammate 并没有再造一套全新结果类型，而是：

```text
直接复用 AgentTool 那套结果语义
```

这也再次证明 swarm teammate 与 AgentTool 在底层执行能力上是高度同构的。

而 `progress?: AgentProgress` 也说明：

```text
teammate 的进度模型和本地 agent task 的进度模型是复用的
```

所以这一组代表的是：

```text
执行输出面 / 可观测进度面
```

---

### 第 4 组：会话与展示层
关键位置：`src/tasks/InProcessTeammateTask/types.ts:51-63`

这里包括：

- `messages?`
- `inProgressToolUseIDs?`
- `pendingUserMessages`
- `spinnerVerb?`
- `pastTenseVerb?`

这是最容易混在一起的一组，但其实也能拆开。

#### A. `messages?`
注释非常关键：

- 这是给 zoomed view 的 conversation history
- **不是 mailbox messages**
- mailbox messages 在 `teamContext.inProcessMailboxes`

这句话必须记住。

它说明：

```text
task.messages ≠ mailbox
```

`task.messages` 是 UI transcript 镜像；而 in-process mailbox/消息通信另有存放位置。

#### B. `inProgressToolUseIDs?`
这是 transcript view 动画/展示用状态。

说明 task state 里不只放纯业务逻辑字段，还放了一部分 UI 需要的瞬时展示状态。

#### C. `pendingUserMessages`
这个字段前面已经见过很多次了：

- 这是“发给 teammate 但尚未被执行层消费”的待处理输入队列

它本质上是：

```text
leader / UI -> teammate runner 的输入缓冲区
```

#### D. `spinnerVerb?` / `pastTenseVerb?`
这是纯 UI 文案状态，但却被写进 task state。

原因也很合理：

```text
要在跨组件、多次 render 中保持稳定
```

所以这组字段总体上说明：

```text
InProcessTeammateTaskState 不只是执行状态，
也是 transcript/UI 表示层的一部分数据源
```

---

### 第 5 组：生命周期层
关键位置：`src/tasks/InProcessTeammateTask/types.ts:65-71`

这里包括：

- `isIdle`
- `shutdownRequested`
- `onIdleCallbacks?`

这一组非常关键，因为它直接反映“这个 teammate 当前处在什么生命周期相位”。

#### A. `isIdle`
这不是 task status 的替代，而是另一条维度。

前面已经看到：

```text
running != busy
```

所以这里再次确认：

- `status` 代表任务生命周期是否还活着
- `isIdle` 代表当前是否在等待工作

#### B. `shutdownRequested`
这表示 shutdown 是一个显式的等待消费信号。

不是立刻 kill，而是先把关闭请求存入状态里。

#### C. `onIdleCallbacks?`
注释写得很直接：

- Used by leader to efficiently wait without polling

也就是说，leader 可以不是反复轮询，而是把 callback 注册进去，等 teammate 真正 idle 时被唤醒。

所以这组字段的本质是：

```text
生命周期协调层
```

---

### 第 6 组：通知 delta 计算层
关键位置：`src/tasks/InProcessTeammateTask/types.ts:73-75`

这里包括：

- `lastReportedToolCount`
- `lastReportedTokenCount`

这两个字段很小，但很能说明问题。

它们不是核心业务字段，而是为了：

```text
做 progress/notification 的增量计算
```

也就是说系统不会每次都重新从 0 解释完整累计值，而是会记住“上次已经报告到哪里”，再算 delta。

这是一种很典型的 UI/通知优化状态。

---

### `isInProcessTeammateTask(...)` 的价值
位置：`src/tasks/InProcessTeammateTask/types.ts:78`

这个 type guard 很朴素：

- 是 object
- 非 null
- 有 `type`
- `type === 'in_process_teammate'`

但它其实是前面很多 helper 的基础：

- `findInProcessTeammateTaskId(...)`
- `findTeammateTaskByAgentId(...)`
- `getAllInProcessTeammateTasks(...)`

说明这一层虽然简单，却是：

```text
in-process teammate 支线进入 task 多态世界的最小门槛
```

---

### `TEAMMATE_MESSAGES_UI_CAP = 50` 为什么尤其值得记
位置：`src/tasks/InProcessTeammateTask/types.ts:101`

这段注释信息量很大，值得重点记。

核心意思是：

- `task.messages` 只是 UI 镜像
- 完整对话在 runner 本地 `allMessages` 和 transcript 文件里
- 如果把完整副本一直挂在 AppState，会非常吃内存
- 真实线上分析发现大规模 swarm 时 RSS 很夸张
- 曾出现 292 agents / 36.8GB 这种极端会话

所以这里把 UI 镜像 cap 成：

- `50`

它背后的架构结论非常重要：

```text
AppState 里的 teammate transcript 是“展示缓存”，
不是完整真相源
```

这也说明这个类型文件不仅定义字段，还内置了对内存上限的明确设计约束。

---

### `appendCappedMessage(...)` 的真正作用
位置：`src/tasks/InProcessTeammateTask/types.ts:108`

这个函数会：

- 若无旧数组 -> `[item]`
- 若已达到 cap -> 丢掉最旧的，只保留最近 `49` 条再追加新项
- 否则正常 append

而且始终返回新数组，保持 AppState immutability。

这里值得记住的是：

```text
message cap 并不是调用方各自记得遵守，
而是被收口成统一 helper 强制执行
```

这避免了不同调用点自己裁剪时出现不一致。

---

### 这个文件体现出的 8 个架构事实

1. `InProcessTeammateTaskState` 是在通用 `TaskStateBase` 上扩展出来的专用 teammate 变体。
2. teammate 身份被拆成 `identity` 子对象，与运行时 `TeammateContext` 保持形状一致但语义不同。
3. in-process teammate 至少有两级中断语义：整个 teammate abort 和当前工作 abort。
4. teammate 独立维护自己的 `permissionMode` 与 `awaitingPlanApproval` 状态。
5. `messages` 是 UI transcript 镜像，不是 mailbox，也不是完整历史真相源。
6. `pendingUserMessages` 是 leader/UI 到 teammate runner 的待消费输入缓冲区。
7. `isIdle`、`shutdownRequested`、`onIdleCallbacks` 共同构成生命周期协调面。
8. `TEAMMATE_MESSAGES_UI_CAP` 明确体现了这个状态模型对大规模 swarm 内存占用的防御设计。

---

### 现在把第 20-23 站串起来

到这里，in-process teammate 支线已经能形成一个很清晰的结构：

```text
types.ts
  -> 定义状态模型本体

InProcessTeammateTask.tsx
  -> 提供状态操作 helper

spawnInProcess.ts
  -> 负责 spawn / kill 生命周期编排

inProcessTeammateHelpers.ts
  -> 把 leader/runtime 事件桥接进 task 状态
```

所以现在可以把这条线概括成：

```text
state schema
  -> types.ts
state ops
  -> InProcessTeammateTask.tsx
lifecycle
  -> spawnInProcess.ts
glue
  -> inProcessTeammateHelpers.ts
```

这四层合起来，in-process teammate 的状态控制面已经非常清楚了。

---

### 这一站最该记住的 8 句话

1. `types.ts` 定义了 in-process teammate 在 AppState 里的完整状态模型。
2. `identity` 是持久化身份视图，不是 AsyncLocalStorage 的运行时上下文对象。
3. in-process teammate 同时拥有 whole-teammate abort 和 current-work abort 两级中断控制。
4. `permissionMode` 与 `awaitingPlanApproval` 说明 teammate 有独立权限/计划审批状态面。
5. `messages` 只是 UI transcript 镜像，不代表完整消息真相源。
6. `pendingUserMessages` 是发给 teammate 但尚未被执行层消费的输入队列。
7. `isIdle` / `shutdownRequested` / `onIdleCallbacks` 共同描述生命周期协调状态。
8. `TEAMMATE_MESSAGES_UI_CAP = 50` 是为大规模 swarm 内存压力专门做的硬性约束。

---

### 下一站建议

下一站最顺的是：

```text
src/utils/swarm/inProcessRunner.ts
```

原因：

- 你已经把 state schema、状态操作、spawn/kill、glue helper 都补齐了
- 现在最缺的就是“谁真正消费这些状态字段并驱动 teammate 跑起来”

下一步就该看：

**真正的 in-process runner 如何读取 `pendingUserMessages`、处理 idle/active 切换、等待 plan approval、执行 agent loop。**

---

---

### 元

问题：**这一站真正想解决的架构问题是什么？**

回答：这一站定义 in-process teammate 在 AppState 里的状态模型，区分 `TeammateIdentity` 这类可持久化 plain data 和运行中的任务扩展状态。它是多站协作后终于显形的 schema 中心。

### 反

问题：**如果把这一站的设计反过来，会发生什么？**

回答：如果状态模型不清晰，上层 helper、runner 和 UI 都会各自理解 teammate。字段语义一旦漂移，问题不会立刻爆炸，但会持续积累。

### 空

问题：**跳出当前文件名，这一站背后更大的问题是什么？**

回答：更大的问题是，运行时实体怎样被压缩成稳定可序列化的状态视图。schema 设计看似静态，实际上决定了后续所有协作边界。

## 第 24 站：`src/utils/swarm/inProcessRunner.ts`

### 这是什么文件

`src/utils/swarm/inProcessRunner.ts` 是 in-process teammate 支线里真正的**执行引擎**。

前面几站已经把几层基础都铺好了：

- `types.ts` 定义状态模型
- `InProcessTeammateTask.tsx` 提供 task 层状态操作
- `spawnInProcess.ts` 负责 spawn / kill 生命周期
- `inProcessTeammateHelpers.ts` 负责桥接 plan / permission 响应

而这一站终于把链路闭合成一句话：

```text
inProcessRunner.ts = 真正消费这些状态、驱动 teammate 持续运行的主循环
```

它不是简单的“调用一次 `runAgent()` 就结束”。

它做的是一个更完整的运行时职责包：

- 把 teammate 放进独立上下文里执行
- 给 teammate 接上权限询问通道
- 跑真正的 agent loop
- 把进度与消息镜像回 AppState
- 在 idle 时等待下一条 prompt / shutdown 请求 / task list 任务
- 必要时做 compact、清理、终止上报

所以这一站本质上是在回答：

```text
in-process teammate 到底是怎么从“一次 spawn”变成“可持续协作 worker”的
```

---

### 先抓住这个文件的 5 个角色

#### 1. 权限适配器
位置：`src/utils/swarm/inProcessRunner.ts:128`

`createInProcessCanUseTool(...)` 把 teammate 的 tool permission 决策接到了 leader 侧的真实权限体系上。

也就是说，in-process teammate 虽然和 leader 跑在同一进程里，但它并不是直接绕过权限系统，而是走一层专门的 teammate 适配。

---

#### 2. 空闲等待器
位置：`src/utils/swarm/inProcessRunner.ts:689`

`waitForNextPromptOrShutdown(...)` 负责在 teammate 做完一轮工作后继续活着，并等待：

- 新的用户消息
- mailbox 消息
- shutdown 请求
- 共享 task list 中的新任务

这一层决定了它不是“一次性 agent”，而是“持续驻留 worker”。

---

#### 3. 主执行循环
位置：`src/utils/swarm/inProcessRunner.ts:883`

`runInProcessTeammate(...)` 是真正的 runner 主体。

它会：

- 组装 system prompt
- 构造 agent definition
- 建立 conversation buffer
- 反复执行 `runAgent()`
- 每轮结束后转入 idle 等待
- 收到新 prompt 再进入下一轮

所以这就是整个 in-process teammate 的 runtime heart。

---

#### 4. AppState 镜像同步器
它在多处通过 `updateTaskState(...)` 把运行时事件同步回 task state：

- progress
- transcript 镜像
- in-progress tool use IDs
- current work abort controller
- idle / running 切换
- terminal cleanup

这说明 runner 不只是执行器，还是 task/UI 可观测面的维护者。

---

#### 5. 收尾与存活管理器
它还负责：

- 失败时发 idle/failure notification
- 成功/失败时 evict output 与 terminal task
- 关闭 SDK bookend
- 注销 perfetto agent

所以这个文件也承担了完整的**执行生命周期闭环**。

---

### `createInProcessCanUseTool(...)` 在解决什么问题
位置：`src/utils/swarm/inProcessRunner.ts:128`

这段是整站最关键的入口之一。

前面我们已经知道：

- in-process teammate 也会调用工具
- 工具权限不能凭空跳过
- 但它又不像真正后台子进程那样只能完全依赖文件通信

所以这里要解决的核心问题是：

```text
同进程 teammate 如何在不破坏统一权限模型的前提下，向 leader 请求工具授权
```

它的策略分成三层。

#### 第一层：先跑标准 permission check
它先调用：

- `hasPermissionsToUseTool(...)`

如果结果已经是 `allow` / `deny`，就直接返回。

这说明 teammate 并没有旁路特殊规则，而是仍然复用主权限引擎。

#### 第二层：对 Bash 先尝试 classifier auto-approval
位置：`src/utils/swarm/inProcessRunner.ts:156-176`

如果：

- feature 开启 `BASH_CLASSIFIER`
- 当前工具是 Bash
- `result.pendingClassifierCheck` 存在

那么它会先等待 `awaitClassifierAutoApproval(...)`。

而且注释点得很明白：

- teammate 会**等待 classifier 结果**
- 而不是像主线程那样和用户交互竞争 race

这说明 in-process teammate 的 Bash 审批策略比 leader 主循环更偏“同步等待型”。

#### 第三层：ask 时优先走 leader 的 ToolUseConfirm 队列
位置：`src/utils/swarm/inProcessRunner.ts:195`

如果 leader 侧可用：

- `getLeaderToolUseConfirmQueue()`

那就把一个带 `workerBadge` 的确认项塞进 leader 的 UI queue。

这非常关键，因为它意味着：

```text
in-process teammate 的权限确认弹窗，复用了 leader 自己那套 ToolUseConfirm UI 管道
```

所以 Bash diff、FileEdit diff 等 UI 能力，不需要为 teammate 再单独做一套。

#### UI 队列分支里还做了 3 件重要的事

1. 记录 permission wait 时间，并回传给调用方，用来从 elapsed time 中扣掉等待审批的时间。
2. `onAllow(...)` 不只返回 allow，还会：
   - `persistPermissionUpdates(...)`
   - 通过 `getLeaderSetToolPermissionContext()` 把 permission update 写回 leader 的共享 context
   - 并且显式 `preserveMode: true`，防止 worker 的 transformed context 污染 coordinator 的 mode
3. `recheckPermission()` 支持重新检查，如果审批期间规则已经变化，可能无需继续弹窗就自动 allow。

这说明这个分支不是“简单弹个框”，而是完整接进 leader 权限状态机。

---

### mailbox fallback 为什么仍然保留
位置：`src/utils/swarm/inProcessRunner.ts:336`

如果 UI bridge 不可用，它会退回到 mailbox 模式。

这里的流程是：

1. `createPermissionRequest(...)` 构造请求
2. `registerPermissionCallback(...)` 注册本地回调
3. `sendPermissionRequestViaMailbox(request)` 发给 leader
4. 每 500ms 轮询 teammate 自己的 mailbox
5. 收到对应 `request.id` 的 permission response 后：
   - 标记已读
   - `processMailboxPermissionResponse(...)`
   - 由 callback resolve promise

这里要特别记住两点。

#### 1. 同进程 teammate 仍然保留“像进程外 worker 一样”的降级通信路径
这说明架构上非常强调：

```text
UI bridge 是优化路径，不是唯一真相路径
```

#### 2. mailbox fallback 和 tmux teammate 的协议是对齐的
所以无论 backend 是 tmux 还是 in-process，leader 侧都能复用同一套 permission request/response 协议。

这就是前面多站一直在建立的那条统一 swarm 协议线。

---

### `formatAsTeammateMessage(...)` 为什么很重要
位置：`src/utils/swarm/inProcessRunner.ts:457`

这个函数把消息包装成：

```xml
<teammate-message teammate_id="..." color="..." summary="...">
...
</teammate-message>
```

它的意义不是格式漂亮，而是：

```text
让 in-process teammate 看到的消息格式，与 tmux teammate 保持一致
```

这会直接影响两件事：

- 模型读取消息时的角色/来源理解一致性
- transcript view / message rendering 的协议一致性

所以这个函数其实是协议统一层，不是简单字符串 helper。

---

### `findAvailableTask(...)` / `tryClaimNextTask(...)` 说明了什么
位置：`src/utils/swarm/inProcessRunner.ts:595`, `src/utils/swarm/inProcessRunner.ts:624`

这两个函数把 in-process teammate 接到了 shared task list 上。

#### `findAvailableTask(...)`
可领取任务的条件是：

- `status === 'pending'`
- 没有 `owner`
- `blockedBy` 里没有未完成依赖

这里很值得注意的一点是：

- 它先构造 `unresolvedTaskIds`
- 然后用它判断依赖是否仍未解开

这说明 task availability 的判断不是只看当前 task 自己，而是看整个 task list 的未完成集合。

#### `tryClaimNextTask(...)`
它会：

- `listTasks(taskListId)`
- 找到第一个可领取任务
- `claimTask(...)`
- 成功后再 `updateTask(..., { status: 'in_progress' })`
- 最后把任务格式化成 prompt

这里有两个重要的架构点：

1. claim 和 status 更新是分开的两步，所以 runner 会显式再补一次 `in_progress`，让 UI 立即同步。
2. 返回值不是 task 对象，而是 `formatTaskAsPrompt(...)` 产物，说明 task list 最终会被注入为新 prompt 进入 agent loop。

也就是说：

```text
task list -> runner claim -> prompt text -> runAgent()
```

这是 team task system 与 agent loop 的连接点。

---

### `waitForNextPromptOrShutdown(...)` 是这一站最该看透的函数
位置：`src/utils/swarm/inProcessRunner.ts:689`

这个函数定义了 teammate 在 idle 态到底怎么活着。

它每 500ms poll 一次，但内部优先级是精心排过的。

#### 优先级 1：先看 `pendingUserMessages`
位置：`src/utils/swarm/inProcessRunner.ts:705-739`

它每次循环先读 AppState 中当前 task：

- 如果 `pendingUserMessages.length > 0`
- 就取第一条
- 再把队列头 pop 掉
- 返回 `type: 'new_message'`，`from: 'user'`

这说明：

```text
直接来自 transcript / UI 注入的用户消息，优先级高于 mailbox
```

也就是说，查看 transcript 时用户直接继续对 teammate 说话，是最快路径，不需要先绕回文件邮箱。

#### 优先级 2：shutdown request 高于普通 mailbox 消息
位置：`src/utils/swarm/inProcessRunner.ts:760-804`

它会先扫描所有 unread mailbox，优先找：

- `isShutdownRequest(m.text)`

并且源码注释已经明确说了：

- shutdown request 优先于 regular messages
- 防止 peer-to-peer chatter 让 shutdown 饥饿

这是个很重要的 runtime 决策。

#### 优先级 3：team-lead 消息高于 peer message
位置：`src/utils/swarm/inProcessRunner.ts:806-845`

如果没有 shutdown，请求下一层是：

- 先找 `m.from === TEAM_LEAD_NAME`
- 否则才退化为 first unread message

这说明消息优先级不是纯 FIFO，而是：

```text
shutdown > team-lead > peer FIFO
```

这是整个 teammate 空闲消息调度最关键的排序规则。

#### 优先级 4：没有消息时再去 task list 抢任务
位置：`src/utils/swarm/inProcessRunner.ts:853-861`

如果 mailbox 没东西，再去：

- `tryClaimNextTask(taskListId, identity.agentName)`

这说明 task list 不是压过用户和协调消息的最高优先级，而是 idle 时的补充工作来源。

所以整个 idle loop 的心智模型可以总结成：

```text
pendingUserMessages
  > shutdown mailbox message
  > team-lead mailbox message
  > peer mailbox message
  > task-list claim
```

这就是 in-process teammate 的待机调度策略。

---

### `runInProcessTeammate(...)` 如何把一切串起来
位置：`src/utils/swarm/inProcessRunner.ts:883`

这是全文件主角。

可以把它拆成 7 个阶段。

#### 阶段 1：构造 AgentContext 与 system prompt
位置：`src/utils/swarm/inProcessRunner.ts:908-1001`

它先构造：

- `AgentContext`

用于 analytics attribution。

然后按 `systemPromptMode` 组装 teammate system prompt：

- `replace`：完全替换
- 默认：`getSystemPrompt(...) + TEAMMATE_SYSTEM_PROMPT_ADDENDUM`
- 若有 custom agent prompt，再追加 `# Custom Agent Instructions`
- `append`：在默认 prompt 后再拼额外 prompt

这里还会在 custom agent 有 memory 时上报：

- `tengu_agent_memory_loaded`

说明 in-process teammate 不是简化版 agent，而是能吃到完整 prompt / memory / analytics 体系的。

#### 阶段 2：构造 `resolvedAgentDefinition`
位置：`src/utils/swarm/inProcessRunner.ts:972-1001`

这里有个非常重要的注释：

- `permissionMode` 初始设成 `'default'`
- 目的是让 teammate 不受 leader 当前 permission mode 的直接限制

但这并不意味着 teammate 永远固定 default。

因为后面每轮真正执行前，会从当前 task state 读取 `permissionMode`，再构造 `iterationAgentDefinition`。

也就是说：

```text
resolvedAgentDefinition = 基础模板
iterationAgentDefinition = 每轮按当前 task state 注入实时 permissionMode 的执行体
```

另外它还会强制注入 team-essential tools：

- `SendMessage`
- `TeamCreate`
- `TeamDelete`
- `TaskCreate`
- `TaskGet`
- `TaskList`
- `TaskUpdate`

即使显式限制了 tools，这些团队协作核心工具也会被补进去。

这个设计非常关键，因为否则 teammate 可能连回应 shutdown、发消息、更新 task list 的能力都被裁掉。

#### 阶段 3：初始化全局消息缓冲与首轮 prompt
位置：`src/utils/swarm/inProcessRunner.ts:1003-1034`

这里定义：

- `allMessages`：完整历史缓冲
- `wrappedInitialPrompt`：用 `<teammate-message>` 包装 leader 初始 prompt
- `currentPrompt`
- `shouldExit`

然后还会立刻尝试：

- `tryClaimNextTask(identity.parentSessionId, identity.agentName)`

注释已经点明：

- 这是为了让 UI 从一开始就能显示活跃任务
- 使用的是 `parentSessionId`，因为 leader 的 task list 挂在 session ID 下，不挂在 team name 下

这个点非常容易读漏，但很重要：

```text
team task list 的 ID = parentSessionId，不是 teamName
```

#### 阶段 4：建立 content replacement / compact 保护机制
位置：`src/utils/swarm/inProcessRunner.ts:1035-1126`

这里说明 runner 已经考虑到长生命周期 teammate 的上下文增长问题。

关键点有三个：

1. `teammateReplacementState` 是 per-teammate 的内容替换状态，避免多轮 `runAgent()` 时前后 replacement 决策不一致。
2. token 超过 `getAutoCompactThreshold(...)` 时，会用隔离版 `ToolUseContext` 调 `compactConversation(...)`，避免污染主会话的 readFileState cache 和 UI 回调。
3. compact 后会同步更新：
   - `allMessages`
   - `task.messages`
   - microcompact state
   - replacement state

这说明 runner 不只是“能跑”，还认真处理了长期驻留 worker 的上下文内存与 prompt 稳定性问题。

#### 阶段 5：每轮创建独立 work abort controller
位置：`src/utils/swarm/inProcessRunner.ts:1053-1063`

每一轮工作都会建一个新的：

- `currentWorkAbortController`

并写进 task state。

这正好对应上一站看到的两级中断语义：

- lifecycle abort：杀整个 teammate
- work abort：只停当前这一轮

源码注释明确说明：

- Escape 只停止 current work
- 不杀掉整个 teammate

这是“长期存活 worker”能有好交互体验的关键。

#### 阶段 6：真正跑 `runAgent()`
位置：`src/utils/swarm/inProcessRunner.ts:1159-1277`

这里是最核心的执行内环。

几个最重要的点：

1. 它包在：
   - `runWithTeammateContext(...)`
   - `runWithAgentContext(...)`

   里执行，说明 teammate 上下文隔离与 analytics 归因都是真正参与运行的。

2. 它调用的是**和 AgentTool/subagent 一样的 `runAgent()`**。

   这说明 in-process teammate 并没有单独发明另一套 agent loop，而是复用主 agent 执行核心。

3. 关键参数包括：
   - `canUseTool: createInProcessCanUseTool(...)`
   - `isAsync: true`
   - `canShowPermissionPrompts: allowPermissionPrompts ?? true`
   - `forkContextMessages`
   - `override: { abortController: currentWorkAbortController }`
   - `preserveToolUseResults: true`
   - `allowedTools`
   - `contentReplacementState: teammateReplacementState`

4. message 流出来之后，它会：
   - 追加到 `iterationMessages`
   - 追加到 `allMessages`
   - `updateProgressFromMessage(...)`
   - 更新 `progress`
   - 维护 `inProgressToolUseIDs`
   - 把消息镜像进 `task.messages`

其中 `inProgressToolUseIDs` 的更新也很细：

- assistant `tool_use` block 出现时加入集合
- user `tool_result` block 出现时从集合里删掉对应 `tool_use_id`

这说明 transcript 里的工具执行动画是 runner 增量维护出来的。

#### 阶段 7：一轮结束后转入 idle，并等待下一轮
位置：`src/utils/swarm/inProcessRunner.ts:1279-1417`

每轮 `runAgent()` 结束后，它会：

- 清空 `currentWorkAbortController`
- 如果只是 work abort，则在 transcript 里追加 `ERROR_MESSAGE_USER_ABORT`
- 把 task 标成 `isIdle: true`
- 触发 `onIdleCallbacks`
- 如有必要，发 `idle notification`
- 然后进入 `waitForNextPromptOrShutdown(...)`

收到等待结果后：

- `shutdown_request`：包装成 teammate-message，再送回模型决定批准/拒绝 shutdown
- `new_message`：若来自 `user` 就直接纯文本；否则包装成 teammate-message
- `aborted`：退出 while loop

这里最关键的设计思想是：

```text
shutdown 不是 runner 直接拍板，而是把请求交给模型决策
```

注释里已经说得很清楚：

- model will use approveShutdown or rejectShutdown tool

这意味着关闭 teammate 本身也是 agent 协作协议的一部分。

---

### 成功退出和失败退出分别怎么收口
位置：`src/utils/swarm/inProcessRunner.ts:1419-1533`

#### 正常退出
它会：

- 只在 `task.status === 'running'` 时改成 `completed`
- 避免覆盖 `killInProcessTeammate(...)` 已经写下的 terminal 状态
- 保留最后一条 message，其余丢弃
- 清空 pendingUserMessages / inProgressToolUseIDs / abort controllers
- `evictTaskOutput(taskId)`
- `evictTerminalTask(taskId, setAppState)`
- `emitTaskTerminatedSdk(taskId, 'completed', ...)`
- `unregisterPerfettoAgent(identity.agentId)`

这里又体现了两个很成熟的细节：

1. runner 知道 kill path 可能已经先一步落 terminal state，所以不会盲目覆盖。
2. terminal task 会被主动 evict，避免长时间占住 AppState。

#### 失败退出
catch 分支会：

- 置 `status: 'failed'`
- 写入 `error`
- 标记 `isIdle: true`
- 清空各类运行时字段
- evict task output / terminal task
- close SDK bookend
- 通过 mailbox 发一条 `idleReason: 'failed'` 的 idle notification
- `failureReason` 也一并带给 leader

所以失败不是静默挂掉，而是会进入 leader 可观察的失败通知链。

---

### `startInProcessTeammate(...)` 为什么单独存在
位置：`src/utils/swarm/inProcessRunner.ts:1544`

这个函数只是 fire-and-forget 地调用：

- `runInProcessTeammate(config).catch(...)`

但注释很值得记：

- 先把 `agentId` 抽出来
- 避免 catch closure 长时间持有完整 `config`
- 否则会把 `toolUseContext` 等大对象一起 retain 住，持续几个小时

这说明作者对长生命周期 teammate 的**闭包保活 / 内存滞留**问题是有明确意识的。

这个细节非常工程化，也很值得学习。

---

### 这个文件体现出的 10 个架构事实

1. in-process teammate 复用了主权限引擎，只有在 `ask` 时才走专门 teammate 适配层。
2. 同进程权限请求优先复用 leader 的 `ToolUseConfirm` UI 队列，mailbox 只是降级路径。
3. permission update 不只影响当前工具调用，还会写回 leader 的共享 permission context。
4. teammate message 使用统一的 XML 包装协议，以对齐 tmux teammate 行为。
5. idle 态消息调度优先级是：`pendingUserMessages > shutdown > team-lead > peer > task-list`。
6. in-process teammate 真正执行时复用了标准 `runAgent()`，不是另一套独立 agent loop。
7. teammate 拥有 per-turn abort 与 whole-lifecycle abort 两级中断控制。
8. runner 内置 compact、content replacement state、AppState transcript cap 同步，说明它被设计成可长期存活的 worker。
9. shutdown 请求不是 runner 直接执行，而是交还给模型通过专门工具作出批准/拒绝。
10. 成功/失败/kill 收尾都显式考虑了 SDK bookend、task eviction、perfetto 注销等执行后清理。

---

### 现在把第 20-24 站彻底串起来

到这里，in-process teammate 这一整条链已经完整了：

```text
types.ts
  -> 定义状态模型

InProcessTeammateTask.tsx
  -> 提供 task 状态操作与查询 helper

spawnInProcess.ts
  -> 创建/销毁 teammate，接上 AppState 与 cleanup

inProcessTeammateHelpers.ts
  -> 把 leader/runtime 事件桥接进 task state

inProcessRunner.ts
  -> 真正执行 runAgent 循环、处理 idle/wait、权限、shutdown、task claim
```

所以现在可以把这条支线压缩成一句话：

```text
spawn 层负责“把 worker 生出来”，
runner 层负责“让 worker 持续活着并真正干活”。
```

这就是 in-process teammate 架构的完整闭环。

---

### 这一站最该记住的 10 句话

1. `inProcessRunner.ts` 是 in-process teammate 的真实执行主循环，不只是简单包装 `runAgent()`。
2. teammate 的权限询问优先复用 leader 的 `ToolUseConfirm` UI，mailbox 是降级路径。
3. `formatAsTeammateMessage(...)` 的意义是统一 in-process 与 tmux teammate 的消息协议。
4. `waitForNextPromptOrShutdown(...)` 决定了 teammate idle 态如何持续存活。
5. idle 态的消息优先级是：用户注入 > shutdown > lead 消息 > peer 消息 > task list。
6. task list 中的任务最终会被格式化成 prompt，再喂回 `runAgent()`。
7. `runInProcessTeammate(...)` 每轮都会建立独立 `currentWorkAbortController`，支持只中断当前工作不杀死 teammate。
8. 长生命周期 teammate 会主动做 compact，并同步收缩 AppState 中的 transcript 镜像。
9. shutdown 需要交给模型通过工具决定，而不是 runner 直接硬编码批准。
10. 成功和失败路径都显式做 task eviction、SDK 收尾、idle/failure 通知与 perfetto 清理。

---

### 下一站建议

现在最顺的下一站是：

```text
src/tools/SendMessageTool/SendMessageTool.ts
```

原因：

- 你已经把 teammate 侧的“收消息、等消息、跑起来”读透了
- 下一步最自然就是回到“消息是怎么被主动发出去的”这一侧

下一站重点就看：

**leader / teammate 主动发送消息时，`SendMessageTool` 如何决定目标、写 mailbox、以及与 team/task 协作语义如何对齐。**

---

---

### 元

问题：**这一站真正想解决的架构问题是什么？**

回答：这一站是真正的 in-process teammate 执行引擎，负责权限桥接、主循环、空闲等待、消息恢复和状态镜像。它把“被 spawn 出来”变成“持续驻留并能继续协作”。

### 反

问题：**如果把这一站的设计反过来，会发生什么？**

回答：如果没有这层 runner，in-process teammate 只能算一次性任务，无法 idle 后继续等待新 prompt。那就失去了作为常驻协作 worker 的意义。

### 空

问题：**跳出当前文件名，这一站背后更大的问题是什么？**

回答：更大的问题是，共享进程 agent 如何既保持独立身份，又持续挂在同一宿主里运行。in-process runner 就是在回答这种“轻隔离、长驻留”的问题。

## 第 25 站：`src/tools/SendMessageTool/SendMessageTool.ts`

### 这是什么文件

`src/tools/SendMessageTool/SendMessageTool.ts` 是 swarm / agent-team 体系里的**主动发信控制面**。

如果上一站 `src/utils/swarm/inProcessRunner.ts` 解决的是：

```text
teammate 怎么收消息、等消息、继续跑
```

那么这一站解决的就是：

```text
leader 或 teammate 怎么把消息主动送出去
```

它并不只是一个“往 mailbox 写一条文本”的小工具。

它同时负责：

- 普通文本消息
- 广播消息
- shutdown request / response
- plan approval response
- 跨 session 的 bridge / uds 发送
- 发给 in-process local agent 的直接排队/恢复

所以一句话先记：

```text
SendMessageTool.ts = Claude Code swarm 协议里的统一出站消息路由器
```

---

### 先抓住这个文件的 4 条主线

#### 主线 1：它支持两类 payload
位置：`src/tools/SendMessageTool/SendMessageTool.ts:46-87`

输入里的 `message` 可以是：

- `string`
- `StructuredMessage`

而 `StructuredMessage` 又只允许三类：

- `shutdown_request`
- `shutdown_response`
- `plan_approval_response`

这说明 SendMessageTool 并不是一个完全开放的协议层，而是：

```text
自由文本 + 少量强约束的协议消息
```

也就是说，普通沟通和关键控制信号共用一个工具，但控制信号的 schema 是明确锁死的。

---

#### 主线 2：目标地址不只是一种
位置：`src/tools/SendMessageTool/SendMessageTool.ts:67-86`, `src/tools/SendMessageTool/SendMessageTool.ts:612`

`to` 支持：

- teammate name
- `*` 广播
- `uds:<socket-path>`
- `bridge:<session-id>`

所以这不是单纯 team mailbox 工具，而是统一覆盖：

- 当前 swarm 内部队友
- 本机 UDS peer
- Remote Control peer session

这说明 SendMessageTool 的抽象层次已经高于“写某个 teammate inbox”。

---

#### 主线 3：它会按目标类型走完全不同的发送路径
后面真正 `call(...)` 时会分流到：

- bridge 发送
- UDS socket 发送
- local agent task queue / background resume
- 普通 mailbox 单播
- team broadcast
- structured protocol handling

所以它本质是 routing layer，不只是 data formatter。

---

#### 主线 4：它承载的是 swarm 协议，而不只是聊天
尤其是：

- shutdown request / approve / reject
- plan approval / rejection

说明 message system 直接承载了团队协作控制协议。

这点很关键，因为它意味着：

```text
agent team 的控制面，本质上就是通过消息协议串起来的
```

---

### 顶部 schema 在表达什么设计意图

#### `StructuredMessage`
位置：`src/tools/SendMessageTool/SendMessageTool.ts:46`

它只定义三类结构化消息：

1. `shutdown_request`
2. `shutdown_response`
3. `plan_approval_response`

这里最值得注意的是：

- 没有开放任意 object 结构
- 没有让调用方自己发任意协议类型
- 只允许少数已知系统控制消息

这说明 SendMessageTool 虽然是协议工具，但依然在做非常强的 surface 收缩。

#### `inputSchema`
位置：`src/tools/SendMessageTool/SendMessageTool.ts:67`

几个值得记的点：

- 普通 string message 要求 `summary`
- structured message 不需要 summary
- `to` 的说明文字会受 `UDS_INBOX` feature 影响

也就是说，schema 本身已经携带了运行环境感知能力，而不是静态文档。

---

### `handleMessage(...)` 与 `handleBroadcast(...)` 是最基础的 mailbox 路径
位置：`src/tools/SendMessageTool/SendMessageTool.ts:149`, `src/tools/SendMessageTool/SendMessageTool.ts:191`

这两个函数先建立了最朴素的 swarm 发送语义。

#### `handleMessage(...)`
它会：

- 从 `context.getAppState()` 取当前 teamName
- 根据运行身份推导 senderName：`getAgentName()` / teammate / `TEAM_LEAD_NAME`
- 取 senderColor
- 调 `writeToMailbox(...)`
- 返回一份带 `routing` 的结果对象

这里要记住：

```text
普通点对点 teammate 消息，真相写入面就是 file-based mailbox
```

并且返回值里会把：

- sender
- senderColor
- target
- targetColor
- summary
- content

都整理出来，说明工具结果不仅服务模型，也服务 UI 展示。

#### `handleBroadcast(...)`
它会：

- 先验证当前真的在 team context 中
- `readTeamFileAsync(teamName)`
- 从 team file 枚举成员
- 排除 sender 自己
- 对每个 recipient 逐一 `writeToMailbox(...)`

这说明 broadcast 不是特殊底层原语，而是：

```text
基于 team file 成员列表展开成多次单播写入
```

所以 team file 在这里再次扮演了“团队成员真相源”。

---

### shutdown 协议是怎么通过 SendMessageTool 发出去的

#### `handleShutdownRequest(...)`
位置：`src/tools/SendMessageTool/SendMessageTool.ts:268`

流程非常清楚：

- `generateRequestId('shutdown', targetName)`
- `createShutdownRequestMessage(...)`
- JSON 序列化后写进目标 mailbox

这说明 shutdown 请求本质上不是专门 RPC，而是：

```text
一条带 requestId 的结构化 mailbox message
```

这和前面读到的 `useInboxPoller.ts` / `inProcessRunner.ts` 完全对上了。

#### `handleShutdownApproval(...)`
位置：`src/tools/SendMessageTool/SendMessageTool.ts:305`

这是整文件里最重要的协议处理函数之一。

它先：

- 读取当前 `teamName`、`agentId`、`agentName`
- 从 team file 查自己成员信息
- 取出 `tmuxPaneId` 与 `backendType`
- 构造 `createShutdownApprovedMessage(...)`
- 发回 `TEAM_LEAD_NAME`

这里已经能看出一个很成熟的设计：

```text
shutdown approval 不只表示“同意”，还会把执行 backend/pane 身份回传给 leader
```

这样 leader 才能知道后续该清理哪个 pane / backend 资源。

#### 它还有关键的 backend 分流
批准 shutdown 后，它不是只发一条消息就完事，而是继续看自己是什么 backend：

- 如果 `ownBackendType === 'in-process'`
  - 直接通过 `findTeammateTaskByAgentId(...)` 找 task
  - 调 `task.abortController.abort()`
- 否则
  - 如果居然还能在 AppState 里找到 in-process task，就走 fallback abort
  - 否则 `setImmediate(() => gracefulShutdown(0, 'other'))`

这说明：

```text
SendMessageTool 既负责发“协议确认”，也负责触发本地执行实体真正退出
```

也就是说它是协议层和本地 lifecycle 控制层的交汇点。

#### `handleShutdownRejection(...)`
位置：`src/tools/SendMessageTool/SendMessageTool.ts:401`

这里就简单很多：

- 构造 `createShutdownRejectedMessage(...)`
- 发给 team lead
- 返回“继续工作”的结果

这再次说明 shutdown 是一个可协商协议，而不是 leader 单方面 kill。

---

### plan approval 协议为什么放在 SendMessageTool 里
位置：`src/tools/SendMessageTool/SendMessageTool.ts:434`, `src/tools/SendMessageTool/SendMessageTool.ts:478`

#### `handlePlanApproval(...)`
它首先强约束：

- 只有 team lead 才能 approve

如果不是 lead，直接报错。

然后它会：

- 读 `leaderMode = appState.toolPermissionContext.mode`
- 如果 leader 当前是 `plan`，则给 worker 继承 `default`
- 构造 `plan_approval_response`
- 把 `permissionMode` 一起写回 teammate mailbox

这非常关键。

它说明 plan approval 并不只是一个 yes/no 信号，而是还会把：

```text
approval 之后 teammate 应该进入什么 permission mode
```

一并传回去。

这和前面 `useInboxPoller.ts`、`inProcessTeammateHelpers.ts`、`inProcessRunner.ts` 看到的 plan approval 状态切换完全闭环。

#### `handlePlanRejection(...)`
也是只有 team lead 才能做。

它发的是：

- `approved: false`
- `feedback`

说明 rejection 不只是停住，还把 revision feedback 一起回传给 teammate。

所以 SendMessageTool 在这里承担的是：

```text
plan mode 审批协议的出站半边
```

---

### `buildTool(...)` 这一层最值得看的几个点
位置：`src/tools/SendMessageTool/SendMessageTool.ts:520`

#### 1. `shouldDefer: true`
这很值得记。

说明 SendMessageTool 虽然是团队协作关键工具，但在 tool orchestration 里仍然属于 deferred tool。

也就是说它的执行不是最简单的“同步直接跑完”模型，而是参与了更完整的工具调度/UI 生命周期。

#### 2. `isReadOnly(input)`
位置：`src/tools/SendMessageTool/SendMessageTool.ts:539`

只有当 `input.message` 是 string 时才视作 read-only。

这说明系统把普通文本消息视为较低风险，而 structured control messages（shutdown / approval）则不被视为只读。

这个安全建模很合理。

#### 3. `backfillObservableInput(...)`
位置：`src/tools/SendMessageTool/SendMessageTool.ts:543`

这个函数会把原始输入补出更适合 UI/observability 的字段：

- `type`
- `recipient`
- `request_id`
- `approve`
- `content`

这说明工具系统里的“可观测输入”并不总是原始 schema，而是可以做语义投影，方便日志/UI 理解这次调用干了什么。

#### 4. `toAutoClassifierInput(...)`
位置：`src/tools/SendMessageTool/SendMessageTool.ts:571`

它把各种消息都转成短文本摘要，例如：

- `to X: hello`
- `shutdown_request to X`
- `shutdown_response approve req123`
- `plan_approval reject to X`

说明 SendMessageTool 也纳入了权限/安全分类器的统一文本表示层。

---

### 它的权限与输入校验体现了哪些安全边界

#### `checkPermissions(...)`
位置：`src/tools/SendMessageTool/SendMessageTool.ts:585`

这里最重要的是 bridge 场景。

如果 `to` 是：

- `bridge:<session-id>`

它不会自动 allow，而是返回：

- `behavior: 'ask'`
- `decisionReason.type = 'safetyCheck'`
- `classifierApprovable: false`

注释还明确写了：

- 这是 cross-machine prompt injection 风险
- 必须 bypass-immune

这说明：

```text
跨机器发 prompt 被视为高敏感操作，必须显式用户同意
```

而且这个保护优先级比 auto-mode allowlist/classifier 更高。

#### `validateInput(...)`
位置：`src/tools/SendMessageTool/SendMessageTool.ts:604`

这部分信息量非常大，核心边界包括：

1. `to` 不能为空
2. `bridge:` / `uds:` 地址 target 不能为空
3. `to` 不能带 `@`，因为每个 session 只有一个 team
4. bridge 场景下：
   - structured messages 禁止跨 session
   - Remote Control 未连接时拒绝发送
5. uds 场景下：
   - 只允许 plain string message 跨 session
6. string message 必须带 summary（除非 UDS 跨 session）
7. structured message 不能广播到 `*`
8. structured message 不能发到 cross-session 地址
9. `shutdown_response` 只能发给 `team-lead`
10. 拒绝 shutdown 时必须给 `reason`

这说明 SendMessageTool 的校验不是在做表面 schema 检查，而是在强约束：

```text
哪些消息类型允许跨边界、哪些必须留在当前 team 协议域内
```

这是很重要的协议边界设计。

---

### `call(...)` 真正的路由顺序是什么
位置：`src/tools/SendMessageTool/SendMessageTool.ts:741`

这个函数是整个工具的真实路由器。

顺序非常值得记。

#### 路由 1：先处理 bridge / UDS 的 plain text send
如果 `UDS_INBOX` 开启且 `message` 是 string：

- `bridge:`
  - 再次检查 bridge handle 和 active 状态，避免 permissions prompt 期间连接断开
  - 动态 require `postInterClaudeMessage`
  - 发给 remote peer session
- `uds:`
  - 动态 require `sendToUdsSocket`
  - 发给本机 UDS socket

这里有个非常好的工程细节：

- 前面 `validateInput()` 检查过一次连接状态
- 但真正 `call()` 里又**再检查一次**

因为权限确认等待可能持续很久，之前状态可能已经过期。

这就是典型的 TOCTOU 防御思路。

#### 路由 2：再尝试路由到本地 agent task
位置：`src/tools/SendMessageTool/SendMessageTool.ts:800-874`

如果是 plain string 且不是广播，它会：

- 看 `appState.agentNameRegistry.get(input.to)`
- 或尝试 `toAgentId(input.to)`
- 若能定位到本地 agent task：
  - 如果 task 正在 running，直接 `queuePendingMessage(...)`
  - 如果 task 已停止，则 `resumeAgentBackground(...)`
  - 如果 task 已被 evict，也尝试从 transcript resume

这条路径特别关键，因为它说明：

```text
SendMessageTool 不只是 swarm teammate mailbox 工具，
也能给本地 background agent 继续投递 prompt
```

所以它其实统一了两类“给另一个 Claude 实体发后续消息”的场景：

- team teammate
- local background agent

#### 路由 3：普通 team mailbox 文本消息
如果还没被前面拦截：

- `to === '*'` -> `handleBroadcast(...)`
- 否则 -> `handleMessage(...)`

#### 路由 4：structured protocol message
最后才进入：

- `shutdown_request`
- `shutdown_response`
- `plan_approval_response`

这说明 call() 的大体顺序是：

```text
cross-session text
  -> local agent delivery/resume
  -> normal mailbox text
  -> structured swarm protocol
```

---

### 这个文件体现出的 10 个架构事实

1. `SendMessageTool` 是 swarm 协议统一的出站路由层，而不是单纯 mailbox helper。
2. 它把普通文本消息和少量强约束的控制协议消息合并进同一工具面。
3. team broadcast 的真相实现是基于 team file 展开成多次单播 mailbox 写入。
4. shutdown approval/rejection 不只是协议响应，还会触发本地 backend 的真实退出动作。
5. plan approval response 会把 permissionMode 一起回传，说明审批协议会影响 teammate 后续执行模式。
6. bridge/UDS 跨 session 通信与 team mailbox 通信共用一个表层工具，但安全边界不同。
7. bridge 发送被视为跨机器 prompt 注入风险，需要显式用户同意，且不能被 bypass 掉。
8. `call()` 中对 bridge 连接状态做了二次检查，体现了对等待期状态变化的防御。
9. SendMessageTool 还能给本地 background agent 直接排队消息或从 transcript 恢复执行。
10. structured messages 被严格限制在当前 team 协议域内，不能随意 broadcast 或跨 session 扩散。

---

### 现在把第 24-25 站串起来

到这里，agent-team 这一段的收/发两侧已经闭环：

```text
inProcessRunner.ts
  -> teammate 如何消费消息、进入 idle、等待下一轮、继续跑

SendMessageTool.ts
  -> leader / teammate / local agent 如何主动把消息或控制协议发出去
```

所以可以把这两站合并成一句话：

```text
runner 负责“收与等”，
SendMessageTool 负责“发与路由”。
```

这就是 swarm message loop 的完整双向面。

---

### 这一站最该记住的 10 句话

1. `SendMessageTool.ts` 是 agent-team/swarm 体系的统一出站消息路由器。
2. 它支持普通文本和三种结构化控制消息：shutdown request、shutdown response、plan approval response。
3. `to` 不只支持 teammate 名，还支持广播、UDS peer、Remote Control bridge session。
4. 普通 teammate 单播/广播的真相写入面仍然是 file-based mailbox。
5. shutdown approval 不只发确认消息，还会根据 backend 类型真正触发本地退出。
6. plan approval response 会把 leader 的 permission mode 派生后一起传回 teammate。
7. `shouldDefer: true` 说明这个工具也是完整工具调度体系的一部分，不是简单 helper。
8. bridge 发送是高敏感安全边界，必须显式 ask，且 structured messages 禁止跨 session。
9. `call()` 会优先尝试 bridge/UDS、本地 agent queue/resume，再退到普通 mailbox 路由。
10. SendMessageTool 统一了“给 teammate 发消息”和“给本地 background agent 继续发 prompt”两类语义。

---

### 下一站建议

现在最顺的下一站是：

```text
src/utils/teammateMailbox.ts
```

原因：

- 你已经把发送侧 `SendMessageTool` 和消费侧 `inProcessRunner` 都读了
- 现在最自然就是下沉到双方共同依赖的 mailbox 协议底座

下一站重点就看：

**mailbox 文件格式、read/write/mark-as-read 语义、以及 shutdown / permission / idle notification 这些结构化消息到底怎样编码与解析。**

---

---

### 元

问题：**这一站真正想解决的架构问题是什么？**

回答：这一站是 swarm 的统一出站消息路由器，不只发普通文本，也发 shutdown request/response、plan approval response，并支持 teammate、广播、UDS、bridge 等多种目标。它的抽象层次已经高于“写 inbox”。

### 反

问题：**如果把这一站的设计反过来，会发生什么？**

回答：如果每种消息、每种目标都各自实现发送路径，协议控制信号和普通沟通会迅速分裂。统一工具缺失后，消息系统会越来越难解释。

### 空

问题：**跳出当前文件名，这一站背后更大的问题是什么？**

回答：更大的问题是，协作系统中的“发消息”到底是聊天功能还是控制面能力。这里的答案是两者兼具，所以才需要统一路由层。

## 第 26 站：`src/utils/teammateMailbox.ts`

### 这是什么文件

`src/utils/teammateMailbox.ts` 是 agent-team / swarm 体系里的**消息协议底座**。

如果上一站 `SendMessageTool.ts` 解决的是：

```text
消息怎么被主动发出去
```

那么这一站解决的是：

```text
这些消息到底写到哪里、怎样并发安全地读写、以及协议消息如何被编码/识别
```

所以一句话先记：

```text
teammateMailbox.ts = file-based inbox 存储层 + swarm 协议消息定义层
```

它一半是存储实现，一半是协议定义中心。

---

### 先抓住这个文件的 3 层职责

#### 1. inbox 文件存储层
它定义每个 teammate 的 inbox 文件路径、读写、加锁、已读标记。

#### 2. 协议消息定义层
它集中定义了大量 structured mailbox message：

- idle notification
- permission request / response
- sandbox permission request / response
- plan approval request / response
- shutdown request / approved / rejected
- task assignment
- team permission update
- mode set request

#### 3. 协议识别与辅助层
它还负责：

- `isStructuredProtocolMessage(...)`
- 各类 `isXxx(...)` 解析器
- `formatTeammateMessages(...)`
- `getLastPeerDmSummary(...)`

这说明 mailbox 在 Claude Code 里不是“简单消息列表文件”，而是整个 swarm 协议的公共边界层。

---

### 文件顶部最重要的设计事实

#### inbox 路径是按 team + agent name 定位的
位置：`src/utils/teammateMailbox.ts:56`

`getInboxPath(agentName, teamName?)` 最终路径是：

```text
~/.claude/teams/{team_name}/inboxes/{agent_name}.json
```

而且：

- `teamName` 和 `agentName` 都会经过 `sanitizePathComponent(...)`
- inbox key 用的是 **agent name**，不是 UUID

这和文件头注释完全一致：

```text
Inboxes are keyed by agent name within a team.
```

这点非常关键，因为前面很多状态和 task 是按 `agentId` 找，但 mailbox 层是按 `agentName` 寻址。

---

### `TeammateMessage` 是最底层消息信封
位置：`src/utils/teammateMailbox.ts:43`

最基础的 mailbox 存储对象只有这些字段：

- `from`
- `text`
- `timestamp`
- `read`
- `color?`
- `summary?`

这里最该记的是：

```text
mailbox 存的是统一信封；真正的协议类型主要编码在 text 里
```

也就是说：

- 普通文本消息：`text` 就是自由文本
- 协议消息：`text` 里放 JSON 序列化后的 structured message

因此 mailbox storage 层和协议层是“外层统一信封 + 内层 text payload”两层结构。

---

### 读写邮箱为什么要显式加锁
位置：`src/utils/teammateMailbox.ts:31-41`, `src/utils/teammateMailbox.ts:134`

顶部先定义了：

- `LOCK_OPTIONS`
- 带 retry + backoff

注释点得很明确：

- 多个 Claude / 多个 swarm worker 可能并发写同一个 inbox
- async lock API 不会像 sync API 那样直接阻塞 event loop
- 所以需要显式 retries 才能逼近原来的串行语义

这说明 mailbox 的并发模型是：

```text
单 inbox 文件 + proper-lockfile 串行化修改
```

而不是 append-only log、也不是数据库。

这是一个很朴素但工程上足够稳的设计。

---

### `readMailbox(...)` / `readUnreadMessages(...)` 的语义很直接
位置：`src/utils/teammateMailbox.ts:84`, `src/utils/teammateMailbox.ts:115`

#### `readMailbox(...)`
- 直接读 inbox JSON
- `ENOENT` 时返回 `[]`
- 非 ENOENT 则记录错误，也返回 `[]`

这说明 mailbox 读取是**容错偏保守**的：

```text
读失败时宁可当成没有消息，也不把整个上层流程炸掉
```

#### `readUnreadMessages(...)`
- 只是 `readMailbox()` 后筛 `!m.read`

所以 unread 并不是单独存储视图，而是 mailbox 全量数组上的过滤结果。

---

### `writeToMailbox(...)` 是最核心的存储原语
位置：`src/utils/teammateMailbox.ts:134`

这个函数很值得仔细记，因为几乎所有出站消息最后都汇聚到这里。

它的流程是：

1. `ensureInboxDir(teamName)`
2. 先尝试用 `flag: 'wx'` 创建空 inbox 文件 `[]`
3. 对 inbox 文件加锁
4. **拿到锁后重新读一遍最新 messages**
5. push 一条 `read: false` 的新消息
6. 整个数组重写回文件
7. finally 释放锁

这里最关键的是第 4 步：

```text
必须在拿到锁后重新读最新状态
```

否则就会出现典型的 lost update：两个并发写者都基于旧数组 append，再互相覆盖。

所以 mailbox 写入虽然简单，但并没有忽视并发一致性。

---

### 已读标记为什么也要加锁
位置：`src/utils/teammateMailbox.ts:201`, `src/utils/teammateMailbox.ts:279`

#### `markMessageAsReadByIndex(...)`
它会：

- 锁住 inbox 文件
- 重新读取最新 messages
- 检查 index 是否越界
- 如果消息还没读，就把该 index 的 `read` 置为 true
- 重写文件

#### `markMessagesAsRead(...)`
流程类似，只是把所有消息都标成 read。

这里体现的关键思想是：

```text
“已读状态”本身也是共享可变状态，不能不加锁
```

否则消息消费方和其他写入方/消费方会互相踩。

#### `markMessagesAsReadByPredicate(...)`
位置：`src/utils/teammateMailbox.ts:1101`

这又更进一步：

- 不是只能“全读”或“按 index 读”
- 还支持“只把匹配某条件的消息标已读”

这非常适合前面看到的那种场景：

- 某类协议消息要路由给专门 handler
- 其他普通消息还要保留给 LLM attachment 路径

也就是说 mailbox 层已经为“选择性消费”准备好了原语。

---

### `clearMailbox(...)` 的细节也很讲究
位置：`src/utils/teammateMailbox.ts:349`

它不是无脑写 `[]`，而是用：

- `flag: 'r+'`

注释已经说明：

- 如果文件不存在，应该抛 `ENOENT`
- 不要为了清空而意外创建一个本来不存在的 inbox

这是个小细节，但说明作者对“操作不存在 inbox”和“清空已有 inbox”是有语义区分的。

---

### `formatTeammateMessages(...)` 解释了 attachment 为什么是 XML
位置：`src/utils/teammateMailbox.ts:373`

它会把 mailbox message 渲染成：

```xml
<teammate-message teammate_id="..." color="..." summary="...">
...
</teammate-message>
```

这和前面 `inProcessRunner.ts` 的 `formatAsTeammateMessage(...)` 正好形成一致协议。

这说明：

```text
无论消息来自 inbox attachment 还是 in-process runner 注入，最终都尽量对齐成同一种 teammate-message XML 形态
```

所以 XML 不是偶然选择，而是整个 swarm message rendering 的统一表现协议。

---

### structured protocol message 有哪些大类

这个文件中后半段几乎就是一张 swarm 协议字典。

#### 1. `idle_notification`
位置：`src/utils/teammateMailbox.ts:394`

用于 worker 通知 leader：

- 自己进入 idle
- idle 原因：`available` / `interrupted` / `failed`
- 最近 DM summary
- 是否完成了 task / 失败原因

这和上一站 `runInProcessTeammate(...)` 发 idle notification 完全闭环。

#### 2. `permission_request` / `permission_response`
位置：`src/utils/teammateMailbox.ts:453`, `src/utils/teammateMailbox.ts:468`

这里很关键的一点是注释：

- field names align with SDK `can_use_tool`
- response shape mirrors SDK control schemas

这说明 swarm mailbox 并不是自创一套完全不同的权限协议，而是在尽量贴 SDK 既有语义。

#### 3. `sandbox_permission_request` / `sandbox_permission_response`
位置：`src/utils/teammateMailbox.ts:576`, `src/utils/teammateMailbox.ts:597`

这是网络访问 host allow/deny 的专门协议。

说明 sandbox runtime 触发的网络授权，也能通过 mailbox 在 leader 和 worker 间同步。

#### 4. `plan_approval_request` / `plan_approval_response`
位置：`src/utils/teammateMailbox.ts:684`, `src/utils/teammateMailbox.ts:702`

其中 request 会带：

- `planFilePath`
- `planContent`
- `requestId`

response 会带：

- `approved`
- `feedback?`
- `permissionMode?`

这和前面读到的 plan 审批链完全一致。

#### 5. `shutdown_request` / `shutdown_approved` / `shutdown_rejected`
位置：`src/utils/teammateMailbox.ts:720`, `src/utils/teammateMailbox.ts:737`, `src/utils/teammateMailbox.ts:755`

shutdown 协议不仅有 request / reject，还有 approved，并且 approved 还能回带：

- `paneId?`
- `backendType?`

这说明 shutdown 协议天然关心后续清理执行体。

#### 6. 其他控制消息
- `task_assignment`
- `team_permission_update`
- `mode_set_request`

这说明 mailbox 已经不只是“聊天信箱”，而是整个 team 协调总线。

---

### 为什么有一堆 `isXxx(...)` 解析器

这个文件几乎为每种 structured message 都配了一个：

- `isIdleNotification(...)`
- `isPermissionRequest(...)`
- `isPermissionResponse(...)`
- `isSandboxPermissionRequest(...)`
- `isSandboxPermissionResponse(...)`
- `isShutdownRequest(...)`
- `isShutdownApproved(...)`
- `isShutdownRejected(...)`
- `isPlanApprovalRequest(...)`
- `isPlanApprovalResponse(...)`
- `isTaskAssignment(...)`
- `isTeamPermissionUpdate(...)`
- `isModeSetRequest(...)`

这说明 mailbox 协议的消费模式不是“统一大 switch schema parser”，而是：

```text
各运行时模块按需调用对应识别函数，局部消费自己关心的消息类型
```

这和前面 `useInboxPoller.ts` 的分类路由逻辑完全一致。

---

### `isStructuredProtocolMessage(...)` 为什么非常关键
位置：`src/utils/teammateMailbox.ts:1073`

这段注释信息量很大。

它在解决的问题是：

```text
哪些消息应该走 useInboxPoller 的专门 handler，
哪些消息才应该作为原始 teammate 文本上下文被 LLM 看见
```

如果 structured protocol message 被 `getTeammateMailboxAttachments` 先消费成普通 raw text attachment，后面的专门 handler 就拿不到了。

所以这个函数的真正作用是：

```text
把“控制协议消息”从“普通对话消息”里隔离出来
```

这是 swarm 协议正确路由的关键防线。

---

### `sendShutdownRequestToMailbox(...)` 说明什么
位置：`src/utils/teammateMailbox.ts:831`

这个 helper 把 shutdown request 的核心逻辑从 tool/UI 中抽出来复用。

流程是：

- 解析 team name
- 取 senderName（支持 AsyncLocalStorage 下的 in-process teammate）
- `generateRequestId('shutdown', targetName)`
- `createShutdownRequestMessage(...)`
- `writeToMailbox(...)`

这说明 mailbox 层并不只是“被动定义 message type”，还直接提供部分协议发送原语。

---

### `getLastPeerDmSummary(...)` 很小，但特别像运行时胶水
位置：`src/utils/teammateMailbox.ts:1149`

它会逆向扫描 messages，找最后一次 assistant 的 `SendMessage` tool_use：

- 目标不是 `*`
- 也不是 `team-lead`
- `message` 是 string

然后返回：

```text
[to {name}] {summary}
```

这个摘要会被拿去放进 idle notification。

所以它的意义不是“消息分析”，而是：

```text
给 leader 一个 teammate 刚刚最后在和谁沟通、说了什么的大致摘要
```

这非常适合 idle/teammate 状态 UI。

---

### 这个文件体现出的 10 个架构事实

1. teammate mailbox 是按 `teamName + agentName` 落到文件系统中的，而不是按 agentId。
2. mailbox 的底层存储模型是“统一消息信封 + text payload”，协议主要编码在 `text` 里。
3. 所有写入与已读变更都依赖 lockfile 做串行化，避免多 agent 并发踩写。
4. inbox 修改必须在拿到锁后重新读取最新状态，避免 lost update。
5. mailbox 不仅承载普通 teammate 文本消息，也承载整套 swarm 控制协议消息。
6. 权限、sandbox、plan approval、shutdown、mode change、team permission update 都复用了同一个 mailbox 总线。
7. 各类 `isXxx(...)` 让不同运行时模块可以按需局部解析协议消息，而不是强耦合到单一总解析器。
8. `isStructuredProtocolMessage(...)` 是避免控制协议消息被误当作普通 LLM 上下文的关键分流器。
9. XML `teammate-message` 是 attachment 展示层与 in-process runner 注入层共享的统一表现协议。
10. `getLastPeerDmSummary(...)` 说明 mailbox 工具层还直接服务 idle notification / teammate 状态概览等上层运行时体验。

---

### 现在把第 24-26 站串起来

到这里，swarm message loop 的三层已经很完整了：

```text
SendMessageTool.ts
  -> 主动发送与路由

teammateMailbox.ts
  -> mailbox 存储底座 + structured protocol schema

inProcessRunner.ts
  -> 消费 mailbox / pending messages / idle wait / 再进入 agent loop
```

所以现在可以把这段体系压缩成一句话：

```text
SendMessageTool 负责“发”，
teammateMailbox 负责“存和定义协议”，
inProcessRunner/useInboxPoller 负责“取、分流、执行”。
```

这就是 agent-team 消息闭环的核心骨架。

---

### 这一站最该记住的 10 句话

1. `teammateMailbox.ts` 是 swarm 的 mailbox 存储层与协议定义中心。
2. inbox 文件路径按 `teamName + agentName` 定位，不按 agentId。
3. mailbox 的基础信封是 `TeammateMessage`，协议类型主要序列化在 `text` 字段里。
4. inbox 写入和已读变更都需要文件锁，并且必须在拿锁后重新读最新状态。
5. `readUnreadMessages(...)` 只是全量 mailbox 上的过滤，不是单独数据结构。
6. XML `teammate-message` 是 teammate 消息进入 LLM / UI 展示时的统一包装协议。
7. mailbox 同时承载普通文本消息与权限、sandbox、plan、shutdown 等 structured control protocol。
8. `isStructuredProtocolMessage(...)` 是控制协议与普通对话上下文分流的关键守门函数。
9. mailbox 层不仅定义消息 schema，也直接提供 `sendShutdownRequestToMailbox(...)` 这类复用发送原语。
10. `getLastPeerDmSummary(...)` 说明 mailbox 辅助逻辑还直接服务 idle 通知和 teammate 状态概览。

---

### 下一站建议

现在最顺的下一站是：

```text
src/utils/permissions/permissions.ts
```

原因：

- 你刚把 swarm 协议里的 permission request/response mailbox 链路读透
- 下一步最自然就是回到 permission 决策本体，看“ask / allow / deny”到底在哪里被判出来

下一站重点就看：

**`hasPermissionsToUseTool(...)`、规则匹配、mode 语义、以及这些 permission 结果怎样被 tool loop 和 swarm 协议复用。**

---

---

### 元

问题：**这一站真正想解决的架构问题是什么？**

回答：这一站再次回到 mailbox，但焦点更偏协议底座与文件式 inbox 存储。它强调消息写到哪里、怎样并发安全读写，以及结构化协议消息怎样被识别。

### 反

问题：**如果把这一站的设计反过来，会发生什么？**

回答：如果 mailbox 只是原始消息列表，没有协议识别层，上层就难以区分普通聊天与权限、shutdown 等控制信号。通信会有内容，却没有可靠语义。

### 空

问题：**跳出当前文件名，这一站背后更大的问题是什么？**

回答：更大的问题是，通用消息载体怎样承载越来越多的控制协议而不崩坏。答案不是换传输介质，而是把协议边界定义清楚。

## 第 27 站：`src/utils/permissions/permissions.ts`

### 这是什么文件

`src/utils/permissions/permissions.ts` 是 Claude Code 工具权限系统里的**总判定引擎**。

如果前面几站读到的是：

- `SendMessageTool.ts`：消息如何发出去
- `teammateMailbox.ts`：权限请求/响应如何通过 mailbox 编码和传递
- `inProcessRunner.ts`：teammate ask 权限时怎么接回 leader

那么这一站终于回到根问题本身：

```text
一个 tool_use 到底为什么会被 allow、ask、deny
```

所以一句话先记：

```text
permissions.ts = 工具权限判定流水线 + auto mode classifier 协调中心
```

它不是单纯“查一下 mode”的文件，而是把这些层都串起来：

- rule matching
- tool 自身 `checkPermissions(...)`
- bypass / plan / dontAsk / auto 等 mode 变换
- classifier fast-path / classifier 主调用
- headless agent fallback
- denial tracking
- permission context sync / rule 编辑

---

### 先抓住这个文件的 4 条主线

#### 主线 1：权限判定不是单一步骤，而是一条分层流水线
这一点从 `hasPermissionsToUseToolInner(...)` 的分段注释就能看出来：

- 1a / 1b / 1c ...
- 2a / 2b
- 最后再把 `passthrough` 转成 `ask`

这说明 Claude Code 的权限系统不是“mode 优先”或“tool 优先”的简单模型，而是多层叠加。

---

#### 主线 2：tool 自身也参与权限判定
位置：`src/utils/permissions/permissions.ts:1208`, `src/utils/permissions/permissions.ts:1113`

这里会显式调用：

- `tool.checkPermissions(parsedInput, context)`

也就是说，全局权限引擎不会试图独立理解每个工具的细粒度风险，而是把：

```text
全局规则判断 + 各工具自己的领域权限判断
```

组合起来。

这对 Bash、Edit、MCP 等工具尤其关键。

---

#### 主线 3：auto mode 不是简单自动放行，而是一个 classifier 驱动的第二层决策器
位置：`src/utils/permissions/permissions.ts:518-927`

在 `ask` 结果出现后，auto mode 还会再跑一整套逻辑：

- safetyCheck 特判
- PowerShell 特判
- acceptEdits fast-path
- safe allowlist fast-path
- classifier 主调用
- denial tracking
- denial limit fallback

所以 auto mode 其实是：

```text
“先得到 ask，再尝试用 classifier / fast-path 把 ask 转成 allow 或 deny”
```

而不是简单的“auto = 全自动允许”。

---

#### 主线 4：headless / async agent 有专门权限降级路径
位置：`src/utils/permissions/permissions.ts:400`, `src/utils/permissions/permissions.ts:929`

如果当前上下文不能弹 permission prompt：

- 先跑 `PermissionRequest` hooks
- hook 没有决策才 auto-deny

这说明 headless agent 不是粗暴地完全没法过权限，而是还有一层 hook 机会。

---

### 最先要记住的基础函数：规则提取层

#### `getAllowRules(...)` / `getDenyRules(...)` / `getAskRules(...)`
位置：`src/utils/permissions/permissions.ts:122`, `src/utils/permissions/permissions.ts:213`, `src/utils/permissions/permissions.ts:223`

这三组函数会把 `ToolPermissionContext` 中来自多个 source 的字符串规则统一展开成 `PermissionRule[]`。

来源包括：

- settings sources
- `cliArg`
- `command`
- `session`

也就是说，权限规则在内存里的判定视图，不是原始字符串数组，而是：

```text
带 source / behavior / parsed ruleValue 的统一规则对象流
```

这让后续匹配逻辑可以统一工作，而不关心规则最初来自哪里。

---

### `toolMatchesRule(...)` 说明了权限规则如何理解“一个工具”
位置：`src/utils/permissions/permissions.ts:238`

这个函数解决的是：

```text
某条规则是否匹配整个工具本体，而不是内容细分规则
```

关键点有两个：

1. `ruleContent !== undefined` 时不匹配整个工具
   - 也就是说像 `Bash(prefix:*)` 这种不是“工具级规则”，而是内容级规则

2. MCP 工具要用 `getToolNameForPermissionCheck(tool)` 来匹配
   - 因为 MCP 工具在 skip-prefix 模式下显示名可能和 builtin 冲突
   - 所以权限匹配必须看“权限检查名”，不是单纯 UI 展示名

另外它还支持：

- `mcp__server1`
- `mcp__server1__*`

这种 server-level MCP 规则。

所以这一步的本质是：

```text
先把 tool 映射到权限世界里的规范名称，再做工具级规则匹配
```

---

### agent deny 规则是单独处理的
位置：`src/utils/permissions/permissions.ts:304`, `src/utils/permissions/permissions.ts:325`

`getDenyRuleForAgent(...)` / `filterDeniedAgents(...)` 支持：

```text
Agent(Explore)
```

这种语法。

这说明 agent 权限并没有完全依赖通用 toolName 匹配，而是允许：

```text
对 AgentTool 下的具体 agentType 再做一层 deny 过滤
```

`filterDeniedAgents(...)` 还专门做了优化：

- 先把 deny 规则里 `Agent(x)` 的 content 收集成 `Set`
- 再一次性 filter agent 列表

避免对每个 agent 都重新扫描/解析全部规则。

这是个典型的“热路径小优化”。

---

### `createPermissionRequestMessage(...)` 是 prompt 文案生成层
位置：`src/utils/permissions/permissions.ts:137`

这个函数非常值得记，因为用户看到的“为什么要你批准”提示很多都来自这里。

它会根据 `decisionReason.type` 生成不同解释：

- `classifier`
- `hook`
- `rule`
- `subcommandResults`
- `permissionPromptTool`
- `sandboxOverride`
- `workingDir`
- `safetyCheck`
- `other`
- `mode`
- `asyncAgent`

特别值得注意的是 `subcommandResults`：

- 对 Bash 会去掉 output redirection 展示噪音
- 只列出真正需要审批的子操作

这说明权限系统不仅有“判定逻辑”，还有一层：

```text
把内部判定原因翻译成用户可理解审批提示
```

---

### `runPermissionRequestHooksForHeadlessAgent(...)` 在补什么洞
位置：`src/utils/permissions/permissions.ts:400`

这个函数服务的是：

- 后台 agent
- async subagent
- 无法弹 permission prompt 的上下文

流程是：

1. 跑 `executePermissionRequestHooks(...)`
2. 如果 hook 给出 allow：
   - 可带 `updatedInput`
   - 可带 `updatedPermissions`
   - 并且会 `persistPermissionUpdates(...)`
   - 再把更新写回 AppState 的 `toolPermissionContext`
3. 如果 hook 给 deny：
   - 还可 `interrupt`
   - 会直接 abort 当前 context
4. 如果 hooks 失败，不炸整个流程，而是 fall through 到 auto-deny

这说明 headless agent 的 permission 模型是：

```text
先给 hooks 一个自动处理机会，再决定是否拒绝
```

这比“一刀切 auto deny”更灵活，也更适合自动化工作流。

---

### `hasPermissionsToUseTool(...)` 是 outer wrapper，不是最终判定主体
位置：`src/utils/permissions/permissions.ts:473`

这一层先调用：

- `hasPermissionsToUseToolInner(...)`

然后在外层做三类后处理：

1. allow 时重置 denial tracking
2. ask 时应用 `dontAsk` 模式变换
3. ask 时进入 auto mode / headless fallback 逻辑

所以可以把它理解成：

```text
inner = 基础权限流水线
outer = 模式变换 + classifier + headless fallback
```

这个层次划分很重要。

---

### `hasPermissionsToUseToolInner(...)` 才是基础权限流水线主干
位置：`src/utils/permissions/permissions.ts:1158`

这个函数最值得背的是它的顺序。

#### 第 1 段：规则和工具自判先跑
顺序大致是：

1. **1a** 整个工具被 deny rule 拒绝
2. **1b** 整个工具有 ask rule
3. **1c** 调 tool 自己的 `checkPermissions(...)`
4. **1d** tool 自己直接 deny
5. **1e** `requiresUserInteraction()` 的 ask 必须保留
6. **1f** content-specific ask rule 必须保留
7. **1g** safetyCheck ask 必须保留

这说明：

```text
rule/tool/safety 这些“硬约束”优先于 mode fast-path
```

特别是 1f 和 1g，注释反复强调它们对 bypass 也是 immune 的。

#### 第 2 段：再看 mode / alwaysAllow
接下来才是：

- **2a** bypassPermissions（以及 plan 中继承 bypass 的场景）
- **2b** 整个工具 always allow rule

也就是说 bypass 不是最顶层的万能通行证，前面已经有一批更高优先级的阻挡条件。

#### 第 3 段：最后把 `passthrough` 统一收口成 `ask`
如果前面都没明确 allow/deny，就把：

- `passthrough`

转成：

- `ask`

并生成审批提示文案。

所以 inner pipeline 的整体心智模型是：

```text
先处理硬性拒绝/必须询问条件
  -> 再处理 bypass / alwaysAllow
  -> 最后默认 ask
```

---

### 哪些 ask 是 bypass-immune 的
这个文件有两个地方都在强调这件事：

- content-specific ask rules
- safety checks

特别是 safety checks 注释已经写得很直接：

- `.git/`
- `.claude/`
- `.vscode/`
- shell configs

这类路径即使在 bypassPermissions 模式下也必须 prompt。

这说明 Claude Code 的 bypass 模式不是完全无条件，而是仍保留一层：

```text
对敏感路径/敏感编辑的不可绕过审批
```

这个设计非常关键。

---

### auto mode 的真正执行顺序
位置：`src/utils/permissions/permissions.ts:518-927`

这里是本文件最复杂也最关键的一段。

它只在基础结果已经是 `ask` 时触发。

#### 第 1 步：先保护 non-classifier-approvable 的 safetyCheck
如果：

- `decisionReason.type === 'safetyCheck'`
- 且 `classifierApprovable === false`

那么：

- interactive 场景保留原 ask
- headless 场景直接 deny

这说明有些 safetyCheck 甚至连 classifier 都无权自动批准。

#### 第 2 步：某些需要用户交互的工具直接保留 ask
`tool.requiresUserInteraction?.()` 时不进 classifier。

#### 第 3 步：PowerShell 特判
如果 PowerShell auto mode feature 没开：

- interactive 保留 ask
- headless deny

注释里也说明了这是为了避免 PowerShell 下载执行等高风险模式被 classifier 快速放行。

#### 第 4 步：acceptEdits fast-path
它会用一个伪造的 `toolPermissionContext.mode = 'acceptEdits'` 再跑一次：

- `tool.checkPermissions(...)`

如果在 acceptEdits 下会 allow，就直接跳过 classifier，按 auto allow。

但它特意排除了：

- `Agent`
- `REPL`

因为这两个工具若套 acceptEdits 语义会造成静默绕过 classifier。

这个 fast-path 的本质是：

```text
先用更便宜的“若处于 acceptEdits 会不会安全通过”来短路大量低风险操作
```

#### 第 5 步：safe-tool allowlist fast-path
如果工具在 `isAutoModeAllowlistedTool(...)` 白名单里，也直接 allow。

这一步和 acceptEdits fast-path 共同构成了 classifier 之前的两层节流。

#### 第 6 步：真正调用 classifier
- `formatActionForClassifier(tool.name, input)`
- `setClassifierChecking(toolUseID)`
- `classifyYoloAction(...)`
- finally `clearClassifierChecking(toolUseID)`

并且会记录大量 telemetry：

- token 使用
- cache 命中
- duration
- prompt lengths
- classifier cost
- stage1/stage2 指标

说明 classifier 在这里不是黑箱，而是被高度观测化的运行时组件。

#### 第 7 步：处理 classifier block / unavailable / success
如果 block：

- transcriptTooLong：interactive 回退到 manual approval；headless 直接 abort
- unavailable：按 feature gate 决定 fail-closed 还是 fail-open
- 普通 block：更新 denial tracking，必要时触发 denial limit fallback

如果 success：

- 重置 denial tracking
- 返回 allow

所以 auto mode 的真正模型可以概括成：

```text
ask
  -> safety / tool special cases
  -> acceptEdits fast-path
  -> safe allowlist fast-path
  -> classifier
  -> denial tracking / fallback
```

而不是“一个 classifier 决定全部”。

---

### denial tracking 为什么存在
位置：`src/utils/permissions/permissions.ts:483`, `src/utils/permissions/permissions.ts:878`, `src/utils/permissions/permissions.ts:984`

这个文件会在 auto mode 下维护：

- `consecutiveDenials`
- `totalDenials`

成功时 `recordSuccess(...)`
，失败时 `recordDenial(...)`。

当超限时：

- interactive：回落成 ask，让用户人工 review
- headless：直接 abort

这说明 auto mode 不是让 classifier 无限次悄悄拒绝，而是有一个：

```text
拒绝太多 -> 需要人接管
```

的保险丝。

这也是很成熟的自治系统设计。

---

### `checkRuleBasedPermissions(...)` 为什么要单独抽出来
位置：`src/utils/permissions/permissions.ts:1071`

注释已经说得很明确：

- 这是只检查“规则相关那部分流水线”的子集
- 供 bypassPermissions 模式尊重
- 不跑 classifier
- 不跑 dontAsk/auto/asyncAgent 变换
- 不跑 bypassPermissions 自己的 allow fast-path

也就是说它是：

```text
基础硬性规则子流水线的可复用切片
```

这对某些调用方只想知道“有没有规则层面的阻挡”非常有用。

---

### 后半段是 permission context 的编辑与同步层

#### `deletePermissionRule(...)`
位置：`src/utils/permissions/permissions.ts:1329`

它会：

- 禁止删除只读来源：`policySettings` / `flagSettings` / `command`
- 对可编辑来源执行 `removeRules`
- 若来源是磁盘设置，还会删除对应 settings 中的实际规则
- 最后同步 React state

这说明权限系统不是只会判定，也提供规则编辑能力。

#### `convertRulesToUpdates(...)`
位置：`src/utils/permissions/permissions.ts:1375`

它把 `PermissionRule[]` 按：

- source
- behavior

分组，转成 `PermissionUpdate[]`。

这是“规则静态视图”到“上下文变更操作”的桥。

#### `applyPermissionRulesToPermissionContext(...)`
位置：`src/utils/permissions/permissions.ts:1408`

用于初始加法式装载规则。

#### `syncPermissionRulesFromDisk(...)`
位置：`src/utils/permissions/permissions.ts:1419`

这个函数很重要。

它先清掉磁盘来源的旧规则，再应用新的 replaceRules，避免：

```text
settings 里删掉某条规则，但内存里旧规则还残留
```

注释把这个 stale rule 问题解释得很清楚。

另外如果 `allowManagedPermissionRulesOnly()` 开启，还会先把非 policy 源清空。

这说明 permission context sync 已经考虑了：

- 磁盘规则删减
- managed-only 模式
- source/behavior 级别精细替换

---

### 这个文件体现出的 12 个架构事实

1. `permissions.ts` 是 Claude Code 工具权限系统的总判定流水线，而不是单纯 mode 判断器。
2. 权限规则先被统一展开成带 source / behavior / ruleValue 的 `PermissionRule` 视图，再进入匹配。
3. 工具权限判定由“全局规则 + tool 自己的 `checkPermissions(...)`”共同决定。
4. content-specific ask rule 和 safetyCheck ask 都是 bypass-immune 的高优先级约束。
5. `hasPermissionsToUseToolInner(...)` 负责基础判定主干，外层 `hasPermissionsToUseTool(...)` 再做 dontAsk/auto/headless 变换。
6. auto mode 只在基础结果为 `ask` 时介入，而且先跑多层 fast-path，再决定是否调用 classifier。
7. acceptEdits fast-path 通过伪造 `mode: 'acceptEdits'` 重跑 tool.checkPermissions 来省 classifier 成本。
8. PowerShell、需要用户交互的工具、某些 safetyCheck 都会跳过 classifier 自动批准路径。
9. denial tracking 是 auto mode 的保险丝：拒绝太多后，interactive 要回退给用户，headless 要直接 abort。
10. headless/async agent 并非直接无条件拒绝，而是先给 PermissionRequest hooks 机会作自动 allow/deny。
11. `checkRuleBasedPermissions(...)` 是基础规则子流水线的可复用切片，专门服务只关心硬规则的场景。
12. 文件后半段还承担 permission context 编辑与磁盘规则同步职责，说明它既是 runtime 判定层，也是权限状态维护层的一部分。

---

### 现在把第 26-27 站串起来

到这里，swarm 权限链条已经能闭环：

```text
permissions.ts
  -> 产生 allow / ask / deny 的原始权限判定

teammateMailbox.ts
  -> 把 permission_request / response 编码成 mailbox 协议消息

inProcessRunner.ts / useInboxPoller.ts
  -> 在 teammate 与 leader 间传递 ask/allow/deny 结果
```

所以可以把这一段压缩成一句话：

```text
permissions.ts 决定“该不该问”，
mailbox 协议负责“怎么传”，
runner / poller 负责“怎么落地执行”。
```

这就是 Claude Code agent-team 权限协作链的核心骨架。

---

### 这一站最该记住的 12 句话

1. `permissions.ts` 是工具权限判定流水线的总控制中心。
2. 工具权限不是单看 mode，而是 rule、tool 自判、mode、classifier、hooks 叠加后的结果。
3. `tool.checkPermissions(...)` 是权限系统中的一等公民，不是附属逻辑。
4. bypassPermissions 也不能绕过 content-specific ask rule 和 safetyCheck。
5. outer `hasPermissionsToUseTool(...)` 主要负责 dontAsk、auto mode、headless fallback 等后处理。
6. auto mode 的本质是“把 ask 进一步交给 fast-path + classifier 再判一次”。
7. acceptEdits fast-path 和 safe allowlist fast-path 是为了减少 classifier 成本。
8. 某些 safetyCheck、PowerShell、需要用户交互的工具不会轻易走 classifier 自动批准路径。
9. denial tracking 防止 auto mode 无限次静默拒绝；超过阈值后必须回退给人接管或直接 abort。
10. headless agent 在 auto-deny 前，会先给 PermissionRequest hooks 一次自动处理机会。
11. `checkRuleBasedPermissions(...)` 提供了只看硬规则、不跑模式变换/分类器的权限子流水线。
12. 文件后半段还负责 permission rule 的删除、分组更新、磁盘同步，是权限状态维护的重要组成部分。

---

### 下一站建议

现在最顺的下一站是：

```text
src/hooks/useCanUseTool.ts
```

原因：

- 你已经把“权限结果是怎么判出来的”读透了
- 下一步最自然就是看这个判定函数在真实 tool loop / UI 交互里是怎么被包起来和消费的

下一站重点就看：

**`useCanUseTool` 怎样接 `hasPermissionsToUseTool(...)`、怎样处理确认弹窗/用户反馈/permission updates，以及它如何成为运行时工具调用前的最后一道门。**

---

### 元

问题：**这一站真正想解决的架构问题是什么？**

回答：这一站回到权限总判定引擎，但更强调它是一条分层流水线，尤其把 tool 自带 `checkPermissions(...)` 与 auto mode classifier 协调在一起。它说明 auto mode 并不是简单自动放行。

### 反

问题：**如果把这一站的设计反过来，会发生什么？**

回答：如果只按全局 mode 决策而不让工具参与细粒度判断，Bash、Edit、MCP 这类高语义工具会失去关键安全边界。若把 auto mode 当成粗暴 allow，则更危险。

### 空

问题：**跳出当前文件名，这一站背后更大的问题是什么？**

回答：更大的问题是，权限系统如何同时具备全局统一性和局部语义感知。真正稳定的权限设计，往往都要允许两层共存。

