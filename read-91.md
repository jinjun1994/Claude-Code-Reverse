## 第 91 站：`src/tools/AgentTool/forkSubagent.ts`

### 这是什么文件

`forkSubagent.ts`（211 行）是子代理 fork 实验的协议定义：`FORK_AGENT` 合成 agent 定义、消息构建、递归 fork 保护和 prompt 缓存一致性机制。

---

### Fork 是什么

Fork 是一个实验性特性（`FORK_SUBAGENT` feature flag）：当 model 调用 `Agent` 时**省略 `subagent_type` 参数**，不选择特定 agent 类型，而是创建一个**继承父代理完整对话历史的子代理**。

目的：prompt 缓存命中——fork 子代理的 API 请求前缀与父代理字节相同。

---

### `FORK_AGENT` 合成定义

```ts
FORK_AGENT = {
  agentType: 'fork',
  tools: ['*'],            // 通配符但 useExactTools=true 直接传父工具池
  maxTurns: 200,
  model: 'inherit',        // 继承父代理模型
  permissionMode: 'bubble', // 权限提示冒泡到父终端
  getSystemPrompt: () => '', // 实际未使用——用 override.systemPrompt 传父代理已渲染字节
}
```

`getSystemPrompt` 返回值实际**不被使用**。Fork 路径通过 `override.systemPrompt` 传入父代理的 `renderedSystemPrompt`（已渲染的字节）。原因：重新生成可能因 GrowthBook 冷→热状态变化而与父代理字节不一致，导致 prompt cache miss。直接传字节是确定性的。

---

### 启用条件

```ts
isForkSubagentEnabled():
  feature('FORK_SUBAGENT') &&
  !isCoordinatorMode() &&          // coordinator 有自己的委托模型
  !getIsNonInteractiveSession()    // 非交互 session 禁用
```

---

### Prompt 缓存一致性机制

`buildForkedMessages()` 构造字节相同的 API 请求前缀：

```
[..., 父完整历史,
 assistant(all original tool_use blocks),     // 原封不动
 user(
   tool_result{id=use1, "Fork started..."},  // 统一占位符
   tool_result{id=use2, "Fork started..."},
   ...
   "<fork_boilerplate>You are a forked worker... directive</fork_boilerplate>"
 )
]
```

关键点：
1. **所有 tool_result 使用相同的 `FORK_PLACEHOLDER_RESULT`**——不因任务内容变化，避免 token 前缀变化
2. **只有最后的 directive 文本不同**——这是每个 fork 子代理唯一的变量
3. 因为 Anthropic API 的 prompt cache 按前缀匹配，所有 fork 子代理在前置部分都命中缓存

---

### 递归 fork 保护

`isInForkChild(messages)` 检测消息历史中是否已有 `<fork_boilerplate>` 标签：
```ts
messages.some(m => m.text.includes(`<${FORK_BOILERPLATE_TAG}>`))
```
Fork 子代理的 Agent tool 保留在工具池中（cache identity），但 `call()` 入口会检查此标签并拒绝再 fork。

---

### `buildChildMessage()` — 给子代理的指令

10 条非协商规则：
1. 你是 forked worker——不要 spawn 子代理，直接执行
2. 不要对话、提问、建议下一步
3. 不要添加评论
4. 直接使用工具
5. 修改文件前先 commit
6. 工具调用间不输出文本
7. 严格限定 scope
8. 报告 <500 词
9. 以 "Scope:" 开头
10. 报告结构性事实

---

### `buildWorktreeNotice()`

当 fork 子代理运行在 worktree 隔离环境时，注入路径翻译提示和隔离说明。

---

### 常量和 XML 标记

| 常量 | 用途 |
|---|---|
| `FORK_BOILERPLATE_TAG` | `<fork_boilerplate>` XML 标签（检测递归 fork） |
| `FORK_DIRECTIVE_PREFIX` | `<fork_directive>` 前缀（标识指令） |
| `FORK_PLACEHOLDER_RESULT` | `"Fork started — processing in background"` |

---

### 读完这一站后，你应该抓住的 5 个事实

1. Fork 子代理是一个实验性功能——当 model 省略 `subagent_type` 时触发，创建一个继承父代理完整对话历史的子代理。
2. 整个文件的核心设计目标是 prompt 缓存一致性：所有 fork 子代理产生字节相同的 API 请求前缀，只有最后的 directive 文本不同。
3. `FORK_AGENT.getSystemPrompt()` 返回值不被使用——fork 路径通过 `override.systemPrompt` 传入父代理的已渲染字节，避免 GrowthBook 状态变化导致的 cache miss。
4. `buildForkedMessages()` 对所有 tool_result 使用统一的占位符文本 `"Fork started — processing in background"`——不是因为工具结果不重要，而是为了让每个 fork 子代理的请求前缀完全相同。
5. 递归 fork 保护通过扫描消息历史中的 XML boilerplate 标签实现——fork 子代理保留 Agent tool（为了 cache identity），但入口处拒绝再 fork 的请求。
