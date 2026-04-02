## 第 189 站：System Prompt + Thinking + Effort + Betas + Fast Mode

### systemPrompt.ts——系统提示构建

**优先级链** `buildEffectiveSystemPrompt()`：
1. Override 替换一切
2. 协调器模式提示
3. 代理提示（主动模式追加，否则替换默认）
4. 自定义提示
5. 默认提示

始终在末尾追加 `appendSystemPrompt`。

死代码消除通过 feature-flagged 懒 require（`PROACTIVE`, `KAIROS`）。

---

### thinking.ts——思考管理

**`ThinkingConfig`**——`{ type: 'adaptive' | 'enabled' | 'disabled' }`

**`modelSupportsThinking()`**——1P 和 Foundry：所有 Claude 4+（包括 Haiku 4.5）。3P：Opus 4+ 和 Sonnet 4+。

**`modelSupportsAdaptiveThinking()`**——仅 Opus 4-6 和 Sonnet 4-6。未知模型在 1P/Foundry 上默认 true。

**`hasUltrathinkKeyword(text)`**——检测文本中的 "ultrathink" 关键词（带位置以用于 UI 高亮）。

### effort.ts——努力程度参数

**`EffortLevel`**——`'low' | 'medium' | 'high' | 'max'`

**优先级链** `resolveAppliedEffort()`：env → appState → 模型默认值。

**特殊规则**：`max` 仅 Opus 4-6 公开支持。对非 Opus 模型强制 max→high。`'max'` 是会话级（非外部用户持久化），其他可持久化。

**Pro 用户默认**——Opus 4-6 上默认 medium。启用 ultrathink 的用户默认 medium。

---

### betas.ts——Beta HTTP 头

**`getAllModelBetas(model)`**——memoized，构建完整 beta 头列表：
- claude-code 头
- OAuth 头
- 1M 上下文
- 交错思考
- 脱敏思考
- 连接器文本摘要
- 上下文管理
- 结构化输出
- Token 高效工具
- Web 搜索
- 提示缓存范围

`getMergedBetas()`——合并 SDK 提供的 betas 与自动检测的模型 betas。

`filterAllowedSdkBetas()`——根据白名单过滤 SDK betas，对不允许的头警告。

**Provider 感知**——第一方/Bedrock/Vertex/Foundry 有不同头。Bedrock extra-params 头仅 Bedrock 提供者可用。

---

### fastMode.ts——Fast Mode 管理

**Fast Mode** = Opus 4.6 加速速度。

**不可用原因**：免费账户、组织偏好、extra_usage_disabled、网络错误、SDK 限制、提供者限制（仅 1P）。

**运行时状态**——`{ status: 'active' } | { status: 'cooldown', resetAt, reason }`

**Signal 事件系统**——`onCooldownTriggered`, `onCooldownExpired`, `onOrgFastModeChanged` 用于状态变更通知。

**`prefetchFastModeStatus()`**——异步组织级 fast mode 状态检查：
- 30s 节流
- 飞行中请求去重
- OAuth token 401 刷新
- 失败时缓存回退

**`handleFastModeRejectedByAPI()`**——API 拒绝时永久禁用。

---

### autoModeDenials.ts——Auto Mode 拒绝追踪

有界 FIFO（最多 20 条）， newest first。`recordAutoModeDenial()` 前插到数组。用于 UI Recent Denials tab 显示。Feature-gated（`TRANSCRIPT_CLASSIFIER`）。

---

---

### 元

问题：**这一站真正想解决的架构问题是什么？**

回答：这一站真正聚焦的是模型调用前的能力配置层，也就是系统提示、思考模式、努力程度、beta 头和 fast mode 如何共同决定一次调用的形态。优先级链和 provider-aware betas 说明这不是参数表，而是一套模型能力装配器。

### 反

问题：**如果把这一站的设计反过来，会发生什么？**

回答：如果这些规则散在模型选择、设置读取和请求构造各处，最先失控的就是“这次调用到底启用了什么”。用户看到的会是偶发的行为差异，开发者看到的则是极难复盘的配置组合问题。

### 空

问题：**跳出当前文件名，这一站背后更大的问题是什么？**

回答：更大的问题是，一个 AI 客户端怎样把不断增长的模型能力封装成稳定、可解释、可回放的运行配置。这里讨论的是能力装配治理。

## 第 190 站：Session State + Query Processing + Recovery

### sessionState.ts——会话状态机

**`SessionState`**——`'idle' | 'running' | 'requires_action'`

**`RequiresActionDetails`**——`{ tool_name, action_description, tool_use_id, request_id, input? }`

**`SessionExternalMetadata`**——`{ permission_mode?, is_ultraplan_mode?, model?, pending_action?, post_turn_summary?, task_summary? }`

**单监听器架构**——每种类型一个监听器（可替换）。

**`notifySessionStateChanged()`**——更新全局状态，通知监听器，桥接 pending_action 到元数据，空闲时清除 task_summary。

**`notifyPermissionModeChanged()`**——所有权限模式转换的单一瓶颈（Shift+Tab、斜杠命令、bridge 命令）。

可选 SDK 事件发射（由 `CLAUDE_CODE_EMIT_SESSION_STATE_EVENTS` 门控）。

---

### queueProcessor.ts——队列处理器

**`processQueueIfReady({ executeInput })`**——主处理函数：
- 斜杠命令（以 '/' 开头）和 bash-mode 命令逐个处理——每命令错误隔离、退出码、进度 UI
- 其他命令按相同模式批量处理

**优先级处理**——斜杠/单独命令优先，然后批量。主线程过滤（仅 `agentId === undefined` 的命令）。模式感知批处理（永不混合 prompt vs task-notification）。

---

### conversationRecovery.ts——会话恢复

**`deserializeMessagesWithInterruptDetection()`**——从日志文件反序列化消息：
- 迁移旧版附件类型
- 剥离无效 permissionMode 值
- 过滤未解析的工具使用、孤儿思考消息、仅空白助手消息
- 检测轮次中断，为中断中添加合成的 "从上次断开处继续"

**`detectTurnInterruption()`**——三向分类：`none`、`interrupted_prompt`（用户输入但 CC 未开始）、`interrupted_turn`（模型正在工具执行中）。

**`loadConversationForResume()`**——支持三种恢复来源：未定义（最近）、会话 ID 字符串、或预加载的 LogOption。复制 plan 文件、复制文件历史、恢复技能状态、处理会话启动 hook。

**`loadMessagesFromJsonlPath()`**——链式遍历 transcript JSONL 文件，找到最新的非 sidechain 叶子。

---

### queryHelpers.ts——查询辅助

**`normalizeMessage()`**——生成器，将内部 `Message` 类型转换为 `SDKMessage` 格式，带 bash/powerShell 进度节流（CCR 每 30s 一次）。

**`handleOrphanedPermission()`**——生成器，重新执行先前已授权的工具调用。

**`extractReadFilesFromMessages()`**——两遍分析：第一遍找工具使用，第二遍找对应结果。对编辑从磁盘读取（因为编辑输入是 old_string/new_string，非完整内容）。

**`extractBashToolsFromMessages()`**——从 BashTool 历史提取顶级 CLI 工具名（如 `git`、`aws`、`vercel`）。

---

---

### 元

问题：**这一站真正想解决的架构问题是什么？**

回答：这一站真正要解决的是，会话在运行、等待动作、恢复中断三种状态之间如何保持可追踪的状态机。`SessionState`、queueProcessor 与 conversationRecovery 共同定义了“会话如何继续是同一个会话”。

### 反

问题：**如果把这一站的设计反过来，会发生什么？**

回答：如果这些东西没有被统一建模，权限等待、批处理输入和中断恢复都会各自保有局部状态。那时即使功能都在，系统也会越来越难回答“当前会话到底处于什么状态”。

### 空

问题：**跳出当前文件名，这一站背后更大的问题是什么？**

回答：更大的问题是，会话型 AI 如何把实时执行与事后恢复接成同一条状态历史。这里讨论的是会话连续性，而不是单个恢复函数。

## 第 191 站：Cursor 光标引擎 + Background Housekeeping

### Cursor.ts——文本光标引擎

**全面的终端 REPL 文本光标和操作引擎**——Unicode 感知光标导航、文本编辑、选择、kill ring、Vim 风格词移动。

**`Cursor` 类**：
- **导航**——left/right/up/down/startOfLine/endOfLine/startOfLogicalLine/endOfLogicalLine/firstNonBlankInLine/goToLine/startOfFile/endOfFile
- **词移动**——nextWord/prevWord/endOfWord/nextVimWord/prevVimWord/endOfVimWord/nextWORD/prevWORD/endOfWORD
- **Vim 字符查找**——f/F/t/T 语义
- **文本编辑**——insert/del/backspace/deleteToLineStart/deleteToLineEnd/deleteWordBefore/deleteWordAfter/deleteTokenBefore
- **图片引用**——imageRefEndingAt/imageRefStartingAt/snapOutOfImageRef 原子跳过 `[Image #N]` chip
- **渲染**——render() 带光标字符、掩码（用于密钥）、ghost text、视口滚动

**`MeasuredText` 类**——包装文本带列宽，惰性的换行、图段边界计算、词边界缓存（使用 `Intl.Segmenter`）。

**Kill ring 全局状态**——`pushToKillRing()`, `getLastKill()`, `getKillRingItem()`, `clearKillRing()`, `recordYank()`, `canYankPop()`, `yankPop()`。

**不可变模式**——`Cursor` 方法返回新 `Cursor` 实例（不可变光标模式）。

**Unicode 正确性**——输入 NFC 标准化，全文图段分割器操作。

---

### backgroundHousekeeping.ts——后台维护

**`startBackgroundHousekeeping()`** 编排器：
- **初始化服务**：`initMagicDocs()`, `initSkillImprovement()`, `initExtractMemories()`（feature-gated）, `initAutoDream()`, `autoUpdateMarketplacesAndPluginsInBackground()`
- **10 分钟延迟**的极慢操作（文件清理、版本清理），带**用户交互感知**（如果用户最后一分钟内有交互，推迟 24 小时）
- 为 `ant` 用户设置**每 24 小时重复**清理（npm cache、旧版本），两者都通过标记文件限流和锁

**Fire-and-forget** init（void promises）。用户交互感知：如果用户活跃则推迟重工作。

---

---

### 元

问题：**这一站真正想解决的架构问题是什么？**

回答：这一站把文本光标引擎与后台维护放在一起，看似跨度大，实则都在服务“长期可用的 REPL 体验”。前者让输入编辑具备 Unicode 正确性与 Vim 级操作语义，后者则在用户不打扰时维持文档、记忆、插件和清理任务运转。

### 反

问题：**如果把这一站的设计反过来，会发生什么？**

回答：如果光标逻辑只是字符串索引补丁，复杂输入很快就会在图段、宽字符和图片引用上出错；而后台维护若无统一编排，则会在用户活跃时打断体验。两边都会把“看似不重要的细节”累积成明显摩擦。

### 空

问题：**跳出当前文件名，这一站背后更大的问题是什么？**

回答：更大的问题是，交互式 AI 工具如何同时照顾前台手感与后台自维护。这里讨论的是体验连续性，而不是输入或清理的单点实现。

## 第 192 站：Concurrent Sessions + Portable Session Storage + Installer

### concurrentSessions.ts——并发会话管理

**基于 PID 的会话注册表**——启用 `claude ps` 列出所有活跃会话和元数据。

```typescript
SessionKind: 'interactive' | 'bg' | 'daemon' | 'daemon-worker'
SessionStatus: 'busy' | 'idle' | 'waiting'
```

**`registerSession()`**——写入 PID 文件带会话元数据（pid/sessionId/cwd/startedAt/kind/entrypoint/name/logPath/agent），注册清理。监听会话切换更新文件。

**`countConcurrentSessions()`**——扫描会话目录，过滤陈旧 PID（崩溃的会话），删除文件，返回活跃会话数。

**安全设计**——文件名守卫（`/^\d+\.json$/`）防止意外删除非 PID 文件。WSL 上跳过扫描避免删除活跃 Windows PID。

---

### sessionStoragePortable.ts——便携会话存储

**Node.js 无内部依赖**——CLI 和 VS Code 扩展之间的共享会话存储实用程序。

**UUID 验证**——正则 `validateUuid()`。

**JSON 字段提取**——`unescapeJsonString()`, `extractJsonStringField()`, `extractLastJsonStringField()`——从原始 JSON 文本提取字符串字段无需完整解析（适用于截断数据）。

**Prompt 提取**——`extractFirstPromptFromHead()`——从 JSONL 头块找到第一个有意义的用户 prompt，跳过工具结果、元消息、命令消息和自动生成的模式。

**文件 I/O**——`readHeadAndTail()`, `readSessionLite()`——高效读取文件的首尾 64KB，共享缓冲区。

**分块 transcript 加载**——`readTranscriptForLoad()`——前向分块读取（1MB 块），剥离 attribution-snapshot，在 compact 边界截断，对最后一个 snapshot 重排到 EOF。对跨块接缝的行使用状态机 + 携带缓冲区。

**手动缓冲区管理**——`Sink` 结构体带动态增长，携带缓冲区用于块边界。紧凑标记（`"compact_boundary"`）的字节级边界检测，有界搜索窗口。

---

### localInstaller.ts——本地安装器

**管理 Claude CLI 本地安装**——设置独立包环境，处理安装/更新。

路径到 `~/.claude/local/`。原子如果缺失写（`'wx'` 标志）避免覆盖。惰性路径获取器（memoized）在 `main()` 设置 env 变量后求值。

---

---

### 元

问题：**这一站真正想解决的架构问题是什么？**

回答：这一站真正关注的是 Claude Code 作为多会话本地产品时，怎样管理活跃实例、共享 transcript 存储并自我安装更新。PID 注册、便携 transcript 读取和本地安装器，共同组成了“本机上的 Claude 基础设施层”。

### 反

问题：**如果把这一站的设计反过来，会发生什么？**

回答：如果没有这一层，`claude ps` 无法可靠列活跃会话，CLI 与 VS Code 也难共享稳定会话格式，本地安装目录还会不断踩踏已有状态。那样产品虽然能用，却永远像一组松散工具而不是统一应用。

### 空

问题：**跳出当前文件名，这一站背后更大的问题是什么？**

回答：更大的问题是，一个本地 AI 客户端怎样在多进程、多入口和长期安装状态下保持自身身份。这里讨论的是产品落地到用户机器后的系统学问题。

## 第 193 站：MCP WebSocket + MCP Validation + Embedded Tools + tempfile

### mcpWebSocketTransport.ts——MCP WebSocket 适配器

**实现 MCP `Transport` 接口的 WebSocket 传输适配器**——抽象 Bun 原生 WebSocket 和 Node.js `ws` 包。

**Adapter 模式**——桥接两个 WebSocket API（Bun 原生 vs `ws` 包）。统一错误处理。正确监听器清理防止内存泄漏。基于 promise 的连接初始化。

---

### mcpValidation.ts——MCP 验证和截断

**分辨率和验证 MCP 输出 token 上限**，处理超大 MCP 工具结果的截断。

**`getMaxMcpOutputTokens()`**——优先级：env var `MAX_MCP_OUTPUT_TOKENS` > GrowthBook `tengu_satin_quoll.mcp_tool` > 默认（25000）。

**两层截断**——快速启发式检查（基于大小）先避免不必要的 API token 计数调用，然后接近阈值时使用精确 API 计数（0.5x 系数）。

对过大图片压缩而非盲目截断。字符到 token 转换（maxChars = maxTokens * 4）。

---

### embeddedTools.ts——嵌入式搜索工具

**检测当前构建是否有 bfs/ugrep 搜索工具直接嵌入 bun 二进制**。

检查 `EMBEDDED_SEARCH_TOOLS` env var，排除 SDK/local-agent 入口。

当为真时：`find`/`grep` shell 命令被遮蔽，Glob/Grep 工具从工具注册表移除，关于 find/grep 的提示指导被省略。

---

### tempfile.ts——临时文件路径

**生成临时文件路径**，支持随机 UUID 或内容派生的稳定标识符。

**内容寻址标识符**——相同内容 → 相同路径 → prompt 缓存前缀命中。随机 UUID 回退用于不需要内容稳定性时的唯一性。

用于 temp 文件，其路径出现在发给 Anthropic API 的内容中（如沙盒拒绝列表在工具描述中），UUID 每次子进程产生都变化会使 prompt 缓存前缀失效。

---

---

### 元

问题：**这一站真正想解决的架构问题是什么？**

回答：这一站处理的是 MCP 接入的几个关键边角：跨运行时 WebSocket 适配、超大输出验证截断、嵌入式搜索工具存在性，以及临时文件路径的内容稳定性。它们共同服务的是“让工具生态接进来时不破坏主系统”。

### 反

问题：**如果把这一站的设计反过来，会发生什么？**

回答：如果这些能力四散，Bun/Node 的差异会直接漏到业务层，MCP 输出要么被盲截断要么撑爆上下文，临时路径还会持续破坏 prompt cache。最后问题会表现在工具不稳定，但根因其实是接入层没被建好。

### 空

问题：**跳出当前文件名，这一站背后更大的问题是什么？**

回答：更大的问题是，一个可扩展 AI 客户端怎样吸纳多种工具与运行时，而不把外部复杂性原样倒进主循环。这里讨论的是工具接入边界。

## 第 194 站：Heap Dump + gh PR Status + Misc Utils

### heapDumpService.ts——堆转储

**捕获 V8 堆快照和内存诊断**——手动通过 `/heapdump` 或自动在 1.5GB 堆阈值触发。

**`MemoryDiagnostics`**——堆使用率、RSS、增长率、V8 堆统计（空间、分离上下文）、活跃句柄/请求、文件描述符、smaps_rollup（Linux）、泄漏分析。

**防御性**——先写诊断（在堆快照之前，因为堆快照可能崩溃），然后写入堆快照到 ~/Desktop。

**运行时分支**——Bun 使用 `Bun.generateHeapSnapshot()`（同步）+ `Bun.gc()`；Node.js 使用流式 `getHeapSnapshot()` + pipeline。

**启发式检测**——分离上下文、高句柄计数、native > heap、高增长率、高 FD 计数。

---

### ghPrStatus.ts——GitHub PR 状态

**使用 `gh` CLI 获取当前 git 分支的 GitHub PR 审核状态**。

```typescript
PrReviewState: 'approved' | 'pending' | 'changes_requested' | 'draft' | 'merged' | 'closed'
```

最佳努力 null-on-error（容忍缺失的工具/仓库）。默认分支过滤避免显示误导性的最近合并 PR 数据。超时保护子进程执行（5s）。

---

### CircularBuffer.ts——环形缓冲

**固定大小环形缓冲**——满时自动驱逐最旧项。O(1) 插入带自逐。使用模运算的环形缓冲 + 预分配数组。泛型类型。

---

### controlMessageCompat.ts——控制消息兼容

**规范化 camelCase `requestId` 为 snake_case `request_id`**——维护与旧 iOS app 构建的向后兼容。

原地突变性能（避免分配）。如果两个键都存在，snake_case 获胜。被 `replBridge.ts` 中的 `isSDKControlRequest` 消费。

---

### 元

问题：**这一站真正想解决的架构问题是什么？**

回答：这一站把内存诊断、PR 审核状态和兼容性小工具并列，核心并不是杂项收纳，而是运行期可诊断性。无论是 1.5GB 阈值 heap dump，还是 `gh` 最佳努力查询，都是在为“出了问题时还能看见什么”建立后备能力。

### 反

问题：**如果把这一站的设计反过来，会发生什么？**

回答：如果这类能力只在事故时临时拼凑，真正需要它们时往往已经来不及。特别是堆快照前先写诊断、PR 状态默认分支过滤这类细节，一旦不提前制度化，就只能靠事后猜测。

### 空

问题：**跳出当前文件名，这一站背后更大的问题是什么？**

回答：更大的问题是，复杂系统如何给自己准备足够的自我诊断与外部状态感知手段。这里讨论的是可观测性的尾部能力。

### 读完后应抓住的 3 个事实

1. **System Prompt 优先级链**——Override > Coordinator > Agent > Custom > Default。协调器模式完全替换默认提示；代理模式在主动模式下追加（不替换）。始终追加 `appendSystemPrompt`。这允许每个代理/coordinator 模式有独立系统提示而不需要重建整个提示。

2. **Conversation Recovery 的合成消息注入**——恢复被中断的会话时，`conversationRecovery.ts` 不仅修复损坏的消息——它注入合成消息使 transcript 对 API 有效。中断的轮次添加合成的 "continue from where you left off" 提示；挂起的工具使用添加合成的助手哨兵。这是生产级恢复——用户可以从崩溃中恢复而无需重新开始。

3. **Portable Session Storage 的分块 transcript 加载**——`readTranscriptForLoad()` 以 1MB 块前向读取 JSONL 文件，在 `compact_boundary` 边界截断。使用状态机 + 跨越块接缝的行携带缓冲区。这避免了将整个 transcript 加载到内存——即使对于大会话。
