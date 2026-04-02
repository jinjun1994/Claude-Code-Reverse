## 第 110 站：`src/tools/MCPTool/`（4 个文件，~76K）

### 这是什么

MCPTool 是 Model Context Protocol 工具的基础模板——与具体工具（如 FileEditTool 有完整实现）不同，MCPTool 是一个薄层适配器。实际的 MCP 工具（如 `mcp__github__search_code`、`mcp__slack__send_message`）在运行时由 `src/services/mcp/client.ts` 中的 `fetchToolsForClient()` 动态创建，通过 spread `MCPTool` 并覆盖关键字段。

---

### MCPTool 模板：核心结构

```ts
export const MCPTool = buildTool({
  isMcp: true,
  name: 'mcp',                         // 被 mcpClient.ts 覆盖
  async description() { return DESCRIPTION },   // 被覆盖
  async prompt() { return PROMPT },             // 被覆盖
  inputSchema: z.object({}).passthrough(),      // 接受任意输入
  outputSchema: z.string(),                     // 输出是字符串
  async call() { return { data: '' } },         // 被覆盖
  async checkPermissions() {
    return { behavior: 'passthrough', message: 'MCPTool requires permission.' }
  },
  userFacingName: () => 'mcp',           // 被覆盖
  // ... UI renderers
})
```

MCPTool 本身不做任何事情——它是作为所有 MCP 工具的基类，被 spread 到具体工具对象中。

---

### MCP 工具适配链（`fetchToolsForClient()` in `client.ts`）

```ts
// 对于 MCP server 返回的每个 tool：
const adaptedTool = {
  ...MCPTool,
  name: `mcp__${serverName}__${toolName}`,     // 完全限定名
  mcpInfo: { serverName, toolName },
  async description() { return tool.description ?? '' },
  async prompt() { return truncatedDescription },
  inputJSONSchema: tool.inputSchema,
  async call(args, context, _canUseTool, parentMessage, onProgress) {
    return callMCPToolWithUrlElicitationRetry(args, context, onProgress)
  },
  userFacingName() {
    return `${client.name} - ${tool.annotations?.title || tool.name} (MCP)`
  },
  isOpenWorld() { return tool.annotations?.openWorldHint ?? false },
  isDestructive() { return tool.annotations?.destructiveHint ?? false },
  isReadOnly() { return tool.annotations?.readOnlyHint ?? false },
  isSearchOrReadCommand() {
    return classifyMcpToolForCollapse(client.name, tool.name)
  },
}
```

---

### MCP 提示机制

#### URL Elicitation Retry
```ts
callMCPToolWithUrlElicitationRetry()
```

处理 MCP 错误 `-32042`（URL 需要用户确认）——当 MCP server 需要一个 URL 但模型发送的是错误的或占位的，这个错误会触发重试，让模型重新提供正确的 URL。

#### 实际工具调用
```ts
callMCPTool() {
  // 1. 设置进度日志间隔（5 秒间隔）
  // 2. 应用配置超时
  // 3. client.callTool({ name, arguments, _meta }, onprogress)
  // 4. 检查 result.isError
  // 5. processMCPResult() 格式化结果
  // 6. 处理 401 auth 错误、session expired、connection closed
}
```

---

### 分类系统：`classifyForCollapse.ts`

#### 目的
MCP 工具搜索结果在 UI 中被折叠（collapsed），以便不会占据过多界面空间。`classifyMcpToolForCollapse()` 返回 `{ isSearch, isRead }` 来决定折叠行为。

#### 搜索工具（~100 个）
```ts
const SEARCH_TOOLS = new Set([
  'search_files',          // Filesystem MCP
  'brave_web_search',      // Brave Search
  'search_code',           // GitHub MCP
  'search_pubmed',         // PubMed
  'search_dashboards',     // Grafana
  'search_logs',           // Datadog
  'search_jira_issues_using_jql',  // Jira
  ...
])
```

#### 读取工具（~300 个）
```ts
const READ_TOOLS = new Set([
  'read_file',             // Filesystem MCP
  'get_file_contents',     // Various
  'list_directory',        // Filesystem MCP
  'query',                 // Database MCP
  'get_issue',             // GitHub/GitLab
  'get_pull_request',      // GitHub
  'get_user',              // Various
  ...
])
```

#### 规范化
```ts
function normalize(name: string): string {
  // camelCase/kebab-case → snake_case
  return name.replace(/([a-z])([A-Z])/g, '$1_$2')
             .replace(/-/g, '_')
             .toLowerCase()
}
```

未知工具名默认不折叠（保守行为）。

---

### MCP 注解映射

MCP 工具定义中的 `annotations` 字段直接映射到工具属性：

| MCP Annotation | 工具属性 |
|---|---|
| `openWorldHint` | `isOpenWorld()` |
| `destructiveHint` | `isDestructive()` |
| `readOnlyHint` | `isReadOnly()` |
| `title` | `userFacingName()` |

这是 MCP 规范的一部分——服务器在声明工具时应提供这些 hints。

---

### 结果处理：`processMCPResult()`

```ts
function processMCPResult(result: CallToolResult_v20250618_0): string {
  // 处理 content array
  // 处理 TextContent、ImageContent、ResourceContents 等类型
  // Image 内容: [IMAGE: data:image/png;base64,...] 格式
  // 错误: result.isError → throw McpToolCallError
}
```

MCP 结果可以是多部分（`content: [{ type: 'text', ... }, { type: 'image', ... }]`）——需要统一转换为可读文本。

---

### UI.tsx（51K）

MCP 工具的大规模渲染组件组——处理：
- 工具调用的 UI 显示
- 进度更新（带间隔动画）
- 结果格式化
- 错误显示

---

### 命名约定

```ts
// mcpStringUtils.ts
function buildMcpToolName(serverName: string, toolName: string): string {
  return `mcp__${serverName}__${toolName}`
}
// e.g., "mcp__github__search_code"
// e.g., "mcp__slack__send_message"
```

完全限定名确保即使不同服务器有相同工具名也不会冲突。

---

### 文件结构

| 文件 | 用途 |
|---|---|
| `MCPTool.ts`（2.2K） | 基础模板模板——所有 MCP 工具的基类 |
| `classifyForCollapse.ts`（15K） | 搜索/读取工具分类（~400 工具名） |
| `UI.tsx`（51K） | MCP 工具 UI 渲染 |
| `prompt.ts`（119B） | 通用 prompt 常量（被覆盖） |

---

### 元

问题：**这一站真正想解决的架构问题是什么？**

回答：这一站的关键不在具体某个 MCP 工具，而在提供一个薄适配模板，让外部 server 暴露的工具能被动态挂进 Claude Code。名称覆写、注解映射、URL elicitation retry 和结果规范化，体现的是“运行时接入”的架构思路。

### 反

问题：**如果把这一站的设计反过来，会发生什么？**

回答：如果每个 MCP 工具都写成内置特例，扩展性会立刻塌掉，新 server 每加一种能力都要改核心代码。那会把 MCP 从协议变回硬编码插件清单。

### 空

问题：**跳出当前文件名，这一站背后更大的问题是什么？**

回答：更大的问题是，宿主系统如何给未知外部能力留出统一接入口。MCPTool 的价值正在于它几乎什么都不做，却让大量外部工具以同一种方式被理解和治理。

### 读完后应抓住的 5 个事实

1. **MCPTool 是一个动态适配模板，不是具体工具**——MCPTool.ts 本身几乎没有功能。真正的 MCP 工具在 `fetchToolsForClient()` 中由 MCP server 的 `tools/list` 响应动态构建。每个 MCP 工具通过 spread MCPTool 获得基类，然后覆盖 name、schema、description、call 等属性。这是为了让 Claude Code 能运行任何 MCP 服务器暴露的工具，而不需要为每个工具编写专门代码。

2. **输入 schema 是 `z.object({}).passthrough()`**——MCP 工具接受任何输入对象，因为每个 MCP 服务器的工具都有自己的 JSON Schema。验证在 MCP 服务器侧，而不是 Claude Code 侧。这是设计选择：MCP 工具的定义和约束来自服务器，适配层不做强校验。

3. **分类系统有 ~400 个已知工具名**（~100 搜索 + ~300 只读）——这使 UI 可以正确折叠搜索结果。比如 Brave 搜索和 GitHub 搜索会被折叠显示，而不是展开整个响应。未知工具默认不折叠——这是保守但安全的选择，防止新工具意外地过度折叠导致信息隐藏。

4. **URL Elictation Retry 处理 MCP `-32042` 错误**——当 MCP 服务器需要一个 URL 但模型发送了占位值或错误的 URL 时，这个错误代码指示重试。框架捕获它并允许模型重新提供正确的 URL 值。这是人机协作的关键机制：MCP 服务器需要用户的上下文信息，而模型在首次尝试时不知道。

5. **MCP 注解直接映射到工具属性**——`openWorldHint`、`destructiveHint`、`readOnlyHint`、`title` 从 MCP server 的工具声明中直接提取并转换为工具的行为属性。这是 MCP 规范的设计：服务器告诉客户端工具的性质，客户端使用这些 hints 做权限决策和 UI 显示。
