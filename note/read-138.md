## 第 173 站：Memdir 内存目录系统

### 这是什么

Claude 基于文件的持久化内存系统——路径解析、内存类型分类、目录扫描、相关性选择。

---

### paths.ts——路径解析引擎

**`getAutoMemPath()`**——memoized 优先级链：
1. `CLAUDE_COWORK_MEMORY_PATH_OVERRIDE` env（完整路径覆盖）
2. settings.json 中的 `autoMemoryDirectory`（仅信任来源：policy/local/user，排除 projectSettings 防止恶意仓库写访问）
3. `<memoryBase>/projects/<sanitized-git-root>/memory/`（默认）

**安全关键**——`validateMemoryPath()` 拒绝相对路径、根路径、Windows 驱动根、UNC 路径和空字节。`isAutoMemPath()` 提供文件系统写入 carve-out 的包含检查。

---

### memoryTypes.ts——封闭四类型分类

`user`, `feedback`, `project`, `reference`——每种类型有显式 `<scope>`, `<description>`, `<when_to_save>`, `<how_to_use>`, `<examples>` 指导直接嵌入提示文本。

**`WHAT_NOT_TO_SAVE_SECTION`**——代码模式、git 历史、调试方案、CLAUDE.md 内容、临时状态。

**`TRUSTING_RECALL_SECTION`**——eval 验证的指导：从内存推荐前，验证文件/函数/标志是否存在。

---

### teamMemPaths.ts——团队内存路径

团队内存位于 `<autoMemPath>/team/`。

**`validateTeamMemWritePath()`**——两阶段验证：
1. 解析 `..` 段
2. 在最深现有祖先上解析符号链接捕获符号链接逃逸（PSR M22186）

`realpathDeepestExisting()`——从不存在的文件路径向上遍历找到最深现有祖先。

---

### findRelevantMemories.ts——相关性选择

用 `sideQuery()` + Sonnet 从 manifest 选择最多 5 个相关内存。选择器提示明确排除工具已在使用时的工具引用内存（但包含警告/陷阱）。支持 `alreadySurfaced` 过滤跨轮重新预算选择。触发 recall-shape 遥测事件。

---

### memdir.ts——核心提示构建器

```
ENTRYPOINT_NAME = 'MEMORY.md'
MAX_ENTRYPOINT_LINES = 200
MAX_ENTRYPOINT_BYTES = 25000
```

**`truncateEntrypointContent()`**——两阶段截断（先行再字节，到最后换行符）。

**`loadMemoryPrompt()`**——主入口点，分派：
- KAIROS 日志日模式
- TEAMMEM 组合模式
- 仅 auto 模式
- null（已禁用）

---

### 读完后应抓住的 2 个事实

1. **MEMORY.md 是索引而非存储**——MEMORY.md 包含指针/索引，实际记忆存储在单独的文件中。这保持索引小而快，允许独立更新单个记忆文件。

2. **团队内存的两阶段符号链接验证**——不仅字符串规范化路径。两阶段：解析 `..` 后再解析最深现有祖先上的符号链接。这防止通过符号链接链的目录逃逸攻击（PSR M22186）。

---

## 第 174 站：Task 实现系统

### 核心架构

**多态任务注册表**——每种任务类型实现 `Task` 接口（name, type, kill 函数）。任务是受 `AppState` 追踪的状态机。

---

### 7 种任务类型

| 类型 | 实现 | 特点 |
|------|------|------|
| **LocalShellTask** | `LocalShellTask.tsx` | Stall watchdog（5s 检查，45s 阈值）检测 shell 输出停止 + 看起来像交互提示 |
| **LocalAgentTask** | `LocalAgentTask.tsx` | 后台代理，追踪 input/output token 分离、工具计数、最近 5 个活动 |
| **RemoteAgentTask** | `RemoteAgentTask.tsx` | 云代理，追踪远程会话 ID、轮询、ultrareview 进度、ultraplan 阶段 |
| **InProcessTeammateTask** | `InProcessTeammateTask.tsx` | 同进程 teammate， capped messages（50 条，~20MB RSS），独立权限模式 |
| **LocalWorkflowTask** | 工作流任务 | - |
| **MonitorMcpTask** | MCP 监控 | - |
| **DreamTask** | `DreamTask.ts` | 后台内存整合代理，最多 30 轮显示，kill 时回滚整合锁 mtime |

**`LocalMainSessionTask`**——后台主会话。支持 Ctrl+B 双击后台，前台化（`foregroundMainSessionTask()`），独立查询执行 + 隔离 transcript。

### 设计模式

**React-free 核心**——类型和 kill 逻辑被提取以避免 React 进入依赖图。`LocalShellTask/guards.ts` 是纯类型 + 类型守卫，提取后非 React 消费者不拉 React。

**不可变状态转换**——所有任务状态变更使用 `updateTaskState()`，通过 AppState setter 模式原子执行。

**SDK 事件对称性**——任务生命周期事件独立发射 SDK 事件，与 XML 通知对称。

**`stopTask.ts`**——集中 kill 入口。按 ID 查找，验证运行状态，分派到 `taskImpl.kill()`，发射 SDK `task_terminated` 事件。LocalShellTask 特殊处理：抑制 "exit code 137" 噪音通知。

**PillLabel UI**——`pillLabel.ts` 生成后台任务药丸标签：
- `local_bash`：区分 "shell" vs "monitor" 计数
- `in_process_teammate`：显示团队计数（去重）
- `remote_agent`：显示钻石图标 + ultraplan 阶段

---

## 第 175 站：Hook 实现系统

### toolPermission/——四路径权限解析

**`createPermissionContext()`**——构建冻结上下文对象：
- `logDecision()` / `logCancelled()`——分析
- `persistPermissions()`——持久化权限到设置
- `tryClassifier()`——bash 分类器自动批准
- `runHooks()`——执行权限请求 hook
- `buildAllow()` / `buildDeny()`——决策构造器
- `pushToQueue()` / `removeFromQueue()` / `updateQueueItem()`——React 队列桥接

**`createResolveOnce()`**——原子 claim-and-resolve 语义，防止并发解析路径间的竞态。

---

### permissionLogging.ts——集中分析扇出

`logPermissionDecision()` 发射：
- 每源独立分析事件（config/classifier/hook/user permanent/temporary）
- OTel `tool_decision` 事件
- 代码编辑 OTel 计数器（每语言）
- 决策存储到上下文

源类型：`hook`, `user`（permanent/temporary）, `classifier`, `user_abort`, `user_reject`

---

### handlers/——三种权限处理路径

**`coordinatorHandler.ts`**——顺序自动化检查：先 hook（快、本地），然后 classifier（慢、仅 bash）。决策返回或 null 降级。

**`swarmWorkerHandler.ts`**——Swarm worker 权限流。尝试分类器自动批准，然后通过 mailbox 转发到 leader。**在发送请求之前注册回调**以避免竞态。监听 abort 信号以 cancel 决策解析防止 worker 挂起。

**`interactiveHandler.ts`**——主代理交互流。推入 `ToolUseConfirm` 到确认队列，带回调：`onAbort`, `onAllow`, `onReject`, `recheckPermission`, `onUserInteraction`。**竞态四个解析路径**：
1. Bridge 权限（来自 claude.ai Web UI）
2. Channel 权限中继（Telegram, iMessage 等）
3. 权限 hook（异步后台）
4. Bash 分类器（异步后台）

**`claim()` 原子守卫**——所有路径使用 claim() 防止多重解析。

UX 细节：
- 勾选框转换计时器（3s 聚焦，1s 未聚焦）
- 用户交互宽限期（200ms）
- 分类器指示器消动画

---

### useCanUseTool——编排器

React 编译器优化的 hook，产出 `CanUseToolFn`：
```
1. 检查是否 abort
2. hasPermissionsToUseTool()（config allow/deny 列表）
   - allow + classifier: 设置 classifier 批准，解析
   - deny: 记录拒绝，处理 auto-mode 拒绝，解析
3. ask:
   - Coordinator 路径: handleCoordinatorPermission()
   - Swarm worker 路径: handleSwarmWorkerPermission()
   - Interactive 路径: handleInteractivePermission()（推入队列，竞态路径）
   - speculative classifier peek for bash（awaitAutomatedChecksBeforeDialog === false）
```

---

### useTextInput.ts——核心文本输入引擎

Emacs 风格 readline：游标管理、kill ring（yank/yank-pop）、多行编辑、内联 ghost text、图片粘贴、输入模式过滤、修饰键预热（Apple Terminal 兼容）。

---

### useReplBridge.tsx——Bridge 连接管理

**连续故障熔断器**——最多 3 次连续失败。

**入站消息注入**——带文件附件处理。

**持续化**——助理模式崩溃恢复通过 bridge-pointer.json。

**出站消息流式**——UUID 去重。

---

### useTaskListWatcher.ts——外部任务观察

观察任务列表目录的外部创建任务。**用 refs 稳定不稳定的 props**（防止 Bun PathWatcherManager 死锁）。1000ms 文件系统观察防抖。原子认领任务，格式化为提示，提交失败时释放。

---

### fileSuggestions.ts——文件索引

Rust `FileIndex`（nucleo/fuzzy matcher）驱动的文件索引。单例懒构建。追踪：缓存文件、配置文件、目录、忽略模式、git 索引 mtime、已加载签名以避免不必要重建。

---

## 第 176 站：Plugins 和剩余系统

### builtinPlugins.ts——内置插件注册表

`@builtin` 后缀区分于市场插件。启用状态解析：`user_setting > defaultEnabled`（默认：true）。

插件结构：skills, hooks config, MCP servers。

**`getBuiltinPlugins()`**——根据用户设置返回启用/禁用拆分，`defaultEnabled` 回退。

---

### screens/——屏幕组件

| 屏幕 | 用途 |
|------|------|
| **Doctor.tsx** | 诊断/故障排除屏幕——插件健康、系统状态、错误摘要 |
| **REPL.tsx** | 主终端 UI——完整 REPL 循环、消息显示、权限对话框 |
| **ResumeConversation.tsx** | 会话恢复——恢复被中断的对话，重连活跃会话 |

---

## 第 177 站：最终总结

至此，我们已从 `/Users/jinjun/study/Claude-Code-Reverse/src/` 读取并记录了 **177 个架构站点**，覆盖 **137 篇文档**（read.md → read-137）。

### 已覆盖的完整子系统列表

| 站点范围 | 覆盖内容 |
|----------|----------|
| 1-27 | 启动入口、CLI 解析、主循环、命令系统 |
| 28-60 | 工具系统全览（所有内置工具实现） |
| 61-85 | 权限系统、Hook 系统、MCP、Plugin 系统 |
| 86-110 | 内存系统、Agent Team、Computer Use、Compact |
| 111-134 | Forked Agent、Bootstrap State、Schema、API 层、Settings Sync、Tips、Policy Limits |
| 135-137 | Bridge/Remote、UI/Ink、Utils、Voice、Diagnostics、Keybindings、VCR、Analytics、Memdir、Tasks |

### 核心架构模式总结

1. **Feature Flag 分层**——编译时 `feature()` (bun:bundle) + 运行时 GrowthBook + 环境变量。三级门控确保灵活性。
2. **Prompt Cache 意识**——几乎所有子系统都考虑缓存影响：CacheSafeParams 五字段匹配、deterministic 渲染防止 cache bust、sticky-on header latches。
3. **Fail-Open 默认**——API 失败功能保持、hook 失败静默、migration 失败不阻塞启动。HIPAA 场景有 fail-closed 例外。
4. **三优先级消息队列**——now/next/later 控制并发用户输入和系统事件。
5. **隔离-默认共享**——forked agent 隔离所有 mutable 回调，只有三个 opt-in 标志可共享。
6. **基线-差分检测**——diagnostic tracking (编辑前后)、toolSearch delta (工具变动)、mcpInstructions delta (服务器变动)。
7. **异步生成器模式**——query loop、stop hooks、retry engine、VCR streaming 全部用 AsyncGenerator 实现流式控制。
