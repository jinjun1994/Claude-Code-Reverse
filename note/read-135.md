## 第 155 站：Bridge 远程控制系统（bridge/ + remote/，~12 个文件）

### 这是什么

Bridge 实现"Remote Control"——本地 Claude Code CLI 与 claude.ai Web 界面之间的双向集成层。允许用户从 Web 控制本地会话。

---

### 两条并行实现

| 版本 | 架构 | 关键文件 |
|------|------|----------|
| **V1 (env-based)** | 通过 Environments API 派发工作——注册本地为 "environment"，poll/ack/stop/heartbeat/deregister 生命周期 | `bridgeMain.ts`, `replBridge.ts`, `initReplBridge.ts` |
| **V2 (env-less)** | 直连 `/v1/code/sessions` 会话入口层——无 poll/ack 生命周期，SSE + CCRClient | `remoteBridgeCore.ts` |
| **Remote viewer** | WebSocket 直连远程 CCR 会话用于实时监控 | `RemoteSessionManager.ts`, `SessionsWebSocket.ts` |

**V2 连接步骤**：
```
1. POST /v1/code/sessions 创建会话
2. POST /bridge 获取 worker JWT + epoch
3. 创建 v2 transport（SSE + CCRClient）
4. 调度主动 JWT 刷新（过期前 5 分钟）
5. 401 恢复 → 重建 transport + 新 epoch
```

---

### 核心类型

**`BridgeConfig`**——dir, machineName, branch, maxSessions, spawnMode, sandbox, bridgeId, workerType
**`WorkSecret`**——解码 base64url 载荷：`session_ingress_token`, `api_base_url`, git info, auth tokens, env vars
**`SessionHandle`**——运行时子会话句柄：sessionId, done Promise, kill/forceKill, activities ring buffer, writeStdin
**`SpawnMode`**——`'single-session' | 'worktree' | 'same-dir'`（最多 32 并发会话）

---

### 关键设计模式

1. **依赖注入**——`initBridgeCore`/`initEnvLessBridgeCore` 不从全局 bootstrap 读取任何内容。所有回调（`toSDKMessages`, `createSession`, `archiveSession`, `onAuth401`）均为注入，使 daemon/SDK 调用者不需要拉入整个 REPL 树。

2. **Echo 去重**——两个实现都使用 `BoundedUUIDSet`（FIFO 环形缓冲集，容量 ~2000）检测服务器回显刚发出的消息。防止双重处理。

3. **FlushGate**——初始历史刷入新会话时，实时写入通过 `FlushGate` 排队，保证服务器收到 `[history..., live...]` 严格顺序。

4. **JWT 刷新调度器**——过期前 5 分钟主动刷新，代计数器使飞行中的旧刷新失效，降级 30 分钟刷新链，最多 3 次失败上限。

5. **Epoch 管理（V2）**——每次 `/bridge` 调用 bump 服务端 `worker_epoch`。transport 必须用新 epoch 重建——否则旧 CCRClient 心跳到旧 epoch 会在 20 秒内收到 409。

6. **Session ID 兼容 shim**——`cse_*` vs `session_*` 标签 ID 通过 `toCompatSessionId` / `toInfraSessionId` / `sameSessionId` 处理，由 GrowthBook 门控。

7. **多会话 Spawn 模式**：
   - `worktree`：每个会话创建隔离 git worktree
   - `same-dir`：所有会话共享 cwd（可能冲突）
   - `single-session`：一个会话，完成后销毁

---

### Remote Session Viewer

**`SessionsWebSocket`**——通过 `/v1/sessions/ws/{id}/subscribe` 连接，OAuth 认证，5 次重试/2s 延迟退避，30s ping 间隔。

**`sdkMessageAdapter`**——将 CCR 的 `SDKMessage` 转换为内部 REPL `Message` 类型。

**`remotePermissionBridge`**——远程权限请求/响应桥接。

---

### 元

问题：**这一站真正想解决的架构问题是什么？**

回答：这一站真正要解决的是，本地 CLI 与 claude.ai Web 如何形成一条双向会话桥，而不是简单远程命令通道。V1/V2 两套桥接架构、FlushGate 顺序保证、JWT 刷新与 epoch 重建，说明它在处理的是远程控制下的时序、身份和会话一致性。

### 反

问题：**如果把这一站的设计反过来，会发生什么？**

回答：如果没有独立桥接层，消息去重、历史补刷、实时写入、401 恢复都会混进上层业务逻辑。那样一旦 Web 与本地两边都能发消息，最先坏掉的就不是功能，而是“这到底是不是同一个会话”的确定性。

### 空

问题：**跳出当前文件名，这一站背后更大的问题是什么？**

回答：更大的问题是，本地执行环境如何安全地被远端接管而又不丢掉本地上下文。Bridge 讨论的不是连接本身，而是远程协作时代 CLI 的会话主权问题。

### 读完后应抓住的 3 个事实

1. **Bridge V1 vs V2 架构差异**——V1 走 Environments API（poll/ack 生命周期），V2 直连会话入口（SSE + epoch 管理）。V2 由 `tengu_bridge_repl_v2` GrowthBook 门控。Epoch 机制是关键——JWT 刷新不够，epoch 变化必须重建 transport，否则 20s 内 409。

2. **BoundedUUIDSet Echo 去重**——FIFO 环形缓冲集，容量 ~2000。本地发出的消息 ID 被缓存，如果服务器回显同一个消息，检测到重复后跳过。防止桥接层的消息翻倍。

3. **FlushGate 顺序保证**——初始化时历史消息和实时消息必须按严格顺序送达服务器。FlushGate 在历史 flush 完成前排队实时写入，保证 `[history..., live...]` 顺序。

---

## 第 156 站：UI 层架构（Ink + REPL + Components）

### 这是什么

基于 Ink（React 终端渲染器）的交互式 UI 系统——Bootstrap 链、Store 架构、REPL 主组件、消息渲染管线、权限对话框。

---

### Bootstrap 链

```
Stage 1: src/entrypoints/cli.tsx
  → void main()，快速路径分派子命令（--version, daemon, bridge, ps, logs, attach, kill）
  → 动态 import() 最小模块加载
  → 未命中快速路径 → import main.tsx 的 main()

Stage 2: src/main.tsx（~2500+ 行，最大文件）
  → 迁移、settings、会话类型
  → init() → 环境设置（配置、TLS、代理、telemetry）
  → getDefaultAppState() → createStore()
  → showSetupScreens() → renderAndRun()

Stage 3: src/replLauncher.tsx
  → 动态 import App.tsx + REPL.tsx
  → Ink root: <App><REPL /></App>
```

---

### Store 架构（src/state/）

**最小化 Store 模式**——不是 Redux 也不是 Zustand：
```typescript
type Store<T> = {
  getState: () => T
  setState: (updater: (prev: T) => T) => void
  subscribe: (listener) => () => void
}
```

**双状态模式**：
| 层 | 内容 | 原因 |
|---|---|---|
| **全局 AppStateStore** | 权限、MCP、插件、任务、设置 | 配置状态，多组件共享 |
| **REPL 本地 useState** | messages、streamingToolUses、toolUseConfirmQueue、promptQueue | 消息高频更新（每个流式 chunk），不适合全局 store |

**`onChangeAppState`**——AppState 变更的副作用同步器：持久化 verbose/expandedView 到 globalConfig，同步 permission mode 到 CCR/SDK，清除 auth 缓存，应用环境变量。

---

### Ink 渲染引擎（src/ink/）

这是 **深度定制的 Ink fork**：

| 层 | 文件 | 职责 |
|---|------|------|
| Reconciler | `ink.tsx`, `reconciler.ts` | React Fiber 终端 reconciler |
| Renderer | `renderer.ts`, `output.ts` | 纤维树 → 终端输出 |
| Scheduler | `frame.ts` | 帧调度、FPS 追踪 |
| Components | `components/Box`, `Text`, `ScrollBox`, `Button`, `Link`, `RawAnsi` | 基础组件 |
| Hooks | `hooks/useInput`, `useTerminalSize`, `useAnimation` | 终端交互 hook |
| Layout | `layout/` | Yoga flexbox 布局引擎 |
| Termio | `termio/ansi.ts`, `csi.ts`, `dec.ts`, `esc.ts`, `osc.ts`, `sgr.ts` | 终端协议解析 |
| Log | `log-update.ts` | 拦截 console.log 防止与 Ink 输出混合 |

---

### REPL 组件（src/screens/REPL.tsx，~5200+ 行）

**本地状态管理**（不在全局 store 中）：
- `messages`——会话数组 `useState<MessageType[]>`
- `streamingToolUses`, `streamingThinking`
- `toolUseConfirmQueue`——权限对话框队列
- `promptQueue`——hook prompt 队列
- `sandboxPermissionRequestQueue`——网络沙盒审批
- `screen`——prompt/transit 等视图状态
- `streamMode`——requesting/responding/tool-use

**权限对话框管线**：
```
useCanUseTool hook（tool 执行 permission 阶段调用）
  → setToolUseConfirmQueue（入队）
    → PermissionRequest 组件（toolUseConfirmQueue 消费）
      → PermissionDialog（样式边框）+ PermissionRequestTitle
        → 4-way racer system（bridge/channel/hook/classifier）
```

---

### 组件架构（src/components/）

**设计系统**——`ThemeProvider`, `ThemedBox`, `ThemedText`, `Ratchet`, `ProgressBar`, `Divider`, `Dialog`
**消息渲染**——`VirtualMessageList.tsx`（虚拟滚动）、`MessageRow.tsx`（单条渲染）
**消息类型组件**（`src/components/messages/`）——30+ 种消息渲染：AssistantText/Thinking/ToolUse/RedactedThinking、UserText/Prompt/BashInput/Image、HookProgress、RateLimit、CompactBoundary 等

**权限组件**（`src/components/permissions/`）：
- `PermissionDialog.tsx`——带样式边框的视觉容器
- `PermissionRequestTitle.tsx`——标题/副标题，可选 worker badge
- `WorkerPendingPermission.tsx`——Swarm worker 的待定权限
- `SandboxPermissionRequest.tsx`——网络域审批

---

### 元

问题：**这一站真正想解决的架构问题是什么？**

回答：这一站把 Ink、Store、REPL、消息组件和权限对话框放到一张图里，核心在于终端 UI 不是“显示文本”，而是一套完整交互运行时。双状态模式、深度定制 Ink fork、权限 4-way racer 的 UI 管线，都表明它承担的是终端会话产品的表现层骨架。

### 反

问题：**如果把这一站的设计反过来，会发生什么？**

回答：如果只把 UI 当成组件拼装，消息流、权限请求、状态同步和渲染性能会很快互相拖累。尤其 `messages` 这类高频状态若塞进全局 store，表面统一，实际上会把整个交互层拖进低效和混乱。

### 空

问题：**跳出当前文件名，这一站背后更大的问题是什么？**

回答：更大的问题是，终端这种受限媒介如何承载复杂 AI 会话体验。这里讨论的不是某个组件，而是“CLI 也能成为产品界面”的系统性答案。

### 读完后应抓住的 3 个事实

1. **双状态模式（Store vs useState）**——messages 不放在全局 AppState 中，每个流式 chunk 更新 store 代价太高。全局 store 用于"配置"状态（权限、MCP、插件），本地 useState 用于 REPL 专有状态（消息、流式、队列）。这是性能优化设计。

2. **Ink 是深度定制 fork**——不是直接 npm install ink，而是 `src/ink/` 下的完整 fork，包含自定义 reconciler、Yoga 布局、终端协议解析（ANSI/CSI/DEC/ESC/OSC/SGR）、帧调度。这允许对终端渲染做底层优化。

3. **Permission 4-way Racer**——权限决定由 4 个来源竞争：bridge（远程控制）、channel（hook 通道）、hook（权限 hook）、classifier（yolo 分类器）。`claim()` 原子守卫确保 race-free 决定。

---

## 第 157 站：核心 Utils 系统（messages/context/model/toolPool/attachments/claudemd）

### messages.ts（~5500 行）—— 消息编排中心

**角色**——消息构造、格式化、序列化、解释的完整编排层。

**关键常量**：
- `INTERRUPT_MESSAGE`, `CANCEL_MESSAGE`, `REJECT_MESSAGE`, `SUBAGENT_REJECT_MESSAGE`——用户拒绝/中断时注入的精确文本
- `DENIAL_WORKAROUND_GUIDANCE`——模型被拒绝时的替代方案指导
- `SYNTHETIC_TOOL_RESULT_PLACEHOLDER`——`ensureToolResultPairing` 插入的占位，标记为"RLHF 训练数据毒药"
- `MEMORY_CORRECTION_HINT`——auto memory 开启且 `tengu_amber_prism` 标志启用时附加到拒绝消息

**`deriveShortMessageId(uuid)`**——从 UUID 派生确定性短 ID（6 位 base36），用于 "snip" 工具引用。

---

### context.ts（222 行）—— 上下文窗口与 Token 预算

**常量**：
```
MODEL_CONTEXT_WINDOW_DEFAULT = 200,000
CAPPED_DEFAULT_MAX_TOKENS = 8,000     ← 避免过度预留 slot 容量（<1% 请求命中上限，触发后 64K 一次性干净重试）
ESCALATED_MAX_TOKENS = 64,000         ← 升级目标
COMPACT_MAX_OUTPUT_TOKENS = 20,000
MAX_OUTPUT_TOKENS_DEFAULT = 32,000
MAX_OUTPUT_TOKENS_UPPER_LIMIT = 64,000
```

**`getContextWindowForModel()` 优先级链**：
1. `CLAUDE_CODE_MAX_CONTEXT_TOKENS`（仅 Ant 环境变量）
2. 模型 `[1m]` 后缀（客户端显式 opt-in）
3. `getModelCapability()`——API 缓存中 `max_input_tokens >= 100,000`
4. Beta header 包含 1M beta AND `modelSupports1M()`
5. GrowthBook sonnet 4.6 1M 实验（`coral_reef_sonnet`）
6. Ant 模型配置 `contextWindow` 字段
7. 默认 200K

`modelSupports1M()`——仅 `claude-sonnet-4` 和 `opus-4-6`。

---

### model/ 目录—— 模型解析与选项

**`model.ts`**——核心模型解析：
- `parseUserSpecifiedModel()`——中央解析器，剥离 `[1m]` 后缀，解析别名（opus/sonnet/haiku/best/opusplan），支持 Ant 模型别名
- `getRuntimeMainLoopModel()`——上下文感知模型选择：`opusplan` 在 plan 模式（<200K tokens）下使用 Opus，否则 Sonnet；Haiku plan 模式升至 Sonnet
- `firstPartyNameToCanonical()` / `getCanonicalName()`——去除日期/提供者后缀统一为规范名（如 `claude-3-7-sonnet-20250219` 和 Bedrock ID 都映射到 `claude-3-7-sonnet`）
- `resolveSkillModelOverride()`——解析 skill frontmatter 中的 `model:` 字段，携带 `[1m]` 后缀，防止 skill 降级上下文窗口
- `maskModelCodename()`——遮盖内部 codename（"capybara-v2-fast" → "cap*****-v2-fast"）

**`modelCapabilities.ts`**——动态模型能力 API 获取，磁盘缓存到 `~/.claude/cache/model-capabilities.json`。仅 Ant 用户 + 一方 API 可用。

**`modelOptions.ts`**——每个用户层级（Ant/Max/Premium/Pro/PAYG）的模型选择器选项。

---

### toolPool.ts（80 行）—— 工具池合并与去重

```typescript
mergeAndFilterTools(initialTools, assembled, mode):
1. initialTools 优先去重
2. 分区：built-in 工具 vs MCP 工具（built-in 必须是连续前缀以利用服务端缓存策略）
3. 各自按名称字母排序
4. 返回 [built-in..., MCP...]
5. coordinator 模式过滤
```

**React-free**——不导入 React，使 `print.ts` 可在 ink 依赖链外使用。

---

### sessionStart.ts（233 行）—— 会话启动 Hook 编排

**关键模式**：
- 旁路通道 `pendingInitialUserMessage`——模块级变量持有单条初始消息，避免改变 `Promise<HookResultMessage[]>` 返回类型（影响 5 个调用点）
- `shouldAllowManagedHooksOnly()` 安全门——阻止插件 hook 作为"不信任外部代码"
- `"CRITICAL: do not add ANY 'warmup' logic"`——启动性能不容妥协

---

### claudemd.ts（~1480 行）—— CLAUDE.md 发现、解析、注入

**加载优先级**（从低到高）：
1. Managed `/etc/claude-code/CLAUDE.md`——组织级策略
2. User `~/.claude/CLAUDE.md`——个人全局指令
3. Project `CLAUDE.md`, `.claude/CLAUDE.md`, `.claude/rules/*.md`——仓库内
4. Local `CLAUDE.local.md`——项目私有，gitignored

**关键常量**：
```
MAX_MEMORY_CHARACTER_COUNT = 40,000  // 单文件上限
MAX_INCLUDE_DEPTH = 5               // 防止 @include 循环引用
TEXT_FILE_EXTENSIONS = 120+ 种文本扩展名
```

**`@include` 语法**——`@path`, `@./path`, `@~/path`, `@/absolute/path`，仅在叶子文本节，不在代码块内。通过 `processedPaths` 集合防止循环。

**`contentDiffersFromDisk` 追踪**——当 HTML 注释剥离、frontmatter 移除或 MEMORY.md 截断时，保留原始磁盘字节。调用者可缓存 `isPartialView` 状态，使 Edit/Write 操作知道需要先真实 Read。

**`getMemoryFiles()` 处理流程**：
- 从文件系统根向下遍历到 CWD，逐层收集 CLAUDE.md
- git worktree 处理：嵌套 worktree 中跳过主仓库已检入文件
- AutoMem 入口（memory.md）
- TeamMem 入口（TEAMMEM 特性开启时）
- 详细分析日志：时长、文件数、每类型计数、内容长度

**`getDateChangeAttachments()`**——检测跨午夜并发送 `date_change` 附件，同时为 `/dream` 技能刷新会话 transcript。

---

### attachments.ts（~2000+ 行）—— 附件工厂

**`Attachment` 类型**——90+ 种变体的庞大类型联合：

| 类别 | 类型示例 |
|------|----------|
| 文件/内容 | FileAttachment, PDFReference, edited_text_file, edited_image_file |
| 记忆 | nested_memory, relevant_memories, current_session_memory |
| 模式/状态 | plan_mode, auto_mode, plan_mode_exit, auto_mode_exit |
| Hook | cancelled, blocking_error, success, permission_decision 等 10 种 |
| 智能体 | agent_mention, task_status, team_context, teammate_mailbox |
| 系统 | critical_system_reminder, todo_reminder, token_usage, budget_usd |
| 技能 | dynamic_skill, skill_listing, skill_discovery |

**`getAttachments()` 三执行层**：
1. **用户输入附件**——at-mentioned 文件、MCP 资源、agent mention、skill discovery
2. **线程附件**（主/子代理共用）——queued commands, date change, deferred tools delta, nested memory, plan mode 等
3. **仅主线程**——IDE selection, IDE opened file, diagnostics, token usage, budget USD

**关键设计**：
- `maybe(label, fn)`——每项附件的错误边界，5% 采样计时/大小日志
- 1 秒超时门控防御慢计算
- `CLAUDE_CODE_DISABLE_ATTACHMENTS` 和 `CLAUDE_CODE_SIMPLE` 短路只保留 queued commands
- **确定性渲染**——Memory 附件头在 attachment 创建时预计算（而非渲染时），因为 `memoryAge()` 调用 `Date.now()` 会导致 "3 days ago" 跨轮变为 "4 days ago"——bust cache
- Agent 列表从工具描述移到 delta 附件——嵌入减少 ~10.2% 的 fleet cache_creation 命中

---

### 元

问题：**这一站真正想解决的架构问题是什么？**

回答：这一站的重点是把消息编排、上下文窗口、模型解析、工具池、CLAUDE.md 和记忆附件这些横切问题收束为基础设施。它们共同服务的不是一个功能点，而是每一轮 query 怎样在 token、缓存、上下文和提示材料之间稳定成形。

### 反

问题：**如果把这一站的设计反过来，会发生什么？**

回答：如果这些能力散在各模块里，最先失控的是一致性：模型名解析会分叉，附件会随机变动，`Date.now()` 一类细节就会不停 bust cache。到了那时，系统不会立刻坏掉，但会越来越难解释为什么同一轮请求总在“悄悄变”。

### 空

问题：**跳出当前文件名，这一站背后更大的问题是什么？**

回答：往大了看，这一站讨论的是 AI 系统如何把“真正发给模型的内容”构造得既丰富又稳定。它背后的长期问题，是提示上下文怎样既动态、又可缓存、还能被治理。

### 读完后应抓住的 4 个事实

1. **Capped max_tokens 8K + 64K 一次干净重试**——默认 8K 而非 32K/64K，因为 BigQuery p99 输出是 4911 tokens——32K 过度预留 8x。不到 1% 请求触达上限，触发后获得 64K 一次性干净重试。这是基于数据的生产调参。

2. **contentDiffersFromDisk 追踪**——CLAUDE.md 处理（去掉 HTML 注释/截断）后的内容与磁盘字节可能不同。标记此状态使 Edit 操作知道需要先 Read 真实内容而不是缓存的版本。这是一个一致性保护。

3. **附件确定性渲染防 cache bust**——Memory 附件头预计算而不是在渲染时计算，因为 `Date.now()` 在跨轮次时变化（"3天前" → "4天前"），会破坏 prompt cache。这是真实 cache miss 事件修复的。

4. **Attachment maybe() 吞错设计**——每项附件都包裹在 `maybe(label, fn)` 中，捕获错误并吞掉返回 `[]`。这是 fail-open 设计——单个附件失败不应阻止整个轮次进行。

---

## 第 158 站：Voice、Diagnostic Tracking、Terminal 工具

### Voice 系统

**三层录音后端**（按优先级）：
1. **Native**——`audio-capture-napi`，链接 CoreAudio（macOS）或 cpal（Linux/Windows），首次按键时懒加载（dlopen 阻塞事件循环 1-8s）
2. **arecord**——ALSA Linux 回退，150ms 超时探测区分真实设备（WSLg）与仅二进制存在（WSL1/headless）
3. **SoX `rec`**——通用回退，16kHz mono 16-bit PCM

**Voice Stream STT**：
- WebSocket 连接 `api.anthropic.com`（非 claude.ai——Cloudflare TLS 指纹阻止非浏览器客户端）
- 二进制音频帧 + JSON 控制消息（KeepAlive 每 8s，CloseStream 结束时）
- 5 种 finalize 触发：`post_closestream_endpoint`（~300ms，快乐路径）/ `no_data_timeout`（1.5s）/ `safety_timeout`（5s）/ `ws_close` / `ws_already_closed`
- `tengu_cobalt_frost` 门控使用 Deepgram `stt_provider: deepgram-nova3`

**Voice Keyterms**——无需模型调用，从 4 个来源构建最多 50 个关键词：全局硬编码术语（MCP/symlink/grep 等）+ 项目根名 + git 分支词（拆分 `feat/voice-keyterms` → `[feat, voice, keyterms]`）+ 最近文件名。标识符拆分支持 camelCase/PascalCase/kebab-case/snake_case。

**语音门控**——`isVoiceModeEnabled()` = `hasVoiceAuth()`（需要 Anthropic OAuth，不支持 API key/Bedrock/Vertex/Foundry）+ `isVoiceGrowthBookEnabled()`（`VOICE_MODE` 构建标志 + GrowthBook 急停开关未开启）

---

### Diagnostic Tracking

**目的**——回答 "Claude 是否破坏了什么？"——在文件编辑前后捕获 IDE 级诊断（编译器错误、linter 警告），检测**新增**诊断。

**基线-差异模式**：
```
编辑前: beforeFileEdited() → getDiagnostics RPC → 存入 baseline
编辑后: getNewDiagnostics() → 重新获取 → 与 baseline 差分 → 仅返回新增项
```

**双源诊断**——处理 `file://` URI 和 `_claude_fs_right:` / `_claude_fs_left:` URI（Claude 内部文件系统比较协议）。

**IDE 集成通过 MCP**——所有诊断来自 IDE RPC 调用（`callIdeRpc('getDiagnostics', ...)`）。从 MCP 客户端数组中找到已连接的 IDE 客户端。

---

### Terminal 工具

**tmux Socket 隔离（`tmuxSocket.ts`）**：
- 创建隔离 socket `claude-<PID>`（`-L` 标志）
- `getClaudeTmuxEnv()` 返回 `"socket_path,server_pid,pane_index"` 格式
- Shell 覆盖 TMUX 环境变量到所有子进程
- 懒初始化——首次使用 Tmux 工具时创建
- 关闭时 `registerCleanup` 销毁

**tmux Swarm Backend（`TmuxBackend.ts`）**：
- tmux 内：leader 30%，teammates 70% 水平分割；额外 teammate 在 70% 区域内平铺
- tmux 外：创建 `claude-swarm` session，全平铺布局
- 每个 agent 彩色边框分隔、异步锁防止并行创建冲突

**`terminal.ts`**——文本截断工具，3 行可见窗口 + "... +N lines (ctrl+o to expand)" 提示，ANSI-aware 切片。

---

### 元

问题：**这一站真正想解决的架构问题是什么？**

回答：这一站把语音输入、编辑后诊断差分和 tmux/terminal 能力放在一起，说明它处理的是“会话外围感知能力”。无论是三层录音后端、诊断基线差分，还是隔离 tmux socket，本质上都在让 Claude 不只会说话，还能感知与操作外部环境。

### 反

问题：**如果把这一站的设计反过来，会发生什么？**

回答：如果这些能力只是零散外挂，语音会在无设备环境里卡住，诊断会变成噪音式全量报告，终端会话也会和用户现有 shell 环境互相污染。最先出问题的不是功能缺失，而是这些外围能力无法可靠地接入主会话。

### 空

问题：**跳出当前文件名，这一站背后更大的问题是什么？**

回答：更大的问题是，AI 助手如何稳妥地借用人类已有的输入、IDE 和终端生态。这里讲的不是录音或 tmux 本身，而是 Claude Code 与外部工具世界的接口设计。

### 读完后应抓住的 2 个事实

1. **Voice 三层后端 + 150ms arecord 探测**——Native → arecord → SoX rec 按优先级排列。arecord 的 150ms 超时是关键：它区分 WSLg（真实音频设备，150ms 内响应）和 WSL1/no audio（二进制存在但无设备，超时）。防止在无音频的环境中挂起。

2. **Diagnostic Tracker 的基线-差异模式**——编辑前快照 + 编辑后差分 = 只报告新增诊断。这不是持续监控——只在 Claude 编辑文件前后捕获。这是精确的归因设计：只报告 Claude 引入的破坏。
