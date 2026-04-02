## 第 150 站：Forked Agent 与 Bootstrap 状态（2 个文件，~3K）

### 这是什么

Forked Agent 是 Claude Code 的同进程子代理引擎——在同一个 Node.js 进程中运行独立的 agent loop，同时复用父进程的 prompt cache 以节省成本。Bootstrap state 是全局单例状态。

---

### forkedAgent.ts —— 同进程子代理引擎

**核心问题**：如何生成一个子代理 loop，完全隔离上下文，但共享父进程的 prompt cache？

**CacheSafeParams**——prompt cache 共享的五个字段必须完全相同：
```ts
type CacheSafeParams = {
  systemPrompt, userContext, systemContext,
  toolUseContext, forkContextMessages
}
```

> Anthropic API 的 cache key 由 system prompt、tools、model、message prefix、thinking config 组成。任何不同都导致 cache miss。

**`runForkedAgent()` 流程**：
```
1. 解构 cacheSafeParams
2. createSubagentContext()——创建隔离 ToolUseContext
3. 构建 initialMessages = forkContextMessages + promptMessages
4. fork-specific agentId（如 'session_memory'）
5. 调用共享 query() 函数
6. 积累 totalUsage（从 message_delta usage）
7. 增量记录 transcript
8. finally: 清除 readFileState + 截断 messages
9. 记录 tengu_fork_agent_query 分析事件
```

**`createSubagentContext()`——隔离引擎**：

默认**完全隔离**：
- `setAppState` → 空操作 `() => {}`（子代理的状态变更不影响父进程）
- `abortController` → 新 controller，链接到父进程（父进程 abort 传播到子进程）
- `getAppState` → 包装为 `shouldAvoidPermissionPrompts: true`（后台代理不应请求权限）
- `readFileState` → 克隆（不是新建！保留父进程的 tool-use UUID 决策以维持 cache 稳定）
- `queryTracking` → 新 UUID，`depth` 从父进程 +1

**选择共享标志**：
- `shareSetAppState`——允许写入父进程 AppState
- `shareAbortController`——共享 abort（交互式代理需要）
- `shareSetResponseLength`——共享 UI 响应长度追踪

**关键隔离细节**：
- `setAppStateForTasks`——**始终**到达根 store（防止后台 bash 任务的 PPID=1 僵尸进程）
- `localDenialTracking`——每个异步子代理独立的拒绝计数状态
- `contentReplacementState`——从父进程克隆（关键——关于哪些工具结果被克隆的决策产生相同的 message prefix）
- UI 回调全部 `undefined`（子代理不能控制父进程 UI）
- 新空集合：`nestedMemoryAttachmentTriggers`、`loadedNestedMemoryPaths`、`dynamicSkillDirTriggers`、`discoveredSkillNames`

---

### 调用方

| 用途 | forkLabel | 特点 |
|---|---|---|
| Session memory 提取 | `session_memory` | 仅允许 FILE_EDIT_TOOL |
| 对话压缩 | - | forked agent 摘要上下文 |
| Post-stop hooks | - | `/btw`、prompt 建议、post-turn 摘要 |
| Skill/命令 | 从 command.agent 解析 | 带 allowed-tools 白名单 |
| AgentTool | - | 子代理委派 |

**`lastCacheSafeParams` 模块槽**——Post-stop hooks 使用模块级 `lastCacheSafeParams` 槽而不通过参数传递。这是 hook 注册和执行之间的侧通道。

---

### bootstrap/state.ts —— 全局会话状态

**State 类型 ~200 字段**。DAG 叶子——导入很少，其他所有东西都依赖它。

注释：`DO NOT ADD MORE STATE HERE - BE JUDICIOUS WITH GLOBAL STATE`

**关键状态组**：

| 组 | 示例 |
|---|---|
| 目录/身份 | `originalCwd`, `projectRoot`, `cwd` |
| 会话 ID | `sessionId`（UUID, `/clear` 时重新生成）, `parentSessionId` |
| 成本/会计 | `totalCostUSD`, `totalAPIDuration`, per-model `modelUsage` |
| 模型状态 | `mainLoopModelOverride`, beta header latches |
| Telemetry | OpenTelemetry providers, counters |
| 认证 | `sessionIngressToken`, `oauthTokenFromFd` |
| Feature flags | `kairosActive`, `promptCache1hAllowlist` |
| 生命周期 | `sessionBypassPermissionsMode`, `scheduledTasksEnabled` |
| Hook 注册 | `registeredHooks` |

**关键模式**：

1. **会话切换原子性**：`switchSession()` 同时更改 `sessionId` 和 `sessionProjectDir`，防止漂移。触发 `sessionSwitched` 信号。

2. **Sticky-on header latches**——Beta 功能头（AFK mode, fast mode, cache editing, thinking-clear）使用 `boolean | null` 三态——`null`="未评估"，`true/false`=锁定。触发后保持不变，防止中途切换 bust cache。

3. **Turn-level 预算**：`snapshotOutputTokensForTurn()` 捕获 turn 开始的 output token 基线，`getTurnOutputTokens()` 计算每 turn 消费。

4. **延迟交互时间戳**：`updateLastInteractionTime()` 设脏标志，Ink 渲染周期一次刷新（避免每次按键都调用 Date.now()）。

5. **`SessionCronTask`**——内存中 cron 任务（不持久化到磁盘）：
```ts
{ id, cron, prompt, createdAt, recurring?, agentId? }
```
`agentId` 路由到 teammate 而非主 REPL。

---

### 读完后应抓住的 4 个事实

1. **Cache 共享需要五字段匹配**——`CacheSafeParams`（systemPrompt/userContext/systemContext/toolUseContext/forkContextMessages）必须在父子请求间完全相同。任何不同都导致 cache miss。在 fork 上设置 `maxOutputTokens` 会被警告——它改变 `budget_tokens` 从而改变 thinking config。

2. **隔离-默认，选择共享**——所有可变回调默认隔离（`setAppState` → no-op）。只有三个 opt-in 标志可以共享（`shareSetAppState`、`shareAbortController`、`shareSetResponseLength`）。这是安全设计：子代理不会意外影响父进程。

3. **readFileState 是克隆不是新建**——子代理继承父进程已读文件状态。这很关键：如果子代理独立判断某文件已读，会产生不同的 UUID 决策，破坏 cache 稳定性。

4. **Sticky-on latches 防止 cache bust**——Beta header latches（thinking-clear、fast mode 等）一旦设为 `true` 就锁定——中 session 切换不会 bust cache。`/clear` 和 `/compact` 时 `clearBetaHeaderLatches()` 重置所有为 `null`。

---

## 第 151 站：Schema 系统（1 个文件）

### schemas/hooks.ts —— Hook 配置 Schema

**唯一目的**——打破 `settings/types.ts` ↔ `plugins/schemas.ts` 的循环依赖。

**`lazySchema` 模式**——延迟 schema 构建直到首次使用，允许 schema 互相引用而不用循环求值：
```ts
const HookCommandSchema = lazySchema(() =>
  z.discriminatedUnion('type', [
    BashCommandHookSchema,
    PromptHookSchema,
    AgentHookSchema,
    HttpHookSchema,
  ])
)
```

**四层嵌套**：
```
HooksSchema (top-level: event → matcher[])
  └─ HookMatcherSchema (matcher: string, hooks: HookCommandSchema[])
      └─ HookCommandSchema (4 types discriminated union)
          └─ Individual schemas (Bash/Prompt/Agent/Http + optional `if` field)
```

**安全模式**：
- HTTP headers 只允许 `allowedEnvVars` 白名单中的变量插值——防止任意 env var 泄露
- AgentHook 默认 Haiku 模型——低成本验证器
- 注释警告不要在 hook schema 中添加 `.transform()`——之前的 bug（gh-24920）中 transform 在 round-trip 中丢弃了用户的 prompt。

**24+ Hook 事件**——从 `HOOK_EVENTS` 枚举定义，覆盖 PreToolUse 到 InstructionsLoaded 的全生命周期。

**三层系统**：
1. 持久化层：`schemas/hooks.ts`（可 JSON 化的 schema）
2. 验证层：`settings/types.ts`、`plugins/schemas.ts`（消费 schema）
3. 运行时层：`types/hooks.ts`（回调、进度等内存类型）

---

### 读完后应抓住的 2 个事实

1. **lazySchema 打破循环依赖**——Zod discriminated unions 需要完整 schema 求值。如果两个模块互相引用 schema，循环求值会失败。lazySchema 延迟到首次使用才求值。

2. **HTTP hook 的 env var 白名单**——不是所有环境变量都可插值——只有 `allowedEnvVars` 中声明的变量才解析。这是安全设计：防止 hook 配置通过环境变量泄露敏感信息。
