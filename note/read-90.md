## 第 90 站：`src/tools/AgentTool/runAgent.ts`

### 这是什么文件

`runAgent.ts`（974 行）是子代理的执行引擎：构建完整的子代理运行时环境（MCP / hooks / skills / 权限 / 上下文隔离），然后通过 `query()` 启动主循环，将每次迭代的产出作为 async generator 逐条 yield 回 AgentTool。

---

### 核心职责

`runAgent()` 是一个 `AsyncGenerator<Message, void>`：

```
初始化环境 → query(主循环) → for...await → yield 每条 message → finally 清理
```

---

### 初始化链（11 步）

#### 1. 模型解析
```ts
resolvedAgentModel = getAgentModel(
  agentDefinition.model,    // agent frontmatter 定义
  toolUseContext.options.mainLoopModel,  // 父代理模型
  model,                     // AgentTool 调用时的 model 参数
  permissionMode            // 权限模式影响 haiku 选择
)
```

#### 2. 消息准备
- Fork 路径：`forkContextMessages`（父代理完整历史）+ `promptMessages`（子代理指令）
- 非 Fork 路径：仅 `promptMessages`
- `filterIncompleteToolCalls()` 剔除缺少 `tool_result` 配对的 `tool_use` 块

#### 3. 文件状态缓存
- Fork 路径：`cloneFileStateCache(parent)` — 共享父代理已读取文件的缓存
- 普通路径：`createFileStateCacheWithSizeLimit()` — 全新空缓存

#### 4. 上下文精简（token 优化）
- `omitClaudeMd`：Explore/Plan 等只读代理省略 CLAUDE.md 中的 commit/PR/lint 规则（省 ~5-15 Gtok/周）
- `omitGitStatus`：Explore/Plan 省略 stale gitStatus（省 ~1-3 Gtok/周）

#### 5. 权限上下文代理
`agentGetAppState()` 包装父 `getAppState()`，覆盖：
- `mode`：用 agent 定义的 `permissionMode`（但 bypass/acceptEdits/auto 优先）
- `shouldAvoidPermissionPrompts`：异步代理默认 true
- `awaitAutomatedChecksBeforeDialog`：后台代理在弹权限前先跑自动检查
- `alwaysAllowRules.session`：用 `allowedTools` 隔离替代父级 session 规则
- `effortValue`：agent 定义的 effort 级别

#### 6. 工具解析
```ts
resolvedTools = useExactTools
  ? availableTools            // fork 路径直接使用父工具池
  : resolveAgentTools(...)   // 普通路径经过过滤/通配符解析
```

#### 7. 系统提示构建
```ts
agentSystemPrompt = override?.systemPrompt
  ?? enhanceSystemPromptWithEnvDetails(
       [agentDefinition.getSystemPrompt()],
       resolvedAgentModel,
       additionalWorkingDirectories,
       enabledToolNames
     )
```

#### 8. 中止控制器
- 异步代理：创建新的 `AbortController`（独立于父级）
- 同步代理：共享父级 `abortController`

#### 9. SubagentStart hooks
```ts
for await (const hookResult of executeSubagentStartHooks(...)) {
  additionalContexts.push(...hookResult.additionalContexts)
}
```
hook 注入的结果作为 `attachment` 消息追加到初始消息。

#### 10. Frontmatter hooks 注册
```ts
if (agentDefinition.hooks && hooksAllowedForThisAgent) {
  registerFrontmatterHooks(..., isAgent=true)  // Stop → SubagentStop
}
```

#### 11. Skill 预加载
```ts
for (const skillName of agentDefinition.skills ?? []) {
  const content = await skill.getPromptForCommand('')
  initialMessages.push(createUserMessage({ content, isMeta: true }))
}
```
支持三种解析策略：精确匹配 → 插件前缀匹配 → 后缀匹配。

#### 12. MCP 服务器初始化
```ts
const { clients, tools, cleanup } = initializeAgentMcpServers(
  agentDefinition,
  toolUseContext.options.mcpClients  // 父级客户端
)
```
- 字符串引用：复用父级 memoized 客户端
- 内联定义：新建连接，仅 agent 生命周期结束后清理
- 合并后去重 `uniqBy([...resolvedTools, ...agentMcpTools], 'name')`

---

### AgentOptions 构建

子代理的 `ToolUseContext.options`：

| 字段 | 值 |
|---|---|
| `isNonInteractiveSession` | useExactTools 继承父级，否则异步=true/同步=false |
| `tools` | allTools（解析 + MCP 合并） |
| `commands` | `[]`（无 CLI 命令） |
| `mainLoopModel` | resolvedAgentModel |
| `thinkingConfig` | useExactTools 继承父级，否则 disabled |
| `mcpClients` | mergedMcpClients |
| `querySource` | 仅 useExactTools 时传递（fork 递归保护） |

---

### 主循环执行

```ts
for await (const message of query({
  messages: initialMessages,
  systemPrompt: agentSystemPrompt,
  userContext: resolvedUserContext,
  systemContext: resolvedSystemContext,
  canUseTool,
  toolUseContext: agentToolUseContext,
  querySource,
  maxTurns: maxTurns ?? agentDefinition.maxTurns,
})) {
  // 过滤 stream_event（只向前推送 TTFT 指标）
  // 过滤 attachment（如 max_turns_reached）
  // 记录 transcript（侧链持久化）
  // yield 给 AgentTool
}
```

---

### 清理链（finally 块）

1. `mcpCleanup()` — 关闭 agent 专属 MCP 连接
2. `clearSessionHooks(agentId)` — 移除 agent 的 frontmatter hooks
3. `cleanupAgentTracking(agentId)` — 清理 prompt cache 跟踪状态
4. `readFileState.clear()` — 释放克隆的文件状态缓存
5. `initialMessages.length = 0` — 释放 fork 上下文消息
6. `unregisterPerfettoAgent(agentId)` — 释放性能追踪
7. `clearAgentTranscriptSubdir(agentId)` — 释放转录目录映射
8. `root.todos[agentId]` 删除 — 防止 agent 泄漏积累的内存泄漏
9. `killShellTasksForAgent(agentId)` — 杀死子代理的后台 shell 任务
10. `killMonitorMcpTasksForAgent(agentId)` — 杀死 MCP 监控任务

---

### `filterIncompleteToolCalls()`

过滤掉有 `tool_use` 但没有对应 `tool_result` 的 assistant message：
1. 遍历所有 user message，收集已有的 `tool_result.tool_use_id`
2. 剔除包含没有配对 result 的 `tool_use` 的 assistant message
3. 防止 API 报错孤儿 tool call

---

### `resolveSkillName()`

skill 名称解析有三种策略：
1. 精确匹配：`hasCommand(skillName)` — 检查 name / userFacingName / aliases
2. 插件前缀：`pluginPrefix:skillName` — 从 agentType 提取插件前缀
3. 后缀匹配：找任意以 `:skillName` 结尾的命令

---

### 读完这一站后，你应该抓住的 6 个事实

1. `runAgent()` 是一个 async generator——它不直接返回结果，而是逐条 yield query() 主循环产出的 message。AgentTool（或 resumeAgent）通过 `for await (const msg of runAgent())` 消费这些消息来积累 agent 产出。
2. 初始化链 12 步中有 4 步专注于 token 优化：cloneFileStateCache（复用父代理已读文件缓存）、omitClaudeMd（只读代理跳过 CLAUDE.md 冗余信息）、omitGitStatus（Explore/Plan 省略 stale git 状态）、thinkingConfig disabled（非 fork 子代理不启用思考模式，控制输出 token 成本）。
3. 权限上下文通过一个代理函数 `agentGetAppState()` 实现动态覆盖——父级 session 规则被替换为 `allowedTools`（隔离），但 `cliArg` 级别规则（SDK `--allowedTools`）被保留。
4. 清理链非常长（10 步），涵盖 MCP 连接、hooks、prompt cache 跟踪、文件缓存、perfetto 追踪、转录目录、todos、shell 任务、监控任务——每步都是资源释放，避免 whale 会话中数百个子代理产生累积泄漏。
5. `filterIncompleteToolCalls()` 是 fork 子代理安全性的重要保障——复制父代理消息历史时可能包含未完成的 tool_use 块（父代理仍在循环中），这些块直接发送到 API 会报错。
6. `useExactTools` 是 fork 路径的关键开关：它跳过工具过滤、继承父级 thinkingConfig 和 isNonInteractiveSession、传递 querySource（递归 fork 保护），目的是让fork 子代理的 API 请求前缀与父代理字节相同，触发 prompt cache hit。
