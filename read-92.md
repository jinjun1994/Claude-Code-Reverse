## 第 92 站：`src/tools/AgentTool/loadAgentsDir.ts`

### 这是什么文件

`loadAgentsDir.ts`（755 行）是 agent 定义加载和解析中枢：从 6 个来源（built-in / plugin / user / project / policy / flag settings）加载 agent，解析 markdown frontmatter 或 JSON，合并去重，建立 memoized 缓存。

---

### Agent 定义的 6 个来源

| 来源 | `source` 值 | 说明 |
|---|---|---|
| Built-in | `'built-in'` | 代码内置的 agent |
| Plugin | `'plugin'` | 插件提供的 agent |
| User | `'userSettings'` | settings.json 中用户定义的 |
| Project | `'projectSettings'` | 项目级别定义的 |
| Policy | `'policySettings'` | 管理员策略定义的 |
| Flag | `'flagSettings'` | 环境变量/配置标志定义的 |

---

### 三级 Agent 类型体系

```
AgentDefinition =
  BuiltInAgentDefinition   (source: 'built-in', getSystemPrompt 带 toolUseContext 参数)
  CustomAgentDefinition    (source: 'userSettings'/'projectSettings'/'policySettings'/'flagSettings', getSystemPrompt 无参数)
  PluginAgentDefinition    (source: 'plugin', 带 plugin 字段)
```

区别：`BuiltInAgentDefinition` 的 `getSystemPrompt` 接收 `{ toolUseContext }` 参数以动态构建提示；`Custom`/`Plugin` 的 `getSystemPrompt` 通过闭包捕获静态 prompt 文本。

---

### `BaseAgentDefinition` 字段全览

| 字段 | 用途 |
|---|---|
| `agentType` | 唯一标识 |
| `whenToUse` | 能力描述——model 选择 agent 时的依据 |
| `tools?` | 允许的工具列表（支持通配符 `*`） |
| `disallowedTools?` | 禁止的工具 |
| `skills?` | 预加载的 skill 列表 |
| `mcpServers?` | 代理专属 MCP 服务器（引用名或内联定义） |
| `hooks?` | 代理生命周期内的 session-scoped hooks |
| `color?` | UI 显示颜色 |
| `model?` | 模型（`sonnet/opus/haiku/inherit`） |
| `effort?` | 投入级别 |
| `permissionMode?` | 权限模式 |
| `maxTurns?` | 最大轮数 |
| `background?` | 是否始终后台运行 |
| `isolation?` | 隔离级别（`worktree` 或 `remote`—ant-only） |
| `memory?` | 持久内存作用域（`user/project/local`） |
| `requiredMcpServers?` | MCP 前置依赖检查 |
| `initialPrompt?` | 首轮追加的开场 prompt |
| `omitClaudeMd?` | 是否省略 CLAUDE.md（只读代理省 token） |
| `criticalSystemReminder_EXPERIMENTAL?` | 每轮重新注入的系统提醒 |

---

### 加载链

```ts
getAgentDefinitionsWithOverrides(cwd) — memoized

1. 如果 CLAUDE_CODE_SIMPLE → 仅返回 built-in agents
2. loadMarkdownFilesForSubdir('agents', cwd) — 加载用户/项目 markdown 文件
3. parseAgentFromMarkdown() — 解析每个文件为 CustomAgentDefinition
4. loadPluginAgents() — 并发加载插件 agent
5. initializeAgentMemorySnapshots() — 如果有 agent 内存快照，并发初始化
6. getBuiltInAgents() — 加载内置 agent
7. 合并 [builtIn, plugin, custom]
8. getActiveAgentsFromList() — 按优先级去重
9. 初始化 UI 颜色
```

**去重逻辑**（`getActiveAgentsFromList`）：按 `agentType` 建立 Map，按 `[builtIn, plugin, user, project, flag, managed]` 顺序写入，后写的覆盖先写的——相同 agentType 时，用户定义的 agent 覆盖内置 agent。

---

### `parseAgentFromMarkdown()` 关键解析

#### 工具注入（Agent Memory）
```ts
if (isAutoMemoryEnabled() && memory && tools !== undefined) {
  // 自动注入 Write/Edit/Read——内存 agent 需要这些来读写记忆
  tools = [...tools, FILE_WRITE_TOOL_NAME, FILE_EDIT_TOOL_NAME, FILE_READ_TOOL_NAME]
}
```

#### 动态系统提示
```ts
getSystemPrompt: () => {
  // 如果启用了 agent 内存，把内存 prompt 附加到正文
  return systemPrompt + (memoryEnabled ? '\n\n' + loadAgentMemoryPrompt(...) : '')
}
```

---

### `parseAgentFromJson()`

用于 settings.json 或插件中 JSON 格式的 agent 定义，解析路径为 `AgentJsonSchema` Zod 验证后的结果。构建方式与 markdown 解析相同——`getSystemPrompt` 通过闭包捕获 prompt 文本。

---

### MCP 前置依赖

`hasRequiredMcpServers()` 检查 agent 是否依赖特定 MCP 服务器：
```ts
agent.requiredMcpServers.every(pattern =>
  availableServers.some(server =>
    server.toLowerCase().includes(pattern.toLowerCase())
  )
)
```
模式匹配而非精确匹配——`"slack"` 匹配 `"my-slack-server"`。

---

### 缓存与刷新

```ts
getAgentDefinitionsWithOverrides = memoize(async (cwd) => ...)
clearAgentDefinitionsCache() → 清除 memoized 缓存 + plugin agent 缓存
```

---

### 容错策略

- 非 agent Markdown 文件（缺少 `name` frontmatter）静默跳过
- 解析失败的文件进入 `failedFiles` 列表，不阻断其他 agent 加载
- 全局异常时至少返回 built-in agents 作为 fallback

---

### 读完这一站后，你应该抓住的 6 个事实

1. Agent 定义来自 6 个来源，按 `[builtIn, plugin, user, project, flag, managed]` 优先级去重——相同 `agentType` 时，后加载的覆盖前面的。
2. `BuiltInAgentDefinition` 与 `CustomAgentDefinition` 的核心区别在于 `getSystemPrompt` 签名——内置 agent 接收 `{ toolUseContext }` 参数动态构建提示，自定义 agent 从闭包中返回缓存提示。这是因为内置 agent 需要感知当前环境（工具池/MCP/cwd）而动态调整提示。
3. 如果启用了 agent 内存（`AGENT_MEMORY_SNAPSHOT`），加载链为每个有 `memory` 的 agent 注入 `Write/Edit/Read` 工具——即使 agent 的 frontmatter 中没有声明这些工具。这是因为内存 agent 需要读写 `.claude/memory` 文件。
4. MCP 前置依赖使用模式匹配——`requiredMcpServers: ["slack"]` 匹配任何包含 `"slack"` 的已连接服务器名（如 `"my-slack-server"`）。这是为了解决同一个 MCP 服务器可能有多个实例的场景。
5. `CLAUDE_CODE_SIMPLE` 模式下跳过所有自定义 agent 加载——直接返回 built-in agents，与 tools 子系统的 SIMPLE 模式一致。
6. 解析函数对非 agent Markdown 文件做静默兼容——缺少 `name` frontmatter 的文件可能是 co-located 参考文档，不会报错。只有有 `name` 但解析失败的文件才进入 `failedFiles`。
