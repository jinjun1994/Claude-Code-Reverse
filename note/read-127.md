## 第 143 站：CLI 入口与模式路由（6 个文件，~2.5K）

### 这是什么

CLI 入口是 Claude Code 的启动路由器——从 `cli.tsx` 快速路径分发到 TUI 交互模式、headless 模式、或各种内部子命令。

---

### cli.tsx —— 快速引导路由器

**关键模式：14+ 快速路径，零成本导入**

```
--version          → console.log(version)，零导入
--claude-in-chrome-mcp  → 启动 Chrome MCP
--computer-use-mcp      → 启动 Computer Use MCP
--daemon-worker=<kind>  → 瘦 worker
remote/rc/bridge        → bridgeMain()
daemon                  → daemonMain()
ps/logs/attach/kill     → 后台处理
new/list/reply          → templatesMain()
--tmux --worktree       → execIntoTmuxWorktree()
fall through            → main.tsx（完整 CLI）
```

所有导入都是动态的——不匹配的路径零模块成本。

**`feature('FLAG')` 是编译时门**——`bun:bundle` 的 feature macro 允许构建时 DCE。外部构建剥离所有 Ant-only 和企业分支。

---

### main.tsx —— 完整 CLI 指挥器

**Module-level 并行预热**（lines 1-271）：
```ts
startMdmRawRead()      // 并行
startKeychainPrefetch() // 并行
// ~135ms import 时间内完成
```

**PreAction Hook**（在任何命令前运行）：
```ts
program.hook('preAction', async () => {
  await ensureMdmSettingsLoaded()       // 等待并行子进程
  await init()                          // 共享引导
  process.title = 'claude'
  await runMigrations()                 // 同步迁移
  // 以下 fire-and-forget:
  // remote managed settings
  // policy limits
  // upload user settings
})
```

**交互 vs 非交互决定**：
```ts
const isNonInteractive = hasPrintFlag || hasInitOnlyFlag || hasSdkUrl || !process.stdout.isTTY
```

**Client Type 推断**：
```ts
clientType = env.GITHUB_ACTIONS      → 'github-action'
CLAUDE_CODE_ENTRYPOINT === 'sdk-ts'  → 'sdk-typescript'
CLAUDE_CODE_ENTRYPOINT === 'remote'  → 'remote'
default                              → 'cli'
```

**Deferred Prefetch**（REPL 渲染后并行触发）：
`initUser()`, `getUserContext()`, `prefetchSystemContextIfSafe()`, `countFilesRoundedRg()`, `refreshModelCapabilities()`, event loop stall detector。

---

### init.ts —— 共享引导

**10 步引导**：
```
1. enableConfigs()
2. applySafeConfigEnvironmentVariables()
3. applyExtraCACertsFromConfig()
4. setupGracefulShutdown()
5. Fire-and-forget: logging, GrowthBook, OAuth, JetBrains, GitHub, remote settings, policy limits
6. recordFirstStartTime()
7. configureGlobalMTLS(), configureGlobalAgents(), preconnectAnthropicApi()
8. CCR upstream proxy relay
9. 平台设置 + swarm team 清理注册
10. Scratchpad directory
```

**ConfigParseError 处理**：交互模式 → React `InvalidConfigDialog`；非交互 → stderr + exit 1。

---

### commands.ts —— 命令注册

**100+ 命令，多种来源**：
1. 内置（config, mcp, memory, help, model, theme, login, doctor...）——静态导入
2. Feature-gated（proactive, bridge——require 在 feature macro 内）
3. 认证依赖（!isUsing3PServices() ? [logout, login()] : []）
4. Ant-only（~25 内部命令）

**可用性过滤**：
- `claude-ai`：需要 Claude.ai 订阅
- `console`：需要 API key（非 Claude.ai）+ 第一方端点

**安全命令集**：
- `REMOTE_SAFE_COMMANDS`：远程 TUI 允许的（session, exit, clear, help...）
- `BRIDGE_SAFE_COMMANDS`：移动端/网页 bridge 允许的（compact, clear, cost, summary...）

---

### print.ts —— 无头模式

**三种输出格式**：
- `text`：纯文本 stdout
- `json`：完成时单个 JSON 结果
- `stream-json`：实时流式结构化事件

**关键模式**：`createCanUseToolWithPermissionPrompt()`——非交互模式下权限提示通过特殊工具或自动 deny 处理。

**`--include-hook-events`**：在 hook 事件输出中包含 hook 生命周期事件。

---

### 读完后应抓住的 4 个事实

1. **14+ 快速路由**——`cli.tsx` 是真正的入口，每个特殊路径（Chrome MCP, daemon, bridge, templates）都有独立的 fast-path，不加载完整 CLI。所有导入动态化。

2. **init() 是共享前置**——通过 Commander `preAction` hook 保证在任何命令前运行。配置、迁移、telemetry 都在这里。

3. **Deferred Prefetch 模式**——REPL 渲染后立即并行触发 6-7 个异步操作，不等结果。这减少了首响延迟。

4. **Client Type 由 env 决定**——`CLAUDE_CODE_ENTRYPOINT` 环境变量区分 CLI/SDK/remote/github-action。这影响工具可用性、输出格式、和权限行为。

---

## 第 144 站：Settings 与 MCP 配置系统（7 个文件，~4K）

### 这是什么

Settings 合并引擎决定 Claude Code 的所有配置行为，MCP 配置系统在多个来源之间合并服务器定义并应用企业策略。

---

### 设置合并优先级

```
pluginSettings（base）→ userSettings → projectSettings → localSettings → flagSettings → policySettings
```

**Policy Settings 级联**（"第一个来源胜出"）：
```
Remote managed (API) > HKLM/macOS plist > managed-settings.json + .d/ > HKCU
```

**Managed file settings 的 drop-in 约定**：
```
managed-settings.json              → 基础
managed-settings.d/*.json          → 按字母序合并（后面的赢）
```

例如 `10-otel.json`, `20-security.json` 独立团队发布策略片段。

---

### settings.ts —— 合并引擎

**数组合并**：
```ts
function settingsMergeCustomizer(objValue, srcValue) {
  if (Array.isArray(objValue) && Array.isArray(srcValue))
    return mergeArrays(objValue, srcValue)  // concat + dedup
  return undefined  // lodash 默认深合并
}
```

**`updateSettingsForSource`** 写入特性：
- `undefined` 值触发 key 删除
- 数组始终替换（不合并）
- 标记 "internal write" 防止文件变更检测器自触发
- 自动将 local settings 添加到 `.gitignore`
- 写入后重置会话缓存

**安全读取**——防 RCE 设计：
```ts
// 刻意排除 projectSettings——防止恶意项目文件自动绕过安全对话框
hasSkipDangerousModePermissionPrompt()
hasAutoModeOptIn()
getUseAutoModeDuringPlan()
```
只读 userSettings 和 userSettings 等可信来源。

---

### remoteManagedSettings —— 远程管理设置

**HTTP 缓存**：
```ts
export function computeChecksumFromSettings(settings) {
  const sorted = sortKeysDeep(settings)
  const hash = createHash('sha256').update(jsonStringify(sorted)).digest('hex')
  return `sha256:${hash}`
}
```

必须与服务端 Python `json.dumps(settings, sort_keys=True, separators=(",",":"))` 匹配。

**后台轮询**：每小时一次，`setInterval` 带 `.unref()`（不阻塞进程退出）。

**30 秒超时**——`waitForRemoteManagedSettingsToLoad()` 防止测试中死锁。

---

### MCP 配置 —— 9 种传输类型

| 类型 | 描述 |
|---|---|
| `stdio` | 子进程（最常见） |
| `sse` | Server-Sent Events + OAuth |
| `sse-ide` | IDE 内部 SSE |
| `http` | Streamable HTTP (MCP 2025-03-26) |
| `ws` | WebSocket |
| `ws-ide` | IDE WebSocket |
| `sdk` | SDK 占位符 |
| `claudeai-proxy` | Anthropic 代理 |

---

### config.ts —— MCP 配置合并

**环境变量扩展**：`${VAR}` 在 command/args/URL/headers 中扩展，追踪缺失变量。

**去重**：`getMcpServerSignature()` 计算 `stdio:[cmd,args]` 或 `url:unwrappedUrl`。插件重复被抑制。

**MCP 企业策略**：
```
1. 检查 denylist（优先）→ denied
2. 无 allowlist → 全部允许
3. 空 allowlist → 全部阻止
4. stdio 必须匹配命令
5. 远程必须匹配 URL（支持通配符）
6. 回退到名称匹配
```

---

### MCP Client —— 连接生命周期

**超时 + 重连**：
```ts
// 连接超时 race
const connectPromise = client.connect(transport)
const timeoutPromise = setTimeout(() => reject(new Error('timed out')), getConnectionTimeoutMs())

// 连续终端错误触发重连
MAX_ERRORS_BEFORE_RECONNECT = 3

// 优雅关闭（stdio）
SIGINT (100ms) → SIGTERM (100ms) → SIGKILL
```

**On-close 缓存清理**：
```ts
client.onclose = () => {
  fetchToolsForClient.cache.delete(name)
  fetchResourcesForClient.cache.delete(name)
  fetchCommandsForClient.cache.delete(name)
  connectToServer.cache.delete(key)
}
```

**Auth 失败处理**：
- 401 → server 设为 `needs-auth` 状态
- `mcp-needs-auth-cache.json`（15 分钟 TTL）防止重复提示
- Claude.ai proxy 特殊包装：OAuth refresh + 单次 401 重试

---

### 读完后应抓住的 4 个事实

1. **6 级合并，policy 是"第一个胜出"**——普通设置是深合并（从低到高），但 policy settings 是第一个非空来源胜出（不合并）。这是设计：高优先级策略不应被低优先级意外覆盖。

2. **安全读取排除 projectSettings**——`hasAutoModeOptIn()` 等函数刻意不读 `projectSettings`——防止恶意项目文件自动绕过安全对话框（RCE 风险）。

3. **MCP drop-in 级联关闭**——SIGINT → 100ms → SIGTERM → 100ms → SIGKILL，50ms 轮询间隙。这是优雅的三阶清理。

4. **Managed settings .d/ 约定**——`managed-settings.d/*.json` 按字母序合并，允许不同团队（安全、OTel、合规）独立发布策略片段。

---

## 第 145 站：Tool 接口与注册系统（4 个文件，~1.5K）

### 这是什么

Tool 接口定义了每个工具必须实现的 ~50+ 属性和方法。注册系统将几十种内置工具与 MCP 工具合并为最终的工具池。

---

### Tool.ts —— 核心类型

**Tool 接口 ~230 行，~50+ 属性**：

| 类别 | 属性 |
|---|---|
| 身份 | `name`, `aliases`, `searchHint` |
| 执行 | `call(input, context, canUseTool, parentMessage, onProgress)` |
| Schema | `inputSchema` (Zod), `inputJSONSchema` (MCP), `outputSchema` |
| 描述 | `description()`, `prompt()` |
| 安全 | `isConcurrencySafe()`, `isReadOnly()`, `isDestructive()`, `checkPermissions()`, `validateInput()` |
| UI | `renderToolUseMessage()`, `renderToolResultMessage()`, `renderToolUseProgressMessage()` |
| 延迟加载 | `shouldDefer`（需要 ToolSearch 加载）, `alwaysLoad`（不延迟） |
| MCP | `isMcp`, `mcpInfo: {serverName, toolName}` |
| 中断 | `interruptBehavior()` → `'cancel'` 或 `'block'` |

**`buildTool` factory** 填充默认值：
```ts
const TOOL_DEFAULTS = {
  isConcurrencySafe: () => false,  // 默认失败关闭（fail-closed）
  isReadOnly: () => false,
  isDestructive: () => false,
  checkPermissions: (input) => ({ behavior: 'allow', updatedInput: input }),
}
```

**ToolUseContext** 携带：`options`（commands/flags/model/tools/MCP/agents/budget）、`abortController`、`getAppState/setAppState`、`setToolJSX`、`addNotification`、`messages`、`requestPrompt`、`queryTracking`。

---

### tools.ts —— 工具注册与组装

**条件导入（三种门）**：
1. `process.env.USER_TYPE === 'ant'`——内部工具（REPLTool, SuggestBackgroundPRTool, ConfigTool, TungstenTool）
2. `feature('FLAG_NAME')`——实验性/beta 工具（SleepTool, MonitorTool...）
3. `require()`——懒加载（TeamCreateTool, SendMessageTool——防止循环依赖）

**`getAllBaseTools()`** 是源文件真相。 notable gating:
- GlobTool/GrepTool 在 `hasEmbeddedSearchTools()` 时跳过（ant-native 用 bfs/ugrep）
- Task V2 工具在 `isTodoV2Enabled()` 时替换旧版
- Team 工具在 `isAgentSwarmsEnabled()` 时启用
- TestingPermissionTool 仅在 `NODE_ENV === 'test'`

**`getTools(permissionContext)`** 运行时过滤：
1. Simple mode（`CLAUDE_CODE_SIMPLE`）→ 仅 `[BashTool, FileReadTool, FileEditTool]`
2. Normal mode → 移除特殊工具、按 deny rules 过滤、隐藏 REPL-only 工具

**`assembleToolPool(permissionContext, mcpTools)`** 最终的内置+MCP 合并：
```ts
1. getTools() 获取内置
2. filterToolsByDenyRules() 过滤 MCP
3. uniqBy(..., 'name') 去重（内置优先）
4. 排序保证 prompt-cache 稳定性
```

**排序的深层原因**：内置工具作为连续前缀，MCP 工具跟随。服务器的 `claude_code_system_cache_policy` 在最后一个内置工具后放置缓存断点——如果混合排列会使缓存键失效。

---

### toolPool.ts —— Coordinator 模式过滤

**`applyCoordinatorToolFilter(tools)`**——coordinator 只需要编排工具，不要 worker 工具。

**`mergeAndFilterTools(initialTools, assembled, mode)`**——纯函数，在 React-free 文件中（`print.ts` 导入不需要拉 React/Ink）。

---

### context.ts —— 上下文注入

**System Context**：
- Git 状态（分支、main branch、前 5 个 commit）
- 2000 字符截断，超出时提示用 BashTool `git status`
- Ant-only cache-breaking 注入

**User Context**：
- CLAUDE.md 文件（目录树遍历读取）
- 当天日期
- CLAUDE.md 内容缓存在 `.gitignore`（内存目录）

---

### 读完后应抓住的 3 个事实

1. **~50+ 属性的 Tool 接口**——每个工具必须实现 call/schema/permissions/permission rendering/error handling/interrupt。`buildTool` factory 填充安全默认值，`isConcurrencySafe` 默认 `false`（fail-closed）。

2. **Prompt-Cache 稳定性排序**——工具池合并时，内置工具必须作为连续前缀出现。缓存断点在最后内置工具之后，混合排列会使缓存键失效。这不是风格选择，是缓存正确性问题。

3. **三种门决定工具可见性**——`USER_TYPE === 'ant'`（构建时人员门）、`feature()`（构建时功能门）、`isEnabled()`（运行时启用检查）。一个工具必须通过三种门才能在 API 中出现。
