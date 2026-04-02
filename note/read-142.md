## 第 195 站：Entrypoints + CLI + Init

### entrypoints/cli.tsx——进程入口

**真正的进程入口点**——运行在顶级 `void main()` 调用中。

**快速路径分派**——专用子命令（`--version`, `daemon`, `bridge`, `ps`, `logs`, `attach`, `kill`, templates, environment-runner, self-hosted-runner）使用动态 `import()` 最小模块加载。

**如果未命中快速路径**——导入并调用 `src/main.tsx` 的 `main()` 函数。

**早期输入捕获**——`startCapturingEarlyInput()` 在模块加载期间捕获输入。

### entrypoints/init.ts——环境初始化

**初始化管道**——在首次渲染前调用。~477 行。

**关键步骤**：
- Node >= 18 版本检查
- 会话 ID 设置
- UDS 消息（UDS_INBOX 特性门控）
- 上下文压缩初始化（CONTEXT_COLLAPSE 特性门控）
- 归属钩子（COMMIT_ATTRIBUTION 特性门控）
- 团队内存监视器（TEAMMEM 特性门控）
- 终端备份恢复
- CWD 设置
- Hook 配置快照
- Worktree 创建
- 后台作业启动（10 分钟延迟 + 用户交互感知）
- 预取
- 权限绕过验证
- 会话退出分析

**Bare Mode**——`--bare`/SIMPLE 跳过 UDS、teammate 快照、发布说明预取、归属钩子等用于脚本/自动化的最小开销路径。

### cli/print.ts——打印模式

**`--print` / `-p` 模式**——非交互 headless 模式。直接输出结果到 stdout，无 REPL。使用 headlessProfiler 进行每轮延迟分析。

---

---

### 元

问题：**这一站真正想解决的架构问题是什么？**

回答：这一站真正回答的是：**Claude Code 到底怎样从“一个被启动的进程”变成“一个已经准备好工作的系统”？** 这里把 `entrypoints/cli.tsx`、`entrypoints/init.ts`、`cli/print.ts` 放在一起，不是在罗列入口文件，而是在说明启动链的三件事：谁先接住进程、谁把环境准备好、谁决定走交互还是 headless 路径。只有把这三层拆清楚，后面的 REPL、Bridge、print 模式才有共同起点。

### 反

问题：**如果把这一站的设计反过来，会发生什么？**

回答：如果没有这条清晰的入口链，而是让各种子命令、模式分支、初始化逻辑彼此穿插，系统很快就会失去启动秩序。快速路径会和完整启动路径互相污染，初始化前提会被重复执行或遗漏，print 模式和交互模式也会各走各的私有流程。入口层一旦散掉，整个系统最先损失的不是功能，而是“从哪里开始才算真正开始”的确定性。

### 空

问题：**跳出当前文件名，这一站背后更大的问题是什么？**

回答：跳出这些入口文件后，更大的问题其实是：**一个复杂 CLI 产品如何同时支持多种启动姿态，却又不把自己拆成多套系统。** Claude Code 现在用入口分派 + 初始化管道 + 模式分流来回答这个问题；以后具体路径可以调整，但“多入口共享同一启动骨架”这件事会一直存在。

## 第 196 站：Commands 系统

### Commands 目录

斜杠命令的注册和执行。涵盖 MCP 命令、插件命令等。

### MCP 命令（commands/mcp/）

MCP 服务器的 CLI 管理命令——添加/删除/启动/停止/配置 MCP 服务器。从 `mcp.json` 配置文件读取。

---

---

### 元

问题：**这一站真正想解决的架构问题是什么？**

回答：这一站真正要回答的是：**当功能越来越多时，用户入口如何不退化成一堆零散开关，而仍然保持成一套可发现、可扩展、可治理的命令面。** Commands 系统的意义，不只是“能执行 slash command”，而是把命令注册、子命令组织、MCP 管理入口这些东西纳入同一套秩序，让新增能力知道该挂在哪里，用户也知道该从哪里进入。

### 反

问题：**如果把这一站的设计反过来，会发生什么？**

回答：如果反过来做，不建立命令系统，而是让每个功能自己暴露入口，那么开始时会觉得灵活，后来就会变成混乱：有的能力藏在主命令参数里，有的藏在斜杠命令里，有的又另起一套管理方式。尤其像 MCP 这种需要添加、删除、启动、停止、配置的一整组动作，如果没有统一命令面，维护成本会指数上升。

### 空

问题：**跳出当前文件名，这一站背后更大的问题是什么？**

回答：跳出 Commands 目录之后，更大的问题其实是：**一个不断生长的智能体系统，怎样把“能力”稳定地暴露给用户。** 命令系统只是当前答案；以后可能有新的交互方式，但“能力入口必须有组织”这个问题永远都在。

## 第 197 站：Constants 常量系统

### 完整的常量文件

| 文件 | 内容 |
|------|------|
| **tools.ts** | 工具名常量 |
| **oauth.ts** | OAuth 范围、端点、客户端 ID |
| **common.ts** | 通用常量 |
| **messages.ts** | 消息字符串常量 |
| **figures.ts** | Unicode 字符/图标常量 |
| **cyberRiskInstruction.ts** | 网络安全风险指令 |
| **errorIds.ts** | 错误 ID 映射 |
| **keys.ts** | 加密/配置键 |
| **outputStyles.ts** | 输出样式常量 |
| **product.ts** | 产品名称/版本 |
| **system.ts** | 系统级常量 |
| **turnCompletionVerbs.ts** | 轮次完成动词（Baked/Brewed/Churned/Cogitated/Cooked/Crunched/Sauted/Worked） |
| **xml.ts** | XML 标记常量 |
| **spinnerVerbs.ts** | 200+ 个 whimsical 现在时动词用于加载 spinner |
| **toolLimits.ts** | 工具结果大小限制（50K chars, 100K tokens, 400KB max） |
| **files.ts** | 文件路径常量 |
| **github-app.ts** | GitHub App 集成常量 |
| **betas.ts** | Beta 头常量 |
| **systemPromptSections.ts** | 系统提示段工厂 |

---

---

### 元

问题：**这一站真正想解决的架构问题是什么？**

回答：这一站真正要处理的是：**跨模块共享的词汇、限制、文案和协议边界，怎样才能始终指向同一个意思。** `tools.ts`、`messages.ts`、`toolLimits.ts`、`systemPromptSections.ts` 这些文件放在一起，说明常量系统不是“收纳杂项”的地方，而是整套系统的公共语义层。没有它，很多模块虽然还能跑，但说的已经不是同一种语言。

### 反

问题：**如果把这一站的设计反过来，会发生什么？**

回答：如果反过来做，把常量散落到各自模块里，短期内会显得更就近、更顺手；但随着系统变大，同一个概念会开始出现多份版本。工具名、消息文案、限制阈值、Beta 头、输出样式一旦各自维护，最后最难发现的 bug 反而是“看起来都对，但彼此不再一致”。

### 空

问题：**跳出当前文件名，这一站背后更大的问题是什么？**

回答：跳出常量文件本身，更大的问题其实是：**一个复杂系统怎样维护自己的共享语义，不让跨模块协作建立在隐含约定上。** 常量系统只是最显性的载体，真正长期存在的，是“系统公共语言如何保持稳定”这个问题。

## 第 198 站：UI 组件架构完整图

### Ink Framework Root（ink/components/App.tsx）

**不是 Claude Code 应用根**——这是 Ink 框架根，终端渲染引擎基础。

**关键能力**：
- Kitty 键盘协议、鼠标追踪（DEC 私有模式）
- 光标可见性（默认隐藏，卸载或无障碍模式时显示）
- 带超时刷新的 escape 序列解析（50ms 正常，500ms 粘贴）
- 多击检测——500ms 超时，1-cell 距离容差
- **终端恢复检测**——检测 >5 秒 stdin 间隔（tmux 分离、SSH 重连、笔唤醒）并重新断言终端模式
- 错误边界——`getDerivedStateFromError` 捕获渲染错误并显示 `ErrorOverview`

**嵌套 Context Provider 栈**：
```
TerminalSizeContext → AppContext → StdinContext → TerminalFocusProvider → ClockProvider → CursorDeclarationContext
```

---

### Messages 系统（~30+ 消息组件）

**联合判别器模式**——对话历史中每种消息类型映射到特定组件。

**用户消息**——UserPromptMessage, UserBashInputMessage, UserBashOutputMessage, UserCommandMessage, UserLocalCommandOutputMessage, UserMemoryInputMessage, UserPlanMessage, UserImageMessage, UserResourceUpdateMessage, UserTeammateMessage, UserChannelMessage, UserTextMessage

**助手消息**——AssistantTextMessage, AssistantThinkingMessage, AssistantToolUseMessage, AssistantRedactedThinkingMessage

**工具结果**——UserToolResultMessage 子目录（success/error/rejected/canceled 变体，共享 utils.tsx）

**系统消息**——HookProgressMessage, RateLimitMessage, SystemAPIErrorMessage, SystemTextMessage, TaskAssignmentMessage, PlanApprovalMessage, ShutdownMessage, AttachmentMessage, CompactBoundaryMessage, GroupedToolUseContent, CollapsedReadSearchContent

---

### PromptInput（21 个文件）

**主要输入子系统**——完整的 REPL 输入体验。

| 组件 | 用途 |
|------|------|
| **PromptInput.tsx** | 主输入组件 |
| **ShimmeredInput.tsx** | 带微光/动画效果的输入 |
| **HistorySearchInput.tsx** | 命令历史搜索 |
| **PromptInputFooter.tsx** | 输入下方状态栏 |
| **PromptInputFooterSuggestions.tsx** | 建议 chips |
| **PromptInputHelpMenu.tsx** | 帮助菜单/浮层 |
| **PromptInputModeIndicator.tsx** | 当前输入模式指示 |
| **PromptInputQueuedCommands.tsx** | 排队命令显示 |
| **PromptInputStashNotice.tsx** | Git stash 警告 |
| **VoiceIndicator.tsx** | 语音模式指示器 |
| **Notifications.tsx** | 输入内通知 |
| **SandboxPromptFooterHint.tsx** | 沙盒模式提示 |

**Hooks**——useMaybeTruncateInput, usePromptInputPlaceholder, useShowFastIconHint, useSwarmBanner

---

### Markdown（三层架构）

```
Markdown（检查语法高亮设置）
  → Suspense（无高亮回退）
    → MarkdownWithHighlight（懒加载 getCliHighlightPromise()）
      → MarkdownBody（实际渲染器）
```

**Token 缓存**——模块级 `Map<string, Token[]>`，LRU 驱逐（最大 500 条目），按内容哈希（非完整字符串）键控避免内存保留。

**快速路径**——`hasMarkdownSyntax()` 检查前 500 字符的正则 `/[#*\\`|[>\\\\-_~]|\n\n|^\d+\. |\n\d+\. /`。如果没有找到 markdown，返回单个段落 token 数组而不调用 `marked.lexer`。

**Hybrid 渲染**——表格渲染为 `<MarkdownTable>` React 组件；其他所有内容通过 `formatToken()` 格式化为 ANSI 字符串，通过 `<Ansi>` 渲染。

**`StreamingMarkdown`**——独立导出用于流式内容。使用 `useRef` 支持的 `stablePrefixRef` 单调前进。每个 delta 只对不稳定后缀重新词法分析，保持稳定前缀在嵌套 `<Markdown>` 内 memoized。

---

### Spinner 系统

**SpinnerWithVerb**——在 `BriefSpinner` 和 `SpinnerWithVerbInner` 之间门控（基于 Kairos/brief 特性标志）。

**SpinnerGlyph**——渲染单个 spinner 字符：
- **减少运动模式**——慢慢闪烁橙色点（2 秒周期，交替暗/亮）
- **停顿检测**——当 `stalledIntensity > 0`，平滑从主题色插值到 ERROR_RED（{r:171, g:43, b:63}）。如果 RGB 解析失败（ANSI 主题），回退到二进制开关到 "error" 颜色当强度 > 0.5

**TeammateSpinnerTree**——显示运行中子代理的树（agent swarms）。团队 "lead" 行带框线字符（`\u2552\u2550` / `\u250c\u2500`），映射 teammate 任务到 `TeammateSpinnerLine`。包含 `HideRow` 子组件用于折叠树。显示 token 计数、活动提示和 "enter to view" 指导。

---

### TaskListV2

**动态截断**——基于终端行数动态计算 `maxDisplay`（`rows <= 10 ? 0 : Math.min(10, Math.max(3, rows - 14))`）。超出时优先：最近完成（30s TTL）、进行中、待处理、然后是较旧已完成。

**完成追踪**——使用 refs 追踪 `completionTimestampsRef`（Map<string, number>）和 `previousCompletedIdsRef`（Set<string>）检测新完成转换。为下次到期间安排 `setTimeout` 强制重新渲染。

**Agent Swarms 集成**——构建 `teammateColors` 和 `teammateActivity`（来自最近工具使用的活动描述）。

**排序**——待处理任务按被阻塞状态先排序（未阻塞在被阻塞前），然后按 ID。其他所有按 ID 排序。

隐藏摘要——截断时显示 "... +N pending, M in progress, K completed"。

---

### Theme System

**三状态存储**——`themeSetting`（用户偏好）、`previewTheme`（临时选择器预览）、`systemTheme`（从 `$COLORFGBG` 或 OSC 11 检测的终端主题）。

**解析**——`currentTheme = activeSetting === 'auto' ? systemTheme : activeSetting`。选择器打开时预览获胜。

**`AUTO_THEME` 特性**——当启用且 activeSetting 为 'auto' 时，动态导入并运行 `watchSystemTheme()` 轮询终端主题变更。从 `getSystemThemeName()` 种子避免闪烁。

**TextHoverColorContext**——React context 覆盖子树中未着色文本（跨越 Box 边界，绕过 Ink 缺乏 CSS 级联）。

**ThemedBox / ThemedText**——带主题色解析的 Ink `<Box>` / `<Text>` 包装器。颜色优先级：显式 `color` > `TextHoverColorContext` > `dimColor`（使用 theme.inactive）。

---

### Dialog Launchers

异步启动器，用于一次性对话框 JSX 站点，动态导入其组件：

| 函数 | 用途 |
|------|------|
| `launchSnapshotUpdateDialog` | 代理内存快照合并/保留/替换提示 |
| `launchInvalidSettingsDialog` | 设置验证错误显示 |
| `launchAssistantSessionChooser` | Bridge 会话选择器 |
| `launchAssistantInstallWizard` | 新安装向导 |
| `launchTeleportResumeWrapper` | 交互式 teleport 会话选择器 |
| `launchTeleportRepoMismatchDialog` | 本地 checkout 选择器用于 teleport 目标仓库 |
| `launchResumeChooser` | 挂载 ResumeConversation 屏幕 |

---

### Interactive Helpers

**`showDialog<T>(root, renderer)`**——核心对话框原语——渲染 `renderer(done)` 返回的 JSX，当 `done(result)` 调用时解析 Promise。

**`showSetupDialog<T>()`**——封装 `showDialog` 带 `AppStateProvider` 和 `KeybindingSetup` 提供者减少样板。

**`showSetupScreens()`**——顺序设置对话框管道：
1. **入职**（主题选择 + 首次运行向导）
2. **TrustDialog**——工作空间信任边界（不可信仓库警告），已信任时自动跳过
3. GrowthBook 重新初始化（信任后）
4. **MCP 服务器批准**（`mcp.json`）
5. **Claude.md 外部包含对话框**
6. GitHub 仓库路径映射更新
7. 环境变量应用
8. 信任后遥测初始化
9. **GroveDialog**（策略同意）如果合格

---

### 元

问题：**这一站真正想解决的架构问题是什么？**

回答：这一站真正要回答的是：**终端里的复杂会话体验，怎样被拆成一套可渲染、可输入、可流式更新、可组合的 UI 体系。** 所以这里不是在堆组件目录，而是在给出一张 UI 总图：从 Ink framework root，到 Messages、PromptInput、Markdown、Spinner、Theme、Dialog launchers，整套界面并不是“显示结果”这么简单，而是 Claude Code 作为终端产品的表现层骨架。

### 反

问题：**如果把这一站的设计反过来，会发生什么？**

回答：如果反过来做，不把 UI 当成一套完整架构，而只把它看成若干显示组件，那么最先碎掉的就是一致性。消息渲染会各写各的，流式 Markdown 会和普通文本分家，输入区、任务区、提示区会形成各自的局部约定。到最后，用户看到的不是一套会话界面，而是一堆拼起来还能工作的终端片段。

### 空

问题：**跳出当前文件名，这一站背后更大的问题是什么？**

回答：跳出这张 UI 图之后，更大的问题其实是：**在终端这种受限媒介里，复杂智能体系统怎样稳定地表达状态、历史、反馈和操作入口。** 这就是为什么这里既讨论 Ink root，也讨论 Markdown、Spinner、TaskList 和 Setup Screens——因为真正的主题不是组件，而是终端交互如何成为产品能力。

### 读完后应抓住的 3 个事实

1. **Ink 是深度定制 fork 而非 npm 包**——`src/ink/` 是完整的 fork，包含自定义 reconciler、Yoga 布局、终端协议解析。App.tsx（框架根）处理 Kitty 键盘协议、鼠标追踪、多击检测和终端恢复检测（>5s stdin 间隔触发重新断言）。这是生产设计——这些不是标准 Ink 提供的能力。

2. **Markdown 两阶段词法分析优化**——StreamingMarkdown 使用 `stablePrefixRef` 单调前进——只对新数据重新词法分析，保持前缀 memoized。快速路径 `hasMarkdownSyntax()` 检查前 500 字符，如果没有 markdown 语法跳过 `marked.lexer`。这些是流式 Markdown 渲染的性能优化。

3. **Setup Screens 管道顺序**——Onboarding → TrustDialog → GrowthBook 重新初始化 → MCP 批准 → Claude.md 外部包含 → GitHub 路径映射 → 环境变量 → 遥测 → GroveDialog。关键：GrowthBook 在 TrustDialog 之后重新初始化，因为信任决定了哪些 features 可用。遥测在信任之后初始化因为需要用户的 telemetry consent。
