## 第 168 站：Query 模块子目录（config/deps/stopHooks/tokenBudget）

### 这是什么

`src/query/` 目录是 `query()` 主循环的支持模块——从 query.ts 中分离出配置、依赖注入、stop hooks 和 token 预算。

---

### query/config.ts——不可变配置快照

**`QueryConfig`**——`sessionId` + `gates`（streamingToolExecution, emitToolUseSummaries, isAnt, fastModeEnabled）。

**`buildQueryConfig()`**——在 `query()` 入口时读取会话 ID，评估 feature gates，返回冻结配置对象。

**设计决策**：
- 显式分离不可变配置（快照一次）与每轮可变状态（`State`）和 `ToolUseContext`
- 使未来提取纯 `step(state, event, config)` 归约器可行
- 故意排除 `feature()` gates（bun 编译时 bundle 标志）——只有运行时 gates（env vars, Statsig）
- 从 `fastMode.ts` 内联 `fastModeEnabled` 检查——避免拉入重模块依赖图（axios, settings, auth, model, oauth, config）到测试分片

---

### query/deps.ts——依赖注入

**`QueryDeps`**——4 个依赖：
```typescript
{
  callModel: typeof queryModelWithStreaming
  microcompact: typeof microCompactMessages
  autocompact: typeof autoCompactIfNeeded
  uuid: () => string
}
```

**`productionDeps()`**——返回真实实现的工厂函数。

使用 `typeof fn` 模式使测试代码获得签名检查的 mock 而无需额外样板。范围有意限制为 4 个依赖——注释说明未来 PR 可扩展到 `runTools`, `handleStopHooks`, `logEvent` 等。

---

### query/stopHooks.ts——Turn 结束 Hook

**`handleStopHooks()`**——异步生成器：
```
1. 构建 REPLHookContext（messages/systemPrompt/contexts/tool-use 状态）
2. 保存 cache-safe params 供主会话查询用（子代理不执行）
3. 模板作业分类（CONDITIONAL TEMPLATES + CLAUDE_JOB_DIR env）
4. 后台任务（prompt suggestion/memory extraction/auto-dream）——非 bare 模式
5. 清理 computer-use 会话（CHICAGO_MCP 特性）
6. 执行 executeStopHooks（异步生成器，消费进度消息）
7. 追踪 hook 计数、toolUseIDs、错误、时长
8. 执行期间检查 abort
9. 如有 hook 运行，创建汇总系统消息
10. 执行 executeTaskCompletedHooks（当前 teammate 的在进任务）
11. 执行 executeTeammateIdleHooks
12. 返回 { blockingErrors, preventContinuation }
```

**设计决策**：
- 异步生成器模式处理流式 `StreamEvent` 结果，允许增量生成进度消息
- Feature-gated 条件导入（`EXTRACT_MEMORIES`, `TEMPLATES`, `CHICAGO_MCP`）带懒 `require()` 树摇
- 仅主线程守卫：子代理不能覆盖 cache params、运行后台任务、释放 computer-use 锁
- 作业分类器与 60 秒超时竞速（`.unref()` 不阻塞退出）
- computer-use 清理静默失败（实验特性，非关键路径）
- 每 hook 时长匹配——命令 + 首个未分配条目匹配（hook 并行运行）

---

### query/tokenBudget.ts——Token 预算决策

**`BudgetTracker`**——`continuationCount`, `lastDeltaTokens`, `lastGlobalTurnTokens`, `startedAt`

**`checkTokenBudget()`**——返回决策：
```
常量：COMPLETION_THRESHOLD = 90%,  DIMINISHING_THRESHOLD = 500 tokens

1. 如果有 agentId（子代理）或 budget 为 null/0 → 停止（无完成事件）
2. 计算预算消耗百分比 + 上次检查以来的增量
3. diminishing returns: continuationCount >= 3 AND 最近增量都 < 500 tokens
4. 未 diminishing AND < 90% 预算 → 继续
5. diminishing 或已继续至少一次 → 停止
6. 否则 → 停止（无完成事件）
```

**Diminishing returns 检测**——防止模型产生微小增量输出时的浪费继续。

---

---

### 元

问题：**这一站真正想解决的架构问题是什么？**

回答：这一站把 `query()` 主循环的支撑件拆成 config、deps、stopHooks、tokenBudget，说明它关心的是把“查询循环”从一大坨流程拆成可治理的小层。不可变配置快照、受限依赖注入、停止 hook 异步生成器与 token 预算决策，都是为了让主循环更像引擎而不是脚本。

### 反

问题：**如果把这一站的设计反过来，会发生什么？**

回答：如果这些东西都继续堆在 `query.ts` 里，任何 feature gate、hook、预算规则变更都会直接拖垮主循环可读性。那时最先失去的不是功能正确性，而是 query 作为系统核心的可演化性。

### 空

问题：**跳出当前文件名，这一站背后更大的问题是什么？**

回答：更大的问题是，一次 AI 调用的主循环怎样在越来越多附属职责中维持可拆解结构。这里探讨的是 query 作为运行时核心的模块化治理。

## 第 169 站：Context Providers（React 上下文提供者）

### 这是什么

`src/context/` 目录全部是 React context providers/hooks——供 Ink 终端 UI 使用。

---

### 9 个 Context 提供者

| Context | 用途 |
|---------|------|
| **voice.tsx** | 语音录制/播放状态管理，用 Store 而非 React state |
| **mailbox.tsx** | 全局 `Mailbox` 实例（消息/通知工具） |
| **modalContext.tsx** | 全屏模态对话框内的尺寸和滚动上下文 |
| **overlayContext.tsx** | 追踪哪些 overlay 组件活跃，协调 Escape 键处理 |
| **QueuedMessageContext.tsx** | 消息布局和队列状态 |
| **notifications.tsx** | 优先级通知队列 + 超时 + 折叠 + 取消 |
| **promptOverlayContext.tsx** | 浮层 portal 机制——从 prompt 输入上方溢出 |
| **fpsMetrics.tsx** | FPS 指标 getter 函数传递 |
| **stats.tsx** | 遥测/指标商店（counter/gauge/histogram/set） |

---

### 关键模式

**Provider/Hook 惯用法**——每个 context 文件遵循相同模式：创建 Context → Provider 组件 → hook（`useX`, `useGetX`, `useSetX`）。一致且可预测。

**React Compiler**——所有 context 文件用 React 编译器预编译（`react/compiler-runtime` 的 `$` 缓存变换）。原始源码使用标准 React 模式（useState/useMemo/useCallback）。

**State 管理拆分**：
- 应用级状态通过 `AppStoreContext`/`AppState`（overlayContext, notifications）
- 专用外部订阅商店（voice.tsx 用 `useSyncExternalStore`，stats.tsx 用自己的商店）

**Stats Store 设计**——Reservoir sampling（Algorithm R），大小 1024，近似百分位计算（p50/p95/p99）无存储无限数据。`getAll()` 展平为单一 `Record<string, number>`。进程 `exit` 时持久化到项目配置 `lastSessionMetrics`。

**Prompt Overlay Portal**——4 个 context（data/setter 对）。Split setter 对使写者从不因自己的写操作重新渲染（setter contexts 稳定）。解决 Ink 渲染问题：使用 `position: absolute bottom="100%"` 的浮层被 Ink 的 clip stack 裁切到约 1 行。

---

---

### 元

问题：**这一站真正想解决的架构问题是什么？**

回答：这一站真正要解决的是 Ink 终端 UI 的上下文状态如何被拆成一组稳定的 React provider，而不是把所有 UI 状态塞进一个大 store。voice、modal、overlay、notifications、stats 等 context 分工清晰，说明它要的是“共享状态的边界设计”。

### 反

问题：**如果把这一站的设计反过来，会发生什么？**

回答：如果没有这层 provider 组织，UI 共享数据要么层层 prop drilling，要么被粗暴并入全局状态。最后既会造成不必要重渲染，也会让像 prompt overlay 这种依赖布局技巧的机制变得极难维护。

### 空

问题：**跳出当前文件名，这一站背后更大的问题是什么？**

回答：更大的问题是，终端 UI 如何在没有浏览器 DOM 的前提下仍获得现代组件树的状态协作能力。Context provider 讨论的不是 React 习惯用法，而是 Ink 应用的状态分层。

## 第 170 站：Misc Services（awaySummary/AgentSummary/analytics/notifier/tokenEstimation/rateLimits）

### awaySummary.ts——"While You Were Away" 卡片

取最近 30 条消息 + 摘要提示词 → 小/快模型（无流式/思考/工具）。`skipCacheWrite: true` 避免污染 prompt cache。查询 session memory 获取更广上下文。静默失败（abort/空 transcript/错误时返回 null）。

---

### AgentSummary/——Coordinator Worker 摘要

每 ~30 秒 fork 子代理从 worker transcript 生成 3-5 词的现在时进度摘要。通过 `canUseTool` 拒绝工具（非 `tools: []`，保持 cache key 匹配）。`skipTranscript: true` 不记录 transcript。

---

### Analytics——多层分析系统

**`index.ts`**——公共 API。`AnalyticsSink` 接口：`logEvent()` / `logEventAsync()`。Sink 附加前的事件排队并通过 `queueMicrotask` 排空。类型守卫 `AnalyticsMetadata_I_VERIFIED_THIS_IS_NOT_CODE_OR_FILEPATHS = never` 防止意外记录代码/文件路径。

**`datadog.ts`**——批量发送日志到 Datadog HTTP intake。批次 100 或定时器 15s 刷新。~60 个事件名的白名单（`DATADOG_ALLOWED_EVENTS`）。用户分桶（30 桶，SHA256 哈希 mod 30）隐私保护基数。MCP 工具名归一化为 "mcp" 降低基数。

**`growthbook.ts`**——完整 GrowthBook feature flag 客户端：
- `getFeatureValue_CACHED_MAY_BE_STALE()`——非阻塞，内存 > 磁盘 > 默认
- `getFeatureValue_INTERNAL()`——初始化阻塞（弃用）
- `checkGate_CACHED_OR_BLOCKING()`——磁盘 fast-true，无缓存则阻塞等服务器
- 定期刷新（Ant 20 分钟，其他 6 小时）
- `CLAUDE_INTERNAL_FC_OVERRIDES` 环境变量覆盖

**`firstPartyEventLoggingExporter.ts`**——OpenTelemetry `LoggerProvider` + `BatchLogRecordProcessor`。`/api/event_logging/batch` POST + 二次退避重试 + JSONL 文件持久化失败事件 + 后台重试 + 401 无 auth 降级 + 每次 POST 检查 killswitch。

**`sinkKillswitch.ts`**——每 sink killswitch 通过 GrowthBook `tengu_frond_boric`。形状：`{ datadog?: boolean, firstParty?: boolean }`。Fail-open（缺少配置 = sink 保持开启）。

---

### notifier.ts——终端通知

**6 通道**——auto/iterm2/iterm2_with_bell/kitty/ghostty/terminal_bell/notifications_disabled。`sendAuto` 检测终端类型选最佳方法。Kitty 用随机 ID 追踪通知。执行通知 hook（`executeNotificationHooks`）后发送。

---

### preventSleep.ts——防睡眠

macOS `caffeinate -i -t 300`。**引用计数**——`startPreventSleep()` 递增，`stopPreventSleep()` 递减。每 4 分钟重启（在 5 分钟超时前自愈——如果 Node 被 SIGKILL 杀死）。仅 Darwin。`unref()` 不让它保持 Node 存活。SIGKILL 立即终止（非 SIGTERM）。

---

### tokenEstimation.ts——Token 计数

**三层策略**：
1. API `countTokens` 端点（主路径，带 VCR 缓存）
2. Bedrock AWS SDK `CountTokensCommand`（动态导入 ~279KB）
3. Haiku 回退——发 `max_tokens: 1` 请求，sum input + cache_creation + cache_read
4. 粗估——`content.length / bytesPerToken`（JSON = 2, 其他 = 4）

`stripToolSearchFieldsFromMessages()`——计数前移除 `caller` 和 `tool_reference` 字段。

---

### Rate Limits（claudeAiLimits + rateLimitMessages）

**`ClaudeAILimits`**——`status`（allowed/allowed_warning/rejected）, `resetsAt`, `rateLimitType`（five_hour/seven_day/seven_day_opus/seven_day_sonnet/overage）, `utilization`。

**两层预警检测**：
1. Header 基础：`surpassed-threshold` header
2. 时间回退：利用率 >= 阈值 AND 时间进度 <= 阈值（5h: 90% util at 72% time; 7d: 75%/60%, 50%/35%, 25%/15%）

**`makeTestQuery()`**——`max_tokens: 1` 独立配额检查。`rawUtilization` 暴露给 statusline 脚本。

**`rateLimitMessages.ts`**——3 种消息：限额错误/早期警告/超额通知。按订阅类型追加升级文案（`/extra-usage` 团队/企业，`/upgrade` pro/max）。

---

### remoteManagedSettings——企业托管设置

**`index.ts`**——从 `${BASE_API_URL}/api/claude_code/settings` 获取。Checksum ETag（`If-None-Match`）。5 次指数退避重试。每小时后台轮询。优雅降级（获取失败用旧缓存）。应用前安全检查危险设置变更。

**`securityCheck.tsx`**——远程设置含危险变更时显示阻塞对话框（如允许的命令/权限）。非交互模式跳过对话框。拒绝时 `gracefulShutdownSync(1)`。

**资格检查**——一方 Anthropic 提供者、非自定义 base URL、非 cowork 模式、API key 或 OAuth 且有企业/团队订阅。

---

---

### 元

问题：**这一站真正想解决的架构问题是什么？**

回答：这一站把 away summary、agent summary、analytics、notifier、token estimation、rate limits 和托管设置放到一起，展示的是一组“围绕主循环但不属于主循环”的运营性服务。它们共同承担的是总结、观测、告警、估算与策略同步，让会话系统更像产品而不只是模型壳。

### 反

问题：**如果把这一站的设计反过来，会发生什么？**

回答：如果这些服务各自长在业务分支里，缓存污染、事件排队、隐私约束、通知通道与限额判断都会变成局部约定。久而久之，系统就会出现很多“能工作但无法统一解释”的产品行为。

### 空

问题：**跳出当前文件名，这一站背后更大的问题是什么？**

回答：更大的问题是，AI 产品如何把运营、分析、限额和企业治理嵌进实时会话中而不打断主体验。这里真正讨论的是外围服务层怎样长期伴随核心能力成长。

## 第 171 站：VCR、Tool Limits、Prompts、System Prompt Sections

### VCR 测试夹具

**`withVCR()`**——非流式录制。SHA1 哈希归一化消息，缓存/加载夹具，从缓存回放。

**`dehydrateValue()`**——归一化平台特定数据：CWD 路径 → `[CWD]`, config home → `[CONFIG_HOME]`, 数值 → `[NUM]`, `[DURATION]`, `[COST]`。

**`withStreamingVCR()`**——缓冲流结果，委托给 `withVCR()`。

CI 中丢失夹具会抛出说明；本地自动录制。

---

### Tool Limits 常量

| 常量 | 值 | 用途 |
|------|-----|------|
| DEFAULT_MAX_RESULT_SIZE_CHARS | 50,000 | 单工具结果磁盘持久化前 |
| MAX_TOOL_RESULT_TOKENS | 100,000 | 单工具 token 上限 (~400KB) |
| MAX_TOOL_RESULT_BYTES | 400,000 | 推导自 token 限制 |
| MAX_TOOL_RESULTS_PER_MESSAGE_CHARS | 200,000 | 每消息聚合 tool_result 预算 |
| TOOL_SUMMARY_MAX_LENGTH | 50 | 紧凑视图摘要最大字符 |

---

### Prompts（prompts.ts）——系统提示组装

**`SYSTEM_PROMPT_DYNAMIC_BOUNDARY`**——分离静态（跨组织可缓存）与会话特定内容。

**Section 构建函数**——`getSimpleIntroSection()`, `getSimpleDoingTasksSection()`, `getUsingYourToolsSection()`, `getActionsSection()`, `getLanguageSection()`, `getMcpInstructionsSection()` 等。

**Feature-gated 模块导入**——`require()` 配合 `bun:bundle` 死代码消除。

**Ant 特定指令**——`USER_TYPE === 'ant'` 门控：注释风格指导、完成前验证、通过 `/issue` 或 `/share` 报告 bug。

**前沿模型名**——Claude Opus 4.6。模型族 ID：Opus 4.6, Sonnet 4.6, Haiku 4.5。

**`isReplModeEnabled()`**——改变工具使用指导（隐藏仅 REPL 可用工具）。

---

### systemPromptSections.ts——System Prompt Section 工厂

```typescript
systemPromptSection(name, compute)
  → 缓存，计算一次，缓存到 /clear 或 /compact

DANGEROUS_uncachedSystemPromptSection(name, compute, reason)
  → 每轮重新计算，标记 cacheBreak: true 打破 prompt cache
```

`resolveSystemPromptSections()`——评估所有 section，支持单 section 缓存。`clearSystemPromptSections()`——也清除 beta header latches。

---

---

### 元

问题：**这一站真正想解决的架构问题是什么？**

回答：这一站的核心是把“发给模型的提示词”和“验证这套提示词的测试装置”一起收束。VCR 夹具、工具结果上限、系统提示组装和 section 级缓存，说明它在处理提示内容的可测性、体积边界与缓存稳定性。

### 反

问题：**如果把这一站的设计反过来，会发生什么？**

回答：如果没有这一层，提示模板、测试录制、动态 section 和工具预算会分散在各模块里，各自对字节变化敏感却没人总负责。最先出问题的会是 prompt cache 和回归测试，因为这些东西最怕“看起来没变，实际上已经变了”。

### 空

问题：**跳出当前文件名，这一站背后更大的问题是什么？**

回答：更大的问题是，大型 AI 应用如何把系统提示从神秘字符串变成可测试、可组合、可缓存的工程对象。这里讨论的是 prompt engineering 的产品化基础设施。

## 第 172 站：剩余服务（Magic Docs, Stats, OverlayContext, MCP Server Approval）

### Magic Docs——自动文档更新

**自动更新带有 `# MAGIC DOC: [title]` 头的 Markdown 文件**。

**架构**：
- 用 regex 检测魔术文档头
- `Map` 追踪文件
- 注册 `FileReadListener` 在读取时检测
- `registerPostSamplingHook` 后采样 hook 顺序更新所有追踪文档
- 用 `runAgent()` + 受限工具（仅 `FileEditTool`）
- 克隆文���状态缓存做隔离

**设计哲学**——简洁写作、概览非代码巡视、WHY 优于 WHAT。支持自定义提示词 `~/.claude/magic-docs/prompt.md`。

---

### Stats Context——UI 遥测商店

**Counter/Gauge/Histogram/Set 四种指标类型**。
- Reservoir sampling（Algorithm R, 大小 1024）近似百分位
- `percentile(sorted, p)`——线性插值百分位计算
- `getAll()` 展平为 `Record<string, number>`：原始计数器、直方图统计（count/min/max/avg/p50/p95/p99）、集合大小
- 进程退出时持久化到项目配置 `lastSessionMetrics`，用于跨会话对比
- 每类型专用 hook（`useCounter`, `useGauge`, `useTimer`, `useSet`）返回 memoized 回调——组件永不因这些重渲染

---

### 元

问题：**这一站真正想解决的架构问题是什么？**

回答：这一站把 Magic Docs、Stats、OverlayContext 与 MCP 批准等剩余能力并到一起，重点不在“剩余”而在这些服务都负责补足主系统的长期运维感。自动文档更新和指标存储，都在把 Claude 的工作从瞬时响应延伸成持续维护。

### 反

问题：**如果把这一站的设计反过来，会发生什么？**

回答：如果缺少这层，像 MAGIC DOC 这样的延迟更新能力会混入普通文件流程，指标采样也会和 UI 渲染逻辑纠缠。最后这些看似辅助的能力会成为最难定位、最难测试的隐性副作用。

### 空

问题：**跳出当前文件名，这一站背后更大的问题是什么？**

回答：更大的问题是，系统怎样把“使用时响应”扩展为“使用后继续整理和观测”。这里谈的不是零散工具，而是 Claude Code 的后台维护能力如何被制度化。

### 读完后应抓住的 3 个事实

1. **Stop Hooks 的异步生成器模式**——`handleStopHooks()` 是异步生成器。这允许它在 hook 流式产出进度消息时增量地将它们 yield 回主循环。Hook 并行运行，每 hook 的持续时间匹配命令和首个未分配条目。这是生产设计——并行 hook 的时序分析需要精确的归属。

2. **Token Budget 的 Diminishing Returns 检测**——当连续 3 次继续 AND 最近增量都 < 500 tokens 时停止。这防止模型在几乎完成时浪费产出微小增量。90% 预算阈值作为硬限制。这是基于成本的决策。

3. **System Prompt 的两级缓存**——`systemPromptSection` 缓存到 /clear 或 /compact。`DANGEROUS_uncachedSystemPromptSection` 每轮重新计算，标记 `cacheBreak: true` 打破 prompt cache。这是性能与安全的权衡——大多数 section 缓存，只有动态变化的不安全 section 每轮重新计算。
