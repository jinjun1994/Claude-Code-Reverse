# Claude Code 完整系统架构图

基于 `outputs/claude-cli-clean.js` 中真实可见的 CLI 入口 `VLY()/NLY()/yLY()`、Commander 路由、交互式 turn runtime、tool inner loop、MCP/Chrome bridge、agent team、compact、async wrapper、settings/auth/persistence 与横切基础设施实现整理。

## 1. 完整系统架构图

```mermaid
flowchart TD
    USER[用户] --> VLY[主进程入口]
    VLY --> NLY[入口类型分类器]
    VLY --> YLY[构建 Commander 根命令与 preAction]

    YLY --> ROUTE[Commander 路由层<br/>interactive / print / subcommands / special modes]

    ROUTE --> INTERACTIVE[交互式会话路径]
    ROUTE --> SUBCMDS[子命令<br/>mcp / auth / plugin / doctor / update / agents / auto-mode / setup-token]
    ROUTE --> SPECIAL[特殊宿主模式<br/>bridge / chrome / mcp native host / tmux / worktree / remote]

    INTERACTIVE --> SETTINGS[设置、认证、提示词装配<br/>config / auth / memory / permissions / hooks / plugins]
    SETTINGS --> SK[回合包装层]
    SK --> OS[回合引擎包装]
    OS --> PF[核心回合状态机]

    PF --> API[createMessage 流式采样路径]
    API --> CONTENT[助手内容块]
    CONTENT --> TOOLBRANCH[tool_use 与 mcp_tool_use 分支]
    TOOLBRANCH --> TOOLCALL[工具调用内循环<br/>本地执行器 / 浏览器桥接 / tools/call]
    TOOLCALL --> TRANSCRIPT[tool_result 配对与 transcript 回注]
    TRANSCRIPT --> PF

    PF --> AUTOCOMPACT[核心循环内的自动压缩决策]
    AUTOCOMPACT --> NZ6[压缩管线]
    NZ6 --> PF

    SK --> SIDECHAIN[侧链 transcript 记录与事件过滤]
    SIDECHAIN --> RG8[异步任务封装层]
    RG8 --> TASKS[任务通知、输出文件、用量状态]

    TOOLCALL --> TOOLSYS[工具系统<br/>内置 / 延迟 / ToolSearch / MCP 工具]
    TOOLCALL --> PERM[权限系统]
    TOOLCALL --> HOOKS[hook 系统]
    TOOLCALL --> MCP[MCP 集成与传输工厂]

    PF --> TEAM[Agent 团队运行时<br/>TeamCreate / 任务 / 邮箱 / TeamDelete]
    TEAM --> RG8

    SUBCMDS --> SETTINGS
    SUBCMDS --> MCP
    SUBCMDS --> TEAM
    SPECIAL --> MCP
    SPECIAL --> TEAM
    SPECIAL --> TOOLCALL

    SETTINGS --> MEMORY[记忆提示词注入]
    SETTINGS --> PLUGINS[插件市场与 pluginConfigs]
    SETTINGS --> PERSIST[持久化、认证状态、transcript 保留]

    PF -.-> TELEMETRY[遥测与诊断]
    TOOLCALL -.-> TELEMETRY
    SETTINGS -.-> TELEMETRY
```

## 2. 总体说明

这份总图现在按源码真实主链重画，不再把 Claude Code 只当成“CLI + 主循环 + 工具”三层抽象系统。

更准确的结构是：

1. `VLY()` 作为真正的主入口
2. `NLY()` 先决定 entrypoint 类型
3. `yLY()` 构建 Commander 根命令与 preAction 初始化链
4. Router 把请求分成 interactive、subcommand、special modes
5. interactive 路径进入 settings/auth/prompt/runtime 装配
6. `Sk -> OS -> PF_` 构成 turn runtime 主链
7. `tool_use` 进入 tool-call inner loop
8. compact 在 `PF_` 内部作为正式分支存在
9. `RG8(...)` 等外层包装把 turn runtime 收口为后台任务、通知和输出文件

## 3. 模块级详细说明

### 3.1 真正的系统入口是 `VLY()`，不是抽象的 “CLI”

源码里的真实顶层入口是 `VLY()`：`outputs/claude-cli-clean.js:374906-374972`。

它做的事情包括：

- 初始化进程级 handler
- 判断 `print/init-only/sdk-url` 等模式
- 决定是否进入非 TTY / SDK CLI 形态
- 调用 `NLY(...)` 设定 `CLAUDE_CODE_ENTRYPOINT`
- 再调用 `yLY()` 启动 Commander 路由层

因此总图里的第一层应该是 `VLY -> NLY -> yLY`，而不是直接写一个笼统的 `CLI entry`。

### 3.2 `NLY()` 决定 entrypoint 类型，是总路由前的环境分类层

`NLY(A)`：`outputs/claude-cli-clean.js:374890-374904`。

它会根据当前 argv / 环境把 entrypoint 归类为：

- `mcp`
- `claude-code-github-action`
- `sdk-cli`
- `cli`

而 `VLY()` 后面还会继续细分为：

- `sdk-typescript`
- `sdk-python`
- `claude-vscode`
- `local-agent`
- `claude-desktop`
- `remote`

这说明 Claude Code 不是单一 CLI 宿主，而是多宿主入口体系。

### 3.3 `yLY()` 才是 Commander 根命令与 preAction 初始化链

`yLY()`：`outputs/claude-cli-clean.js:374995-375060+`。

这一层做的不是业务执行本身，而是：

- 构建 Commander root command
- 注册 `preAction`
- 在 `preAction` 中加载 managed settings、初始化 sinks、migrations、remote settings、settings sync
- 之后才把请求真正路由到 interactive / subcommands / special handlers

所以源码上的“CLI 层”其实应拆为：

- process entry
- entrypoint kind detection
- commander root + preAction initialization
- concrete route handlers

### 3.4 interactive 路径前还有一整层 settings/auth/runtime 装配

interactive 模式不是 `yLY()` 直接调用 `Sk(...)`。

在进入 turn runtime 前，还要经过：

- settings merge
- auth state / OAuth / API key 决策
- memory prompt 注入
- permission context
- hooks / plugins / MCP tools 装配
- system prompt 构建

这些已经分别在专题文档里展开，但在总图里必须作为 `SETTINGS` 层存在，否则会误以为 main loop 直接裸调模型。

### 3.5 turn runtime 的真实主链是 `Sk -> OS -> PF_`

这一点是当前总图最关键的修正。

源码并不是只有一个 `Sk(...)` 主循环：

- `Sk(...)` 是 wrapper：`outputs/claude-cli-clean.js:177565-177833`
- `OS(...)` 是中层包装
- `PF_(...)` 是更底层的 while-loop state machine：`outputs/claude-cli-clean.js:203992-204488`

因此总图里要把 turn runtime 画成三层，而不是一个单节点。

### 3.6 `PF_` 内部已经包含 sampling、tool branch 和 autocompact

`PF_` 里至少维护这些状态：

- `messages`
- `toolUseContext`
- `autoCompactTracking`
- `turnCount`
- `pendingToolUseSummary`
- recovery flags

并且在每一轮里会做：

- content replacement / normalization
- `microcompact`
- `autocompact`
- create message stream
- tool branch
- transcript reinjection

所以总图里 `PF_` 不是单薄的“chat loop”，而是 Claude Code runtime 的核心状态机。

### 3.7 tool-call inner loop 是完整的执行内环，不是 `PF_` 的一条小箭头

当 assistant content 里出现 `tool_use` 或 `mcp_tool_use` 时，会进入完整 tool inner loop。

这条链至少覆盖：

- 本地执行器 `IM3`
- CLI beta runner `Zx6`
- `ChromeBridgeClient.callTool`
- `tools/call` / `requestStream`
- `tool_result` pairing validator

所以总图里把 `TOOLCALL` 单独抽成主模块是必要的。

### 3.8 compact 不是外围维护逻辑，而是在 core loop 里通过 `autocompact` 触发 `nZ6(...)`

源码里 `autocompact` 决策就在 `PF_` 内：`outputs/claude-cli-clean.js:204056-204083`。

而 `nZ6(...)` 再负责：

- pre_compact hooks
- summarize
- rebuild attachments / readFileState
- boundary marker
- post_compact / session hooks

对应：`outputs/claude-cli-clean.js:135656-135830`。

所以总图里 `AUTOCOMPACT -> nZ6 -> PF_` 这一段必须显式存在。

### 3.9 `Sk(...)` 还承担 event filter 和 sidechain transcript recorder

`Sk(...)` 在 yield 事件前，还会：

- 对事件做过滤
- 调 `tp([a], ...)` 记录 sidechain transcript
- 再把事件往上 yield

代码依据：`outputs/claude-cli-clean.js:177811-177815`。

因此 `Sk` 不是只做参数装配，也承担了 runtime event gateway 的职责。

### 3.10 `RG8(...)` 是后台任务 / agent 的外层壳，不是 turn core

`RG8(...)`：`outputs/claude-cli-clean.js:139625-139733`, `200250-200275`。

它负责：

- 消费 `Sk(...)` 的事件流
- 更新 usage / activity
- 维护 task state
- 生成 output file
- 派发 completion notification

因此总图里应把它放在 `Sk/PF_` 上方的 **outer async wrapper** 位置，而不是和 `PF_` 混成一个循环。

### 3.11 subcommands 和 special modes 会绕过 interactive 主链，直接连接对应子系统

真实架构里：

- `auth` 子命令更直接连 auth/settings
- `mcp` 子命令更直接连 MCP config/factory
- `agents`/`tmux`/`worktree` 更直接连 team/task runtime
- bridge / Chrome host 模式会直接接入 tool / MCP / browser bridge

所以总图中 subcommands 和 special modes 不应全部汇入同一个 main loop。

### 3.12 Chrome / MCP / bridge 不是边缘模块，而是系统级宿主分支

源码里有完整的 `ChromeBridgeClient`、Claude in Chrome MCP server、browser automation tool overrides 等实现。

这说明：

- Chrome bridge
- Chrome MCP
- remote bridge

都不是“插件例子”，而是 Claude Code 的正式运行分支。

## 4. 更偏源码调用链的时序图

```mermaid
sequenceDiagram
    autonumber
    actor User as 用户
    participant Main as 主进程入口
    participant Entry as 入口类型分类器
    participant Cmd as Commander 根命令
    participant Init as preAction 初始化链
    participant Setup as 设置、认证、提示词装配
    participant Sk as 回合包装层
    participant Core as 核心回合状态机
    participant Tool as 工具调用内循环
    participant Compact as 自动压缩管线
    participant Wrap as 异步任务封装层

    User->>Main: 启动 claude 或 SDK 宿主
    Main->>Entry: 分类入口类型
    Main->>Cmd: 构建 Commander 根命令
    Cmd->>Init: preAction 加载受管设置与 sinks

    alt 交互式路径
        Init->>Setup: 加载 settings / auth / memory / hooks / plugins / MCP
        Setup->>Sk: 进入回合包装层
        Sk->>Sk: 解析模型、过滤 forkContext、构建系统提示词
        Sk->>Core: 进入核心回合状态机

        loop 回合循环
            Core->>Core: 消息归一化、微压缩、自动压缩
            alt 需要压缩
                Core->>Compact: 执行自动压缩管线
                Compact-->>Core: 压缩后的 messages 与重建上下文
            end
            Core->>Core: 采样助手消息流
            alt 助手内容包含 tool_use
                Core->>Tool: 进入工具调用内循环
                Tool-->>Core: tool_result 回注到 transcript
            end
            Core-->>Sk: 流事件
            Sk-->>Wrap: 过滤后的 yield 事件
        end

        Wrap-->>User: 最终输出、任务结果、通知
    else 子命令或特殊模式
        Cmd-->>User: 分发到 auth / mcp / plugin / agents / bridge / chrome 处理器
    end
```

## 5. 产品/模块分层简化总览图

```mermaid
graph LR
    A[进程与宿主层<br/>CLI / SDK / VSCode / Desktop / Remote] --> B[路由与控制平面<br/>interactive / subcommands / special modes]
    B --> C[运行时装配层<br/>settings / auth / prompt / memory / permissions / hooks / plugins / MCP]
    C --> D[对话运行时层<br/>回合包装层 → 回合引擎 → 核心状态机]
    D --> E[执行内循环<br/>工具调用循环 / 自动压缩管线]
    E --> F[协作与异步层<br/>异步任务封装 / tasks / agent team / mailbox]
    F --> G[持久化与可观测层<br/>transcripts / auth state / memory / telemetry / output files]
```

## 6. 与专题文档的映射

- CLI 与路由：`Lesson/cli-and-routing-architecture.md`
- 配置认证设置：`Lesson/config-auth-and-settings-architecture.md`
- 权限与安全：`Lesson/permissions-and-safety-architecture.md`
- Hooks：`Lesson/hooks-and-automation-architecture.md`
- 插件：`Lesson/plugin-and-marketplace-architecture.md`
- MCP：`Lesson/mcp-integration-architecture.md`
- Memory：`Lesson/memory-system-architecture.md`
- Turn loop：`Lesson/turn-loop-architecture.md`
- Tool system：`Lesson/tool-system-architecture.md`
- Tool call loop：`Lesson/tool-call-loop-architecture.md`
- Agent team：`Lesson/agent-team-architecture.md`

## 7. 代码依据

- `NLY(...)`：`outputs/claude-cli-clean.js:374890-374904`
- `VLY(...)`：`outputs/claude-cli-clean.js:374906-374972`
- `yLY(...)` 与 Commander root / preAction：`outputs/claude-cli-clean.js:374995-375060+`
- `Sk(...)`：`outputs/claude-cli-clean.js:177565-177833`
- `PF_(...)` core loop：`outputs/claude-cli-clean.js:203992-204488`
- `autocompact` in core loop：`outputs/claude-cli-clean.js:204056-204083`
- `nZ6(...)`：`outputs/claude-cli-clean.js:135656-135830`
- `RG8(...)`：`outputs/claude-cli-clean.js:139625-139733`, `200250-200275`
- `ChromeBridgeClient`：`outputs/claude-cli-clean.js:19063-19703`
- Tool runtime / local executors：`outputs/claude-cli-clean.js:46974-47234`
- MCP integration / factory：`outputs/claude-cli-clean.js:145893-150838`
- Team runtime / `TeamCreate`：`outputs/claude-cli-clean.js:153493-153678`, `199699-199918`
