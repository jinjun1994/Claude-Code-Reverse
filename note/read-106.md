## 第 106 站：`src/tools/GrepTool/`（3 个文件，~48K）

### 这是什么

GrepTool 是 ripgrep（`rg`）的封装——提供代码库内容搜索能力。与 BashTool 中调用 `rg` 不同，它是原生工具，有更结构化的输入 schema（`output_mode`、`head_limit`、`offset`、`-A`/`-B`/`-C` 等），并且内置了 token 节省机制（路径相对化、结果截断、长度限制）。

---

### 输入 Schema
```ts
{
  pattern: string            // 必填：正则表达式
  path?: string              // 可选：搜索路径（默认 cwd）
  glob?: string              // 可选：文件过滤 glob
  output_mode?: 'content' | 'files_with_matches' | 'count'   // 默认 files_with_matches
  -A?: number                // 后上下文行数
  -B?: number                // 前上下文行数
  -C?: number                // 前后上下文行数（alias for context）
  context?: number           // 前后上下文行数
  -n?: boolean               // 显示行号（默认 true）
  -i?: boolean               // 大小写不敏感
  type?: string              // 文件类型（js, py, rust 等）
  head_limit?: number        // 结果数量限制（默认 250，0 = 无限制）
  offset?: number            // 跳过前 N 个结果（分页）
  multiline?: boolean        // 多行模式（rg -U --multiline-dotall）
}
```

---

### 架构特征

#### 只读且并发安全
```ts
isConcurrencySafe() { return true }
isReadOnly() { return true }
```

GrepTool 是只读工具，不会修改文件系统——可以与其他工具并发执行。这也是它可以被 auto-allow 在只读模式下的原因。

---

### 调用链：`call()`

```
1. 构建 ripgrep 参数数组
2. 自动添加 VCS 排除规则（.git, .svn, .hg, .bzr, .jj, .sl）
3. 添加 --max-columns 500（防止 base64/minified 内容污染输出）
4. 添加多行模式标志（如果启用）
5. 添加大小写、模式、类型标志
6. 添加上下文行标志（-C/context 优先于 -A/-B）
7. 处理以 - 开头的模式（用 -e flag 避免被当作选项）
8. 添加 glob 过滤（逗号/空格分隔 + 花括号保留）
9. 注入忽略规则（permission context + plugin cache）
10. 执行 ripGrep(args, absolutePath, abortSignal)
11. 按 output_mode 处理结果
```

---

### 输出模式处理

#### Content Mode
- ripgrep 输出格式：`/absolute/path:line:content` 或 `/absolute/path:line_content`
- **路径相对化**：`toRelativePath()` 将绝对路径转换为相对路径，节省 token
- 先应用 `head_limit` 再 relativize（避免处理将被丢弃的行）

#### Count Mode
- ripgrep 输出格式：`/absolute/path:count`
- 解析总匹配数和文件数
- 路径同样相对化

#### Files With Matches Mode（默认）
- 统计每个文件的 mtime（`Promise.allSettled` 容错——文件可能在 rg 和 stat 之间被删除）
- **按 mtime 排序**：最近修改的文件排在前面——这是关键的 UX 设计，因为模型更可能关心活跃文件
- mtime 相同时按文件名排序
- 排序后应用 `head_limit`、路径相对化

---

### Token 节省机制

#### 1. `DEFAULT_HEAD_LIMIT = 250`
无限制的 grep 可以产生 ~6-24K token 的输出（20KB persist 阈值），严重影响 context 预算。250 对大多数搜索足够，但模型可以通过 `head_limit=0` 显式绕过。

#### 2. `toRelativePath()`
绝对路径（如 `/Users/jinjun/study/Claude-Code-Reverse/src/tools/GrepTool/GrepTool.ts`）被转换为相对路径（`src/tools/GrepTool/GrepTool.ts`）——每行节省大量字符。

#### 3. `--max-columns 500`
单行超过 500 字符（如 base64 编码的数据、minified JS）会被截断，防止一行占据整个输出。

#### 4. `applyHeadLimit()` 在 relativize 之前
对宽泛模式（可能返回 10k+ 行），先截断再逐行做路径相对化——避免不必要的字符串处理。

---

### 分页机制

`head_limit` + `offset` 的组合支持结果分页：
- `head_limit: 50, offset: 0` → 前 50 个结果
- `head_limit: 50, offset: 50` → 下一页 50 个结果
- `offset` 等效于 `| tail -n +N | head -N`

当结果被截断时，输出中包含 `appliedLimit` 信息告知模型有更多结果可用。

---

### 忽略规则注入

```ts
const ignorePatterns = getFileReadIgnorePatterns(appState.toolPermissionContext)
```

GrepTool 自动注入以下忽略模式：
1. 权限上下文中的文件读取忽略规则
2. Plugin cache 孤儿目录排除（`getGlobExclusionsForPluginCache()`）
3. VCS 目录（.git, .svn, .hg, .bzr, .jj, .sl）

这确保搜索不会深入到版本控制元数据或插件缓存目录中。

---

### 路径验证

`validateInput()`：
- 如果提供了 `path`，检查路径是否存在
- UNC 路径直接跳过 fs 操作（Windows SMB 认证安全）
- 路径不存在时，`suggestPathUnderCwd()` 提供建议（模糊匹配 cwd 下的相似文件名）

---

### WSL 超时处理

```ts
// WSL has severe performance penalty for file reads (3-5x slower on WSL2)
// The timeout is handled by ripgrep itself via execFile timeout option
```

在 WSL2 上文件读取性能显著下降（3-5 倍）。超时由 ripgrep 内部处理而非 AbortController——因为中断 agent loop 不好。如果 rg 超时，抛出 `RipgrepTimeoutError` 传播到模型，模型知道搜索未完成而不是认为没有匹配。

---

### 元

问题：**这一站真正想解决的架构问题是什么？**

回答：这一站把内容搜索从 shell 命令升级成结构化检索工具，重点不在能不能搜，而在怎样控制结果规模、排序方式和输出形态。`output_mode`、分页、mtime 排序和 token 节省机制，使搜索成为面向模型的能力，而不是面向终端的噪音。

### 反

问题：**如果把这一站的设计反过来，会发生什么？**

回答：如果仍让搜索停留在原始 `rg` 文本输出，模型会频繁被大结果、长路径和无序匹配淹没。那样搜索不是帮助理解代码，而是在不断挤占上下文预算。

### 空

问题：**跳出当前文件名，这一站背后更大的问题是什么？**

回答：更大的问题是，代码搜索在 AI 系统里必须同时服务“发现”与“压缩”。GrepTool 的设计说明，检索能力本身也要为 token 经济和推理节奏让路。

### 读完后应该抓住的 5 个事实

1. **GrepTool 是结构化 ripgrep 封装，不是原始 shell 调用**——输入 schema 提供 `output_mode`、`head_limit`、`offset`、`-A`/`-B`/`-C` 等参数，输出是结构化对象（`numFiles`、`filenames`、`content`、`numMatches`）而非纯文本。这让模型可以更精确地控制搜索行为和结果量。

2. **结果按 mtime 排序**——在 files_with_matches 模式下，每个文件被 stat 获取 mtime，结果按修改时间降序排列。这意味着模型搜索结果时，最活跃的文件排在最前面——这是一个 UX 设计，让模型更容易找到相关的（最近修改的）文件。

3. **Token 节省有三层机制**：`--max-columns 500` 限制行长、`toRelativePath()` 缩短路径、`DEFAULT_HEAD_LIMIT = 250` 限制结果数量。这三层加起来防止单次 grep 消耗大量 context token。

4. **先截断再相对化**——对宽泛模式的性能优化：`applyHeadLimit()` 在 `toRelativePath()` 之前调用。宽泛模式可以返回 10k+ 行，但 `head_limit` 只保留 ~30-100 行。如果先相对化，就要对全部 10k 行做字符串处理；先截断就只需要处理 ~100 行。

5. **`Promise.allSettled` 处理 ENOENT 容错**——文件可能在 ripgrep 扫描和 stat 之间被删除。使用 `allSettled` 确保单个文件不存在不会拒绝整个批处理。失败的 stat 排序为 mtime 0（最旧），这样文件被移除后自然沉底。
