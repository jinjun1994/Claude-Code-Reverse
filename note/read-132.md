## 第 152 站：核心类型系统（7 个文件）

### 这是什么

`src/types/` 是整个代码库的类型基础词汇——ID 品牌化、命令注册、hook I/O、权限决策管线、plugin 生命周期、transcript 日志格式、和消息队列。

---

### ids.ts —— 品牌化 ID 类型

```ts
type SessionId = string & { readonly __brand: 'SessionId' }
type AgentId = string & { readonly __brand: 'AgentId' }
```

**`AGENT_ID_PATTERN`**——`/^a(?:.+-)?[0-9a-f]{16}$/`
以 `a` 开头 + 可选前缀 + 16 位 hex。`toAgentId()` 验证返回 `null`——防止队友名称或非代理字符串被误认为 agent ID。

---

### command.ts —— 命令类型

**标签联合**：
```ts
Command = CommandBase & (PromptCommand | LocalCommand | LocalJSXCommand)
```

**PromptCommand 的 `context` 字段**：
- `inline`——技能展开在当前对话中
- `fork`——生成独立子代理，独立 token budget

**`LocalJSXCommandContext`**——重型运行时依赖桥接：`canUseTool?`、`setMessages`、MCP 管理、IDE 扩展、API key 更改。

---

### hooks.ts —— Hook 系统类型

**Zod/SDK 类型等价断言**：
```ts
type _assertSDKTypesMatch = Assert<IsEqual<SchemaHookJSONOutput, HookJSONOutput>>
```

编译时验证——Zod 推断的类型与 SDK 类型必须完全匹配。

**15+ 事件特定响应**——每个事件有自己的输出字段。`PreToolUse` 有 `permissionDecision`；`PostToolUse` 有 `updatedMCPToolOutput`（hook 可以重写 MCP 结果）。

**HookCallback**——`type: 'callback'` 的内部钩子直接调用 JS 函数，而不是生成进程。`internal?: true` 排除指标。

---

### permissions.ts —— 权限类型

**10 种 PermissionDecisionReason**：`rule`, `mode`, `subcommandResults`, `permissionPromptTool`, `hook`, `asyncAgent`, `sandboxOverride`, `classifier`, `workingDir`, `safetyCheck`, `other`。

**YoloClassifierResult**——两阶段 XML 管线（fast + thinking 阶段），每阶段计时、token、请求 ID、错误转储路径。

---

### plugin.ts —— Plugin 和 Marketplace 类型

**27+ PluginError 变体**——覆盖 git/网络/manifest/MCP/LSP/Hook/MCPB/policy 全生命周期错误。

关键错误：
- `mcp-server-suppressed-duplicate`——两个插件提供同一个 MCP 服务器
- `dependency-unsatisfied`——依赖未满足
- `marketplace-blocked-by-policy`——企业策略阻止

**PluginComponent 联合**——`'commands' | 'agents' | 'skills' | 'hooks' | 'output-styles'`

---

### logs.ts —— Transcript 日志类型

**TranscriptMessage**：
```ts
type TranscriptMessage = SerializedMessage & {
  parentUuid: UUID | null
  logicalParentUuid?: UUID | null  // parentUuid 被 null 化时保留逻辑父级
  isSidechain: boolean
  agentId?: string, teamName?: string
  agentName?: string, agentColor?: string
}
```

**Context Collapse（"marble-origami"）**——代码名混淆的特性门控系统：
- `ContextCollapseCommitEntry`——归档消息跨度，用摘要替换
- `ContextCollapseSnapshotEntry`——last-wins 语义的快照

**AttributionSnapshot**——字符级代码贡献追踪，SHA-256 内容哈希，`permissionPromptCount`, `escapeCount`。

---

### textInputTypes.ts —— 输入与队列类型

**三优先级队列**：
```ts
type QueuePriority = 'now' | 'next' | 'later'
// now: 立即中断（终止 in-flight tool call）
// next: mid-turn drain（当前工具完成后，下一次 API 调用前）
// later: end-of-turn drain（等 turn 完成，作为新查询处理）
```

**PromptInputMode**——`bash | prompt | orphaned-permission | task-notification`

**QueuedCommand**——`agentId?` 路由到特定子代理（undefined = 主线程）。`workload?` 排队时记账 header。`bridgeOrigin?`——通过 `isBridgeSafeCommand()` 过滤。

**`OrphanedPermission`**——连接 assistant message 和待定权限结果。当消息被压缩或截断时，权限对话框需要关联回原始请求。

---

### 元

问题：**这一站真正想解决的架构问题是什么？**

回答：这一站把 `src/types/` 定义成全局共享词汇表，真正要解决的是命令、权限、hook、plugin、transcript、消息队列都说同一种语言。品牌化 ID、三优先级队列、Zod/SDK 等价断言，都说明它是运行时秩序的静态骨架。

### 反

问题：**如果把这一站的设计反过来，会发生什么？**

回答：若没有集中类型层，`AgentId`、`PermissionDecisionReason`、`QueuedCommand` 这类关键概念会在不同模块各自漂移。最后最难排查的问题不是编译失败，而是各处都“差不多对”，却已经不是同一套协议。

### 空

问题：**跳出当前文件名，这一站背后更大的问题是什么？**

回答：更大的问题是，复杂智能体系统怎样让扩展增长时仍保有统一语义。这里的类型不是注释替代品，而是整个系统防止概念分叉的长期约束机制。

### 读完后应抓住的 4 个事实

1. **三优先级队列 + 4 输入模式**——`now/next/later` 控制消息何时被 drain。`now` 中断 in-flight 工具；`next` 等当前工具完成后插队；`later` 等 turn 完全结束后作为新请求。这是 Claude Code 处理并发用户输入和系统事件的核心机制。

2. **Context Collapse "marble-origami"**——代码名混淆的特性门控上下文压缩。通过 UUID 稳定的边界归档消息跨度，替换为摘要，resume 时回放。Commit + Snapshot 双模型。

3. **Zod/SDK 编译时等价断言**——`Assert<IsEqual<SchemaHookJSONOutput, HookJSONOutput>>` 在编译时验证 Zod schema 与 SDK 类型完全匹配。这防止运行时验证和静态类型之间的漂移。

4. **Transcript 的 isSidechain 标记**——区分主线程和子代理/teammate 消息。agentId + teamName 允许消息树重建和按代理过滤。
