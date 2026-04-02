## 第 118 站：`src/tools/ToolSearchTool/`（3 个文件，~28K）

### 这是什么

ToolSearchTool 是延迟（deferred）工具发现工具——当模型需要但不在当前工具列表中的工具时，它可以用关键字搜索或精确选择来找到并加载这些工具。这与直接调用已加载工具不同，ToolSearchTool 本身不执行任何工具操作，只是"查找工具的工具"。

---

### 输入 Schema

```ts
{
  query: string         // "select:<tool_name>" 精确选择 或 关键字搜索
  max_results?: number  // 默认 5
}
```

### 输出 Schema

```ts
{
  matches: string[]              // 匹配的工具名
  query: string                  // 原始查询
  total_deferred_tools: number   // 可用延迟工具总数
  pending_mcp_servers?: string[] // 仍在连接的 MCP 服务器
}
```

---

### 两种搜索模式

#### Select 模式：精确选择

```ts
// query: "select:FileEditTool" 或 "select:FileEditTool,GrepTool"
const selectMatch = query.match(/^select:(.+)$/i)
```

- 支持逗号分隔的多选：`select:A,B,C`
- 先在 deferred tools 中查找，再 fallback 到完整 tools 集
- 如果工具已加载（非延迟），仍然返回——让模型继续而不重试抖动

#### Keyword 模式：关键字搜索

```ts
searchToolsWithKeywords(query, deferredTools, tools, maxResults)
```

评分系统：

| 匹配类型 | 非 MCP 分数 | MCP 分数 |
|---|---|---|
| 工具名部分精确匹配 | 10 | 12 |
| 工具名部分包含 | 5 | 6 |
| 全名回退 | 3 | 3 |
| searchHint 匹配 | 4 | 4 |
| description 匹配 | 2 | 2 |

MCP 工具名称权重更高——因为 `mcp__server__action` 格式中 server 名是更好的信号。

---

### 搜索优化

#### 快速路径：精确工具名匹配

```ts
const exactMatch = deferredTools.find(t => t.name.toLowerCase() === queryLower)
if (exactMatch) return [exactMatch.name]
```

如果查询正好是工具名，直接返回而不运行评分。处理模型使用裸工具名而不是 `select:` 前缀的情况（从子代理/压缩后见过）。

#### MCP 前缀快速路径

```ts
if (queryLower.startsWith('mcp__') && queryLower.length > 5) {
  // 查找所有以 mcp__server 开头的工具
}
```

模型经常只搜索服务器名（如 `mcp__github`）来找该服务器的所有工具。

#### 必需/可选术语

```ts
// "+slack +send" 要求同时包含 slack 和 send
// "slack send" 两者都有就加分
if (term.startsWith('+') && term.length > 1) {
  requiredTerms.push(term.slice(1))
} else {
  optionalTerms.push(term)
}
```

模型可以用 `+` 前缀标记"必须有"的术语。

#### 预编译正则模式

```ts
function compileTermPatterns(terms: string[]): Map<string, RegExp> {
  for (const term of terms) {
    patterns.set(term, new RegExp(`\\b${escapeRegExp(term)}\\b`))
  }
  return patterns
}
```

搜索前编译一次所有正则模式，避免 tool×term×2 次重复编译。

---

### Memoized 描述缓存

```ts
const getToolDescriptionMemoized = memoize(
  async (toolName: string, tools) => tool.prompt(...),
  (toolName) => toolName  // 缓存 key
)
```

工具描述异步获取（`tool.prompt()`），但按工具名缓存。这样搜索时不需要重复调用 prompt。

#### 缓存失效检测

```ts
function maybeInvalidateCache(deferredTools: Tools) {
  const currentKey = getDeferredToolsCacheKey(deferredTools)
  if (cachedDeferredToolNames !== currentKey) {
    getToolDescriptionMemoized.cache.clear?.()
  }
}
```

当延迟工具集合改变时（MCP 服务器连接/断开），自动清除描述缓存。

---

### 输出格式：tool_reference blocks

```ts
return {
  content: content.matches.map(name => ({
    type: 'tool_reference',
    tool_name: name,
  })),
}
```

当找到匹配工具时，结果使用 `tool_reference` 类型——这是 Anthropic API 的特殊格式，客户端收到后会自动加载该工具的 schema 和 prompt。当没有匹配时返回纯文本错误消息。

---

### MCP 服务器连接状态

```ts
function getPendingServerNames(): string[] | undefined {
  const pending = appState.mcp.clients.filter(c => c.type === 'pending')
  return pending.length > 0 ? pending.map(s => s.name) : undefined
}
```

当搜索无结果时，结果中包含仍在连接的 MCP 服务器列表。这告诉模型"不是没有这个工具，只是 MCP 服务器还没连好"。

---

### 延迟工具是什么

Deferred tools 是 `shouldDefer: true` 的工具——它们不在首轮工具列表中暴露给模型，而是等模型请求时才通过 ToolSearchTool 查找并加载。这减少了首轮 prompt 大小，提高 API 缓存效率。常见延迟工具包括不太常用的专用工具（如 NotebookEdit、ScheduleCron 等）。

---

### 读完后应抓住的 4 个事实

1. **两种搜索模式**：`select:ToolName` 精确选择和关键字搜索。Select 模式是确定性的——模型知道要什么就直接选。Keyword 模式是概率的——模型描述功能，ToolSearchTool 评分排序。模型通常用 select 因为它已经通过 prior knowledge 知道工具名

2. **评分系统细致**：工具名匹配权重最高（10-12 分），searchHint 次之（4 分），description 最低（2 分）。这是故意的——searchHint 是精心策划的能力短语（如 "schedule a recurring prompt"），比长 prompt 更有信号。MCP 工具的搜索权重普遍高于内置工具。

3. **Tool_reference 输出格式**：匹配结果不是纯文本而是 `tool_reference` block——Anthropic API 的特殊格式，让客户端自动加载工具 schema。这让 ToolSearchTool 不只是"搜索结果"，而是实际的工具加载机制。

4. **Memoized 描述 + 自动失效**：工具描述异步获取（`tool.prompt()` 调用 LLM 相关操作），所以必须缓存。缓存 key 是工具名列表的排序连接——当 MCP 服务器连接/断开导致工具集变化时，缓存自动失效。
