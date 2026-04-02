## 第 111 站：`src/tools/WebFetchTool/`（5 个文件，~52K）

### 这是什么

WebFetchTool 是 URL 内容抓取工具——通过 `getURLMarkdownContent()` 获取网页内容，然后应用模型提供的 prompt 对内容进行摘要或提取。这是 Claude Code 从互联网获取信息的核心途径。

---

### 输入 Schema
```ts
{
  url: string    // 必填：必须是可以解析的 URL
  prompt: string // 必填：用于在获取内容后执行的 prompt
}
```

### 输出 Schema
```ts
{
  bytes: number         // 内容大小
  code: number          // HTTP 状态码
  codeText: string      // HTTP 状态文本
  result: string        // 应用 prompt 后的结果
  durationMs: number    // 耗时
  url: string           // 实际请求的 URL
}
```

---

### 权限机制：域名级别

#### Preapproved Host
```ts
isPreapprovedHost(hostname, pathname)
```

预批准域名列表可自动允许，包含：
- Anthropic 自有域名（platform.claude.com、modelcontextprotocol.io 等）
- 编程语言文档（docs.python.org、go.dev、doc.rust-lang.org 等）
- 框架文档（react.dev、angular.io、nextjs.org 等）
- 数据库文档（ mongodb, redis, postgresql, sqlite 等）
- 云服务文档（AWS、GCP、Azure、Kubernetes 等）

路径前缀检查（如 `github.com/anthropics`）：
```ts
// Enforce path segment boundaries:
// "/anthropics" must not match "/anthropics-evil/malware"
pathname === prefix || pathname.startsWith(prefix + '/')
```

#### 权限规则内容
权限检查使用的是 `domain:hostname` 格式（而非完整 URL）：
```ts
function webFetchToolInputToPermissionRuleContent(input) {
  const hostname = new URL(url).hostname
  return `domain:${hostname}`
}
```

权限规则链：deny → ask → allow → 默认 ask。

---

### 执行流程

```
1. getURLMarkdownContent(url, abortController)
2. 处理重定向（跨 host 重定向提示用户重试）
3. 如果是预批准 URL + text/markdown + 长度 < MAX_MARKDOWN_LENGTH
   → 直接返回内容（不用 LLM 处理）
4. 否则 → applyPromptToMarkdown(prompt, content, signal)
5. 二进制内容保存到磁盘并提示路径
6. 返回结果
```

---

### 重定向处理

当遇到跨主机重定向时（`301`、`307`、`308`），WebFetch 不会自动跟随，而是返回一个特殊的消息格式：
```
REDIRECT DETECTED:
Original URL: xxx
Redirect URL: yyy
Status: 301 Moved Permanently

请重新使用 WebFetch 调用，url: "yyy", prompt: "..."
```

这是故意的安全设计：重定向可能将流量引导到恶意主机，模型应该显式检查并决定如何处理。

---

### 提示词安全约束

```ts
prompt: `IMPORTANT: WebFetch WILL FAIL for authenticated or private URLs.
Before using this tool, check if the URL points to an authenticated
service (e.g. Google Docs, Confluence, Jira, GitHub). If so, look
for a specialized MCP tool that provides authenticated access.`
```

这个提示始终包含在 prompt 中，无论 `ToolSearch` 是否在工具列表中。如果条件性地切换会导致 API 缓存失效（连续两次 cache miss）。

---

### 缓存优化考虑

工具提示中有一个重要的优化设计：认证警告**始终包含**，即使 `ToolSearch`（可以发现更好的 MCP 工具）不在当前工具列表中。条件性地添加/移除这段前缀会使 API prompt 变化，导致 Anthropic API 缓存失效率加倍。

---

### `preapproved.ts` 安全注释

```
SECURITY WARNING: These preapproved domains are ONLY for WebFetch (GET requests only).
The sandbox system deliberately does NOT inherit this list for network restrictions,
as arbitrary network access (POST, uploads, etc.) to these domains could enable data exfiltration.
Some domains like huggingface.co, kaggle.com, and nuget.org allow file uploads
and would be dangerous for unrestricted network access.
```

这些预批准域名仅用于 WebFetch 的 GET 请求。沙箱系统不会继承这个列表——sandbox 中的网络连接仍然需要显式用户权限。

---

### 二进制内容处理

当 WebFetch 遇到二进制内容（PDF、图片等）时：
1. 内容保存到临时磁盘路径
2. 通过 MIME 类型推导文件扩展名
3. 结果中包含指向该路径的提示：`[Binary content (application/pdf, 2.5MB) also saved to /tmp/...]`

这使得即使 Haiku 的摘要不够详细，模型仍可以用 Read/Bash 工具直接检查原始文件。

---

### `utils.ts`（17K）

核心工具函数：
- `getURLMarkdownContent()` — 实际 HTTP 请求 + HTML→Markdown 转换
- `applyPromptToMarkdown()` — 使用 Haiku 模型根据 prompt 处理内容
- `isPreapprovedUrl()` — URL 预批准检查（host + path）
- `MAX_MARKDOWN_LENGTH` — markdown 内容最大长度限制

---

### 元

问题：**这一站真正想解决的架构问题是什么？**

回答：这一站不是“抓网页”这么简单，而是把单 URL 获取、域名授权、跨主机重定向和内容后处理组合成受控网络入口。它特别强调认证 URL 不该乱碰，说明 WebFetch 更像一个安全化的 GET 读取器。

### 反

问题：**如果把这一站的设计反过来，会发生什么？**

回答：如果把任意网页访问都交给 curl/wget 风格能力，重定向、认证资源和二进制落盘都会迅速把风险外溢。那样信息抓取看似灵活，其实缺少任何可信边界。

### 空

问题：**跳出当前文件名，这一站背后更大的问题是什么？**

回答：往大处看，这一站在讨论 AI 访问互联网时如何把“连得上”改造成“拿得稳”。WebFetch 的真正主题，是网络读取的最小可信面。

### 读完后应抓住的 5 个事实

1. **WebFetch 是 GET-only 的简单获取**——与 BashTool 中的 curl/wget 不同，它不能做 POST 请求或复杂交互。它只获取单个 URL 的文本或二进制内容，然后应用 prompt 提取信息。认证/私有 URL 会失败——这是为什么提示中说应该优先用 MCP 工具进行认证访问。

2. **权限检查在域名级别**——规则格式是 `domain:hostname`（`domain:react.dev`），不是完整 URL。这使得用户可以允许整个域名的所有路径，而不需要逐条 URL 授权。预批准域名跳过权限检查直接允许，但沙箱网络限制不继承这个列表。

3. **重定向是显式返回的而不是自动跟随**——跨主机重定向时，WebFetch 不跟随，而是生成一条消息让模型重新调用该工具。这是为了防止恶意重定向把流量劫持到未知主机。模型看到重定向消息后可以决定是否重试新 URL。

4. **预批准 URL + markdown 内容直接返回不经过 LLM**——当内容类型是 `text/markdown` 且大小 < `MAX_MARKDOWN_LENGTH` 时，整个内容直接返回而不调用 Haiku。这节省了 API 成本。预批准 URL 的内容被认为足够安全不需要 LLM 过滤。

5. **API 缓存稳定性是设计约束**——WebFetch 的 prompt 中始终包含认证警告，即使 ToolSearch 不在工具列表中。条件性地添加/移除这段文本会导致 API 缓存失效，每切换一次会触发两次 cache miss。这是 Claude Code 对 API 成本的细致优化。
