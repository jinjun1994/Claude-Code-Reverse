## 第 108 站：`src/tools/LSPTool/`（6 个文件，~92K）

### 这是什么

LSPTool 是 Language Server Protocol 的工具封装——提供代码分析能力（跳转到定义、查找引用、悬停信息、文档符号、工作区符号等）。这使 Claude Code 可以直接查询语言服务器（如 TypeScript Language Server）来获取类型、符号、引用等代码上下文信息，而不需要通过文本搜索。

---

### 输入 Schema
```ts
{
  operation: 'goToDefinition' | 'findReferences' | 'hover' |
              'documentSymbol' | 'workspaceSymbol' |
              'goToImplementation' | 'prepareCallHierarchy' |
              'incomingCalls' | 'outgoingCalls'
  filePath: string        // 文件路径
  line: number            // 行号（1-based，编辑器友好）
  character: number       // 字符偏移（1-based，编辑器友好）
}
```

---

### 核心机制

#### 只读且并发安全
```ts
isConcurrencySafe() { return true }
isReadOnly() { return true }
isLsp: true
shouldDefer: true
isEnabled() { return isLspConnected() }
```

LSPTool 是只读的，可以并发调用。`shouldDefer: true` 表示如果不是 LSP 连接的状态，工具会被延迟注册。`isEnabled()` 检查 LSP 是否已连接，没有 LSP 时不暴露此工具。

---

### 完整的调用链

```
1. 等待 LSP 初始化（waitForInitialization，如果还在 pending）
2. 确保文件已在 LSP 中打开
   - isFileOpen() 检查
   - 未打开时读取文件内容 → openFile()
   - 文件大小检查（10MB 上限）
3. 映射操作到 LSP method + params
4. sendRequest() 到 LSP server
5. Call hierarchy 两步处理（incoming/outgoing calls）
6. 过滤 gitignore 的结果
7. formatResult() 格式化
8. 返回 { operation, result, resultCount, fileCount }
```

---

### LSP 操作 → Method 映射

| 操作 | LSP Method |
|---|---|
| goToDefinition | textDocument/definition |
| findReferences | textDocument/references |
| hover | textDocument/hover |
| documentSymbol | textDocument/documentSymbol |
| workspaceSymbol | workspace/symbol（空 query 返回所有符号） |
| goToImplementation | textDocument/implementation |
| prepareCallHierarchy | textDocument/prepareCallHierarchy |
| incomingCalls | prepareCallHierarchy → callHierarchy/incomingCalls |
| outgoingCalls | prepareCallHierarchy → callHierarchy/outgoingCalls |

---

### Call Hierarchy 两步处理

`incomingCalls` 和 `outgoingCalls` 需要两个 LSP 请求：
1. `textDocument/prepareCallHierarchy` → 获取 `CallHierarchyItem[]`
2. `callHierarchy/incomingCalls` 或 `callHierarchy/outgoingCalls` → 使用 item[0] 获取实际调用关系

如果 prepare 返回空 array，直接返回 "No call hierarchy item found at this position"。

---

### Git Ignore 过滤

结果中会过滤掉 `.gitignore` 中的文件——使用批处理 `git check-ignore`（每批 50 个文件路径，超时 5 秒）：

```ts
const result = await execFileNoThrowWithCwd('git', ['check-ignore', ...batch], {
  cwd,
  preserveOutputOnError: false,
  timeout: 5_000,
})
// Exit code 0 = 至少一个路径被 ignore
// Exit code 1 = 都不被 ignore
// Exit code 128 = 不是 git 仓库
```

过滤应用在：findReferences、goToDefinition、goToImplementation、workspaceSymbol。

---

### 位置标准化

#### 1-based → 0-based 转换
```ts
// 模型看到的是 1-based 行号/字符（编辑器风格）
// LSP 协议使用 0-based
const position = {
  line: input.line - 1,
  character: input.character - 1,
}
```

这是关键的 UX 设计——模型看到的提示和返回的结果都是编辑器风格的行号，所以模型在引用错误信息时不需要额外的坐标转换。

---

### URI 处理

```ts
const uri = pathToFileURL(absolutePath).href  // 文件路径 → file:// URI
```

LSP 协议使用 `file://` URI 格式，所有路径在发送到 LSP 之前会被转换。

---

### Location 和 LocationLink 统一化

LSP 服务器可以返回两种位置格式：
- `Location`：`{ uri, range }`
- `LocationLink`：`{ targetUri, targetRange, targetSelectionRange }`

`toLocation()` 将 LocationLink 转换为 Location 格式，以便统一处理。

---

### 文件大小限制

LSP 工具拒绝分析超过 10MB 的文件：
```ts
const MAX_LSP_FILE_SIZE_BYTES = 10_000_000
```

这防止 LSP 服务器处理过大文件时导致性能问题或内存溢出。

---

### 容错和错误处理

```ts
// LSP 服务器返回 undefined 时的处理
if (result === undefined) {
  // 记录日志但不报错
  logForDebugging(...)
  // 返回 "No LSP server available for file type: .ext"
}
```

LSPTool 对错误有宽容的处理——如果 LSP 服务器未初始化或请求失败，返回格式化的错误消息而不是抛出异常。

---

### `formatters.ts`（17KB）

格式化各种 LSP 结果为可读文本：
- `formatGoToDefinitionResult()` — 跳转定义结果
- `formatFindReferencesResult()` — 查找引用结果
- `formatHoverResult()` — 悬停信息
- `formatDocumentSymbolResult()` — 文档符号
- `formatWorkspaceSymbolResult()` — 工作区符号
- `formatIncomingCallsResult()` — 调用者
- `formatOutgoingCallsResult()` — 被调者
- `formatPrepareCallHierarchyResult()` — 调用层级准备

---

### `symbolContext.ts`（3.3KB）

符号上下文处理——提取符号的完整层级结构（父符号、子符号路径等）用于更精确的上下文传递。

---

### `schemas.ts`（6KB）

LSPTool 的完整输入验证 schema——支持 discriminated union 模式，对不同的 operation 验证不同的必需字段。

---

### 元

问题：**这一站真正想解决的架构问题是什么？**

回答：这一站把“读代码”从文本匹配提升到语义查询，重点是定义、引用、悬停和调用层级这些语言服务器能回答的问题。它的价值在于让 Claude Code 不再只看字符串，而开始沿着符号关系理解工程。

### 反

问题：**如果把这一站的设计反过来，会发生什么？**

回答：如果只靠 grep 模拟这些能力，结果会充满近似匹配和语义误判，尤其在类型系统和跨文件引用里最容易失真。那样工具看似够用，但理解复杂代码时会始终隔着一层雾。

### 空

问题：**跳出当前文件名，这一站背后更大的问题是什么？**

回答：更大的问题是，AI 读代码究竟依赖文本还是依赖语言结构。LSPTool 的存在说明，真正的代码理解迟早要接入语言基础设施，而不能永远停在关键字层。

### 读完后应抓住的 5 个事实

1. **LSPTool 提供了代码级别的语义理解**，而不只是文本搜索。通过跳转定义、查找引用、悬停信息等操作，模型可以获取类型信息、函数签名、符号关系——这些是 grep 等文本工具无法提供的。这对理解复杂代码结构和类型系统至关重要。

2. **1-based 坐标是编辑器友好的 UX 设计**——模型输入的行号和字符偏移都是 1-based（就像编辑器显示的那样），内部自动转换为 LSP 的 0-based 坐标。这意味着模型在引用错误信息、编译器输出、IDE 提示时不需要做坐标转换。

3. **Call hierarchy 是两步操作**——先 `prepareCallHierarchy` 获取符号的层级项，再用该项查询实际的调用关系。这与 findReferences 直接一步查询不同，因为调用关系需要先知道"哪个符号"再"谁的调用关系"。

4. **Git ignore 过滤使用批处理 check-ignore**——每批 50 个路径用一次 `git check-ignore` 调用，超时 5 秒。这比逐个检查每个路径更高效，同时避免结果中包含被忽略的文件（如 node_modules、build/ 等）。

5. **LSPTool 是懒初始化的**——`shouldDefer: true` 和 `isEnabled() = isLspConnected()` 确保 LSP 工具只在 LSP 服务器实际连接后才注册为可用工具。如果 LSP 没有连接，工具不会出现在工具池中。此外，`call()` 中检查 `getInitializationStatus()` 如果是 pending 则等待初始化完成才开始操作。
