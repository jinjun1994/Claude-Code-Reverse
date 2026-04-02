## 第 105 站：`src/tools/FileWriteTool/`（3 个文件，~64K）

### 这是什么

FileWriteTool 是全量文件写入工具——与 FileEditTool 的 `old_string`/`new_string` 增量编辑不同，Write 是完整内容替换。模型提供 `file_path` 和 `content`，系统写入整个文件。通常用于创建新文件或模型选择全量重写而不使用增量 edit 的场景。

---

### 输入 Schema
```ts
{
  file_path: string     // 必填：必须是绝对路径
  content: string       // 必填：完整文件内容
}
```

### 输出 Schema
```ts
{
  type: 'create' | 'update'
  filePath: string
  content: string
  structuredPatch: StructuredPatchHunk[]
  originalFile: string | null
  gitDiff?: ToolUseDiff
}
```

---

### 验证链：`validateInput()`

1. **Secret 扫描**：`checkTeamMemSecrets()` 检测内容中的密钥，防止写入包含敏感信息的文件
2. **Deny 规则匹配**：`matchingRuleForInput()` 检查路径是否在 deny 目录中
3. **UNC 路径跳过**：`\\` 或 `//` 开头 → 直接返回 true（Windows SMB 认证安全风险）
4. **文件存在性检查**：`fs.stat()` 获取 mtime，不存在则允许（新文件创建）
5. **读取状态验证**：文件必须在 `readFileState` 中且不是 partial view —— 未读取的文件不能写入
6. **时间戳检查**：`lastWriteTime > readTimestamp` → 文件已被修改，拒绝写入

---

### 写入链：`call()`

```
1. 展开路径（~ → 绝对路径）
2. 技能发现（fire-and-forget）：discoverSkillDirsForPaths()
3. 条件技能激活：activateConditionalSkillsForPaths()
4. diagnosticTracker.beforeFileEdited()
5. mkdir 父目录（原子写入前）
6. fileHistoryTrackEdit() 备份（如启用）
7. readFileSyncWithMetadata() 读取当前内容 + 时间戳验证
8. 内容比较回退：full read 时比较内容避免 cloud sync/antivirus 导致的误报
9. writeTextContent(fullFilePath, content, enc, 'LF') 原子写入
10. LSP 通知：clearDeliveredDiagnostics → changeFile → saveFile
11. VS Code 通知：notifyVscodeFileUpdated(fullFilePath, oldContent, content)
12. 更新 readFileState 时间戳
13. CLAUDE.md 特殊事件：tengu_write_claudemd
14. Git diff 获取（远程模式 + feature flag）
15. 生成 patch 并返回结果
16. countLinesChanged() 追踪变更行数
17. logFileOperation() analytics
```

---

### 关键设计：行结束符处理

#### `writeTextContent(fullFilePath, content, enc, 'LF')`

第四个参数 `'LF'` 是关键的——它告诉写入函数：**不要改模型发送的行结束符，使用 `LF`**。

之前的行为是：
- 对于已存在文件：沿用旧文件的行结束符
- 对于新文件：用 ripgrep 采样仓库中其他文件的行结束符

但这导致了两个问题：
1. 如果模型发送了包含 `\r\n` 的 bash 脚本到 Linux 系统，保留旧文件的 CRLF 会导致脚本静默失败
2. 仓库中的二进制文件可能"污染"行结束符采样结果

现在 `content` 中模型发送了什么行结束符就写入什么——模型对自己的输出负责。

---

### 与 FileEditTool 的关键差异

| 方面 | FileEditTool | FileWriteTool |
|---|---|---|
| 粒度 | 增量 `old_string`/`new_string` | 全量覆盖 |
| 引号处理 | `normalizeQuotes` + `preserveQuoteStyle` | 无（模型发送什么就用什么） |
| 反消毒 | `desanitizeMatchString()` 15 对映射 | 无 |
| 路径创建 | 需要先读再写 | 自动 mkdir 父目录 |
| 新文件 | 不接受（文件必须存在） | 接受（`type: 'create'`） |
| Patch 生成 | `getPatchForEdit()` | `getPatchForDisplay()` 模拟 diff |
| 共享 | TOCTOU 防护、LSP 通知、时间戳验证、`FILE_UNEXPECTEDLY_MODIFIED_ERROR` | 相同 |

---

### CLAUDE.md 特殊处理

```ts
if (fullFilePath.endsWith(`${sep}CLAUDE.md`)) {
  logEvent('tengu_write_claudemd', {})
}
```

写入 `CLAUDE.md` 时触发专门的 analytics 事件——这是因为 CLAUDE.md 是 prompt 注入文件，写入它会直接影响下一次模型调用的上下文。追踪这个事件可以分析 CLAUDE.md 的使用频率和编辑模式。

---

### LSP 通知链路（与 FileEditTool 相同）

```ts
clearDeliveredDiagnosticsForFile(`file://${fullFilePath}`)  // 清除旧诊断
lspManager.changeFile(fullFilePath, content)                 // didChange
lspManager.saveFile(fullFilePath)                            // didSave
```

这确保 LSP 服务器（如 TypeScript Language Server）在文件写入后立即重新计算诊断信息并同步到 IDE。

---

### 读完这一站后，你应该抓住的 4 个事实

1. **FileWriteTool 是全量覆盖而 FileEditTool 是增量修改**——Write 不接受 `old_string` 匹配检查，只需要文件被读取过且未被修改。这意味着模型可以写入它之前读过的任何文件，但不需要匹配特定内容。相比之下 Edit 更精准但约束更多。

2. **行结束符不再被系统覆盖**——之前系统会保存文件旧行结束符或采样仓库行结束符，导致模型发送的内容被静默改写（比如 bash 脚本收到 CRLF 在 Linux 上失败）。现在 `writeTextContent(content, 'LF')` 直接使用模型发送的行结束符，不对内容做行结束符规范化。

3. **Write 可以创建新文件，Edit 不可以**——FileWriteTool 对不存在的文件返回 `{ result: true }`（允许写入），因为新文件不需要先读。FileEditTool 的 Edit 要求文件必须存在且有内容才能匹配 `old_string`。Write 在写入前自动 `mkdir` 父目录，这是唯一自动创建目录的工具。

4. **TOCTOU 防护机制与 FileEditTool 共享**——两个工具都使用 `readFileState` + 时间戳 + 内容比较回退来防止在读取和写入之间文件被外部修改。`FILE_UNEXPECTEDLY_MODIFIED_ERROR` 在两个工具间共享（从 FileEditTool/constants.ts 导入），确保错误消息一致。
