## 第 104 站：`src/tools/FileReadTool/FileReadTool.ts`（~1184 行）

### 这是什么

FileReadTool 是 Claude Code 的文件读取工具——支持多种内容格式（文本、图片、PDF、Notebook）的统一读取入口。与 BashTool 的 `cat` 不同，它是类型感知的：针对不同文件类型走不同的处理管线，并且有去重、缓存、token 预算控制。

---

### 输入 Schema
```ts
{
  file_path: string          // 必填：文件路径
  offset?: number            // 可选：读取起始行号
  limit?: number             // 可选：读取行数上限
  description?: string       // 可选：读取目的描述
}
```

---

### 完整生命周期

```
1. validateInput() 安全验证
2. 设备路径拦截：BLOCKED_DEVICE_PATHS
3. 内容类型检测：notebook → image → PDF → text
4. 去重检查：readFileState.get() 比较 offset/limit + mtime
5. 读取执行：按类型分派
6. 输出格式化为 ToolResultBlockParam
7. 事件注册：registerFileReadListener() pub-sub
8. Analytics 日志：session file detection
```

---

### 安全门：设备路径拦截

#### `BLOCKED_DEVICE_PATHS`
阻止特殊设备路径防止进程挂起：
```
/dev/null
/dev/zero
/dev/random
/dev/urandom
/dev/fd/0, /dev/fd/1, /dev/fd/2
/dev/stdin, /dev/stdout, /dev/stderr
```

这些路径如果被读取会导致阻塞（`/dev/random` 等待熵）或无效数据（`/dev/null` 为空），不应该进入正常读取流程。

---

### 内容类型分派管线

#### `callInner()` 中的 5 种内容类型
按优先级检测：

1. **Notebook（.ipynb）**：JSON 解析 → JSONL 格式输出 cells
2. **Image**：`readImageWithTokenBudget()` 管线
3. **PDF（范围）**：`extractPDFPages()` + 页码参数
4. **PDF（全文）**：完整提取
5. **Text**：`readFileInRange()` 默认路径

---

### 图片处理管线

#### `readImageWithTokenBudget()` 压缩链
```
single-read → standard resize → token check → compress → fallback (sharp 400×400 JPEG q20)
```

每个阶段检查 token 预算：
- 单步读取后如果 token 超限 → 尝试缩小尺寸
- 标准压缩后仍超限 → 更激进的压缩
- 最终回退：sharp 库生成 400×400、quality 20 的 JPEG

#### macOS 截图路径处理
macOS 截图文件名中 AM/PM 前可能是 thin space（U+2009）而非 regular space——需要规范化处理才能正确找到文件。

---

### PDF 处理管线

#### `extractPDFPages()`
- 支持页码范围参数（如 `pages: "1-5"`）
- 基于大小阈值回退：文件过大时减少提取内容
- 页码数量上限控制：防止超大 PDF

---

### 文本读取管线

#### `readFileInRange()`
- 支持 offset/limit 参数进行范围读取
- Token 验证：`validateContentTokens()` 确保内容在预算内
- Cyber Risk Mitigation：对大文件显示 `CYBER_RISK_MITIGATION_REMINDER`（Opus 4.6 除外）

#### `validateContentTokens()`
使用 API-based token counting 对大文件进行 token 计数验证——确保内容不会超出模型的 context 预算。

---

### 去重机制：Read Dedup

#### `readFileState.get()` 比较逻辑
```ts
// 缓存命中条件：
- 相同文件路径
- 相同的 offset/limit
- 文件 mtime 未变
```

如果命中缓存，返回 `file_unchanged` stub 而非重新读取——大约节省 18% 的 `cache_creation` token。

#### Session File Detection
检测是否为 session 相关文件（如 transcript、memory 文件），用于 analytics 日志记录。

---

### 事件系统：`registerFileReadListener()`

Pub-sub 模式注册文件读取事件监听器——允许其他组件响应文件读取行为（如追踪哪些文件被读取过，用于 FileEditTool 的 TOCTOU 防护）。

---

### 输出格式化

#### `mapToolResultToToolResultBlockParam()`
6 种输出类型映射：

| 类型 | 用途 |
|---|---|
| `image` | 图片数据（base64 编码） |
| `notebook` | JSONL 格式的 notebook cells |
| `pdf` | PDF 页码提取结果 |
| `parts` | 多部分内容片段 |
| `file_unchanged` | 去重复读的 stub 响应 |
| `text` | 普通文本内容 |

---

### 读完这一站后，你应该抓住的 5 个事实

1. **文件类型感知读取**：FileReadTool 不是简单的 `cat`，而是根据文件扩展名和内容分派到不同的处理管线。图片走压缩链，PDF 走页面提取，notebook 走 JSONL 序列化——每个管线有自己的 token 预算和回退机制。这确保无论读什么文件都不会超出 context 预算。

2. **Read Dedup 节省 ~18% cache_creation tokens**：通过 `readFileState` 追踪每个文件的读取状态（offset/limit + mtime），如果模型重复读取同一文件的相同范围且文件未变，直接返回 `file_unchanged` stub。这在模型循环读取文件验证编辑结果时特别有效。

3. **图片压缩链是渐进式的**：single-read → resize → token check → compress → sharp fallback。每一步都检查 token 预算，只有超限时才进入下一步。最激进的压缩是 sharp 400×400 JPEG quality 20，确保即使是高分辨率图片也能被处理。

4. **设备路径阻止是必要的**：`/dev/random`、`/dev/zero`、`/dev/null` 等设备路径如果被读取会导致阻塞、无限数据或空结果。`BLOCKED_DEVICE_PATHS` 在读取入口处拦截这些路径，防止进程挂起或模型接收到无效内容。

5. **Pub-sub 事件系统与 TOCTOU 防护协作**：`registerFileReadListener()` 允许 FileEditTool 等组件追踪文件读取状态。当 FileEditTool 执行编辑前，它检查 `readFileState` 确认文件是否被读过、何时读过、内容是什么——这是 TOCTOU 防护的底层依赖。
