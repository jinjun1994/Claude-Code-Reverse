## 第 146 站：AppState、OAuth 与类型系统（6 个文件，~2.5K）

### 这是什么

AppState 是全局状态容器，OAuth 是认证流，权限类型定义决定整个权限系统的类型安全边界，MCP 字符串工具处理 `mcp__server__tool` 双向解析。

---

### AppStateStore

**DeepImmutable + plain 混合**：

```ts
type AppState = DeepImmutable<{
  // 核心运行时字段（冻结防止意外修改）
  settings, toolPermissionContext, teamContext, ...
}> & {
  // 需要函数类型的可变字段
  tasks: { [taskId: string]: TaskState }
  mcp, plugins, agentNameRegistry: Map<...>
}
```

`DeepImmutable` 冻结大部分状态，但 `tasks`/`mcp`/`plugins`/`agentNameRegistry` 被排除——它们有函数类型或需要可变语义。

**关键状态段**：

| 段 | 用途 |
|---|---|
| `settings`, `toolPermissionContext` | 会话配置 |
| `tasks` | 统一任务注册表 |
| `mcp` | 客户端、工具、命令、资源、`pluginReconnectKey` |
| `replBridge*` | 本地 CLI ↔ claude.ai 网页双向连接 |
| `speculation` | 推测性下一轮执行 |
| `teamContext` | 代理协作协调 |
| `computerUseMcpState` | Chicago MCP 状态 |
| `inbox` | 代理间消息队列 |
| `workerSandboxPermissions` | Worker sandbox 网络访问批准 |

**`pluginReconnectKey` 是巧妙的响应式触发器**——数值不重要，变化本身触发 MCP 连接 effect。

---

### OAuth 服务（5 文件）

**完整流程**：
```
1. AuthCodeListener（本地 HTTP 捕获 redirect_uri）
2. 生成 PKCE code_verifier + code_challenge（S256）
3. buildAuthUrl（Claude.ai vs Console 两个 IdP）
4. 双重授权路径 race：
   - browser redirect → AuthCodeListener
   - manual paste → manualAuthCodeResolver
5. exchangeCodeForTokens（PKCE verifier）
6. fetchProfile（订阅类型、速率层级）
7. 浏览器重定向到 success/error 页面
```

**Token 刷新优化**：
```ts
// 已有 profile 时跳过 fetch——每天减少 ~7M fleet-wide 请求
const haveProfileAlready =
  config.oauthAccount?.billingType !== undefined &&
  config.oauthAccount?.accountCreatedAt !== undefined &&
  existing?.subscriptionType != null &&
  existing?.rateLimitTier != null
```

**5 分钟 buffer**：`isOAuthTokenExpired()` 在过期前 5 分钟视为已过期，防止 token 在请求中途失效。

**AuthCodeListener 的 deferred redirect 模式**：
- HTTP 请求到达时捕获 `ServerResponse` 但不立即响应
- OAuth token 交换完成后才重定向浏览器到 success/error
- 这样 token 交换失败时也能重定向到 error 页面

---

### Permission Types

**三种模式层级**：
```
External: 'acceptEdits' | 'bypassPermissions' | 'default' | 'dontAsk' | 'plan'
Internal: 添加 'auto' | 'bubble'
```

`'auto'` 只在 `feature('TRANSCRIPT_CLASSIFIER')` 时存在。

**PermissionDecisionReason**（7 种）：
- `rule`——匹配 PermissionRule
- `mode`——基于模式
- `hook`——权限 hook 拦截
- `classifier`——AI 分类器决定
- `safetyCheck`——安全检查（带 `classifierApprovable` 标志）
- `subcommandResults`——复合命令逐子命令结果
- `workingDir`/`sandboxOverride`/`asyncAgent`/`permissionPromptTool`/`other`

**ToolPermissionContext**——运行时权限上下文：
```ts
{
  mode: PermissionMode
  additionalWorkingDirectories: Map
  alwaysAllowRules: RulesBySource  // { [source]?: string[] }
  alwaysDenyRules: RulesBySource
  alwaysAskRules: RulesBySource
  isBypassPermissionsModeAvailable: boolean
  strippedDangerousRules?: RulesBySource  // auto mode 中被剥离的
}
```

---

### Permission Rule Parser

**双向解析**：
```
"ToolName(content)" ↔ { toolName, ruleContent }
```

**Legacy 别名**：`Task` → `AGENT_TOOL_NAME`, `KillShell` → `TASK_STOP_TOOL_NAME` 等。确保重命名的工具仍然匹配现有的权限规则。

**未转义括号检测**（关键解析逻辑）：
```ts
// 未转义 = 前面有偶数个反斜杠
let backslashCount = 0
while (str[j] === '\\') { backslashCount++; j-- }
if (backslashCount % 2 === 0) return i
```

**转义顺序**：反斜杠 → `(` → `)`。如果先转 `(` 再转 `\` 会变成 `\\(` 而不再匹配。

---

### MCP Normalization & String Utils

**命名约定**：`mcp__serverName__toolName`

**双向解析**：
```ts
mcpInfoFromString("mcp__github__add_issue") → { serverName: "github", toolName: "add_issue" }
buildMcpToolName("github", "add_issue") → "mcp__github__add_issue"
```

**已知的 `__` 限制**：`mcp__my__server__tool` 会错误解析为 server="my", tool="server__tool"。实践中罕见。

**权限检查名称**：`getToolNameForPermissionCheck()` 使用完全限定的 `mcp__server__tool` 名——防止针对内置工具的 deny rule 意外匹配同名的 MCP 工具。

---

### 元

问题：**这一站真正想解决的架构问题是什么？**

回答：这一站把全局状态、认证流程、权限类型和 MCP 名称解析这些基础件放在一起看，强调它们共同构成运行时底座。`DeepImmutable` 与可变区的边界、PKCE OAuth、RulesBySource，都显示这里处理的是系统一致性而非业务功能。

### 反

问题：**如果把这一站的设计反过来，会发生什么？**

回答：如果这些底座概念散落在各处，状态冻结边界、认证刷新逻辑和权限类型含义都会慢慢漂移。那种漂移短期不显眼，长期却最容易把系统拖向难以维护的形态。

### 空

问题：**跳出当前文件名，这一站背后更大的问题是什么？**

回答：更大的问题是，一套复杂 CLI 需要怎样的“基础语言”来描述自己。146 站正是在构建这门语言：状态怎么存，权限怎么说，MCP 名字怎么算。

### 读完后应抓住的 4 个事实

1. **DeepImmutable  AppState**——大部分状态被 `DeepImmutable` 冻结防止意外修改。`tasks`/`mcp` 等因为函数类型排除在外。不是 Zustand store 本身——是形状和默认值的定义层。

2. **OAuth 双重授权 race**——browser redirect 和 manual paste 并行竞速，先到的赢。SDK/headless 场景中 caller 用 `skipBrowserOpen: true` 拿到两个 URL 自己处理。Profile 刷新优化每天省 ~7M 请求。

3. **PermissionRule 的 RulesBySource**——每个规则的来源（userSettings/projectSettings/localSettings/policySettings 等）都被单独追踪，使 UI 能显示规则溯源。

4. **Legacy 别名解析器支持转义内容**——权限规则内容可以含括号（如 `Bash(git commit -m "feat(auth)")`），通过反斜杠转义。Legacy 别名（Task→AgentTool, KillShell→TaskStopTool）确保旧规则不失效。
