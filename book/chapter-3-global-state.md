# 第 3 章：全局状态与启动装配

## 3.1 State 管理与循环依赖防护

`bootstrap/state.ts`（1,758 行）是整个 Claude Code 的全局状态管理核心。它的顶部注释是代码库中最严厉的纪律约束之一：

```typescript
// DO NOT ADD MORE STATE HERE - BE JUDICIOUS WITH GLOBAL STATE
// ...
// ALSO HERE - THINK THRICE BEFORE MODIFYING
// ...
// AND ESPECIALLY HERE
const STATE: State = getInitialState()
```

三行注释，分别位于类型定义、初始化和实例化之前。这不是装饰——是一个明确的架构信号：这个文件是导入 DAG（有向无环图）的叶子，任何新增都可能导致循环依赖。

---

### State 类型的结构设计

`State` 类型约 200 个字段，按语义可分几层：

**身份层**（project identity）：
- `originalCwd` / `projectRoot` / `cwd`——三种路径语义
- `sessionId` / `parentSessionId`——会话身份和链式恢复

**计量层**（telemetry & cost）：
- `totalCostUSD` / `totalAPIDuration` / `totalToolDuration`
- `turnToolCount` / `turnHookCount` / `turnClassifierCount`——Turn-scoped 计数器
- `modelUsage`——按模型名的 token/成本追踪

**遥测层**（OpenTelemetry）：
- `meter` / `sessionCounter` / `costCounter` / `tokenCounter`
- `loggerProvider` / `tracerProvider` / `meterProvider`

**会话控制层**（session lifecycle）：
- `isInteractive` / `kairosActive` / `clientType`
- `scheduledTasksEnabled` / `sessionPersistenceDisabled`
- `hasExitedPlanMode` / `lspRecommendationShownThisSession`

**Prompt 缓存控制层**（cache coherence）：
- `afkModeHeaderLatched` / `fastModeHeaderLatched` / `cacheEditingHeaderLatched` / `thinkingClearLatched`

---

### 为什么是 Sticky-on Latch

`State` 中的四个 `*Latched` 字段共享同一个设计模式：**sticky-on latch**。

```typescript
// State 类型定义
afkModeHeaderLatched: boolean | null,     // AFK 模式 β header
fastModeHeaderLatched: boolean | null,    // 快速模式 β header
cacheEditingHeaderLatched: boolean | null,// 缓存编辑 β header
thinkingClearLatched: boolean | null,     // 思考清除 β header
```

**问题**——这些 β header 控制 API 请求的 `cache_control` 行为（`ttl: 1h` vs 默认）。如果用户在会话中反复切换功能（如快速模式的冷却进入/退出），header 值的变化会导致 prompt cache miss，产生重复的 token 成本（~50-70K tokens 每次）。

**解决方案**——一旦功能首次激活，对应的 latch 设为 `true`，header 持续发送。即使功能被关闭，header 不撤回——cache 不会因为 feature 状态振荡而失效。

```
null → 功能未激活 → 不发送 header
true → 功能已激活 → 始终发送 header（即使功能已关闭）
```

这是一个缓存一致性（cache coherence）的工程实践——用内存中的 latch 维护远端 API cache 的稳定性。

---

### 信号（Signal）发布/订阅系统

`bootstrap/state.ts` 通过 `createSignal()` 提供了 9 个事件信号：

```typescript
// signal.ts - 极简发布/订阅原语
export function createSignal<Args extends unknown[] = []>(): Signal<Args> {
  const listeners = new Set<(...args: Args) => void>()
  return {
    subscribe(listener) {
      listeners.add(listener)
      return () => { listeners.delete(listener) }  // 取消订阅
    },
    emit(...args) {
      for (const listener of listeners) listener(...args)
    },
    clear() { for reset }
  }
}
```

**为什么不用事件发射器**——Node.js 的 `EventEmitter` 有内存泄漏风险（未取消订阅的 listener 永远不 GC）、类型不够精确（`any` 事件参数）。`createSignal` 是一个 15 行的纯函数实现，提供类型安全的 `Args extends unknown[]`。

可用的信号包括：

| 信号 | 触发场景 | 订阅者 |
|------|---------|-------|
| `onSessionSwitch` | `switchSession()` | `concurrentSessions.ts`（PID 文件同步） |
| `onSessionStateChanged` | 会话状态变更 | UI 组件 |
| `onProjectRootUpdate` | 项目根更新 | Skills/历史系统 |
| `onLastInteractionTime` | 用户交互 | 超时检测 |
| `onPermissionModeChanged` | 权限模式变更 | 权限 UI |
| `onStatsStoreAvailable` | 遥测就绪 | 统计组件 |

---

### 循环依赖防护：依赖注入

`bootstrap/state.ts` 是导入图的叶子——所有模块几乎都导入它。因此它不能导入任何业务模块。对于需要回调的场景，使用依赖注入：

```typescript
// bootstrap/state.ts:15-17
// Indirection for browser-sdk build (package.json "browser" field swaps
// crypto.ts for crypto.browser.ts). Pure leaf re-export of node:crypto —
// zero circular-dep risk.
import { randomUUID } from 'src/utils/crypto.js'
```

`crypto` 是一个 leaf——只 re-export `node:crypto`，不导入任何项目模块。这是 bootstrap-isolation 规则的例外，明确记录。

---

### Forked Agent 隔离

`State` 支持 forked agent 的隔离。有三个 opt-in 标志控制哪些状态共享：

- `shareSetAppState`——共享应用状态
- `shareAbortController`——共享中止控制器
- `shareSetResponseLength`——共享响应长度

这是安全设计。子 Agent 默认拥有隔离的 state，只有在显式配置时才与主 Agent 共享特定部分。

---

## 3.2 Channel 注册与 Dev-Channel 门控

### Channel 类型与注册表

`ChannelEntry` 是插件和 MCP 服务器的注册标识：

```typescript
export type ChannelEntry =
  | { kind: 'plugin'; name: string; marketplace: string; dev?: boolean }
  | { kind: 'server'; name: string; dev?: boolean }
```

**为何区分 kind**——两种 kind 有不同的信任模型：
- `plugin`：通过 marketplace 验证 + GrowthBook allowlist 检查
- `server`：手动配置的 MCP 服务器，allowlist 检查总是失败（schema 是插件专用）

`dev` 字段标记通过 `--dangerously-load-development-channels` 加载的开发频道。这是安全的分离——allowlist gate 检查每个条目的 `dev` 标志，而非会话级别的标志。

---

### Channel 注册函数

```typescript
// bootstrap/state.ts
export function tryAddPluginChannel(entry: ChannelEntry): boolean {
  // 检查是否已存在
  const exists = STATE.allowedChannels.some(
    e => e.kind === entry.kind && e.name === entry.name
  )
  if (exists) return false
  STATE.allowedChannels.push(entry)
  channelsChanged.emit()
  return true
}

export function tryRemovePluginChannel(name: string): boolean {
  const idx = STATE.allowedChannels.findIndex(
    e => e.kind === 'plugin' && e.name === name
  )
  if (idx === -1) return false
  STATE.allowedChannels.splice(idx, 1)
  channelsChanged.emit()
  return true
}
```

注册是幂等的——相同的频道注册两次，第二次返回 `false`。这使得上层不需要做 exists-before-add 检查。

---

## 3.3 启动装配序列

### `init()` 的核心职责

`entrypoints/init.ts` 的 `init()` 函数是启动装配的中心节点：

1. **认证**：检查 API key / OAuth token
2. **策略限流**：检查组织级 policy limits
3. **模型解析**：确定当前使用的模型（支持 `--model` 和设置覆盖）
4. **会话初始化**：创建或恢复会话 ID
5. **遥测**：注册 OpenTelemetry provider

`main.tsx` 的 `run()` 函数在 Commander `preAction` hook 中调用 `await init()`。这保证了 init 只在实际命令执行时运行，而不是在帮助输出时。

---

### 迁移系统

启动时运行多个数据库迁移函数：

```typescript
// main.tsx 中的迁移序列
function runMigrations(): void {
  migrateAutoUpdatesToSettings()
  migrateBypassPermissionsAcceptedToSettings()
  migrateEnableAllProjectMcpServersToSettings()
  migrateFennecToOpus()
  migrateLegacyOpusToCurrent()
  migrateOpusToOpus1m()
  migrateReplBridgeEnabledToRemoteControlAtStartup()
  migrateSonnet1mToSonnet45()
  migrateSonnet45ToSonnet46()
  resetAutoModeOptInForDefaultOffer()
  resetProToOpusDefault()
}
```

每个迁移函数负责将旧配置格式转换为新格式。这些是一次性迁移——一旦用户的配置被更新，函数成为 no-op。

---

### 设置变更检测器

```typescript
// main.tsx:422-425
void settingsChangeDetector.initialize()
if (!isBareMode()) {
  void skillChangeDetector.initialize()
}
```

设置检测器在启动后监视设置文件的变更（`.claude/settings.json`）。这是热重载机制——设置变更不需要重启会话。

在 `--bare` 模式下，技能检测器不初始化（因为没有 hooks，没有技能目录发现）。

---

### 事件循环停滞检测器

```typescript
// main.tsx:428-429 (仅内部)
if ("external" === 'ant') {
  void import('./utils/eventLoopStallDetector.js')
    .then(m => m.startEventLoopStallDetector())
}
```

`eventLoopStallDetector` 检查主线程是否被阻塞超过 500ms。这是内部开发工具，外部构建中通过 feature flag 消除。

---

## 3.4 State 类型的完整分层

`getInitialState()` 构建约 200 个字段，但这些字段不是任意排列的——它们有清晰的层次：

```typescript
// bootstrap/state.ts - State 类型的核心分层
interface State {
  // ── 身份层 ──
  originalCwd: string          // 用户启动时的目录
  projectRoot: string          // 项目根目录（可能不同）
  cwd: string                  // 当前工作目录
  sessionId: string            // 当前会话 ID
  parentSessionId: string | undefined  // 会链恢复

  // ── 计量层 ──
  totalCostUSD: number         // 累计美元成本
  totalAPIDuration: number     // 累计 API 延迟
  totalToolDuration: number    // 累计工具执行时间
  turnToolCount: number        // 本轮工具调用数
  turnHookCount: number        // 本轮 hook 执行数
  modelUsage: Record<string, ModelUsage>  // 按模型追踪

  // ── 遥测层 ──
  meter: Meter                 // OpenTelemetry Meter
  loggerProvider: LoggerProvider
  tracerProvider: TracerProvider

  // ── 会话控制层 ──
  isInteractive: boolean       // TTY vs 非交互
  clientType: ClientType       // cli / remote / sdk / etc
  hasExitedPlanMode: boolean   // 是否已退出计划模式

  // ── Cache 控制层 ──
  afkModeHeaderLatched: boolean | null
  fastModeHeaderLatched: boolean | null
  cacheEditingHeaderLatched: boolean | null
  thinkingClearLatched: boolean | null
}
```

---

## 3.5 信号发布/订阅系统的实现细节

`createSignal()` 是一个 15 行的纯函数实现，提供类型安全的发布/订阅：

```typescript
// signal.ts
export function createSignal<Args extends unknown[] = []>(): Signal<Args> {
  const listeners = new Set<(...args: Args) => void>()
  return {
    subscribe(listener) {
      listeners.add(listener)
      return () => { listeners.delete(listener) }
    },
    emit(...args) {
      for (const listener of listeners) listener(...args)
    },
    clear() { /* reset all */ }
  }
}
```

**为什么不用 EventEmitter**——Node.js `EventEmitter` 有内存泄漏风险（未取消的 listener 永远不会被 GC）、类型不够精确（`any` 事件参数）。`createSignal` 的类型安全 `Args extends unknown[]` 确保 listener 参数与 emit 调用匹配。

**`onSessionSwitch` 的订阅者**：
- `concurrentSessions.ts`：PID 文件同步——会话切换时更新 `.claude/pid` 文件
- Session 追踪 UI：显示当前活跃会话

**信号的生命周期**——`clear()` 在会话切换时调用，清理旧的监听器。


---

## 3.6 State 类型的完整字段枚举

`State` 类型跨越 `lines 45-257`，共约 200 个字段。以下是按语义分类的完整枚举：

### 身份与路径（10 个字段）

| 字段 | 类型 | 含义 |
|------|------|------|
| `originalCwd` | `string` | 启动时的 realpath（NFC 规范化） |
| `projectRoot` | `string` | 稳定项目根（初始化后不变） |
| `cwd` | `string` | 当前工作目录（可变） |
| `sessionId` | `SessionId` | 当前会话 UUID |
| `parentSessionId` | `SessionId | undefined` | 链式恢复中的父会话 |
| `sessionProjectDir` | `string | null` | 转录文件所在目录 |
| `sessionIngressToken` | `string | null | undefined` | 会话认证令牌 |
| `oauthTokenFromFd` | `string | null | undefined` | FD 读取的 OAuth 令牌 |
| `apiKeyFromFd` | `string | null | undefined` | FD 读取的 API key |
| `directConnectServerUrl` | `string | undefined` | 直连服务器 URL |

### 计量与计数器（15 个字段）

| 字段 | 累积性 | 用途 |
|------|--------|------|
| `totalCostUSD` | 累积 | 累计美元成本 |
| `totalAPIDuration` | 累积 | API 总延迟 |
| `totalToolDuration` | 累积 | 工具总执行时间 |
| `totalLinesAdded/Removed` | 累积 | 代码行变更统计 |
| `turnToolCount/HookCount/ClassifierCount` | Turn-scoped | 每轮重置 |
| `turnToolDurationMs/HookDurationMs/ClassifierDurationMs` | Turn-scoped | 每轮毫秒数 |
| `turnHookCount` | Turn-scoped | 每轮 hook 执行数 |
| `turnClassifierDurationMs` | Turn-scoped | 分类器延迟 |

**Turn-scoped 计数器的生命周期**——每次 turn 开始时这些字段重置为零。这使得每轮的性能分析独立于历史。

### Telemetry 提供者（9 个字段）

| 字段 | 初始值 | 注入时机 |
|------|--------|---------|
| `meter` | `null` | `setMeter()` from `init.ts` |
| `loggerProvider` | `null` | `setLoggerProvider()` |
| `tracerProvider` | `null` | `setTracerProvider()` |
| `meterProvider` | `null` | `setMeterProvider()` |
| `sessionCounter/locCounter/prCounter/commitCounter/costCounter/tokenCounter` | `null` | `setMeter()` 中通过工厂函数创建 |

所有 telemetry 字段以 `null` 初始化——`State` 不直接依赖 OpenTelemetry SDK，避免循环导入。提供者通过 `set*()` 函数注入，调用方在 `init.ts` 中完成初始化。

### Session 控制标志（20+ 个字段）

`isInteractive`、`kairosActive`、`strictToolResultPairing`、`sessionTrustAccepted`、`scheduledTasksEnabled`、`sessionPersistenceDisabled`、`hasExitedPlanMode`、`needsPlanModeExitAttachment`、`lspRecommendationShownThisSession` 等。

---

## 3.7 Beta Header Latch：四路粘性锁存器

```
┌─────────────────────┬──────────────────────┬──────────────────────────┐
│ latch               │ trigger              │ purpose                  │
├─────────────────────┼──────────────────────┼──────────────────────────┤
│ afkModeHeaderLatched├ AFK mode first on    │ 防止 Shift+Tab 振荡 bust  │
│ fastModeHeaderLatch │ Fast mode first on   │ 防止冷却进入/退出 bust   │
│ cacheEditingHeader  │ Cached micro-comp on │ 防止 mid-session toggle  │
│ thinkingClear       │ >1h since last API   │ 清除旧 thinking 并锁定   │
└─────────────────────┴──────────────────────┴──────────────────────────┘

设置: state.ts (lines 1708-1748) - 在 API 请求构建期间
读取: claude.ts (lines 1425-1680) - 决定是否添加 beta header
重置: clearBetaHeaderLatches() - 在 /clear 和 /compact 时调用
```

**为何叫 sticky-on**——一旦 latch 设为 `true`，就不会再回到 `false`。即使对应的功能被关闭，header 仍然发送。这是缓存一致性策略——远端 API 的 prompt cache 由 header 值决定，改变 header 值将导致 ~50-70K tokens 的 cache miss。

**`null` vs `false` 的语义**——`null` 表示"功能从未激活"，`true` 表示"功能已激活且锁定"。没有 `false` 状态——如果功能是关闭的，latch 停留在 `null`，header 不发送。

### clearBetaHeaderLatches 的调用时机

```typescript
// 在 /clear 和 /compact 后
clearBetaHeaderLatches()  // 将所有四个 latch 重置为 null
```

这是正确的——`/clear` 和 `/compact` 开始了新的对话，header 应该重新评估。如果不调用 clear，新对话可能继承旧的不正确 header 状态。

---

## 3.8 Signal 系统的实际使用

`state.ts` 中只有**一个** `createSignal` 调用：

```typescript
// state.ts:481
const sessionSwitched = createSignal<[id: SessionId]>()
// state.ts:489 暴露为公共订阅 API
export const onSessionSwitch = sessionSwitched.subscribe
```

### 唯一的订阅者：concurrentSessions.ts

```typescript
// concurrentSessions.ts:101-103
onSessionSwitch(id => {
  void updatePidFile({ sessionId: id })
})
```

当会话切换时（如 `/resume`），PID 文件需要更新。`~/.claude/sessions/<pid>.json` 的 `sessionId` 字段是 `claude ps` 命令读取当前会话转录文件的唯一途径。如果不更新，PID 文件指向旧的 sessionId，sparkline 显示错误的会话。

**为何其他状态变更没有信号**——State 中的大多数字段（如 `cwd`、`totalCostUSD`、`isInteractive`）通过直接 getter/setter 访问。不需要 pub/sub——消费者直接读取当前值。信号只用于`需要通知外部观察者`的场景——session switch 就是这种场景。

### 代码库中的其他信号

虽然 `state.ts` 只导出一个信号，但代码库中有约 19 个 `createSignal()` 调用：

| 信号 | 位置 | 用途 |
|------|------|------|
| `growthbook` 信号 | `growthbook.ts` | A/B 测试状态变更 |
| `skillChangeDetector` | `changeDetector.ts` | 技能目录变更 |
| `fastMode` 信号 | `fastMode.ts` | 快速模式状态 |
| `tasks` 信号 | `tasks.ts` | 任务变更通知 |
| `mailbox` 信号 | `mailbox.ts` | 邮箱消息 |
| `classifierApprovals` | `classifierApprovals.ts` | 分类器审批 |
| `awsAuthStatusManager` | `awsAuthStatusManager.ts` | AWS 认证状态 |

这些信号分散在各自的功能域中，不集中在 `state.ts`——这是正确的分离关注点。

---

## 3.9 Forked Agent 状态隔离

子 Agent 通过 `createSubagentContext()` 获得隔离的状态视图。三个关键的 opt-in 标志控制共享行为：

```typescript
// forkedAgent.ts:345-462
const agentContext = createSubagentContext(toolUseContext, {
  shareSetAppState: !isAsync,         // 仅共享同步 Agent
  shareSetResponseLength: true,        // 两者都共享贡献
  shareAbortController: false,         // 默认独立
})
```

### 三种旗标的语义

| 标志 | 默认值 | 当 true | 当 false |
|------|--------|---------|----------|
| `shareSetAppState` | `false` | 子 Agent 写入直达根 AppState（交互 Agent更新共享态） | 变为 no-op（静默丢弃） |
| `shareSetResponseLength` | `false` | 子 Agent 贡献到父 Agent 响应长度/API 指标追踪 | 长度独立追踪/no-op |
| `shareAbortController` | `false` | 父 Agent 中止杀死子 Agent | 新 `AbortController` 关联但可独立中止  |

**为何 `shareSetAppState` 只对同步 Agent**——异步 Agent（后台任务）不应该影响前台 AppState，因为它们可能在未来的任意时间运行。同步 Agent（在用户可见的会话中）应该共享状态，以便 UI 实时更新。

**状态克隆语义**——除了上述 3 个选项，子 Agent 上下文的其他一切都是克隆或全新的：
- `readFileState`: `cloneFileStateCache()` 克隆
- `contentReplacementState`: 克隆
- `nestedMemoryAttachmentTriggers`、`dynamicSkillDirTriggers`、`discoveredSkillNames`: 新 Set
- `queryTracking`: 新 chainId，深度 +1
- UI 回调（`addNotification`、`setToolJSX` 等）: `undefined`

### no-op 作为隔离默认

当 flag 为 false 时，`setAppState` 变为 `() => {}`——空函数。调用不会抛出错误，只是被静默丢弃。这是 fail-safe 的隔离——子 Agent 代码即使忘记检查隔离条件，也不会影响父 Agent 状态。

---

## 3.10 Bootstrap 隔离与循环导入防护

`state.ts` 是导入 DAG 的叶子——只有它导入其他模块，没有其他模块被它导入（除 `state.ts` 本身外）。

### 导入约束的执行

```
state.ts 可以导入:
  ✓ bun:bundle (feature flag)
  ✓ fs/promises, path, process (stdlib)
  ✓ lodash-es/sumBy.js
  ✓ src/utils/crypto.js (re-export leaf)
  ✓ src/utils/signal.js (leaf)
  ✓ src/utils/settings/settingsCache.js

ESLint custom rule: custom-rules/bootstrap-isolation
```

**唯一的 eslint-disable**——`import { randomUUID }` 使用 `src/` 路径别（而非 `./` 或 `../`）绕过规则的路径前缀检查。

### 避免 sleep() 导入

```typescript
// state.ts:820
// bootstrap-isolation forbids importing sleep() from src/utils/
await new Promise(r => setTimeout(r, SCROLL_DRAIN_IDLE_MS).unref?.())
```

使用原生 `setTimeout` 而非导入 `sleep()` 工具函数——因为 `sleep()` 在 `src/utils/` 中可能间接导入 `state.ts`，形成循环。

### 回调注入模式

```typescript
// setMeter 接受外部注入的 meter 实例
export function setMeter(
  meter: Meter,
  createCounter: (name: string, opts: CounterOptions) => AttributedCounter
): void {
  STATE.meter = meter
  STATE.sessionCounter = createCounter('sessions', ...)
  // ... 创建所有 8 个计数器
}
```

`state.ts` 不导入 OTel SDK——`init.ts` 初始化所有提供者并通过 `set*()` 函数注入。这是**依赖倒置**——高层模块（`State`）不依赖低层模块（OTel），而是 OTel 作为参数传入。

---

## 3.11 Channel 注册表的真实实现

代码库中实际存在的 Channel 管理机制与初始设计文档中略有不同：

### ChannelEntry 类型与访问器

```typescript
// state.ts:37-39
export type ChannelEntry =
  | { kind: 'plugin'; name: string; marketplace: string; dev?: boolean }
  | { kind: 'server'; name: string; dev?: boolean }
```

`allowedChannels` 和 `hasDevChannels` 通过简单的 getter/setter 管理——**没有 pub/sub 信号**。消费者在需要时调用 `getAllowedChannels()`，而不是订阅变更通知。

### 谁调用 `setAllowedChannels`

| 调用方 | 场景 | 文件 |
|--------|------|------|
| `print.ts:4711` | MCP server 连接时添加条目 | 连接时 append，断开时 remove |
| `main.tsx:1695` | 启动初始化期间设置 | 从 CLI 解析 |
| `interactiveHelpers.tsx:267` | 用户接受 dev channels dialog | `--dangerously-load-development-channels` |

### Gate 评估时机

Channel gate 在连接时评估——不是 allowlist 变更时。`gateChannelServer()` 函数执行多层门检查：

```
capability → disabled → auth → policy → session allowlist 
→ marketplace verification → allowlist
```

如果 allowlist 在连接后更新，连接的服务器不需要重新 gate——gate 只在 `notifications/claude/channel` 注册时评估。这意味着：如果一个通道在连接后被添加到 allowlist，服务器不会自动注册——需要重新连接或触发重新评估。

---

## 3.12 `switchSession()` 的轻量级语义

```typescript
// state.ts:468-479
export function switchSession(
  sessionId: SessionId,
  projectDir: string | null = null,
): void {
  STATE.planSlugCache.delete(STATE.sessionId)  // 清除即将离开的会话的 slug
  STATE.sessionId = sessionId
  STATE.sessionProjectDir = projectDir
  sessionSwitched.emit(sessionId)
}
```

**关键设计决策**——`switchSession()` 只修改 3 个字段：
1. `sessionId`：新 UUID
2. `sessionProjectDir`：新项目目录（可选）
3. `planSlugCache`：清除旧会话的 plan slug

**不重置的字段**——持续时间计数器、成本、modelUsage、遥测状态、以及其余 190+ 字段全部保留。这意味着新会话继承了旧会话的内存状态（成本累积、模型使用计数等）。这是有意的——`/resume` 恢复旧会话时，成本追踪应该是累积的，不应从零开始。

**sessionId+sessionProjectDir 的原子配对**——注释 CC-34（lines 456-466）记录了过去的 drift bug：sessionId 和 sessionProjectDir 曾分开更新，导致两者不匹配。现在配对为 `switchSession(sessionId, projectDir)`，确保更新是原子的。

---

## 3.21 双 Store 系统

Claude Code 有**两套**独立的全局状态系统：

**Layer 1——Bootstrap State**（`bootstrap/state.ts`，约 1758 行，~257 个字段）：裸可变单例。无 React、无 Store 抽象——只有 `const STATE = getInitialState()` 和直接的 getter/setter 函数。

**Layer 2——React AppState**（`state/AppStateStore.ts`，约 140 个字段）：不可变 `DeepImmutable` 对象，包装在类 Redux store（`createStore()`）中，通过 React Context + `useSyncExternalStore` 消费。

两套系统通过 `ToolUseContext['getAppState']` 桥接——工具执行上下文将纯业务层连接至 React UI 层。

---

## 3.22 State 类型结构（Bootstrap）

`State` 接口（`bootstrap/state.ts:45-257`）按语义分层组织：

**身份/路径层**（10 个字段）：
| 字段 | 用途 |
|------|------|
| `originalCwd` | 启动时的路径（NFC 规范化 realpath，处理 iCloud Drive EPERM） |
| `projectRoot` | 稳定项目根，启动时设置一次，**永不**被 `EnterWorktreeTool` 更新 |
| `cwd` | 当前工作目录（可变） |
| `sessionId` | 当前会话 UUID，`/clear` 时重新生成 |
| `parentSessionId` | 链路血缘（plan mode 到 implementation） |

**成本与会计层**（10 个字段）：
- `totalCostUSD`、`totalAPIDuration`、`totalToolDuration`
- `totalLinesAdded`、`totalLinesRemoved`
- `modelUsage: { [modelName: string]: ModelUsage }`——每模型的 token/成本明细
- `hasUnknownModelCost`
- `startTime`（`getTotalDuration()` = `Date.now() - startTime`）

**Turn 级计数器**（6 个字段——每轮重置）：
- `turnHookDurationMs`、`turnToolDurationMs`、`turnClassifierDurationMs`
- `turnToolCount`、`turnHookCount`、`turnClassifierCount`
- 每个有 `reset*Duration()` 将 duration 和 count 都归零

---

## 3.23 Sticky-on Latch 模式

四个 `*HeaderLatched` 字段使用**三态 null-boolean** 模式：

- `null` = "特性从未评估/激活"——不发送 beta 头
- `true` = "特性至少激活过一次"——始终发送头
- **不存在 `false` 状态**

**为何需要**——Anthropic API prompt 缓存键（约 50-70K token）在 beta 头值变化时改变。如果用户在特性关闭和开启之间切换（如 Shift+Tab 在普通和快速模式之间），头来回变化会反复破坏 prompt 缓存。

**状态转移是严格单调的**：
```
null --(首次激活)--> true --(特性关闭)--> true (不变)
```

**重置**——`clearBetaHeaderLatches()` 在 `/clear` 和 `/compact` 后将所有四个重置为 `null`。

被 latch 的四个头：
| Latch | Beta 头作用 |
|-------|-----------|
| `afkModeHeaderLatched` | AFK 模式 beta 头 |
| `fastModeHeaderLatched` | 快速模式 beta 头 |
| `cacheEditingHeaderLatched` | 缓存编辑 beta 头 |
| `thinkingClearLatched` | thinking 清除 beta 头 |

---

## 3.24 State 初始化

**`getInitialState()`**（行 260-426）从零构建整个 `State` 对象：

- `originalCwd` 通过 `realpathSync(cwd()).normalize('NFC')` 解析——处理符号链接和 macOS 的 NFC 规范
- 如果 realpathSync 在 iCloud CloudStorage 挂载点失败（EPERM），回退到原始 `cwd()`
- 所有遥测计数器初始化为 `null`（非零）——计数器必须后续注入
- `sessionId` 在模块加载时初始化为随机 UUID
- 四个 latch 字段全部初始化为 `null`

**模块单例**——第 429 行：`const STATE: State = getInitialState()`。这是唯一实例。该文件是模块单例。

**测试隔离**——`resetStateForTests()` 从全新的 `getInitialState()` 复制每个字段——昂贵但正确。

---

## 3.25 State 变异模式

**Bootstrap State**——直接通过专用 setter 函数变异。没有 `setState()` 函数。每个字段有自己的 getter/setter 对（如 `getTotalCostUSD()`/`setTotalCostUSD()`）。某些字段只读（无 setter），如 `sessionId` 只能通过 `switchSession()` 修改。

**累加器模式**——计数器使用 `+=` setter：
- `addToTotalCostState()`——同时更新总计和每模型使用
- `addToToolDuration()`——同时更新总计、turn duration 和 turn count

**延迟交互时间**——`updateLastInteractionTime()` 默认设置 `interactionTimeDirty = true` 而非调用 `Date.now()`。实际时间戳在 Ink 每个渲染周期前的 `flushInteractionTime()` 中批量处理。这避免每次按键都调用 `Date.now()`。

**会话恢复**——`setCostStateForRestore()` 接受来自磁盘会话记录的一组字段，将 `startTime` 调整为 `lastDuration`，使挂钟累积而不重置。

---

## 3.26 AppState（React Store）不可变性模式

`store.ts` 是 34 行的泛型 store：

```typescript
function createStore<T>(initialState, onChange?) {
  let state = initialState
  const listeners = new Set<Listener>()
  return {
    getState: () => state,
    setState: (updater) => {
      const prev = state
      const next = updater(prev)
      if (Object.is(next, prev)) return  // bailout
      state = next
      onChange?.({ newState: next, oldState: prev })
      for (const listener of listeners) listener()
    },
    subscribe: (listener) => { ... }
  }
}
```

**关键设计决策**——使用 `Object.is(prev, next)` 做 bailout——**不是**结构比较。这强迫调用者始终创建新的对象引用。`DeepImmutable` 类型包装在编译时防止意外变异。

**onChangeAppState** 挂载副作用到 AppState diff：
- 权限模式变更通知 CCR external_metadata 和 SDK 状态流——这被称为"CCR/SDK 模式同步的单一瓶颈点"，修复了之前 8 个变异路径中的 6 个
- `mainLoopModel` null/非 null 转换更新设置
- `expandedView` 变更持久化到 globalConfig
- `settings` 变更清除认证缓存（apiKey、AWS、GCP）并重新应用环境变量

---

## 3.27 Forked Agent 隔离

`createSubagentContext()` 从父级创建隔离的 ToolUseContext：

**默认：完全隔离**：

| 字段 | 默认行为 | 共享覆盖 |
|------|---------|---------|
| `setAppState` | `() => {}`（no-op） | `shareSetAppState: true`——写入 root store |
| `abortController` | 新的父级子级 | `shareAbortController: true`——共享同一控制器 |
| `readFileState` | `cloneFileStateCache()`——深克隆 | N/A |
| `localDenialTracking` | 新的 `createDenialTrackingState()` | `shareSetAppState: true` 时共享 |

**为何克隆 `contentReplacementState`（而非新创建）**——克隆状态包含来自消息前缀的父级 `tool_use_id` UUID。新状态会将它们视为"未见"并做出不同的替换决策，生成不同的有线前缀，导致缓存未命中。克隆对缓存命中做出相同的决策。

**为何 `setAppStateForTasks` 始终到达 root**——即使 `setAppState` 是 no-op，`setAppStateForTasks` 必须到达 root store，使异步 agent 的后台 bash 任务被正确注册和杀死。没有这个，任务变成 PPID=1 僵尸。

---

## 3.28 Session 切换

`switchSession(sessionId, projectDir)` 原子性地改变**仅 3 个字段**：
1. `planSlugCache.delete(STATE.sessionId)`——驱逐新会话的 slug 条目
2. `STATE.sessionId = sessionId`
3. `STATE.sessionProjectDir = projectDir`
4. `sessionSwitched.emit(sessionId)`

**190+ 个字段不重置**——成本追踪、modelUsage、遥测状态、latch 等都继续携带。这是有意的——`/resume` 是继续会话而非重新开始。

**CC-34 问题**——过去因分别更新 `sessionId` 和 `sessionProjectDir` 导致它们不同步的 bug。现在始终在此单一函数中一起修改。

---

## 3.29 信号系统（Pub/Sub）

`utils/signal.ts`——43 行，`createSignal<Args>()` 返回 `{ subscribe, emit, clear }`。

Bootstrap state 中**仅存一个信号**：
- `sessionSwitched = createSignal<[id: SessionId]>()`——由 `switchSession()` 发出
- 暴露为 `onSessionSwitch = sessionSwitched.subscribe`

**唯一的订阅者**：`concurrentSessions.ts`——当 `/resume` 切换会话时更新 PID 文件的 `sessionId`。

**为何其他状态变更没有信号**——大多数字段由消费者按需直接读取。信号保留用于"某事发生，通知外部观察者"的场景，即 State 模块不能导入观察者的情况（DAG 叶子约束）。

---

## 3.30 Bootstrap 隔离约束

文件级注释（行 31-33）：`DO NOT ADD MORE STATE HERE - BE JUDICIOUS WITH GLOBAL STATE`。类似警告出现在行 259（`ALSO HERE - THINK THRICE`）和行 428（`AND ESPECIALLY HERE`）。

**导入约束**——`state.ts` 只能导入：
- 标准库（`fs/promises`、`path`、`process`、`crypto`）
- 特性标记（`bun:bundle`）
- `lodash-es/sumBy.js`
- 叶子重导出：`src/utils/crypto.js`（仅 `node:crypto`）、`signal.ts`、`settingsCache.js`

ESLint 规则 `custom-rules/bootstrap-isolation` 强制执行此约束。

**无 `sleep()` 导入**——行 820-822 使用原始 `setTimeout` 而非 `import { sleep }`——因为 `sleep()` 在 `src/utils/` 中可能间接导入 `state.ts`，形成循环。

---

## 3.39 State Selectors（派生状态模式）

`state/selectors.ts`——AppState 之上的**纯选择器层**：

- `getViewedTeammateTask(appState)`——提取当前查看的团队成员任务，带 null 安全守卫
- `getActiveAgentForInput(appState)`——确定用户输入应路由到哪里（leader vs 查看中的团队成员 vs 本地 agent），返回判别联合类型 `{ type: 'leader' | 'viewed' | 'named_agent' }`

**设计原则**——文件头部注释："selectors are pure"——仅数据提取，无副作用。与 Redux 派生模式一致，是对 `Store<T>` 抽象的补充。

---

## 3.40 React Selector Hook 模式

`state/AppState.tsx`（117-199 行）——四个层次化的 React hooks：

| Hook | 用途 | 重渲染条件 |
|------|------|------------|
| `useAppState<T>(selector)` | 使用 `useSyncExternalStore` + 选择器函数 | 仅选中值通过 `Object.is` 比较变化时 |
| `useSetAppState()` | 返回**稳定引用**——永不变化 | 从不因状态变化重渲染 |
| `useAppStateMaybeOutsideOfProvider<T>` | 在 Provider 外安全使用 | 返回 `undefined`（`NOOP_SUBSCRIBE` 做空操作） |
| `useAppStateStore()` | 返回原始 `Store<AppState>` | 供非 React 代码使用 |

**关键约束**——"`useAppState` 的选择器绝不能返回新对象——`Object.is` 总是认为它们变化了"。这要求选择器只能返回对已有值的直接引用或原始类型。

---

## 3.41 Speculation 状态（推测执行状态机）

`state/AppStateStore.ts`（52-79 行）——嵌入 AppState 的**推测执行状态机**：

```typescript
type SpeculationState =
  | { status: 'idle' }
  | {
      status: 'active'
      id: string
      abort: () => void
      startTime: number
      messagesRef: { current: Message[] }     // Mutable ref——避免每条消息深拷贝
      writtenPathsRef: { current: Set<string> }
      contextRef: { current: REPLHookContext }
      toolUseCount: number
      isPipelined: boolean
    }
```

驱动"预思考"特性——模型在用户还在决策时推测生成后续查询。使用 mutable refs 避免每条消息的数组展开深拷贝。`SpeculationResult` 含 `timeSavedMs` 字段量化推测的收益。

`IDLE_SPECULATION_STATE` 常量提供可复用的空闲状态单例避免重复分配。

---

## 3.42 Session 状态机（Idle / Running / Requires Action）

`utils/sessionState.ts`——独立于 Bootstrap 和 AppState 的 SDK 会话状态机：

```typescript
type SessionState = 'idle' | 'running' | 'requires_action'
```

**三种独立单例监听器**（非 Signal）：`stateListener`、`metadataListener`、`permissionModeListener`——各通过 `set*Listener()` 设置而非 subscribe/emit。

**`SessionExternalMetadata`**（32-45 行）——推送给 CCR Web 侧栏的结构化元数据：`permission_mode`、`is_ultraplan_mode`、`model`、`pending_action`、`task_summary`。

**可选 SDK 事件发射**——受 `CLAUDE_CODE_EMIT_SESSION_STATE_EVENTS` 环境变量门控。

**`notifyPermissionModeChanged()`**——用作 CCR 同步的**单一咽喉点**（single choke point）。

---

## 3.43 Teammate View 状态转换

`state/teammateViewHelpers.ts`——查看团队成员 agent 转录的状态管理：

**`enterTeammateView(taskId, setAppState)`**：
1. 设置 `viewingAgentTaskId`
2. 对 `local_agent` 任务应用 `retain: true`——阻断驱逐，启用流追加，触发磁盘引导
3. 若从另一 agent 切换，**release** 前一个回 stub 形式

**`exitTeammateView(setAppState)`**——丢弃 retain，清除消息回 stub，若任务终止则通过 `evictAfter` 时间戳调度驱逐。

**`release()` 辅助函数**——创建 stub 形式：清除 `messages`、设 `retain: false`、设 `evictAfter: Date.now() + PANEL_GRACE_MS`（30 秒）。细粒度生命周期机制。

---

## 3.44 File History 状态（检查点系统）

`utils/fileHistory.ts`——文件检查点系统：

**`FileHistoryState`**：
- `snapshots[]`——上限 100 个快照
- `trackedFiles: Set<string>`——跟踪的文件集
- `snapshotSequence: number`——单调递增活动信号（`useGitDiffStats` 使用）

**`FileHistorySnapshot`** 关联 `messageId: UUID`——将检查点与对话 turn 绑定。

`fileHistoryRestoreStateFromLog()`——从持久化日志恢复文件历史（`--resume` 时）。

**功能标志门控**——`fileHistoryEnabled()` 检查 `globalConfig` 中的 `fileCheckpointingEnabled` 加上 `CLAUDE_CODE_DISABLE_FILE_CHECKPOINTING` 环境变量。双重开关允许策略级全局禁用。

---

## 3.45 Context 缓存与备忘录化

`context.ts`——使用 `lodash-es/memoize` 缓存昂贵异步计算：

| Memoized 函数 | 说明 | 失效条件 |
|---------------|------|----------|
| `getGitStatus` | 5 个 git 操作并行执行 `Promise.all`，截断 2000 字符 | 仅 `systemPromptInjection` 变化时 |
| `getSystemContext` | 组合 gitStatus + injection 的 `cacheBreaker` | 同上 |
| `getUserContext` | 读取 CLAUDE.md 文件，设置缓存内容 | 同上 |

**`systemPromptInjection`**——手动可变字符串作为缓存破坏器。`setSystemPromptInjection()` 立即调用 `getUserContext.cache.clear()` 和 `getSystemContext.cache.clear()`。

**循环防护**——`getUserContext` 调用 `setCachedClaudeMdContent()` 为自动模式分类器设置缓存（阻断 `permissions/filesystem -> permissions -> yoloClassifier` 的导入循环）。

---

## 3.46 Settings 变更检测的删除重建宽限

`changeDetector.ts`——变更检测器的实现细节：

**删除 + 重建宽限期**——`DELETION_GRACE_MS = 1200ms`：文件被删除时启动宽限定时器。若在此窗口内被重建（自动更新器、另一会话启动常见），删除被取消，照常处理为变更。

**内部写追踪**——`markInternalWrite()` / `consumeInternalWrite(path, 5000ms)`——Claude Code 写自身设置文件后 5 秒窗口内的变更被抑制，避免自触发重载循环。

**ConfigChange Hook 阻断**——扇出前 `executeConfigChangeHooks()` 运行——任何 hook 返回阻断结果（退出码 2 或 `decision: 'block'`）则完全跳过该设置变更。

**单一生产者缓存重置**——修复前，N 个监听器各自防御性地清缓存导致 N 次磁盘重载。现在 `fanOut()` 重置一次（单一生产者），第一个监听器支付 miss 代价，后续全部命中缓存。

---

## 3.47 终端焦点状态信号

`ink/terminal-focus-state.ts`——DECSET 1004 焦点报告的信号实现：

**三态枚举**：`'focused' | 'blurred' | 'unknown'`（默认）。

**双订阅系统**：
| 订阅者 | 用途 |
|--------|------|
| `subscribers: Set<() => void>` | `useSyncExternalStore` 消费者（React） |
| `resolvers: Set<() => void>` | 基于 Promise 的等待（非 React） |

**`setTerminalFocused(v: boolean)`**——失焦时解析所有等待 Promise 并清空调节器集；聚焦时通知 React 订阅者。消费者将 `'unknown'` 等同 `'focused'` 处理（不节流）。

这与 `state.ts` 中 `createSignal()` 模式不同——两套订阅集和双模式通知系统，而非简单 Signal。

---

## 3.48 Session Restore 系统

`utils/sessionRestore.ts`——全面的会话恢复框架：

**`processResumedConversation(result, opts, context)`**——主协调器：协调器模式匹配、会话 ID 设置、agent 恢复、worktree 恢复和初始状态计算。

**`restoreSessionMetadata(result)`**——从转录恢复 `sessionName`、`agentName`、`agentColor`、`tag`、`customTitle`。

**`restoreWorktreeForResume(worktreeSession)`**——恢复时 chdir 到 worktree 目录，TOCTOU 安全的 `process.chdir()` 存在检查。

**`extractTodosFromTranscript(messages)`**——反向扫描转录找最后 TodoWrite tool_use 块，SDK resume 时水合 `AppState.todos`。

**`--fork-session` 区别于 `--resume`**——保持全新启动会话 ID，种子化 `contentReplacement` 记录使新会话知道哪些消息被替换。

---

## 3.49 onChangeAppState 单一咽喉点

`state/onChangeAppState.ts`——CCR/SDK 模式同步的**单一咽喉点**：

**历史问题**——此 hook 之前，模式变更仅通过 8+ 条变更路径中的 2 条中继给 CCR，导致 `external_metadata.permission_mode` 陈旧。

**现在的行为**——任何 `setAppState` 调用若改变 `toolPermissionContext.mode` 都通知：
1. CCR：通过 `notifySessionMetadataChanged`（外部化模式，非内部专有名称）
2. SDK 通道：通过 `notifyPermissionModeChanged`

**副作用**——持久化 `expandedView`、`verbose`、`tungstenPanelVisible` 到 `globalConfig`，设置变更时清除认证相关缓存。

---
