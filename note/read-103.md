## 第 103 站：`src/tools/FileEditTool/`（6 个文件，~96K）

### 这是什么文件

FileEditTool 是 Claude Code 的核心文件编辑工具——通过 `old_string`/`new_string` 匹配替换实现文件内容的精确编辑。与 Write（全量覆盖）不同，Edit 是增量修改，保持文件的其余内容不变。

---

### 输入 Schema
```ts
{
  file_path: string          // 必填：文件路径（相对或绝对）
  old_string: string         // 必填：要替换的文本
  new_string: string         // 必填：替换后的文本
  replace_all?: boolean      // 默认 false：是否替换所有匹配项
}
```

---

### 完整生命周期

#### validateInput()（13 步验证链）
1. 写入权限检查 `checkWritePermissionForTool()`
2. Secret 扫描：`checkTeamMemSecrets()` 检测新字符串中的密钥
3. 空编辑拒绝：`old_string === new_string` → error
4. 路径 deny 规则匹配
5. UNC 路径跳过文件系统操作（`\\` 或 `//` → 直接返回 true）
6. 文件大小上限检查：1 GiB
7. 文件编码检测：BOM → UTF-16LE，否则 UTF-8
8. 文件不存在且 old_string 非空 → 提示类似文件名
9. 空文件检查：old_string 为空但文件有内容 → error
10. Notebook 检测：`.ipynb` → 提示使用 NotebookEditTool
11. 未读取检查：文件未被读取过 → error（`errorCode: 6`）
12. 时间戳检查：文件被修改过 → 回退到内容比较
13. `findActualString()` 搜索匹配：处理引号规范化
14. 多匹配检查：多个匹配但 `replace_all: false` → error
15. 设置文件验证：`validateInputForSettingsFileEdit()`

#### call()（执行链）
```
1. 发现 skill 目录（fire-and-forget）
2. 触发条件 skill
3. 父目录 mkdir（原子操作前）
4. 文件历史备份
5. 读取当前内容进行最终验证
6. findActualString() 查找实际文本
7. preserveQuoteStyle() 保留引号样式
8. getPatchForEdit() 生成结构化 patch
9. writeTextContent() 原子写入
10. LSP 通知（didChange + didSave）
11. VS Code 通知
12. 更新 readFileState 时间戳
13. Analytics 日志记录
14. Git diff 获取（远程模式）
15. 返回结果
```

---

### 关键的字符串处理机制

#### `findActualString()` — 引号匹配的容错搜索
1. 先尝试精确匹配
2. 失败后对双方都运行 `normalizeQuotes()`（花引号 → 直引号）
3. 在规范化后的文件中找到匹配位置
4. 从原始文件中提取该位置的实际字符串
5. 返回原始文本中实际存在的内容

#### `preserveQuoteStyle()` — 保留文件的引号风格
当匹配是通过引号规范化找到的时（文件中有花引号但模型发送了直引号），对 `new_string` 也应用相同的花引号样式：
- 判断文件中是否存在双花引号或单花引号
- `applyCurlyDoubleQuotes()`：根据上下文（前导空格/标点）判断开/闭引号
- `applyCurlySingleQuotes()`：对 `'` 做同样处理，但排除缩写形式（`don't`、`it's`）

#### `normalizeQuotes()`
统一将 `''`、`''`、`""`、`""` 转换为 `'` 和 `"`——因为 Claude 模型无法输出花引号。

#### `desanitizeMatchString()` — 反消毒处理
Claude 在 prompt 中看到的系统标记被消毒（如 `<function_results>` → `<fnr>`），但模型回传的编辑请求使用消毒后的版本。这组映射将消毒版还原为真实字符串：
```
<fnr> → <function_results>
<n> → <name>
<o> → <output>
<s> → <system>
<r> → <result>
...等 15 对映射
```

#### `stripTrailingWhitespace()`
每行去除末尾空白，但保留行结束符。Markdown 文件除外（两个尾部空格是硬换行）。

---

### Patch 生成与显示

#### `getPatchForEdit()` / `getPatchForEdits()`
使用 `structuredPatch()` 生成标准 unified diff 格式：
- 应用 edit 得到新内容
- 对 old/new 内容生成 patch
- Patch 中的 tab 转换为空格（显示友好）
- 多 edit 依次应用，检查中间冲突

#### 冲突检测
当多个 edit 操作同文件时：
```ts
// old_string 不能是之前 new_string 的子串
for (const previousNewString of appliedNewStrings) {
  if (previousNewString.includes(oldStringToCheck)) {
    throw Error('old_string is a substring of a new_string from a previous edit.')
  }
}
```
这防止 edit 操作互相重叠导致的中间状态不一致。

---

### 读取状态验证机制

#### `readTimestamp` 跟踪
`toolUseContext.readFileState` 记录每个文件的读取时间和内容：
```ts
{
  content: string,
  timestamp: number,    // 最后修改时间
  offset?: number,      // 部分读取时存在
  limit?: number,
  isPartialView?: boolean
}
```

- 编辑前文件必须在 `readFileState` 中
- 时间戳比较：`lastWriteTime > readTimestamp.timestamp` → 文件被修改
- 全读取的回退到内容比较（cloud sync/antivirus 可能改变 mtime 但不改内容）

#### TOCTOU 防护
在 `call()` 中，验证和写入之间有一个关键区：
- `readFileForEdit()` 重新读取文件
- 比较时间戳和内容
- 如果文件被外部修改 → 抛出 `FILE_UNEXPECTEDLY_MODIFIED_ERROR`
- 只有验证通过后才执行写入

---

### 权限处理

#### `checkWritePermissionForTool()`
委托给文件系统权限检查——与文件写入共用同一套机制。

#### `preparePermissionMatcher()`
```ts
async preparePermissionMatcher({ file_path }) {
  return pattern => matchWildcardPattern(pattern, file_path)
}
```
支持通配符模式匹配文件路径。

---

### UI 渲染

#### `renderToolUseMessage`
显示编辑摘要：`Editing file_path: old_string → new_string`

#### `renderToolResultMessage`
显示文件编辑完成的结果，包括修改提示、路径、是否全部替换。

#### `getToolUseSummary()`
返回简洁的编辑摘要用于 activity display。

---

### 类型和常量

#### `types.ts`
```ts
type FileEditInput = {
  file_path: string
  old_string: string
  new_string: string
  replace_all?: boolean
}

type FileEditOutput = {
  filePath: string
  oldString: string
  newString: string
  originalFile: string
  structuredPatch: StructuredPatchHunk[]
  userModified: boolean
  replaceAll: boolean
}

type FileEdit = { old_string: string, new_string: string, replace_all: boolean }
```

#### `constants.ts`
```ts
FILE_EDIT_TOOL_NAME = 'Edit'
FILE_UNEXPECTEDLY_MODIFIED_ERROR = 'File has been modified...'
```

---

### 读完这一站后，你应该抓住的 6 个事实

1. FileEditTool 使用 `old_string`/`new_string` 而非整文件写入——这是 Claude Code 的核心文件修改范式。模型只需提供要替换的文本和替换后的文本，系统负责生成 diff、追踪文件历史和通知 LSP。这使得模型可以进行精准的单点修改而不需要掌握整个文件。

2. 引号规范化链（`normalizeQuotes` → `findActualString` → `preserveQuoteStyle`）处理模型和文件间的引号差异。模型只能发送直引号（`'`、`"`），但文件中可能有花引号（`''`、`""`）。编辑通过规范化找到匹配位置，但写入时保留文件原来的花引号样式。这是一个关键的 UX 设计——用户看到的文件中引号不变。

3. 反消毒机制（`desanitizeMatchString()`）是必要的桥接——由于 Claude 系统标记在 prompt 中被消毒，模型回传的 `old_string` 是消毒后的版本。在匹配前必须将其还原为文件中的真实内容。15 对映射覆盖了所有被消毒的系统标记。

4. 读取状态验证防止了 TOCTOU 问题：模型读取文件后，用户可能修改了文件——编辑前的二次检查（时间戳 + 内容回退）确保编辑基于最新内容。如果文件被修改，错误消息要求模型重新读取文件。

5. `getGitDiff()` 在远程模式下获取 diff——这是为了让远程环境下的 Claude 能知道编辑的准确影响范围。本地模式下 diff 不是必须的因为 patch 已经包含完整信息。

6. LSP 通知链路（`didChange` + `didSave` + `clearDeliveredDiagnosticsForFile`）确保 IDE 的 linter 和类型检查在编辑后立即更新——这是编辑后 IDE 体验的基础。没有这些通知，IDE 会显示过时的诊断结果。
