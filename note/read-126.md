## 第 135 站：`src/services/tools/` 工具执行管线（3 个文件，~2.3K）

### 这是什么

工具执行管线是模型返回 `tool_use` blocks 后的完整处理链路——从分区分组、权限校验、hook 执行、实际调用，到结果回写。

---

### toolOrchestration.ts —— 多轮调度器

**职责**：接收模型的 `ToolUseBlock[]`，按并发性分组，分发到并发或串行执行。

```ts
// 分区逻辑
function partitionToolCalls(blocks: ToolUseBlock[]): Batch[] {
  // 连续 isConcurrencySafe 的工具分到同一 batch
  // 非安全工具各自独立一个 batch
}
```

**关键模式——延迟上下文修改器**：并发执行期间，工具返回 `contextModifier` 函数，batch 完成后再依次应用。这避免对共享上下文的竞态。

**并发执行**：`all()` 工具带并发上限（`CLAUDE_CODE_MAX_TOOL_USE_CONCURRENCY`，默认 10）。

---

### toolExecution.ts —— 单工具执行管线

**核心函数 `checkPermissionsAndCallTool()` 分 8 个阶段**：

| 阶段 | 职责 |
|---|---|
| Phase 1 | Zod schema 验证 + `tool.validateInput()` |
| Phase 2 | Bash spec classifier 异步启动（与 hook 并行） |
| Phase 3 | 安全加固：剥离 `_simulatedSedEdit`，回填 observable input，执行 PreToolUse hooks |
| Phase 4 | `resolveHookPermissionDecision()` → allow/deny/ask |
| Phase 5 | `tool.call()` 实际执行 |
| Phase 6 | 结果处理：结构化输出、user feedback、context modifier |
| Phase 7 | PostToolUse hooks |
| Phase 8 | 错误分类（`classifyToolError()`） |

**流式桥接**：`streamedCheckPermissionsAndCallTool()` 将回调风格的进度 API 桥接为 `Stream<MessageUpdateLazy>`。

---

### StreamingToolExecutor.ts —— 流式感知执行器

**与普通管线的区别**：普通管线等模型完整返回后批量执行；StreamingToolExecutor 允许工具调用增量到达、队列化执行。

**Abort 级联架构**：
```
parent (toolUseContext.abortController)
  → siblingAbortController（Bash 报错时触发）
    → toolAbortController（每个工具的子 controller）
```

Bash 工具错误会取消兄弟工具。Read/WebFetch 等视为独立——一个失败不会影响其他。

**合成错误消息**：
- `sibling_error`：「Cancelled: parallel tool call {desc} errored」
- `user_interrupted`：带 memory correction hint 的 REJECT_MESSAGE
- `streaming_fallback`：「Streaming fallback - tool execution discarded」

---

### 元

问题：**这一站真正想解决的架构问题是什么？**

回答：这一站讨论的是模型已经决定调用工具之后，系统如何把这次调用编排成稳定流水线。分 batch、延迟应用 `contextModifier`、权限校验、hook 与错误分类，说明真正关键的是工具调用的执行秩序。

### 反

问题：**如果把这一站的设计反过来，会发生什么？**

回答：如果每个调用点自己决定并发、权限和结果回写，最先崩掉的就是一致性，尤其在多工具并发和流式执行里。工具本身可能没错，但调用框架会让整体行为越来越不可预测。

### 空

问题：**跳出当前文件名，这一站背后更大的问题是什么？**

回答：更大的问题是，模型意图怎样被翻译成受控执行。工具执行管线给出的答案不是“让工具自己管自己”，而是先建立一条统一的运行时主干。

### 读完后应抓住的 3 个事实

1. **两条执行路径**：普通模式（`runTools()` → 分区 → 批量执行）和流式模式（`StreamingToolExecutor` → 队列式增量执行）。两者都调用 `runToolUse()` from toolExecution.ts。

2. **Abort 级联仅限 Bash**——只有 Bash 错误会取消兄弟工具。读取类/WebFetch 类工具错误不影响其他并行工具。

3. **延迟上下文修改器**——并发工具的结果中的 `contextModifier` 不在执行时立即应用，而是等整个 batch 完成后按顺序应用，避免竞态。

---

## 第 136 站：权限运行时系统（3 个文件，~3K）

### 这是什么

权限系统决定模型能否使用某个工具——基于规则、模式、AI classifier、和工具自身的安全检查。

---

### permissionSetup.ts —— 权限模式转换

**危险权限检测**：
```ts
isDangerousBashPermission(ruleContent)
// 匹配：python:*, node:*, python -*, python* 等解释器模式
```

`DANGEROUS_BASH_PATTERNS` 覆盖 python、node、ruby 等解释器。PowerShell 更广泛——`iex`、`invoke-expression`、`.NET` 等。

**Auto mode 过渡**：进入 auto mode 时 `stripDangerousPermissionsForAutoMode()` 剥离危险规则，离开时 `restoreDangerousPermissions()` 恢复。

---

### permissions.ts —— 权限评估引擎

**决策管线 `hasPermissionsToUseToolInner` 按编号顺序**：

```
Step 1a: 整工具被 deny rule → deny
Step 1b: 整工具有 ask rule → ask（sandbox 例外）
Step 1c: tool.checkPermissions() → 委托工具实现
Step 1d: 工具实现显式 deny → deny
Step 1e: 工具需要用户交互 → ask
Step 1f: 内容级 ask rule（如 Bash(npm publish:*)）
Step 1g: 安全检查（.git/、.claude/、shell 配置）→ ask（BYPASS-IMMUNE）

Step 2a: bypassPermissions 模式 → allow
Step 2b: 始终允许列表 → allow
Step 3:  passthrough → ask（默认）
```

**Auto mode classifier 流程**：
1. 先用 `acceptEdits` 模式快速检查安全文件操作
2. 检查安全工具列表
3. 调用 `classifyYoloAction()` 发送 AI classifier
4. `shouldBlock` → 根据 `tengu_iron_gate_closed` 决定 fail closed vs fail open

**BYPASS-IMMUNE 安全检查**：`.git/`、`.claude/`、`.vscode/`、shell 配置文件路径——即使在 `bypassPermissions` 模式下也会弹出确认。

---

### useCanUseTool.tsx —— React Hook 层

将 `hasPermissionsToUseTool` 封装为 React hook——返回 `canUseTool` callback + 权限上下文。UI 根据 `behavior: 'ask'` 显示 PermissionPrompt。

---

### 元

问题：**这一站真正想解决的架构问题是什么？**

回答：这一站把允许、拒绝、询问的判断正式收拢为总闸门，而不是散落在 UI 或工具实现里。危险规则剥离、auto classifier、bypass-immune 路径，说明权限在这里不是附属校验，而是运行时制度。

### 反

问题：**如果把这一站的设计反过来，会发生什么？**

回答：如果权限判断分散在各层，系统会很快出现“双重标准”：有的地方先 ask，有的地方先 deny，有的地方甚至直接放行。最坏的不是严格，而是标准不一致。

### 空

问题：**跳出当前文件名，这一站背后更大的问题是什么？**

回答：更大的问题是，一个会主动采取行动的 AI，边界应当如何制度化。136 站的答案是把权限从边上挪到正中央，让执行永远晚于裁决。

### 读完后应抓住的 3 个事实

1. **7 步编号决策管线**——权限评估严格按 1a→1g→2a→2b→3 顺序。deny 先于 ask 检查；安全检查（1g）在模式决策（2a）之前，保证 bypass-immune。

2. **Auto mode 的剥离/恢复机制**——进入 auto mode 时危险规则被剥离并暂存，退出时恢复。Circuit breaker 触发时，退出 plan 回到 `default` 而非 `auto`。

3. **BYPASS-IMMUNE 安全检查**——保护路径（`.git/`、`.claude/`）和安全文件即使在 bypass 模式下也会要求确认——防止 bypass 绕过核心安全。

---

## 第 137 站：Hook 系统（7 个文件，~7K）

### 这是什么

Hook 系统是生命周期驱动的引擎——在 PreToolUse、PostToolUse、PermissionDenied 等 30+ 事件点上执行用户配置的命令/prompt/http/agent/callback。

---

### hooks.ts —— 执行引擎

**30+ Hook 事件**，覆盖 PreToolUse、PostToolUse、PostToolUseFailure、PermissionDenied、Stop、StopFailure、SessionStart、Setup、Notification、TaskCreated、TaskCompleted、TeammateIdle、UserPromptSubmit、CwdChanged、FileChanged、ConfigChange、InstructionsLoaded、Elicitation、ElicitationResult、SubagentStart、PermissionRequest 等。

**匹配流程**：
```
getMatchingHooks(event, matchQuery):
  1. 按 matchQuery 模式匹配（glob 风格）
  2. 按类型去重（command/prompt/agent/http/callback/function）
     用 pluginRoot/skillRoot 防止跨源冲突
  3. 应用 if 条件过滤（权限规则语法）
```

**快速路径**：当所有匹配的 hook 都是内部 callback 时，零生成器/零进度/零 span 开销，顺序执行。比用户 hook 减少 ~70% 延迟。

**退出码语义**：
- 0 = 成功
- 2 = blocking error（停止执行）
- 其他 = non-blocking error

**异步 hook**：stdout 返回 `{async: true}`。两种模式：
- `asyncRewake`：绕过注册表，退出码 2 时作为 task-notification 唤醒模型
- 标准 async：通过 `AsyncHookRegistry` 跟踪

---

### types/hooks.ts & schemas/hooks.ts —— 类型与配置

**四种外部可配置 hook 类型**：

| 类型 | 执行方式 | 关键字段 |
|---|---|---|
| `command` | shell 命令 | `command`, `shell`, `async`, `asyncRewake` |
| `prompt` | LLM prompt 评估 | `prompt`（含 `$ARGUMENTS`）, `model` |
| `http` | HTTP POST | `url`, `headers`（env var 插值） |
| `agent` | agent 验证器 | `prompt`, `model` |

**Prompt elicitation protocol**：hook 可以在执行中向用户提问（带标签的选项），通过 `promptRequest`/`promptResponse` 类型实现。

---

### hooksConfigManager.ts —— 元数据与分组

提供 hook 事件的元数据描述（summary、description、matcher metadata），按 event+matcher 分组，按优先级排序。

---

### stopHooks.ts —— Stop 阶段处理

**三阶段**：
1. 后台记账（`saveCacheSafeParams`、`executeAutoDream`、`cleanupComputerUseAfterTurn`）
2. 执行 Stop hooks，输出附件消息
3. Teammate hooks（`TaskCompleted`、`TeammateIdle`）

---

### sessionStart.ts —— 会话启动钩子

**关键设计**：`pendingInitialUserMessage` 模块变量作为 side-channel——SessionStart hook 可以注入初始用户消息，而不改变 `processSessionStartHooks` 的返回类型（避免波及 main.tsx/print.ts）。

---

### loadSkillsDir.ts —— 技能目录系统

**Skills 来源（优先级顺序）**：
1. Managed skills（policy settings）
2. User skills（`~/.claude/skills`）
3. Project skills（`.claude/skills`）
4. Additional dir skills（`--add-dir`）
5. Legacy commands（`.claude/commands`，DEPRECATED）

**Skills 与 Hooks 的关系**：Skill 的 frontmatter 可以包含 `hooks` 字段（通过 `HooksSchema` 验证）。加载后 hooks 注册到系统，与普通 hook 同等待遇，仅 `hookSource` 不同便于审计。

**条件激活**：带 `paths` frontmatter 的 skill 不立即激活，当文件操作匹配其 gitignore 风格模式时通过 `activateConditionalSkillsForPaths()` 激活。

---

### 元

问题：**这一站真正想解决的架构问题是什么？**

回答：这一站建立的是 Claude Code 的生命周期事件面，让外部命令、prompt、agent、http 与 callback 能在 30 多个事件点挂接。关键不只是“能加 hook”，而是 hook 被怎样匹配、去重、异步追踪和分类执行。

### 反

问题：**如果把这一站的设计反过来，会发生什么？**

回答：如果 hook 只是零散插桩，事件顺序、退出码语义和异步唤醒都会变得不可控。那时扩展看似灵活，实际却在不断污染主流程。

### 空

问题：**跳出当前文件名，这一站背后更大的问题是什么？**

回答：更大的问题是，系统如何在不改核心代码的前提下暴露可编排生命周期。Hook 系统的意义，就是把“外挂逻辑”升级为一等公民，同时不给主循环失序的机会。

### 读完后应抓住的 3 个事实

1. **30+ 事件点的生命周期引擎**——从 PreToolUse 到 StopFailure，从 SessionStart 到 FileChanged。Hook 通过 matchQuery 匹配，按类型去重，通过 if 条件二次过滤。

2. **Skills 是带 hooks 的 prompt 包**——Skill 本身是 prompt 命令，但可以通过 frontmatter 声明 hook 配置（如 "run-tests" skill 注册 PostToolUse(Bash) hook 验证测试输出）。

3. **快速路径优化**——纯 callback hook 走零生成器快速路径，减少 70% 延迟。这是性能敏感区域（PostToolUse 每轮都触发）的显式优化。

---

## 第 138 站：Agent Team Swarm 系统（5 个文件，~2.5K）

### 这是什么

Agent Swarm 是多代理协作系统——team lead 管理多个 teammate，通过 mailbox 消息路由、共享 task list、和三种后端（tmux/iTerm2/in-process）执行。

---

### teamHelpers.ts —— Team 文件持久化

**TeamFile 结构**：
```ts
type TeamFile = {
  name, description, createdAt,
  leadAgentId, leadSessionId,
  members: [{
    agentId, name, model, color,
    planModeRequired, tmuxPaneId, cwd,
    worktreePath, sessionId,
    backendType, isActive, mode, subscriptions
  }]
}
```

存储路径：`~/.claude/teams/{sanitized-name}/config.json`。

**会话清理**：`registerTeamForSessionCleanup()` 跟踪本轮创建的团队，SIGINT/SIGTERM 时 `cleanupSessionTeams()` 执行两阶段清理：先杀 pane，再删目录。动态导入 `registry.js`/`detection.js` 避免循环依赖。

---

### backends/registry.ts —— 后端检测与抽象

**检测优先级**：
```
1. 在 tmux 内 → tmux（原生 pane）
2. 在 iTerm2 内 → iTerm2（如果有 it2 CLI）否则 tmux
3. 其他终端 → tmux 外部 session
4. 都没有 → in-process 回退
```

**`inProcessFallbackActive` 标志是 sticky**——一旦标记，整个会话不可变。防止环境中途变化。

**teammateMode setting**：
- `'in-process'` → 总是进程内
- `'tmux'` → 总是 pane
- `'auto'` → 不在 tmux/iTerm2 内时用 in-process

---

### useInboxPoller.ts —— Mailbox 消息路由器

**谁轮询**：
- in-process teammate → 不用此 hook（用 `waitForNextPromptOrShutdown()`）
- out-of-process teammate → 读自己的 mailbox
- team lead → 读 leader mailbox

**消息分类路由**：

| 消息类型 | 接收方 | 路由目标 |
|---|---|---|
| Permission request | Team lead | `ToolUseConfirmQueue`（标准 UI） |
| Permission response | Teammate | `processMailboxPermissionResponse()` |
| Shutdown request | Teammate | 通过为普通消息 |
| Shutdown approval | Team lead | 杀 pane，从 teamContext 移除 |
| Plan approval request | Team lead | 自动批准，回写响应 |
| Plan approval response | Teammate | 退出 plan mode |
| Regular message | 忙碌时队列/空闲时直接提交 |

**Shutdown 处理链**：
```ts
killPane(paneId) → removeFromTeamFile() → unassignTeammateTasks() → completed
```

---

### InProcessTeammateTask.ts —— 进程内任务

使用 `AsyncLocalStorage` 实现同进程内的执行上下文隔离。每个 teammate 有自己的消息循环（`waitForNextPromptOrShutdown()`）。

**Side-channel**：`pendingUserMessages` 数组——用户在 leader UI 直接给 teammate 发消息时暂存，teammate 下次轮询时取走。

---

### Task.ts —— 共享类型系统

```ts
type TaskType = 'local_bash' | 'local_agent' | 'remote_agent' | 'in_process_teammate' | 'local_workflow' | 'monitor_mcp' | 'dream'
```

Task ID 格式：1 字节前缀 + 8 随机字符（`36^8 ≈ 2.8 trillion`，抵抗暴力破解）。
- `b` = bash, `a` = agent, `r` = remote, `t` = teammate, `w` = workflow, `m` = mcp, `d` = dream

---

### 元

问题：**这一站真正想解决的架构问题是什么？**

回答：这一站描述的是多代理协作真正落地后的组织基础设施，包括 team file、mailbox、后端检测和 inbox 路由。它让 team lead 与 teammate 的分工、权限请求和计划批准都有固定流向。

### 反

问题：**如果把这一站的设计反过来，会发生什么？**

回答：如果 swarm 只是多个 agent 同时跑而没有中心路由，权限、消息、关闭和任务归属都会迅速变成一团线头。代理越多，协作质量反而越差。

### 空

问题：**跳出当前文件名，这一站背后更大的问题是什么？**

回答：更大的问题是，多智能体不是“多开几个线程”，而是能否形成一种组织形态。138 站说明，真正的 swarm 首先是一套通信和身份制度。

### 读完后应抓住的 4 个事实

1. **Mailbox 是中心路由**——所有代理间通信通过 mailbox。Permission request 从 teammate 转发到 leader 的标准 UI；shutdown/plan approval 通过消息类型路由。

2. **三种后端 + sticky 回退**——tmux > iTerm2 > 外部 tmux > in-process。一旦回退到 in-process，整轮会话不可变。

3. **Task ID 安全前缀**——不同类型有不同任务 ID 前缀，既便于调试（肉眼识别任务类型），又以 `36^8` 的随机空间抵抗暴力文件系统攻击。

4. **Teammate 的 plan 模式自动批准**——Leader 的 inbox poller 收到 plan approval request 后自动批准并回写响应。Leader 不审方案，只确认模式切换。

---

## 第 139 站：Memory 系统（5 个文件，~3K）

### 这是什么

三层 memory 架构：Session Memory（自动摘要）、Auto Memory（模型维护的文件记忆）、Team Memory（跨会话远程同步）。

---

### sessionMemory.ts —— 会话自动记忆

**挂载方式**：`registerPostSamplingHook(extractSessionMemory)`——每次 API 响应后作为 post-sampling hook 触发。

**提取决策**：
- 首次：`initialization threshold`（总上下文 token 数）
- 后续：token 阈值 AND（tool call 阈值 OR 上一轮无 tool call）

**安全约束**：分叉的子代理只允许 `FILE_EDIT_TOOL` 操作记忆文件路径：
```ts
if (tool.name === FILE_EDIT_TOOL && input.file_path === memoryPath)
  return { behavior: 'allow' }
return { behavior: 'deny' }
```

---

### attachments.ts —— 附件注入管线

**40+ 附件类型**，核心 memory 相关：
- `relevant_memories`：LLM 相关性排名选出的最多 5 个文件，4KB/文件，60KB/会话总预算
- `nested_memory`：文件操作触发时，从 CWD 到目标路径的目录层级 CLAUDE.md
- `current_session_memory`：MEMORY.md 内容

**去重机制**：`readFileState`（FileReadTool 已读过的）+ `alreadySurfaced`（扫描历史消息已出现过的）双重去重。

---

### claudemd.ts —— 文件记忆 Prompt 构建

**三路分发**：
```
Priority 1: KAIROS daily-log mode
Priority 2: Team + Auto memory combined
Priority 3: Auto memory only
```

**MEMORY.md** 是索引文件（≤200 行，≤25KB），不直接存储内容——只包含指向各主题 `.md` 文件的链接。

**KAIROS 模式的 `/dream` 技能**：模型向日志文件追加（`logs/YYYY/MM/YYYY-MM-DD.md`），夜间 `/dream` 技能蒸馏到 MEMORY.md。

---

### teamMemorySync/index.ts —— 团队记忆同步

**API**：
```
GET  /api/claude_code/team_memory?repo={owner/repo}
GET  ?view=hashes      （轻量，只要校验和）
PUT                     （delta upsert）
```

**Delta 上传**：只上传本地 hash 与 `serverChecksums` 不同的条目。

**乐观锁 + 冲突重试**：`If-Match` ETag。412 冲突时 → `?view=hashes` 刷新 → 重新计算 delta → 重试。

**Local-wins-on-push, server-wins-on-pull**：推送时本地编辑永不静默丢弃；拉取时服务器内容优先（保留 mtime 避免无变化重写）。

**Secret 扫描**：上传前 `scanForSecrets()`，含密钥的文件跳过。

---

### 元

问题：**这一站真正想解决的架构问题是什么？**

回答：这一站把 memory 分成 Session Memory、Auto Memory 与 Team Memory 三层，分别服务短期总结、文件级记忆和跨会话同步。它的重心不是“记住更多”，而是让不同时间尺度的记忆各有入口与边界。

### 反

问题：**如果把这一站的设计反过来，会发生什么？**

回答：如果所有记忆都塞进同一种机制，提取频率、注入预算、同步冲突和安全限制会互相打架。最终不是记不住，就是记得太乱、太贵、也不可信。

### 空

问题：**跳出当前文件名，这一站背后更大的问题是什么？**

回答：更大的问题是，AI 系统里的记忆究竟该像缓存、像文档还是像知识库。139 站给出的答案不是二选一，而是承认这三种形态必须并存。

### 读完后应抓住的 3 个事实

1. **三层 memory 共存**——Session Memory（自动摘要，post-sampling hook），Auto Memory（模型维护的文件 + MEMORY.md 索引），Team Memory（远程同步，git repo 作用域，local-wins-push / server-wins-pull）。

2. **双重去重**——Memory 注入去重 `readFileState`（模型已读文件）+ `alreadySurfaced`（历史消息已出现），避免 token 浪费。

3. **Delta + 乐观锁的 Team Memory**——不上传全量文件，只计算 hash diff。412 冲突时不需要重新读取文件内容，只要 `?view=hashes` 轻量校验和刷新。

---

## 第 140 站：Plugin 系统（6 个文件，~2K）

### 这是什么

Plugin 系统扩展 Claude Code 的 commands、tools、hooks、MCP servers、skills、和 LSP servers。

---

### 三层 Plugin 模型

| 层级 | 职责 |
|---|---|
| Layer 1 Intent | 哪些 plugin 应激活（settings、CLI、marketplace） |
| Layer 2 Materialization | 克隆到 `~/.claude/plugins/cache/{marketplace}/{plugin}/{version}/` |
| Layer 3 Active Components | 热交换到 AppState（commands/agents/hooks/MCP/LSP） |

**合并优先级**：policy > session > marketplace > builtin。Managed（enterprise policy）名称锁定 `--plugin-dir` 覆盖。

---

### pluginLoader.ts —— 核心发现与加载

```ts
assemblePluginLoadResult():
  Promise.all([marketplaceLoader(), loadSessionOnlyPlugins()])
  → mergePluginSources(policy > session > marketplace > builtin)
  → verifyAndDemote()（依赖检查）
```

**缓存策略**：`loadAllPlugins()`（可能网络获取）和 `loadAllPluginsCacheOnly()`（只读磁盘缓存）分别 memoize。全量加载会预热 cache-only 缓存。

---

### loadPluginCommands.ts —— 命令/技能提取

扫描 `.md` 文件和 `SKILL.md`，解析 frontmatter，转换为 `Command`。

**变量替换**：`${CLAUDE_PLUGIN_ROOT}`、`${CLAUDE_PLUGIN_DATA}`、`${CLAUDE_SKILL_DIR}`、`${CLAUDE_SESSION_ID}`。

**Skill 去重**：目录同时有 `SKILL.md` 和其他 `.md` 时，skill 文件胜出。多个 `SKILL.md` 时取首个。

---

### refresh.ts —— 组件热交换

```ts
refreshActivePlugins():
  1. Clear all caches
  2. Full plugin load (not cache-only)
  3. 并行加载 commands + agents
  4. 并行加载 MCP + LSP
  5. 交换到 AppState
  6. 重新初始化 LSP manager
  7. 重新加载 hooks
  8. 设置 mcp.pluginReconnectKey + 1（触发 MCP 重连）
```

调用来源：`/reload-plugins` 命令、headless 模式首次查询前、新 marketplace 安装后。

---

### PluginInstallationManager.ts —— 后台安装编排

```ts
performBackgroundPluginInstallations():
  diffMarketplaces(declared, materialized)
  → reconcileMarketplaces()
  → 新装：立即 refresh；仅更新：needsRefresh=true
```

新安装自动 refresh（避免 cache-only 路径报 plugin not found）；仅更新设 flag 让用户决定何时应用。

---

### builtinPlugins.ts —— 内置插件注册表

```ts
type BuiltinPluginDefinition = {
  name, description, version, skills, hooks, mcpServers,
  isAvailable(), defaultEnabled
}
```

Skill 命令获取 `source: 'bundled'` 和 `loadedFrom: 'bundled'`（非 `'builtin'`），使它们出现在 Skill 工具列表和分析中。

---

### 元

问题：**这一站真正想解决的架构问题是什么？**

回答：这一站真正讲的是插件从“声明想加载”到“物理到缓存”再到“热交换进 AppState”的三层模型。pluginReconnectKey、新装与更新的不同处理，说明插件系统的重点是运行时接入秩序，而不是安装命令本身。

### 反

问题：**如果把这一站的设计反过来，会发生什么？**

回答：如果插件加载没有分层，市场源、会话源、企业策略和内置能力会互相覆盖得一塌糊涂。尤其刷新时若不能精确触发 MCP/LSP 重连，扩展就会经常处在半更新状态。

### 空

问题：**跳出当前文件名，这一站背后更大的问题是什么？**

回答：更大的问题是，一个长寿命 CLI 如何持续扩展而不越来越脆。Plugin 系统给出的答案是让扩展成为热交换组件，而不是一次性静态拼装。

### 读完后应抓住的 2 个事实

1. **热交换通过 pluginReconnectKey**——`refreshActivePlugins()` 递增 `mcp.pluginReconnectKey`，触发所有 MCP 服务器重连。这是 plugin → MCP 扩展的关键连接点。

2. **新装 vs 更新的不同处理**——新 marketplace 立即 auto-refresh（防止 cache-only 找不到 plugin）；仅更新只设 flag 不自动刷新。这是用户体验的精心设计。

---

## 第 141 站：Computer Use & Chrome 集成（4 个文件，~1K）

### 这是什么

Claude in Chrome（网页控制）和 Computer Use（桌面控制）的 MCP 集成。两者都是通过 in-process MCP server 运行在 CLI 内部，而不启动 325MB 的子进程。

---

### MCP Client 中的 In-Process 传输

```ts
// 相同的模式适用于两者：
const { createServer } = await import('./mcpServer.js')
const { createLinkedTransportPair } = await import('./InProcessTransport.js')
const [clientTransport, serverTransport] = createLinkedTransportPair()
await server.connect(serverTransport)
transport = clientTransport
```

**为什么配置仍是 `type: 'stdio'`**：需要命中 client 传输选择逻辑的正确分支，但 client.ts 按名称拦截，实际不 spawn 子进程。

---

### Computer Use Wrapper（wrapper.tsx）

**核心适配器 `buildSessionContext()`** 返回三组回调：
1. **读取访问器**——从 `AppState.computerUseMcpState` 读取
2. **写回回调**——反馈到 `AppState`（allowed apps、display pin、screenshot dims）
3. **Lock/Gate**——`checkCuLock`/`acquireCuLock`（Esc 热键终止）

**结果归一化**：MCP content blocks → Anthropic API blocks：
```ts
// MCP: { type: 'image', data, mimeType }
// → Anthropic: { type: 'image', source: { type: 'base64', media_type, data } }
```

---

### Chrome 设置

**多门检测 `shouldEnableClaudeInChrome()`**：
1. SDK/CI 默认关闭（除非 `--chrome` flag）
2. `--chrome`/`--no-chrome` CLI
3. `CLAUDE_CODE_ENABLE_CFC` env
4. 全局 config `claudeInChromeDefaultEnabled`
5. 默认 `false`

**Native Host 安装**：
- 写 manifest 到 NativeMessagingHosts 目录
- Windows 额外写注册表
- 创建 wrapper 脚本（因为 Chrome native manifest `path` 字段不能有参数）

---

### 元

问题：**这一站真正想解决的架构问题是什么？**

回答：这一站的核心是把 Claude in Chrome 与 Computer Use 作为 in-process MCP server 嵌进 CLI，而不再额外拉起巨大子进程。它强调的是会话内联接、状态回写和专门审批，而不是单纯“支持浏览器/桌面控制”。

### 反

问题：**如果把这一站的设计反过来，会发生什么？**

回答：如果仍靠外部重量级子进程承载这些能力，启动成本、状态同步和权限 UI 都会变得更松散。看似只是多开一个服务，实际上会把交互延迟与控制权一起外包出去。

### 空

问题：**跳出当前文件名，这一站背后更大的问题是什么？**

回答：更大的问题是，具身交互能力如何被纳入现有工具体系，而不成为另一个孤岛。141 站说明，哪怕是最重的控制能力，也应尽量回到同一个 MCP 与审批框架里。

### 读完后应抓住的 2 个事实

1. **In-Process 传输避免 325MB 子进程**——CU 和 Chrome 都用 `createLinkedTransportPair()` 在进程内运行。配置假装是 stdio 以命中正确分支，但 client.ts 按名拦截。

2. **Computer Use 绕过标准 MCP 权限**——`mcp__computer-use__*` 工具添加到全局 allowedTools 列表，内部 `request_access` 处理审批。这是设计选择：CU 有自己的审批 UI（`ComputerUseApproval` React 组件 + Lock 机制）。

---

## 第 142 站：Turn Loop 主循环（query.ts + 配套文件，~2.3K）

### 这是什么

`query.ts` 是 Claude Code 的核心 `while(true)` 生成器——驱动从用户输入到模型响应、工具执行、压缩、递归的完整生命周期。

---

### 11 阶段管线

```
Phase 1:  预取（memory + skill discovery）
Phase 2:  上下文管理（tool result budget → snip → microcompact → collapse → autocompact）
Phase 3:  阻塞检查（blocking_limit → 终止）
Phase 4:  API 调用（流式 callModel，收集 tool_use blocks）
Phase 5:  流式错误恢复（collapse drain → reactive compact → OTK escalation，最多 3 次重试）
Phase 6:  Stop hooks
Phase 7:  Token budget 检查（生产力低时继续，diminishing 时停止）
Phase 8:  工具执行（StreamingToolExecutor 或 runTools 批量）
Phase 9:  工具摘要（Haiku 非并行调用）
Phase 10: 附件与命令（drain queued commands，consume memory prefetch）
Phase 11: 递归（重建 state，continue）
```

---

### Auto Compact

**阈值**：`effectiveContextWindow - 13,000` tokens。

**Circuit breaker**：连续 3 次失败后跳过。这是真实事件修复的——之前有 1,279 个会话各有 50+ 连续失败，每天浪费 ~250K API 调用。

**抑制规则**：`session_memory`/`compact` 查询源（防递归）、context collapse 启用时（竞争关系）、reactive compact 启用时。

---

### Token Budget

```ts
COMPLETION_THRESHOLD = 0.9          // 90% 预算以下
DIMINISHING_THRESHOLD = 500         // 连续检查间 <500 tokens

// 当 continuationCount >= 3 时，连续两次 <500 tokens → diminishing returns 提前停止
```

防止模型用最小工作量耗光全部 token 预算。

---

### 关键设计模式

1. **State 重建**——可变跨迭代状态在循环顶部解构，每个 continue 站点用 `state = {...}` 重建。
2. **ToolUse 按消息分组**——单个 assistant 消息的所有 tool_use blocks 收集后一起执行，结果一起追加。
3. **错误恢复链式**——prompt-too-long → collapse drain → reactive compact → error surface。

---

### 元

问题：**这一站真正想解决的架构问题是什么？**

回答：这一站是 Claude Code 的主神经中枢，把预取、上下文管理、API 调用、错误恢复、工具执行和递归串成 11 阶段循环。它真正处理的不是单次回答，而是系统如何一轮一轮持续运转而不失控。

### 反

问题：**如果把这一站的设计反过来，会发生什么？**

回答：如果没有这样一条总循环，各种 compact、hooks、tool executor 和 budget 检查只能零散插入，彼此顺序会越来越依赖偶然。那时系统不是没功能，而是没有节拍。

### 空

问题：**跳出当前文件名，这一站背后更大的问题是什么？**

回答：更大的问题是，LLM 应用如何把“会话”做成一台稳定机器。Turn Loop 的存在，就是在为所有局部机制提供统一时间轴。

### 读完后应抓住的 4 个事实

1. **11 阶段管线是确定的**——每个 turn 严格按 Phase 1→11 执行。工具执行在 Phase 8（Stop hooks 之后，递归之前）。

2. **Auto compact circuit breaker 来自真实事件**——1,279 个会话各 50+ 连续失败，每天 ~250K 无效 API 调用。`MAX_CONSECUTIVE_AUTOCOMPACT_FAILURES = 3` 就是答案。

3. **Token Budget 的 diminishing returns**——连续 `continuationCount >= 3` 后，连续两次检查间 < 500 tokens 就触发提前停止，防止模型无意义消耗。

4. **ToolUse 按消息分区而非按单个**——所有来自同一 assistant 消息的 tool_use blocks 一起执行。这意味着如果模型返回 5 个工具的 tool_use，它们都在同一批中处理。
