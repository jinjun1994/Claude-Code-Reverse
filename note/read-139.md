## 第 178 站：Cron 定时任务系统

### 这是什么

会话级 cron 调度器——在会话空闲时定期触发 prompt 执行队列。

---

### 核心文件

| 文件 | 用途 |
|------|------|
| **cron.ts**（308 行） | Cron 表达式解析 + 匹配 |
| **cronJitterConfig.ts**（75 行） | 抖动配置 |
| **cronScheduler.ts**（565 行） | 调度引擎 |
| **cronTasks.ts**（458 行） | 任务管理 + 执行 |
| **cronTasksLock.ts**（195 行） | 任务执行锁 |

---

### cron.ts——Cron 表达式解析

将标准 5 字段 cron 表达式转换为匹配逻辑。支持 `*/N`、`N-M`、`N,M`、`N-M/S` 等格式。

---

### cronScheduler.ts——调度引擎

**核心循环**：
```
1. 从 AppStateStore 获取 cron 任务列表
2. 对每个任务，检查当前时间是否匹配 cron 表达式
3. 匹配 → 检查锁（防止重叠执行）
4. 获取锁 → 将任务推入消息队列
5. 释放锁（任务完成后）
```

**抖动**——使用确定性抖动防止多任务同时触发。从任务 ID 哈希派生偏移量。

---

### cronTasks.ts——任务执行

**执行流程**：
```
1. acquireLock(taskId) ——获取 PID 级锁
2. 将 prompt 推入消息队列（带 agentId 路由）
3. 等待任务完成（通过 inbox 监听）
4. releaseLock(taskId)
```

**锁文件设计**——PID 写入锁文件，死进程检测（PID 不存活 → 回收锁）。

---

## 第 179 站：Graceful Shutdown + Session Restore + AutoUpdater

### gracefulShutdown.ts（529 行）

**目的**——有序关闭 Claude Code 进程——清理资源、保存状态、发射分析事件。

**关键模式**：
- **引用计数**——`startPreventSleep()` 和 `stopPreventSleep()` 匹配
- **清理注册表**——`registerCleanup()` 注册清理回调，关闭时按顺序执行
- **SIGINT/SIGTERM 处理**——注册信号处理器，优雅关闭子进程
- **分析排放**——发射 `tengu_exit` 事件（成本、时长、工具使用统计）
- **超时保障**——如果优雅关闭在超时内未完成，强制进程退出

### sessionRestore.ts（551 行）

**目的**——恢复被中断的会话——从 transcript 文件重建消息历史。

**恢复流程**：
```
1. 扫描最近 transcript 文件，查找未完成的会话
2. 解析 JSONL transcript，重建消息数组
3. 检测工具调用是否完成（tool_use 后是否有匹配的 tool_result）
4. 丢弃不完整消息，保留完整历史
5. 可选从 plan 文件恢复计划文件快照
```

**Teleport 恢复**——跨项目恢复（`crossProjectResume.ts`），支持将会话从一个目录迁移到另一个，路径重新映射。

### sessionTitle.ts（129 行）

**目的**——自动为会话生成标题。调用 forked agent（Haiku 模型），传递最近的消息，生成 3-5 词的现在时标题。后台运行，不阻塞主循环。

### autoUpdater.ts（561 行）

**目的**——Claude Code CLI 自动更新系统。

**关键模式**：
- **更新检查**——使用 npm registry or 内部 API 检查新版本
- **安全验证**——校验下载包的完整性
- **用户通知**——通过 notifier.ts 发送可用更新通知
- **策略门控**——`DISABLE_AUTOUPDATER` env var、企业策略设置、订阅类型
- **后台下载**——更新包在后台下载，准备好后通知用户
- **迁移集成**——旧 `autoUpdates` 设置通过 `migrateAutoUpdatesToSettings` 迁移到 settings.json

---

### advisor.ts（145 行）

**目的**——成本顾问——在 token 使用接近模型预算阈值时给出警告。分析会话成本并在接近限制时建议截断或优化。

---

### sideQuery.ts（222 行）

**目的**——后台 API 查询——生成一个隔离的模型调用而不影响主对话。用于内存扫描、相关内存查找、分类器决策。

**关键设计**：
- 不调用 `query()`——直接调用 API 层
- 不写 transcript
- 不影响主会话缓存状态
- 超时保护（默认 30s）

---

## 第 180 站：Teleport 跨项目恢复（teleport.tsx，1225 行）

### 这是什么

Teleport = 跨项目会话恢复——将正在进行的会话从一个项目目录完全迁移到另一个。

---

### 核心功能

**`teleportSession()`**——主恢复流程：
```
1. 解析源 transcript（JSONL 格式）
2. 识别路径引用（文件路径、glob 模式、git 根）
3. 路径映射：源项目根 → 目标项目根
4. 重写 CLAUDE.md 引用，在目标中重新加载
5. 重建消息数组，跳过不完整的工具调用
6. 可选恢复 plan 文件快照
7. 创建新 transcript，启动新会话
```

**路径检测**——通过内容分析检测消息中引用的所有路径：
- 绝对路径（`/Users/.../project/...`）
- 相对于项目根的路径
- Git 子模块和 worktree 路径
- `@include` 指令中引用的路径

**路径映射策略**：
- 公共子路径识别（如两个项目有相同的 monorepo 结构）
- 无法映射的路径标记为"stale"（陈旧）
- CLAUDE.md 重新解析以匹配目标项目层次结构

**Plan 恢复**——从源 transcript 恢复 plan 文件快照（如果存在）。Plan 内容从工具使用块或附件重建。

---

### 读完后应抓住的 2 个事实

1. **Teleport 不是简单复制**——它不仅复制消息，还智能地映射路径、重新解析 CLAUDE.md 并标记无法匹配目标项目结构的路径为陈旧。这是生产级的恢复——用户可以从开源项目切换到企业仓库，保持上下文。

2. **路径映射是最佳努力的**——Teleport 通过内容分析检测消息中的路径引用。如果路径不存在于目标项目中，标记为"stale"。这是正确的权衡——不保证 100% 保真度，但比从头开始好得多。

---

## 第 181 站：Shell 执行引擎

### Shell.ts——中央 Shell 引擎

**Provider 模式**——`ShellProvider` 抽象接口，dispatch bash vs powershell。

**`getShellConfig()`**——memoized 单例，创建 `BashShellProvider`。优先级：`CLAUDE_CODE_SHELL` env > `$SHELL` env > `which` zsh/bash > 硬编码回退路径。

**两种输出模式**：
- **"File mode"**——直接 fd 写入，零 JS 开销
- **"Pipe mode"**——实时 stdout 回调通过 `onStdout`（hook、实时监控用）

**CWD 持久化**——Shell 通过 `pwd -P` 写入最终 CWD 到临时文件，完成后同步读回更新全局 CWD 状态。符号链接解析 + NFC 标准化（处理 macOS APFS NFD 路径）。

**TOCTOU 安全**——用 `realpath` 同步检查删除的 CWD 恢复，而非 existsSync。

### ShellCommand.ts——ShellCommand 对象

封装运行中的子进程为一等 `ShellCommand` 对象——生命周期管理：结果解析、kill、后台化、清理、超时。

**关键类**——`ShellCommandImpl`：
- Promise 结果解析通过双解析器（`#resultResolver`, `#exitCodeResolver`）
- 超时处理（默认 30 分钟）+ 自动后台化选项
- **大小看门狗**——后台任务每 5 秒轮询文件大小，超过 `MAX_TASK_OUTPUT_BYTES` 时 SIGKILL（防止磁盘填满——引用"768GB 事件"）
- `tree-kill` 杀整个进程树（SIGKILL 137, SIGTERM 143）
- 中断时特殊处理 `'interrupt'` 原因——允许模型看到部分输出而非直接 kill

---

## 第 182 站：文件缓存系统

### fileReadCache.ts——文件读取缓存

**内存内容缓存**，mtime 自动失效。消除 `FileEditTool` 操作中的冗余读取。

**关键设计**：
- Map 存储，最大 1000 条目
- FIFO 驱逐（溢出时删除第一个 key——最老插入顺序）
- 隐式失效：访问时通过 `statSync` mtime 比较检测过时
- 同步 I/O——`readFileSync` + `statSync`——对编辑路径有意为之

### fileStateCache.ts——文件状态缓存

**LRU 缓存**，路径标准化封装，具有条目数和字节大小双重限制。追踪模型已看到的文件状态。

```typescript
FileState: { content, timestamp, offset, limit, isPartialView? }
```

**关键设计**：
- `isPartialView` 标志——内容来自动态注入（CLAUDE.md 等），不匹配磁盘（由于剥离 HTML 注释/frontmatter/截断）。这强制 Edit/Write 先显式 Read。
- 双重大小驱逐：条目数（默认 100）+ 总字节（默认 25MB）
- 时间戳合并：合并两个缓存时较新条目胜出——子代理 fork/join 场景有用
- `cloneFileStateCache()`——通过 `dump()`/`load()` 深拷贝

---

## 第 183 站：MCP Output Storage + Tool Result Storage

### mcpOutputStorage.ts——MCP 输出持久化

**目的**——处理大型或二进制 MCP 工具结果到磁盘，防止消耗模型上下文窗口。

**关键函数**：
- `extensionForMimeType()`——MIME 到扩展名映射（PDF/JSON/CSV/HTML/Office 文档/音频/视频/图像）
- `persistBinaryContent()`——原始字节写入工具结果目录，发射 `tengu_binary_content_persisted` 分析
- `getLargeOutputInstructions()`——生成详细说明，告诉 Claude 分块读取（offset/limit）

### toolResultStorage.ts——工具结果存储

**全面的工具结果管理**——持久化超大工具结果，执行每消息聚合预算，通过替换状态追踪管理提示缓存稳定性。

**`ContentReplacementState`**——追踪每对话线程替换决策：
```typescript
{ seenIds: Set<string>, replacements: Map<string, string> }
```

**预算执行核心算法** `enforceToolResultBudget()`：
```
1. 将候选分区为三类：mustReapply / frozen / fresh
2. 选择最大的 fresh 结果来替换
3. 持久化它们到磁盘
4. 返回替换的消息
```

**关键设计模式**：
- **提示缓存稳定性**——一旦工具结果的命运确定（持久化与否），它被冻结。重新应用使用缓存的替换字符串（零 I/O，逐字节相同）。先前未替换的结果永远不会后来被替换。
- **三路分区**——mustReapply（先前替换的）、frozen（先前看到但保留的）、fresh（新的，可替换候选）
- **GrowthBook 集成**——Feature flags 控制持久化阈值和整个预算执行功能（`tengu_hawthorn_steeple` gate）
- **幂等写入**——使用 `wx` 标志（存在则失败）避免在 microcompact 回放期间每轮重写
- **`768GB 事件`**——大小看门狗在后台任务中防止磁盘填满——引用自真实生产事件

---

## 第 184 站：Tool Schema Cache + Session File Access Hooks + Query Guard

### toolSchemaCache.ts——工具 Schema 缓存

**会话级 memoization**——渲染的工具 schema。防止 mid-session GrowthBook flag 变更、MCP 重新连接、动态工具内容导致字节级 schema 变化破坏 prompt cache。

**设计**：模块级单例 Map。`auth.ts` 清除它而不导入 `api.ts`（避免循环依赖）。**首次渲染锁定**——schema 在首次渲染时缓存；mid-session GrowthBook 刷新不再改变 schema 字节。

### sessionFileAccessHooks.ts——会话文件访问钩子

**PostToolUse 分析钩子**——追踪对会话内存文件、transcript 文件和 memdir 文件的访问。用于观察 Claude 如何与自己的会话数据交互。

**检测**：
- 会话内存访问（`tengu_session_memory_accessed`）
- Transcript 访问（`tengu_transcript_accessed`）
- Memdir 文件访问（`tengu_memdir_accessed` + 细分读取/编辑/写入）
- 团队内存访问（`tengu_team_mem_accessed` + 写入时通知团队内存监视器）

### QueryGuard.ts——查询守卫

**同步三状态机**——防止并发/重叠查询执行。兼容 React `useSyncExternalStore`。

```
状态: idle → dispatching → running → idle
```

**关键方法**：
- `reserve()`——idle 到 dispatching。如果非 idle 返回 false。队列处理器在出列前使用。
- `tryStart()`——从 idle 或 dispatching 到 running。返回 generation 号，null 表示已在运行（并发守卫）。
- `end(generation)`——running 回到 idle，仅当 generation 匹配。返回这个调用者是否应该清理（false 表示新查询已启动）。
- `forceEnd()`——强制结束，不管 generation。递增 generation 使陈旧的 finally 块跳过清理。

**Generation 基于过时检测**——每次 `tryStart()` 递增 generation。`end()` 检查 generation 以检测来自取消查询的陈旧完成。

### sessionActivity.ts——会话活动追踪

**基于引用计数的心跳 keep-alive**——防止远程传输连接在长时间 API 调用和工具执行期间超时。

```
0→1 转换: 启动心跳计时器（每 30 秒）
>0: 心跳持续
1→0 转换: 停止心跳，启动 30 秒空闲计时器（记录 session_idle_30s）
```

**原因追踪**——`activeReasons` Map 追踪每原因计数用于诊断日志。功能门控发送仅当 `CLAUDE_CODE_REMOTE_SEND_KEEPALIVES` 环境变量为真时。在关闭时注册清理回调记录 refcount/活跃原因/活动年龄。

---

### 读完后应抓住的 3 个事实

1. **Tool Result 三路分区预算执行**——`enforceToolResultBudget()` 将工具结果分区为 mustReapply（先前替换的）、frozen（先前看到但保留的）、fresh（新的候选）。选择最大的 fresh 结果来持久化到磁盘。一旦结果命运决定，它被冻结——先前未替换的结果永远不会后来被替换。这是提示缓存稳定性设计。

2. **QueryGuard 三状态机**——idle → dispatching → running。中间 dispatching 状态防止队列处理器在异步间隙期间出列。Generation 号使 `end()` 检测陈旧完成——如果新查询已启动，旧查询的清理被跳过。

3. **ShellCommand 大小看门狗引用 768GB 事件**——后台任务每 5 秒轮询输出文件大小，超过 `MAX_TASK_OUTPUT_BYTES` 时 SIGKILL。这是来自真实生产事件的设计——一个失控的后台任务填满了 768GB 磁盘。
