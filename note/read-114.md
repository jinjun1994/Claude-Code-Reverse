## 第 114 站：`src/tools/NotebookEditTool/`（4 个文件，~40K）

### 这是什么

NotebookEditTool 是 Jupyter Notebook（.ipynb）的单元格级别编辑工具——与 FileEditTool 的文本替换不同，它解析 notebook JSON 并直接操作单元格（cell），支持 replace、insert、delete 三种模式。

---

### 输入 Schema
```ts
{
  notebook_path: string           // 必填：必须是绝对路径
  cell_id?: string                // 可选：单元格 ID
  new_source: string              // 必填：新单元格内容
  cell_type?: 'code' | 'markdown' // 可选：单元格类型
  edit_mode?: 'replace' | 'insert' | 'delete'  // 默认 replace
}
```

---

### 三种编辑模式

#### Replace（默认）
- 修改指定 cell 的 `source` 字段
- 重置 `execution_count` 和 `outputs`（因为是代码单元格）
- 如果 `cell_type` 改变，更新单元格类型

#### Insert
- 在指定 cell 之后插入新单元
- 新生成随机 cell ID（`Math.random().toString(36).substring(2, 15)`）
- `cell_type` 是必填项

#### Delete
- 用 `cells.splice(cellIndex, 1)` 删除单元

---

### Cell ID 解析：`parseCellId()`

Notebook 的 cell ID 格式不固定。`parseCellId()` 尝试两种解析：
1. 精确匹配 `cell.id`（notebook 中的实际 ID）
2. 数字索引解析（`cell-N` 格式 → 提取 N 作为数字索引）

这让模型可以用任意方式引用 cell。

---

### 容错设计

#### Replace 溢出 → Insert
```ts
if (edit_mode === 'replace' && cellIndex === notebook.cells.length) {
  edit_mode = 'insert'  // replace 末尾不存在的 cell → 插入
}
```

如果模型尝试 replace 一个超出 notebook 范围的 cell，会自动变为 insert 而不是报错。

#### 无 cell_id → 默认插入开头
```ts
if (!cell_id) {
  cellIndex = 0  // 插入到 notebook 开头
}
```

---

### 写入注意事项

#### 使用 `jsonParse` 而非 `safeParseJSON`
```ts
// Must use non-memoized jsonParse here: safeParseJSON caches by content
// string and returns a shared object reference, but we mutate the
// notebook in place below. Using the memoized version poisons the cache.
let notebook = jsonParse(content)
```

`safeParseJSON` 是按内容缓存的（memoized）——如果我们就地修改 notebook（`cells.splice`、`targetCell.source = ...`），缓存会被污染。后续的 `validateInput()` 调用会看到突变后的版本。

#### 保留原始编码和行结束符
```ts
const { content, encoding, lineEndings } = readFileSyncWithMetadata(fullPath)
writeTextContent(fullPath, updatedContent, encoding, lineEndings)
```

Notebook 的编码和行结束符与原始文件保持一致，不覆盖。

---

### Cell ID 生成

```ts
if (notebook.nbformat > 4 || (notebook.nbformat === 4 && notebook.nbformat_minor >= 5)) {
  if (edit_mode === 'insert') {
    new_cell_id = Math.random().toString(36).substring(2, 15)
  }
}
```

只有 notebook 格式 >= 4.5 时才生成随机 cell ID——这是 IPython/Jupyter 引入 cell ID 的版本要求。

---

### readFileState 更新

```ts
readFileState.set(fullPath, {
  content: updatedContent,
  timestamp: getFileModificationTime(fullPath),
  offset: undefined,
  limit: undefined,
})
```

这确保后续的 FileReadTool 调用不会返回 `file_unchanged` stub——因为内容改变了但时间戳也会匹配。设置 `offset: undefined` 确保 dedup 不匹配部分读取的缓存。

---

### 验证链

1. 必须为 `.ipynb` 文件
2. UNC 路径跳过（Windows SMB 安全）
3. 编辑模式必须为 replace/insert/delete
4. Insert 模式需要 cell_type
5. 必须先读取文件（readFileState 检查）
6. 时间戳验证
7. 文件必须存在且为有效 JSON
8. cell_id 存在时，必须能在 notebook 中找到

---

### 读完后应抓住的 4 个事实

1. **NotebookEditTool 操作单元格级别而非整个文件**——与 FileEditTool 的 `old_string`/`new_string` 文本替换不同，它解析 notebook 的 JSON 结构，直接修改 `cells` 数组中的 cell 对象。这让模型可以做单元格的插入、删除、替换而不会破坏 notebook 格式。

2. **必须用非 memoized 的 jsonParse**——因为修改就地进行（`splice`、`source = ...`），如果使用缓存版本的解析器，同一内容的后续调用会拿到已被突变的对象引用。这是一个微妙的 bug——如果 `validateInput()` 先缓存了 notebook，随后 `call()` 修改它，后续编辑操作会看到被破坏的状态。

3. **Replace 溢出自动变为 Insert**——这是一个容错设计：如果模型尝试 replace 超出当前 notebook 范围的 cell 索引，工具不会报错而是将其变为 insert。这防止模型因为 notebook 被压缩或重新排序而丢失修改。

4. **Notebook 4.5+ 才生成 Cell ID**——`nbformat > 4 || nbformat >= 4.5` 条件确保只为支持 cell ID 的版本生成 ID。早期版本（4.x < 4.5）没有 ID 字段。这让 notebook 可以在任何版本的 Jupyter 中使用。
