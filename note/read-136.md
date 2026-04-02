## 第 159 站：Buddy 同伴系统

### 这是什么

宠物系统——一个电子宠物（鸭子、鹅、猫、龙等），驻留在用户输入框旁边。有两个阶段：预告期（teaser）和正式期（live）。

---

### 核心类型

```typescript
Rarity: 'common' | 'uncommon' | 'rare' | | 'epic' | 'legendary'  // 权重 60/25/10/4/1
Species: 17 种 (duck, goose, blob, cat, dragon, octopus, owl, penguin, turtle, snail, ghost, axolotl, capybara, cactus, robot, rabbit, mushroom, chonk)
Eye: 6 种 ('·' | '✦' | '×' | '◉' | '@' | '°')
Hat: 8 种（包含 'none'）
StatName: DEBUGGING | PATIENCE | CHAOS | WISDOM | SNARK
```

**CompanionBones**——从 `hashString(userId + SALT)` 用 Mulberry32 PRNG 确定性生成的物理特征（species/eye/hat/rarity/shiny/stats）。**不持久化**——每次读取重新生成。

**CompanionSoul**——LLM 生成的名字+个性，持久化到 config。

---

### 关键设计决策

**防作弊设计**——Bones 从不持久化。用户不能编辑 config 作弊更高稀有度。`SPECIES` 字符串由 `String.fromCharCode()` 构造——将 `"duck"` 等字面量从 bundle 中隐藏——与模型 codename canary 碰撞。

**`getCompanion()`**——组合存储的 soul（来自 config）+ 新鲜 bones。

**`CompanionSprite.tsx`**——React 组件：
- `isBuddyTeaserWindow()`——2026 年 4 月 1-7 日对外部构建活跃
- `isBuddyLive()`——2026 年 4 月起活跃
- `useBuddyNotification()`——如果尚无同伴且预告期内，显示彩虹 `/buddy` 通知
- `findBuddyTriggerPositions(text)`——查找用户输入中的 `/buddy` 位置

---

### 元

问题：**这一站真正想解决的架构问题是什么？**

回答：这一站不是在讲一个彩蛋组件，而是在设计一种可持续、可个性化、又防作弊的轻量陪伴机制。`CompanionBones` 用 userId 哈希确定性生成但不持久化，`Soul` 才落盘，这让 Buddy 既像用户专属角色，又不变成可随意篡改的配置项。

### 反

问题：**如果把这一站的设计反过来，会发生什么？**

回答：如果把稀有度、物种和装饰都直接写进配置，Buddy 很快就会从“陪伴感”变成“可编辑数据”。那样系统最先失去的不是趣味，而是这套角色设定背后的稳定身份和设计节制。

### 空

问题：**跳出当前文件名，这一站背后更大的问题是什么？**

回答：再往外看，这一站关心的是产品怎样在严肃工作流里嵌入轻量人格化元素而不破坏系统秩序。Buddy 讨论的不是宠物，而是功能性工具里的长期情感设计。

### 读完后应抓住的事实

**确定性随机 + 安全防作弊**——Buddy 的物种/稀有度/眼睛/帽子从 userId 哈希 + Mulberry32 PRNG 确定性生成，但不持久化。这意味着：物种改名不会破坏已存储的同伴；用户不能编辑 config 作弊更高稀有度。SPECIES 字符串用 `String.fromCharCode()` 构建以避免字面量进入 bundle（与模型 codename canary 碰撞）。

---

## 第 160 站：Coordinator Mode（多智能体协调器）

### 这是什么

单文件模块，使 Claude 能作为**多智能体协调器**运行——生成 worker 子代理并行研究、实现和验证任务。双重门控：feature flag + 环境变量。

---

### 核心函数

**`isCoordinatorMode()`**——`feature('COORDINATOR_MODE')` 激活 AND `CLAUDE_CODE_COORDINATOR_MODE` env 为真。

**`matchSessionMode(sessionMode)`**——恢复会话时比较存储的模式（`'coordinator' | 'normal'`）与当前 env。不匹配时翻转 `process.env.CLAUDE_CODE_COORDINATOR_MODE` 并返回用户通知。

**`getCoordinatorUserContext()`**——注入 worker 能力上下文：
- 列出 worker 可用工具（`ASYNC_AGENT_ALLOWED_TOOLS` 减内部工具，或简单的 Bash/Read/Edit "simple" 模式）
- 列出已连接的 MCP 服务器
- 暂存盘目录信息（如启用）

**`getCoordinatorSystemPrompt()`**——~370 行系统提示，教导 LLM：
1. 理解自己作为协调器（非执行者）的角色
2. 使用 AgentTool/ SendMessageTool / TaskStopTool
3. 解析 worker 结果的 `<task-notification>` XML
4. 遵循 4 阶段工作流：Research（并行 worker）→ Synthesis（协调器）→ Implementation（worker）→ Verification（worker）
5. 编写自包含的 worker prompt（worker 看不到协调器对话）
6. 决定何时继续 vs 生成新 worker
7. 处理失败、停止 worker、编写精确 prompt

---

### 元

问题：**这一站真正想解决的架构问题是什么？**

回答：这一站真正定义的是 Claude 什么时候从执行者变成协调者，以及这种角色切换如何被显式建模。双重门控、恢复时 `matchSessionMode`、以及 4 阶段协调 prompt，说明它不是“多开几个 agent”，而是给多智能体协作建立专门的工作范式。

### 反

问题：**如果把这一站的设计反过来，会发生什么？**

回答：如果没有单独的协调器模式，主代理与 worker 的职责会混在一起，谁负责研究、谁负责实现、谁负责验证就会不断漂移。那时并行不再提升效率，反而会让上下文隔离和结果汇总一起失序。

### 空

问题：**跳出当前文件名，这一站背后更大的问题是什么？**

回答：更大的问题是，一个 AI 系统如何在内部形成分工，而不是永远只有单线程大脑。Coordinator Mode 回答的是多代理组织结构，而不是单个 agent 的提示词技巧。

### 读完后应抓住的 2 个事实

1. **双重门控**——模型 flag 门控能力，env var 门控每会话激活。这是生产级设计——feature flag 在构建时决定二进制是否有此功能，env var 在运行时决定是否启用。

2. **Prompt-as-Code**——系统提示是一个巨大的模板字符串字面量，从单个工具模块动态插入工具名。协调器有自己完整的 370 行 prompt，与主系统提示分离。

---

## 第 161 站：Server Direct Connect（远程会话代理）

### 这是什么

基于 WebSocket 的客户端，本地 CLI 连接到远程"直连"服务器，使 CLI 能够将会话委托给管理子进程的服务器。

---

### 核心类型

```typescript
SessionState: 'starting' | 'running' | 'detached' | 'stopping' | 'stopped'
SessionIndex: Record<string, SessionIndexEntry>  // 写入 ~/.claude/server-sessions.json
```

**`DirectConnectSessionManager`**——WebSocket 连接管理器：
- `connect()`——打开 WebSocket，设置 auth header
- `sendMessage(content)`——序列化为 `SDKUserMessage` 格式
- `respondToPermissionRequest(requestId, result)`——发送 control_response
- `sendInterrupt()`——发送 control_request subtype: 'interrupt'

---

### 创建流程

```
1. POST ${serverUrl}/sessions { cwd, dangerously_skip_permissions? }
2. 服务器响应 { session_id, ws_url, work_dir? }
3. 创建 DirectConnectSessionManager(config, callbacks)
4. connect() → 打开 WebSocket
5. 双向 JSON 消息：
   入站: control_request（权限）, SDK messages（assistant/result/system）
   出站: user messages, control_response（权限）, control_request（中断）
```

**消息过滤**——过滤 keep_alive、control_cancel_request、streamlined_text、streamlined_tool_use_summary、post_turn_summary 系统消息。未知控制请求子类型获得错误响应（永不挂起）。

---

---

### 元

问题：**这一站真正想解决的架构问题是什么？**

回答：这一站处理的是本地 CLI 把会话托管给远程服务器之后，双方怎样仍保持一个完整的交互协议。WebSocket 管理器、控制请求响应、消息过滤与未知子类型永不挂起，都说明它关注的是远程委托会话的生命线。

### 反

问题：**如果把这一站的设计反过来，会发生什么？**

回答：如果没有专门的直连层，本地 UI、权限请求和远端子进程管理会互相耦合，任何控制消息变种都可能让会话卡死。尤其 keep_alive、permission、interrupt 这些控制流若不先被规整，远程会话只会看似连通、实则脆弱。

### 空

问题：**跳出当前文件名，这一站背后更大的问题是什么？**

回答：更大的问题是，AI 会话如何在“本地体验”和“远端执行”之间稳定穿梭。这里讨论的不是一个 WebSocket 客户端，而是远程代理会话的协商机制。

## 第 162 站：Skill 加载与发现系统

### 这是什么

Skills 是 Claude Code 的可扩展机制——从文件、MCP 服务器、插件、管理目录、bundled 技能等多个来源发现和加载指令。

---

### 6 种技能来源

1. **Bundled skills**——硬编码随 CLI 二进制分发的技能
2. **File-based skills**——从 `.claude/skills/` 目录发现（managed/user/project/additional 范围）
3. **Legacy commands**——`.claude/commands/*.md`（弃用格式）
4. **MCP skills**——MCP 服务器通告的技能
5. **Dynamic/discovered skills**——会话期间触碰文件时从 `.claude/skills/` 发现
6. **Conditional skills**——有 `paths` frontmatter，在匹配文件操作时激活

---

### `loadSkillsDir.ts`——主发现引擎

**`getSkillDirCommands(cwd)`**——主入口，memoized：
```
1. 确定技能目录：managed, user, project（从 cwd 到 home）, additional（--add-dir）
2. 尊重 CLAUDE_CODE_DISABLE_POLICY_SKILLS, CLAUDE_CODE_BARE_MODE, isRestrictedToPluginOnly('skills')
3. 并行 Promise.all 加载所有来源
4. 按 realpath 去重（处理符号链接）
5. 分离无条件技能 vs 条件技能（按 paths frontmatter）
6. 返回无条件技能；存储条件技能供以后激活
```

**动态发现**：
- `discoverSkillDirsForPaths(filePaths, cwd)`——从触碰文件路径向上遍历，找 `.claude/skills/` 目录
- `addSkillDirectories(dirs)`——加载新发现目录的技能，更深路径优先
- `activateConditionalSkillsForPaths(filePaths, cwd)`——使用 gitignore 风格匹配激活匹配被触碰文件的条件技能
- 新技能出现时触发 `skillsLoaded` 信号

**`createSkillCommand()`**——构造 Command 对象：
- 替换参数占位符
- 替换 `${CLAUDE_SKILL_DIR}` 为技能目录
- 替换 `${CLAUDE_SESSION_ID}`
- 执行内联 shell 命令（MCP 技能除外以保安全）

---

### Bundled Skills（主要内置技能）

| Skill | 用途 |
|-------|------|
| **batch** | 编排 5-30 个并行 worker 进行大规模变更 |
| **simplify** | 启动 3 个并行评审代理（reuse/quality/efficiency）审查变更代码 |
| **debug** | 读取会话调试日志，搜索错误/警告 |
| **loremIpsum** | 占位符/生成器文本 |
| **remember** | 记忆持久化技能 |
| **stuck** | 诊断冻结/慢会话，提交到 #claude-code-feedback Slack（仅内部） |
| **updateConfig** | 修改 settings.json |
| **verify** | 验证变更 |
| **loop** | 定时 cron 风格任务调度 |
| **claudeApi** | 构建 Claude API 应用 |

---

### 安全设计

- Bundled 技能文件提取使用 `O_EXCL | O_NOFOLLOW` 和 `0o600` 模式防止符号链接攻击
- 路径验证防止 `..` 遍历
- MCP 技能从不执行内联 shell 命令
- `CLAUDE_SKILL_DIR` 和 `CLAUDE_SESSION_ID` 替换只对非 MCP 技能执行
- Gitignored 目录被跳过动态技能发现

---

### 元

问题：**这一站真正想解决的架构问题是什么？**

回答：这一站真正要解决的是，Skills 作为可扩展指令能力，如何从 bundled、文件、MCP、插件与动态发现等多源进入同一运行时。`getSkillDirCommands()`、条件技能激活、动态目录发现与安全替换规则，说明它承担的是“能力发现与装配”这一层。

### 反

问题：**如果把这一站的设计反过来，会发生什么？**

回答：如果没有统一发现系统，技能会分散在目录扫描、命令兼容、MCP 注入等多条私有路径里。最后新增一个 skill 看似简单，实际上会同时牵动加载顺序、路径安全、缓存与可见性规则。

### 空

问题：**跳出当前文件名，这一站背后更大的问题是什么？**

回答：更大的问题是，Claude Code 怎样把“可扩展能力”真正做成生态，而不是脚本堆。Skill 系统背后长期存在的，是扩展发现、激活时机与安全边界如何统一。

### 读完后应抓住的 2 个事实

1. **条件技能 + 动态发现**——条件技能有 `paths` frontmatter，仅在操作匹配文件时激活。这是上下文感知设计：只有触碰相关文件时，技能才出现。`discoverSkillDirsForPaths` 从触碰文件向上遍历发现新技能目录。

2. **mcpSkillBuilders 循环打破间接**——`loadSkillsDir.ts` 通过 `registerMCPSkillBuilders()` 注册构建函数。MCP 技能发现通过 `getMCPSkillBuilders()` 读取。这避免了 Bun 打包二进制中的运行时动态导入失败和导入图中的循环依赖。write-once 注册表模式。

---

## 第 163 站：Migrations 数据迁移系统

### 这是什么

启动时运行的迁移模块——将用户配置文件从旧格式/模式转换为新格式。每个都是自包含的幂等函数。

---

### 迁移模式

每个迁移：
1. 检查完成标志或旧数据存在性（已迁移则跳过）
2. 从旧位置读取
3. 写入新位置
4. 删除旧字段
5. 记录分析事件
6. 从不抛出——捕获错误记录日志而非破坏启动

---

### 迁移列表

| 迁移 | 操作 |
|------|------|
| **migrateAutoUpdatesToSettings** | 将 `autoUpdates: false` 从全局 config 移到 `settings.json` |
| **migrateBypassPermissionsAcceptedToSettings** | 将 bypassPermissionsModeAccepted 移到 `skipDangerousModePermissionPrompt` |
| **migrateEnableAllProjectMcpServersToSettings** | 迁移 MCP 服务器批准字段 |
| **migrateFennecToOpus** | 将 fennec 模型别名改名为 opus（仅内部用户） |
| **migrateLegacyOpusToCurrent** | 将显式 Opus 4.0/4.1 替换为 `opus` 别名 |
| **migrateOpusToOpus1m** | 将 `opus` 固定到 `opus[1m]`（Max/Team Premium 用户） |
| **migrateReplBridgeEnabledToRemoteControlAtStartup** | 重命名 replBridgeEnabled 为 remoteControlAtStartup |
| **migrateSonnet1mToSonnet45** | 将 `sonnet[1m]` 固定到显式 4.5 版本号 |
| **migrateSonnet45ToSonnet46** | 将 Pro/Max/Team Premium 用户从 4.5 移到 `sonnet` 别名（现为 4.6） |
| **resetAutoModeOptInForDefaultOffer** | 清除旧自动模式对话的接受 |
| **resetProToOpusDefault** | 自动将 Pro 用户迁移到 Opus 4.5 默认值 |

---

### 设计模式

1. **无完成标志的幂等性**——大多数模型迁移只在旧值完全匹配时执行，写入同一来源，避免静默提升到全局默认值
2. **不可逆更改的完成标志**——Sonnet 1M → 4.5 和 Pro → Opus 使用标记因为新别名可能重新创建旧值
3. **一次性 + 分析**——每次迁移记录 `tengu_xxx` 事件追踪
4. **安全失败**——全部 try/catch 包裹——迁移失败不能破坏启动
5. **特定来源读取**——模型迁移只从 `userSettings` 读取（非合并），避免将项目/本地固定提升为全局默认值

---

---

### 元

问题：**这一站真正想解决的架构问题是什么？**

回答：这一站的重点不是迁移了哪些字段，而是把启动时的历史兼容工作组织成幂等、自包含、失败不阻塞的升级管道。模型别名、权限设置、auto update 与 remote control 名称迁移，都只是这种兼容骨架上的实例。

### 反

问题：**如果把这一站的设计反过来，会发生什么？**

回答：如果没有独立迁移层，旧配置兼容会被塞进读取逻辑、默认值逻辑和 UI 逻辑里，任何历史包袱都会长期常驻主代码路径。那样最先恶化的不是用户体验，而是代码库永远无法把“过去”和“现在”分开处理。

### 空

问题：**跳出当前文件名，这一站背后更大的问题是什么？**

回答：再往外看，这一站回答的是产品怎样在持续演进时不抛弃存量用户状态。迁移系统关心的不是一次升级，而是系统如何与自己的历史长期相处。

## 第 164 站：Cost Tracking、History、QueryEngine

### cost-tracker.ts + costHook.ts——成本追踪

**`StoredCostState`**——totalCostUSD, totalAPIDuration, totalToolDuration, totalLinesAdded/Removed, modelUsage（按模型名分组）

**`ModelUsage`**——inputTokens, outputTokens, cacheRead/Creation, webSearchRequests, costUSD, contextWindow, maxOutputTokens

**持久化**——`getStoredSessionCosts()`/`restoreCostStateForSession()`/`saveCurrentSessionCosts()` 读写项目配置文件，按 sessionId 键控。

**`formatTotalCost()`**——格式化总成本摘要（成本、时长、代码变更、按模型使用量）。

**`costHook.ts`**——22 行 React hook。`useEffect` 挂载时注册 `process.on('exit')` 处理器，会话结束时写入成本概览到 stdout。

---

### history.ts——命令历史

**JSONL 历史文件**——`~/.claude/history.jsonl`

```typescript
LogEntry: display text, pastedContents, timestamp, project, sessionId
StoredPastedContent: id, type ('text'|'image'), contentHash, mediaType, filename
```

**关键模式**：
- **写缓冲**——`addToPromptHistory()` 推入 `pendingEntries[]`，`flushPromptHistory()` 用 flock 写入磁盘
- **反向读取**——`readLinesReverse()` O(1) 从 JSONL 文件读取最新条目
- **双读取器**——`getHistory()` 为上箭头（当前会话优先）；`getTimestampedHistory()` 为 Ctrl+R（按显示文本去重，每项目 100 条）
- **粘贴内容外部化**——超过 1024 字符的内容存为哈希 + 外部存储（fire-and-forget）；内联内容直接存在日志条目
- **撤消**——`removeLastFromHistory()` 支持撤消最近历史条目（待处理缓冲弹出或已刷新的 skip-set）

---

### QueryEngine.ts——核心会话引擎

**1296 行类**——中央会话引擎，每次对话一个 QueryEngine。每次 `submitMessage()` 开始一个新 turn。从旧 `ask()` 函数提取，服务于 headless/SDK 和 REPL 路径。

**关键流程**：
```
submitMessage() → AsyncGenerator<SDKMessage>:
1. 处理用户输入（斜杠命令）
2. 记录 transcript
3. 获取系统提示
4. 生成系统初始化消息
5. 进入 query() 循环，迭代消息
6. 每次迭代检查预算/轮次/结构化输出限制
7. 计算结果（success/error_max_turns/error_max_budget_usd/...）
```

**Transcript 写策略**——助手消息 fire-and-forget；用户/compact_boundary 消息 await；生成结果前刷新。

---

### 元

问题：**这一站真正想解决的架构问题是什么？**

回答：这一站把成本记账、提示历史与 QueryEngine 并列，是因为它们共同围绕“一次对话 turn 的真实生命周期”展开。`submitMessage()` 的 AsyncGenerator、按 sessionId 持久化成本、JSONL 历史与反向读取，都在为会话运行和回看建立统一中枢。

### 反

问题：**如果把这一站的设计反过来，会发生什么？**

回答：如果没有这层集中编排，成本统计会和执行流脱节，历史会只剩简单日志，REPL 与 headless 也会各自维护一套会话引擎。那时系统还能跑，但已经没有共同的 turn 边界和统一的运行语义。

### 空

问题：**跳出当前文件名，这一站背后更大的问题是什么？**

回答：更大的问题是，AI 会话怎样既是一次交互过程，又是可累计、可恢复、可计费的长期对象。这里讨论的不是一个类，而是“会话作为基本单位”如何被系统化。

### 读完后应抓住的 2 个事实

1. **QueryEngine 的 AsyncGenerator 模式**——`submitMessage()` 是 `AsyncGenerator<SDKMessage>`。这允许调用者消费流式消息序列（用户、助手、系统、进度、工具摘要），并以结果消息终止。这是 REPL 和 headless SDK 的统一接口。

2. **History 反向读取**——`readLinesReverse()` 从 JSONL 文件尾部向前读取。这是经典设计：用户最近输入最先显示，不需要加载整个文件。Ctrl+R 使用此模式。

---

## 第 165 站：Setup、Task、Vim Mode

### setup.ts——启动初始化管道

**~477 行**，首次渲染前调用：

**关键步骤**：
- Node >= 18 版本检查
- Feature gates：UDS_INBOX（消息服务器）、CONTEXT_COLLAPSE（上下文压缩初始化）、COMMIT_ATTRIBUTION（归属钩子）、TEAMMEM（团队内存监视器）
- Bare mode（`--bare`/SIMPLE）门控：跳过 UDS、teammate 快照、发布说明预取、归属钩子等——最小化启动开销用于脚本使用
- Worktree 流程：验证 git 或 hook，解析规范根，创建 worktree+分支，可选创建 tmux 会话，切换 CWD 和 projectRoot
- 安全检查：`--dangerously-skip-permissions`——拒绝 root/sudo，或 ant 用户除非在 Docker/沙盒且无网络
- 写入 `tengu_exit` 分析事件来自上一次会话成本/时长
- 使用 profile checkpoints 进行启动性能测量

---

### Task.ts——任务抽象

```typescript
TaskType: 'local_bash' | 'local_agent' | 'remote_agent' | 'in_process_teammate' | 'local_workflow' | 'monitor_mcp' | 'dream'
TaskStatus: 'pending' | 'running' | 'completed' | 'failed' | 'killed'
```

**任务 ID 生成**——前缀 + 8 随机字母数字：
- `b` = bash, `a` = agent, `r` = remote, `t` = teammate, `w` = workflow, `m` = monitor, `d` = dream
- `crypto.randomBytes(8)` 映射到 36 字符字母表——约 2.8 万亿种组合

---

### Vim Mode（vim/ 目录）——完整 Vim 状态机

**5 个文件**——types/motions/operators/textObjects/transitions

**`VimState`**——INSERT 模式（跟踪 insertedText）和 NORMAL 模式（跟踪 CommandState）的联合

**`CommandState`**——判别联合：idle, count, operator, operatorCount, operatorFind, operatorTextObj, find, g, operatorG, replace, indent

**`PersistentState`**——lastChange, lastFind, register, registerIsLinewise——持久化跨状态的 Vim 寄存器

**`transition(state, input, ctx)`**——状态转换表。分派到 `fromIdle`, `fromCount`, `fromOperator` 等状态处理器。`handleNormalInput()` 和 `handleOperatorInput()` 是共享分派器。

**Motions**——`resolveMotion()` / `applySingleMotion()` 将键映射为 Cursor 方法调用

**Operators**——`executeOperatorMotion()`, `executeOperatorFind()`, `executeOperatorTextObj()`, `executeLineOp()`（dd/cc/yy）, `executeX()` 等，每个记录变更用于点重复

**Text Objects**——`findTextObject()` 处理词对象（w/W）、引号对象（成对相同字符定界符）、括号对象（深度追踪）。使用图段分割 Unicode 正确性。Inner（`i`）vs Around（`a`）范围控制包含定界符。

**安全设计**——所有操作都是纯函数 + TypeScript 完备性检查。无副作用。点重复通过 `RecordedChange` 联合建模变更，然后回放。

---

### 元

问题：**这一站真正想解决的架构问题是什么？**

回答：这一站关心的是，Claude Code 启动前、运行中和输入层分别需要怎样的稳定抽象。`setup.ts` 管启动管道，`Task.ts` 管后台任务身份，Vim 模式则把复杂编辑交互收束为状态机，它们共同回答的是“系统如何进入并维持可操作状态”。

### 反

问题：**如果把这一站的设计反过来，会发生什么？**

回答：如果没有这些边界，启动逻辑会和运行时互相渗透，后台任务会失去统一身份，输入行为也会变成一堆键位特判。特别是 Vim 这类复杂编辑语义，一旦不做成状态机，就只剩不断加补丁的命令分支。

### 空

问题：**跳出当前文件名，这一站背后更大的问题是什么？**

回答：更大的问题是，终端智能体如何同时具备产品化启动流程、任务化执行模型和编辑器级输入能力。这里串起来的不是三个杂项，而是 Claude Code 作为工具产品的操作基础。

### 读完后应抓住的 2 个事实

1. **Vim 模式是纯状态机**——每个 Vim 操作（运动、运算符、文本对象、转换）是纯函数。`CommandState` 是判别联合，TypeScript 完备性检查确保没有遗漏状态。点重复通过 `RecordedChange` 联合实现——记录变更然后回放。这是正确的设计：没有副作用，测试友好，状态转换可追踪。

2. **Setup 的 Bare Mode 门控**——`--bare`/SIMPLE 跳过 UDS、teammate 快照、发布说明预取、归属钩子。这是为脚本/自动化使用设计的路径——最小启动开销，跳过用户交互相关的所有内容。

---

## 第 166 站：Keybindings 系统（14 个文件）

### 这是什么

完整的键盘快捷键系统——上下文感知解析、多按键和弦支持、用户自定义 JSON 配置、验证。

---

### 数据流

```
1. defaultBindings.ts——静态默认绑定，按上下文组织（18 个上下文）
2. loadUserBindings.ts——加载 ~/.claude/keybindings.json，chokidar 热重载，最后绑定胜出
3. parser.ts——解析按键字符串（"ctrl+shift+k"）为 ParsedKeystroke，和弦（"ctrl+k ctrl+s"）为 Chord 数组
4. match.ts——匹配 Ink Key+input 和 ParsedKeystroke，处理终端怪癖（escape 设置 key.meta）
5. resolver.ts——解析键入到动作，和弦状态追踪（match/none/unbound/chord_started/chord_cancelled）
6. validate.ts——全面验证：解析错误、无效上下文、无效动作、重复键、保留快捷键警告
7. schema.ts——Zod 模式，18 个有效上下文和 60+ 个有效动作标识符
8. reservedShortcuts.ts——不可重绑定（ctrl+c, ctrl+d, ctrl+m）、终端保留（ctrl+z, ctrl+\）
9. useKeybinding.ts——React hooks，插入 Ink 的 useInput
10. KeybindingContext.tsx——React 上下文提供者，处理器注册表
```

---

### 关键模式

**上下文优先级**：注册的活跃上下文（如 Chat, ThemePicker）> 指定上下文 > Global

**和弦**——前缀匹配的"保持和等待"状态，escape 取消等待中的和弦，null-action 取消绑定

**平台特定默认值**——Windows 上 alt+v vs ctrl+v 用于图片粘贴，shift+tab vs meta+m 用于模式循环

**终端 VT 模式检测**——Windows 需要 Node >= 22.17.0/24.2.0 或 Bun >= 1.2.23

**Feature-gated 绑定**——KAIROS/toggleBrief、QUICK_SEARCH/globalSearch、TERMINAL_PANEL/toggleTerminal、MESSAGE_ACTIONS/messageActions、VOICE_MODE/pushToTalk

---

---

### 元

问题：**这一站真正想解决的架构问题是什么？**

回答：这一站真正定义的是终端交互中的“按键语言”，而不是几个默认快捷键。上下文优先级、和弦状态、用户 JSON 配置、验证与热重载，说明它处理的是输入解析、动作路由与用户自定义之间的秩序。

### 反

问题：**如果把这一站的设计反过来，会发生什么？**

回答：如果快捷键逻辑直接散在各组件里，最先崩掉的是上下文切换和多键和弦。那样新增一个绑定不仅会和终端保留键冲突，也会让 Chat、ThemePicker、Global 等上下文之间失去明确优先级。

### 空

问题：**跳出当前文件名，这一站背后更大的问题是什么？**

回答：更大的问题是，复杂 CLI 产品怎样把交互手势做成一套可扩展、可迁移的协议。键位系统讨论的不是按什么键，而是用户如何稳定操控越来越多的能力。

## 第 167 站：Upstream Proxy + Output Styles + Native Bindings

### Upstream Proxy（CCR 容器代理）

**目的**——在 CCR 会话容器内启动本地 CONNECT-over-WebSocket 中继，使容器内进程能通过安全中继向外连接。

**`upstreamproxy.ts`** 流程：
```
1. 从 /run/ccr/session_token 读取会话令牌
2. 调用 prctl(PR_SET_DUMPABLE, 0) 阻止同 UID ptrace
3. 下载上游代理 CA 证书，合并到系统 bundle
4. 启动本地 CONNECT-over-WebSocket 中继
5. 中继启动后取消链接令牌文件
6. 暴露 HTTPS_PROXY / SSL_CERT_FILE 环境变量
7. 任何错误 fail-open
```

**`relay.ts`**——TCP-to-WebSocket 中继，实现 HTTP CONNECT over WebSocket：
- 两阶段：积累 CONNECT 请求直到 CRLFCRLF → 解析 CONNECT 行 → 通过 WS 隧道转发客户端字节
- Bun（Bun.listen）和 Node（net.createServer）运行时检测
- 手动 protobuf 编码 `UpstreamProxyChunk` 避免 protobufjs 依赖
- 处理边缘情况：合并的 CONNECT + ClientHello 包、写反压、30s ping 空闲保活、ws error/close 竞态

---

### Output Styles

**`loadOutputStylesDir.ts`**——从 `.claude/output-styles/` 目录加载基于 Markdown 的输出样式定义。每个 Markdown 文件成为一个命名样式——文件名为样式名、内容为样式提示、frontmatter 定义 name/description/keep-coding-instructions。

**Memoized**——用 `lodash-es/memoize` 缓存。`clearOutputStyleCaches()` 清除 memoize 缓存和插件输出样式缓存。

---

### Native Bindings（native-ts/）

TypeScript 类型安全垫片包裹原生/Bun-优化模块：

| 模块 | 用途 |
|------|------|
| **yoga-layout** | Flexbox 布局引擎（Ink 终端 UI 渲染使用） |
| **file-index** | 原生文件索引/搜索模块 |
| **color-diff** | 颜色差异计算（用于主题/视觉差异渲染） |

这些是薄类型定义层，允许代码库通过标准 TypeScript 类型检查导入原生模块——实际实现是原生 Bun C 绑定或编译库。

---

### 元

问题：**这一站真正想解决的架构问题是什么？**

回答：这一站表面分散，实际都在处理“运行时底座”的边缘能力。无论是 CCR 容器里的 CONNECT-over-WebSocket 代理、Markdown 输出样式目录，还是原生绑定的类型垫片，核心都在于让 Claude Code 能安全接入更低层的网络、显示和原生能力。

### 反

问题：**如果把这一站的设计反过来，会发生什么？**

回答：如果这些能力没有独立边界，代理初始化、证书注入、样式发现和原生模块引用都会渗入业务逻辑。那样系统最先失去的不是功能，而是对底层环境差异的可靠封装。

### 空

问题：**跳出当前文件名，这一站背后更大的问题是什么？**

回答：更大的问题是，一个高级 CLI 如何站在不稳定的操作系统、容器和渲染环境之上仍保持统一行为。这里讨论的是底层适配层，而不是某个代理或样式文件本身。

### 读完后应抓住的 2 个事实

1. **Upstream Proxy 的手动 protobuf**——`encodeChunk()` / `decodeChunk()` 手动实现 protobuf 线缆格式编码（tag 0x0a + varint length + data）。这是因为 protobufjs 依赖过大。这是生产优化——减少运行时依赖链。

2. **Keybindings 和弦状态机**——和弦（如 "ctrl+k ctrl+s"）被解析为 Chord 数组。resolver 跟踪前缀匹配的状态（match/none/unbound/chord_started/chord_cancelled）。escape 取消等待中的和弦。这是终端快捷键的通用设计——支持多步骤组合键。
