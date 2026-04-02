## 第 185 站：Collapse/Streaming UI 优化工具

### streamlinedTransform.ts——流式精简转换

**目的**——将 SDK 消息转换为"抗稀释"精简输出格式。保留文本消息，用累积计数总结工具调用，省略思考内容。

**`ToolCounts`**——`{ searches, reads, writes, commands, other }` 四分类。`SEARCH_TOOLS`, `READ_TOOLS`, `WRITE_TOOLS`, `COMMAND_TOOLS` 四个常量数组。

**工厂/闭包模式**——`createStreamlinedTransformer()` 返回有状态闭包，在文本消息间累积工具计数，遇到文本时重置累加器。

---

### collapseHookSummaries.ts——Hook 摘要合并

合并连续相同 `hookLabel` 的 hook 摘要消息。例如并行工具调用各自发射 hook 摘要时合并为一。

**合并逻辑**：
- `hookCount` 求和
- `hookInfos` 和 `hookErrors` 平展合并
- `preventedContinuation` 和 `hasOutput` 取 `some()`
- `totalDurationMs` 取 `Math.max()`（并行 hook 墙钟时间重叠）

---

### collapseTeammateShutdowns.ts——Teammate 关闭合并

连续 "in-process teammate shutdown" 消息合并为单个 `teammate_shutdown_batch` 附件，带计数。

---

### collapseBackgroundBashNotifications.ts——后台 Bash 通知合并

连续完成的后台 bash 任务通知合并为单个 "N background commands completed" 通知。合成 XML 格式消息：
```xml
<task_notification><status>completed</status><summary>N background commands completed</summary></task_notification>
```

全屏模式且非 verbose 模式时才激活。

---

### collapseReadSearch.ts——读取/搜索操作合并（最复杂，~600 行）

**目的**——合并连续的 Read/Search 操作（Grep/Glob/Read/Bash 搜索/读取/REPL/内存文件操作/MCP 调用/非搜索 Bash 命令）为摘要组。

**`GroupAccumulator`**——大型累加对象，追踪：
- 搜索计数、读取文件路径 Set、读取操作计数
- 内存搜索/读取/写入计数（分离普通内存和团队内存）
- MCP 调用计数、MCP 服务器名称
- Bash 计数、Bash 命令 Map
- Git 操作追踪（commits、pushes、branches、PRs）
- Hook 总时长、Hook 计数、Hook 信息

**主要算法** `collapseReadSearchGroups()`——迭代消息，将可合并的工具使用和结果积累到 `GroupAccumulator`。遇到以下情况时刷新（发射）组：助手文本、不可合并工具使用、或带不可合并结果的用户消息。

**Git 操作浮现**——Bash 非搜索命令的结果被 `scanBashResultForGitOps()` 解析，检测 git commit SHA、PR URL、分支操作、推送事件，浮现到合并摘要中。

**延迟可跳过消息**——可跳过消息（thinking/system）收集到 `deferredSkippable` 中，在合并组后发射，保持视觉顺序。

---

### contextAnalysis.ts——上下文分析

**目的**——分析消息数组产出 token 使用统计，按消息类型和工具分类。特别追踪重复文件读取以获取优化洞察。

**`TokenStats`**——toolRequests/Results Map、human/assistant 消息计数、attachments Map、**重复文件读取** Map（count + tokens）、total。

`tokenStatsToStatsigMetrics()`——转换为扁平指标记录，含百分比分解（如 `human_message_percent`, `tool_request_Read_percent`, `duplicate_read_percent`）。

---

### 读完后应抓住的 2 个事实

1. **Run-length encoding 合并模式**——所有 collapse 工具（hook summaries/teammate shutdowns/bash notifications/read-search）都使用同一次线性扫描的 run-length 分组模式：收集连续匹配项，发射聚合结果。这是经典压缩——减少 UI 视觉噪音同时将结构化数据聚合为可显示摘要。

2. **Git 操作浮现**——collapseReadSearch 不仅合并读取/搜索，还解析 Bash 结果中的 git 操作（commits/PRs/branches/pushes），使它们出现在合并摘要中。这是 UX 设计——用户看到 "committed, pushed, opened PR" 而不是 "ran 3 bash commands"。

---

## 第 186 站：消息队列、活动管理、错误日志

### messageQueueManager.ts——统一优先级命令队列

**所有命令**——用户输入、任务通知、孤立权限——都通过这个队列。**三优先级**——`now: 0`, `next: 1`, `later: 2`。

**关键操作**：
- `enqueue(command)`——默认 `'next'` 优先级
- `dequeue(filter?)`——找到并移除最高优先级命令，同优先级内保 FIFO
- `peek(filter?)`——返回最高优先级不移除
- `popAllEditable()`——提取所有可编辑命令，与当前输入缓冲组合

**设计模式**：
- **模块即单例**——队列状态在模块作用域，避免显式单例类
- **发布/订阅**——`createSignal()` + 冻结快照供 React `useSyncExternalStore` 使用
- **可过滤操作**——dequeue 和 peek 接受可选谓词进行选择处理（如 SDK 仅排空主线程命令）
- **向后兼容别名**——已弃用的 `*PendingNotifications` 包装器为迁移兼容保留

### activityManager.ts——活动追踪

**目的**——分别追踪用户和 CLI 活动时间，自动去重。用于测量活跃/活跃-空闲会话时间。

**`startCLIActivity(operationId)` / `endCLIActivity(operationId)`**——管理活跃 CLI 操作的 Set。空→非空转换时激活 CLI；回到空时记录经过的 CLI 时间。

**优先级层次**——CLI 活动抑制用户活动记录，防止重复计数。

**容错清理**——如果操作已在（表明先前清理失败），先强制清理，宁愿低估也不高估时间。

### errorLogSink.ts——错误日志

**基于文件的错误日志后端**。写入 JSONL 文件到磁盘。延迟初始化直到应用启动完成。

**设计**：
- 每路径延迟初始化日志文件写者
- 使用 `createBufferedWriter()` 带 1 秒刷新间隔和 50 项缓冲区
- 自动恢复 mkdir——如果追加失败（可能目录不存在），创建目录并重试
- **关注点分离**——`log.ts` 是轻量接口和事件队列（避免导入循环）

---

## 第 187 站：Profiling 性能分析系统

### profilerBase.ts——分析基础设施

**共享基础设施**——提供 `getPerformance()`（懒加载 `perf_hooks.performance`）、`formatMs()`、`formatTimelineLine()`。所有分析器使用相同的行格式。

### startupProfiler.ts——启动分析

**测量 CLI 启动初始化阶段的时间**。两种模式：采样 Statsig 日志和详细文件分析。

**阶段定义**——`import_time`, `init_time`, `settings_time`, `total_time`

**采样分析**——决策在模块加载时一次做出；非采样用户零开销。100% Ant 用户，0.5% 外部用户。

**两阶段分析**——轻量采样日志面向所有人；详细文件+内存分析通过 `CLAUDE_CODE_PROFILE_STARTUP` env var 启用。

**阵列优于 Map**——使用与标记平行的阵列因为某些检查点触发多次且 Map 会覆盖。

### headlessProfiler.ts——无头模式分析

**每轮延迟分析**——用于无头/非交互模式（`-p` / print 模式）。测量：

| 阶段 | 含义 |
|------|------|
| `time_to_system_message_ms` | 到系统消息（仅第 0 轮） |
| `time_to_query_start_ms` | 到查询开始 |
| `time_to_first_response_ms` | 到首次响应（TTFT） |
| `query_overhead_ms` | 查询开销 |

命名空间前缀 `headless_`——与其他分析器在共享 `perf_hooks` 时间线上隔离。

---

## 第 188 站：Heatmap + Code Indexing + Session Activity + Teleport

### heatmap.ts——贡献热力图

**GitHub 风格贡献热力图**——7 行（星期）× N 周网格（最多 52 周），月标签、日标签（Mon/Wed/Fri）、图例。

**分位数强度**——5 级（0-4）基于 p25/p50/p75。自适应到用户特定活动分布而非绝对值。

**ANSI 着色**——`chalk.hex('#da7756')`（Claude 橙色）用于热力图字符：灰色 `.` 表示 0，橙色 `░▒▓█` 表示 1-4。

### codeIndexing.ts——代码索引检测

**检测 27 种代码索引/搜索工具**——Sourcegraph、Cody、Aider、Cursor 等——从 bash 命令和 MCP 工具调用。

**CLI 命令**——精确字符串键查找 O(1)，处理 `npx`/`bunx` 前缀命令。

**MCP 服务器**——正则模式数组，灵活匹配（处理连字符、下划线、不区分大小写）。

分类：代码搜索引擎、AI 编码助手、MCP 代码索引服务器、上下文提供者。

---

### 读完后应抓住的 2 个事实

1. **三 Profiler 统一架构**——`profilerBase.ts` 提供共享格式约定，所有三个分析器（`startupProfiler`、`headlessProfiler`、`queryProfiler`）使用相同的时间线格式。这产生跨阶段的可比度量。每种使用 `perf_hooks` 命名空间前缀隔离。

2. **Heatmap 的分位数自适应**——不是绝对阈值（如 "10 消息 = 高"），热力图使用用户自身活动分布的 p25/p50/p75 分位数。这使轻度和重度用户的可视化都有意义——强度是相对的，不是绝对的。
