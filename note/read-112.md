## 第 112 站：`src/tools/WebSearchTool/`（1 个主文件，~32K）

### 这是什么

WebSearchTool 是网络搜索工具——与 WebFetchTool 抓取特定 URL 不同，它执行通用搜索引擎查询。关键区别：它不直接调用搜索引擎 API，而是通过 `queryModelWithStreaming()` 委托 Anthropic 的 API 服务器来执行搜索。搜索由 Claude 服务端完成，客户端只接收结果。

---

### 输入 Schema
```ts
{
  query: string           // 必填：至少 2 字符
  allowed_domains?: string[]   // 只包含这些域名的结果
  blocked_domains?: string[]   // 排除这些域名的结果
}
```

### 输出 Schema
```ts
{
  query: string
  results: (SearchResult | string)[]   // 搜索结果 或 文本摘要
  durationSeconds: number
}
```

---

### 架构模式：服务端搜索，非客户端 API

WebSearchTool **不直接调用任何搜索 API**。流程是：
1. 构建 `web_search_20250305` tool schema
2. 作为 `extraToolSchema` 传给 `queryModelWithStreaming()`
3. Anthropic API 服务器执行实际搜索
4. 客户端流式接收 `server_tool_use` 和 `web_search_tool_result` 事件
5. 提取并格式化结果

---

### 完整执行流程

```
1. 构建 web_search tool schema (allowed_domains, blocked_domains, max_uses=8)
2. 创建 userMessage: "Perform a web search for the query: {query}"
3. 选择模型:
   - Haiku toggle (tengu_plum_vx3): getSmallFastModel() + no thinking
   - Normal: mainLoopModel + thinkingConfig
4. queryModelWithStreaming({ extraToolSchemas: [toolSchema], toolChoice: 'web_search' })
5. 流式处理事件:
   - server_tool_use → 记录 toolUseId
   - input_json_delta → 累积 JSON，提取 query 做进度更新
   - web_search_tool_result → 发送进度事件
   - assistant message → 累积内容
6. makeOutputFromSearchResponse() 解析结果
7. 返回 { query, results, durationSeconds }
```

---

### 启用条件

- **firstParty**（直接 Anthropic API）→ 始终启用
- **Vertex** → 仅 Claude 4.x 模型（opus-4, sonnet-4, haiku-4）
- **Foundry** → 始终启用
- **其他提供商** → 不启用

---

### 流式事件处理

WebSearchTool 通过三种自定义流事件跟踪进度：

| 事件 | 目的 |
|---|---|
| `server_tool_use` | 记录 toolUseId 用于关联 |
| `input_json_delta` | 从 partial_json 提取 query 更新进度 |
| `web_search_tool_result` | 结果到达时发送 `search_results_received` 进度 |

Progress counter 递增，每次新查询或新结果都会触发 UI 更新。

---

### 结果解析器

`makeOutputFromSearchResponse()` 处理混合 block 序列：
- `text` → 累积文本摘要
- `server_tool_use` → 跳过（标记）
- `web_search_tool_result` → 提取 hits（title + url）

结果数组可以是交错的内容块：字符串（文本评论） + SearchResult 对象（带 URL 搜索结果）。

---

### 输出格式化

`mapToolResultToToolResultBlockParam()` 将结果格式化为：
```
Web search results for query: "..."

Links: [{"title":"...", "url":"..."}]

REMINDER: You MUST include the sources above in your response using markdown hyperlinks.
```

输出中**强制要求模型在回复中包含来源链接**——这是一个 prompt 级的约束，确保模型引用的结果有据可查。

---

### 验证

```ts
// 最少查询长度
query.min(2)

// 不能同时指定 allowed_domains 和 blocked_domains
if (allowed_domains?.length && blocked_domains?.length) → error
```

允许/阻止域名不能混合，防止配置歧义。

---

### 最大搜索次数

```ts
max_uses: 8  // Hardcoded to 8 searches maximum
```

单次工具调用最多允许 8 次搜索。这防止搜索循环或过度使用搜索 API 产生不必要的费用。

---

### 读完后应抓住的 5 个事实

1. **搜索由 Anthropic API 执行，不在客户端**——WebSearchTool 只是将请求发送到 API 服务器，由服务器端搜索执行。客户端只负责接收和格式化结果。这与 WebFetchTool（客户端直接发 HTTP 请求）形成鲜明对比。

2. **流式工具事件是进度追踪机制**——`server_tool_use`、`input_json_delta`、`web_search_tool_result` 事件让 UI 可以显示实时进度（"正在搜索 X"、"收到 N 个结果"）。没有这些事件，模型调用搜索时 UI 只有空白动画。

3. **Haiku toggle 可以优化搜索成本**——`tengu_plum_vx3` gate 允许使用 Haiku 而非主模型执行搜索，并且思考模式被禁用（`thinkingConfig: { type: 'disabled' }`）。搜索是直白的不需要推理，所以小模型足够。

4. **结果格式是混合的**——一个查询的结果不是单纯的 URL 列表或单纯的文本，而是交替的 SearchResult 和 string 数组。这是因为 Anthropic API 返回搜索结果时夹杂了 Claude 的评论文本和搜索结果。解析器需要正确跟踪这些状态。

5. **最大搜索次数是 8**——防止搜索循环或过度消耗 API 预算。同时 allowed_domains/blocked_domains 不能同时指定，防止逻辑冲突。
