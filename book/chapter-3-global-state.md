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
