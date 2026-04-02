## 第 131 站：`src/tools/REPLTool/`（2 个文件）

### 这是什么

REPLTool 不是传统意义上的工具，而是一个模式开关——当 CLI 处于 REPL 模式时，它提供 REPL VM 上下文给用户在本地执行工具，同时模型本身看不到这些工具。这是 Claude Code 的本地交互式操作模式。

---

### REPL 模式启用

```ts
function isReplModeEnabled(): boolean {
  if (isEnvDefinedFalsy(CLAUDE_CODE_REPL)) return false
  if (isEnvTruthy(CLAUDE_REPL_MODE)) return true
  return process.env.USER_TYPE === 'ant' && CLAUDE_CODE_ENTRYPOINT === 'cli'
}
```

Ant 员工且 CLI 入口 → 默认开启 REPL 模式。SDK 场景默认关闭——SDK 直接做工具调用，而不是通过 REPL 循环隐藏工具。

---

### REPL_ONLY_TOOLS

```ts
export const REPL_ONLY_TOOLS = new Set([
  FILE_READ_TOOL_NAME,
  FILE_WRITE_TOOL_NAME,
  FILE_EDIT_TOOL_NAME,
  GLOB_TOOL_NAME,
  GREP_TOOL_NAME,
  BASH_TOOL_NAME,
  NOTEBOOK_EDIT_TOOL_NAME,
  AGENT_TOOL_NAME,
])
```

这些工具在 REPL 模式下被从模型的工具列表中隐藏——模型不能直接调用它们。但它们在 REPL VM context 中仍然可访问——用户可以直接在 REPL 环境中调用它们。

### Primitive Tools

```ts
export function getReplPrimitiveTools(): readonly Tool[] {
  return (_primitiveTools ??= [
    FileReadTool, FileWriteTool, FileEditTool,
    GlobTool, GrepTool, BashTool,
    NotebookEditTool, AgentTool,
  ])
}
```

`primitiveTools` 是延迟获取——避免循环依赖（`collapseReadSearch.ts → primitiveTools.ts → FileReadTool.tsx → ... → tool registry`）。这些工具在 REPL 模式下直接注入 VM context 而不经过模型。

---

### 元

问题：**这一站真正想解决的架构问题是什么？**

回答：这一站的重点是 REPL 模式如何把一组原始工具从模型视野里隐藏起来，转而交给本地用户直接操作。它不是传统工具，更像“模型能不能看见工具”的模式开关。

### 反

问题：**如果把这一站的设计反过来，会发生什么？**

回答：如果 REPL 模式下模型仍直接持有这些 primitive tools，用户本地操作与模型自动调用会混在一起，内部交互边界会被打穿。那样 REPL 就不再是本地控制台，而只是另一种普通会话。

### 空

问题：**跳出当前文件名，这一站背后更大的问题是什么？**

回答：更大的问题是，同一套工具在不同交互模式里应不应该对模型可见。REPLTool 的存在说明，工具可见性本身就是一种能力分配。

### 读完后应抓住的 2 个事实

1. **REPL 模式是"模型隐藏工具"模式**——REPL 模式下，文件编辑、glob、grep、bash、agent 等核心工具被从模型可见列表中隐藏。用户直接在 REPL 中操作这些工具，模型只提供建议。这是 Anthropic 员工内部工作模式。

2. **SDK 不走 REPL 默认**——`USER_TYPE === 'ant'` 且 `ENTRYPOINT === 'cli'` 才默认开。SDK 用户直接调用工具，REPL 模式会阻止模型看到工具列表，这对脚本化场景不合适。

---

## 第 132 站：`src/tools/McpAuthTool/`（1 个文件，~8K）

### 这是什么

McpAuthTool 是 MCP 服务器的 OAuth 认证入口——当 MCP server 处于 `needs-auth` 状态时，Claude 不暴露其真实工具，而是创建一个伪工具 `mcp__server__authenticate` 让模型启动 OAuth 流程。

---

### 伪工具生命周期

McpAuthTool 不是静态定义的工具——而是由 `createMcpAuthTool(serverName, config)` 动态创建的伪工具：

```ts
{
  name: `mcp__${serverName}__authenticate`,
  isMcp: true,
  mcpInfo: { serverName, toolName: 'authenticate' },
  description: `The server is installed but requires authentication...`,
}
```

### 执行流程

```ts
// 1. 跳过浏览器打开
const oauthPromise = performMCPOAuthFlow(
  serverName, config,
  u => resolveAuthUrl?.(u),      // 捕获授权 URL
  controller.signal,
  { skipBrowserOpen: true },     // 不发浏览器窗口
)

// 2. 返回 URL 给用户
const authUrl = await Promise.race([
  authUrlPromise,
  oauthPromise.then(() => null),  // 或者 OAuth 已完成返回 null
])

return { status: 'auth_url', authUrl, message: 'Ask user to open...' }

// 3. 后台完成后续操作
void oauthPromise.then(async () => {
  clearMcpAuthCache()
  const result = await reconnectMcpServerImpl(serverName, config)
  // 前缀替换移除伪工具，注入真实工具
  setAppState(prev => ({
    ...prev,
    mcp: {
      tools: [
        ...reject(prev.mcp.tools, t => t.name?.startsWith(prefix)),
        ...result.tools,
      ],
      ...
    },
  }))
})
```

---

### 三种认证状态

| 状态 | 场景 | 返回 |
|---|---|---|
| `auth_url` | 启动 OAuth 成功 | 返回 URL 给用户 |
| `auth_url` (silent) | XAA 缓存 token，静默完成 | "Authentication completed silently" |
| `unsupported` | claudeai-proxy 或非 HTTP 传输 | 引导用户手动 /mcp |
| `error` | OAuth 启动失败 | 错误消息 |

### 不支持的传输类型

- `claudeai-proxy`：用单独的 `/mcp` 认证流程，不通过此工具
- `stdio` 或其他非 HTTP 传输：不支持 OAuth，提示用户手动操作

---

### 元

问题：**这一站真正想解决的架构问题是什么？**

回答：这一站把“需要认证的 MCP server”包装成一个伪工具 `mcp__server__authenticate`，先暴露认证入口，再在后台热替换成真实工具。重点不在 OAuth 细节，而在工具集如何动态变形。

### 反

问题：**如果把这一站的设计反过来，会发生什么？**

回答：如果 needs-auth 状态下仍直接暴露真实工具，模型只会不断撞上 401 与失败调用；若完全不暴露，又失去自助恢复路径。伪工具方案正好填补了这段过渡带。

### 空

问题：**跳出当前文件名，这一站背后更大的问题是什么？**

回答：更大的问题是，外部能力在“尚未就绪”时该如何被系统表述。McpAuthTool 说明，未认证不是简单报错，而是一种需要正式 UI 与工具语义承接的中间状态。

### 读完后应抓住的 3 个事实

1. **McpAuthTool 是动态伪的**——当 MCP server 需要认证时，真实的工具被屏蔽，取而代之的是 `mcp__server__authenticate` 伪工具。模型调用后，后台完成 OAuth 并自动替换为真工具。

2. **Prefix-based 工具替换**——OAuth 完成后，`reject(tools, t => t.name.startsWith(prefix))` 移除所有 `mcp__server__*` 伪工具，替换为真实工具。这是 MCP 工具热切换机制。

3. **三种认证状态**：auth_url（需要用户打开链接）、silent（XAA 缓存 token 无感认证）、unsupported（claudeai-proxy 或 stdio 需手动操作）。

---

## 第 133 站：`src/tools/ListMcpResourcesTool/`（3 个文件，~16K）

### 这是什么

ListMcpResourcesTool 列出所有连接 MCP 服务器的资源——MCP 资源是可读的、类似 URL 的条目（如 `file:///path/to/doc.md`），与 MCP 工具不同。

---

### 输入 Schema

```ts
{
  server?: string    // 可选：过滤特定服务器的资源
}
```

### 输出 Schema

```ts
{
  uri: string
  name: string
  mimeType?: string
  description?: string
  server: string
}[]
```

---

### 缓存与预取

```
fetchResourcesForClient is LRU-cached (by server name) and already
warm from startup prefetch. Cache is invalidated on onclose and on
resources/list_changed notifications.
```

资源列表在启动时预取并 LRU 缓存——`onclose` 和 `resources/list_changed` 通知时失效。`ensureConnectedClient` 是 no-op 当连接健康（memoize hit），连接断开后返回新连接。

---

### 错误隔离

```ts
const results = await Promise.all(
  clientsToProcess.map(async client => {
    try {
      const fresh = await ensureConnectedClient(client)
      return await fetchResourcesForClient(fresh)
    } catch (error) {
      // 一个服务器失败不影响其他服务器
      logMCPError(client.name, errorMessage(error))
      return []
    }
  }),
)
```

一个服务器的重连失败不会级联到其他服务器——`Promise.all` 中 catch 返回空数组而不是 throw。

---

### 元

问题：**这一站真正想解决的架构问题是什么？**

回答：这一站把 MCP 资源和 MCP 工具明确区分开来，前者是可读资源清单，后者是可执行能力。启动预取、LRU 缓存和单服务失败隔离，说明它更像一个稳定的资源索引层。

### 反

问题：**如果把这一站的设计反过来，会发生什么？**

回答：如果每次列资源都重新直连所有 server，不仅慢，还会让单点失败把整体列表一并拖垮。资源发现本该是最平稳的一环，不该跟着连接波动一起颤抖。

### 空

问题：**跳出当前文件名，这一站背后更大的问题是什么？**

回答：更大的问题是，协议世界里“可执行”和“可读取”是否要有不同入口。ListMcpResourcesTool 给出的答案很明确：资源应该先被看见，再被读取。

### 读完后应抓住的 2 个事实

1. **MCP 资源是 URL-like 可读取的**——不是 MCP 工具。ListMcpResourcesTool 列出所有服务器暴露的资源。ReadMcpResourceTool 用于读取具体内容。

2. **LRU 缓存 + 启动预取**——资源列表在启动时预取，`onclose` 和 `resources/list_changed` 通知时失效。这避免每次调用都重新拉取资源列表。

---

## 第 134 站：`src/tools/ReadMcpResourceTool/`（3 个文件，~20K）

### 这是什么

ReadMcpResourceTool（`ReadMcpResource`）读取指定的 MCP 资源——通过 MCP 协议 `resources/read` 方法获取 URI 内容。

---

### 输入 Schema

```ts
{
  server: string    // MCP 服务器名
  uri: string       // 资源的 URI
}
```

### 输出 Schema

```ts
{
  contents: {
    uri: string
    mimeType?: string
    text?: string           // 文本内容
    blobSavedTo?: string    // 二进制内容保存路径
  }[]
}
```

---

### 调用流程

```ts
// 1. 确保服务器连接
const connectedClient = await ensureConnectedClient(client)

// 2. MCP 协议调用
const result = await connectedClient.client.request(
  { method: 'resources/read', params: { uri } },
  ReadResourceResultSchema,
)

// 3. 处理结果
result.contents.map(c => {
  if ('text' in c) return { uri, mimeType, text: c.text }
  if ('blob' in c) {
    // Base64 解码 → 存磁盘 → 返回路径
    const persisted = await persistBinaryContent(
      Buffer.from(c.blob, 'base64'), c.mimeType, persistId
    )
    return { uri, mimeType, blobSavedTo: persisted.filepath }
  }
})
```

---

### 二进制内容处理

```ts
const persistId = `mcp-resource-${Date.now()}-${i}-${random}`
const persisted = await persistBinaryContent(
  Buffer.from(c.blob, 'base64'), c.mimeType, persistId
)
```

MCP 资源的二进制内容（base64 blob）被解码为原始字节并保存到磁盘——防止将大段 base64 编码内容直接注入 prompt。

---

### 前置验证

```ts
if (!client.capabilities?.resources) {
  throw new Error(`Server does not support resources`)
}
```

在读取前检查服务器是否声明支持 resources capability——这是从 MCP handshake 响应中获得的。

---

### 元

问题：**这一站真正想解决的架构问题是什么？**

回答：这一站承接的是 MCP 资源的实际读取，把 `resources/read` 结果规范成文本或落盘的二进制引用。它补齐了 133 站的另一半，让资源列表不止能看，还能真正进入上下文。

### 反

问题：**如果把这一站的设计反过来，会发生什么？**

回答：如果资源读取仍混在普通 MCPTool 调用里，文本资源与 blob 资源的处理就会缺少统一出口。尤其二进制内容若没有稳定持久化路径，资源读取很快会沦为半成品能力。

### 空

问题：**跳出当前文件名，这一站背后更大的问题是什么？**

回答：更大的问题是，协议资源一旦被读取，如何在“内容进入上下文”和“二进制留在磁盘”之间找到平衡。ReadMcpResourceTool 正是在画这条转换线。

### 读完后应抓住的 2 个 fact

1. **MCP 资源是只读 URL-like 数据**——ListMcpResourcesTool 列出它们，ReadMcpResourceTool 读取它们内容。这与 MCP 工具（可执行操作）完全不同。

2. **二进制内容存磁盘**——base64 blob 被解码为文件并存储路径到 prompt——防止大段 base64 注入 prompt。MIME 类型用于推导文件扩展名。

---

## 下一步

还剩 `shared/` 目录未处理。这是工具共享的类型定义、验证工具、和通用逻辑。完成后，整个工具系统架构的遍历就基本完成了。
