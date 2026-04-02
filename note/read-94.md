## 第 94 站：`src/tools/AgentTool/UI.tsx`

### 这是什么文件

`UI.tsx`（871 行，经过 react-forget 编译）是 AgentTool 的渲染中枢：提供工具使用中、进行中、完成、拒绝、错误五种状态的 React 组件，以及子代理进度消息的智能分组/折叠/摘要显示。

---

### 六种渲染函数

| 函数 | 状态 | 说明 |
|---|---|---|
| `renderToolUseMessage` | 工具被调用 | 返回 description 文本 |
| `renderToolUseProgressMessage` | 工具运行中 | 显示最近 3 条进展 + 隐藏计数 |
| `renderToolResultMessage` | 工具完成 | 显示完成统计+转录模式详情 |
| `renderToolUseRejectedMessage` | 工具被拒绝 | 已有进展 + fallback 拒绝消息 |
| `renderToolUseErrorMessage` | 工具错误 | 已有进展 + fallback 错误消息 |
| `renderGroupedAgentToolUse` | 批量 Agent 调用 | 分组显示多个 agents 的统计信息 |

---

### `renderToolUseProgressMessage`

核心渲染逻辑——处理代理运行中的进度消息列表：

#### 1. Condensed 模式
当终端行数不足以渲染完整内容时（`terminalSize.rows < 工具数 * 9 + 7`），显示：
```
In progress… · 4 tool uses · 1,234 tokens · (⌃O expand)
```

#### 2. Normal 模式（非 condensed）
- 只展示最后 `MAX_PROGRESS_MESSAGES_TO_SHOW`（3 条）消息
- ant 构建中，连续的 search/read/REPL 操作被合并为 summary：
  ```
  3 searches · 5 reads · active
  ```
- 隐藏的工具调用显示为 `+2 more tool uses`

#### 3. Transcript 模式
展示完整转录，不截断。

---

### `processProgressMessages`（ant exclusive）

ant 构建特有的智能消息分组：
1. 遍历 agent progress 消息，识别连续的 search/read/REPL 操作
2. 将连续的相同类别操作合并为 `SummaryMessage`
3. 其他消息作为 original 保留
4. 只有 `tool_result` 消息才被计数（避免 `tool_use` + `tool_result` 双重计数）

非 ant 构建跳过此逻辑，直接返回所有消息。

---

### `renderGroupedAgentToolUse`

处理同一轮次多个 Agent 调用的分组显示：
- 计算每个 agent 的 `toolUseCount` 和 `tokens`
- 解析 input schema ��取 agent name/description/subagent_type
- teammate spawn（`isTeammateSpawn`）通过字符串比较判定（因为 `teammate_spawned` 状态被排除在导出 schema 外以进行 DCE）
- teammate spawn 显示为 `@name` 格式
- 检查 `allSameType`——如果所有 agent 类型相同，只显示一次类型名
- 检查 `allAsync`——决定是否显示 "background agents" 文案

---

### `renderToolResultMessage`

根据 agent 状态展示不同结果：

| 状态 | 展示 |
|---|---|
| `remote_launched` | 远程 agent URL + taskId |
| `async_launched` | "Backgrounded agent" + 快捷键提示 |
| `completed` | 统计：`Done (X tool uses · Y tokens · Zs)` |

转录模式下还展示：`AgentPromptDisplay`（原始 prompt）+ `VerboseAgentTranscript`（完整消息流）+ `AgentResponseDisplay`（最终响应）。

---

### 辅助工具

| 函数 | 用途 |
|---|---|
| `userFacingName` | 根据 input 确定显示名（非 `general-purpose` 类型显示类型名，`worker` 类型显示 "Agent"） |
| `userFacingNameBackgroundColor` | 根据 agent 类型确定名称背景色 |
| `extractLastToolInfo` | 从进度消息中提取最后使用的工具信息 |
| `calculateAgentStats` | 计算 toolUseCount 和 tokens |

---

### `VerboseAgentTranscript`

转录模式的核心组件——使用 `buildSubagentLookups` 构建消息索引，逐条渲染 agent 的完整消息流。

---

### React Forgive 编译

这个文件使用了 react-forget（`react/compiler-runtime` 的 `_c` 函数），所有组件都经过编译器自动 memoization。这意味着：
- 每个组件都有 `$[0] === Symbol.for("react.memo_cache_sentinel")` 检查模式
- 输入不变时跳过 JSX 重建

---

### 读完这一站后，你应该抓住的 5 个事实

1. Agent 工具运行时的进度显示有两种模式：condensed（终端小时显示一行统计信息）和 normal（显示最近 3 条进展 + 隐藏计数）。模式切换基于 `terminalSize.rows vs ESTIMATED_LINES_PER_TOOL * toolCount + TERMINAL_BUFFER_LINES` 的公式。
2. `processProgressMessages` 是 ant 构建独有的智能分组——连续的 search/read/REPL 操作被合并为一个 summary，避免大量文件读写操作淹没终端输出。非 ant 构建不做分组。
3. `renderGroupedAgentToolUse` 处理同一轮次多个 Agent 调用的聚合显示——检查 `allSameType` 决定类型是否只显示一次，检查 `allAsync` 决定是否显示 "background agents" 而不是 "agents finished"。
4. Teammate spawn 结果的检测通过字符串比较 `result?.output?.status as string === 'teammate_spawned'`——因为 `teammate_spawned` 状态被排除在导出 schema 外以进行 DCE（在 AgentTool.tsx 中定义为内部 `TeammateSpawnedOutput` 类型）。
5. 整个文件经过 react-forget 自动 memoization 编译——所有组件都有缓存信号量和哨兵检查，不需要手写 `useMemo` 或 `React.memo`。
