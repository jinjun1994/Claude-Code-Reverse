## 第 93 站：`src/tools/AgentTool/resumeAgent.ts`

### 这是什么文件

`resumeAgent.ts`（266 行）是子代理的恢复（resume）路径：从持久化的 transcript 和 metadata 中重建已中断/暂停的子代理状态，然后重新进入 `runAsyncAgentLifecycle()` 继续异步执行。

---

### 核心输入输出

```ts
resumeAgentBackground({
  agentId,        // 已存在子代理的 ID
  prompt,         // 新指令（追加到恢复消息后）
  toolUseContext, // 调用者的上下文
  canUseTool,     // 权限检查函数
  invokingRequestId // 请求 ID（analytics）
}) → ResumeAgentResult { agentId, description, outputFile }
```

---

### 恢复链（7 步）

#### 1. 并发读取持久化状态

```ts
const [transcript, meta] = await Promise.all([
  getAgentTranscript(asAgentId(agentId)),
  readAgentMetadata(asAgentId(agentId))
])
```

- transcript：序列化消息 + `contentReplacements`
- metadata：`{ agentType, description, worktreePath }`

#### 2. 消息过滤和重建

```ts
resumedMessages =
  filterWhitespaceOnlyAssistantMessages(
    filterOrphanedThinkingOnlyMessages(
      filterUnresolvedToolUses(transcript.messages)
    )
  )

resumedReplacementState = reconstructForSubagentResume(
  toolUseContext.contentReplacementState,
  resumedMessages,
  transcript.contentReplacements
)
```

三层过滤：
- `filterUnresolvedToolUses`：移除没有配对 `tool_result` 的 `tool_use`
- `filterOrphanedThinkingOnlyMessages`：移除只有 thinking 的 assistant message
- `filterWhitespaceOnlyAssistantMessages`：移除空白 assistant message

`contentReplacements` 重建用于恢复 prompt cache 稳定性（工具结果已从消息体移到外部存储）。

#### 3. Worktree 恢复

```ts
resumedWorktreePath = meta?.worktreePath
  ? await fsp.stat(meta.worktreePath).then(s => s.isDirectory() ? meta.worktreePath : undefined)
  : undefined
// 如果存在：bump mtime 防止被 stale-worktree 清理删除
if (resumedWorktreePath) {
  await fsp.utimes(resumedWorktreePath, now, now)
}
```

如果 worktree 已被外部删除，降级到父级 cwd 而非崩溃。

#### 4. Agent 解析

```ts
if (meta.agentType === 'fork') → FORK_AGENT
else → 从 activeAgents 中查找，找不到则用 GENERAL_PURPOSE_AGENT
```

跳过 `filterDeniedAgents`——原始 spawn 已经过了权限检查，恢复不应因新增的 deny rule 而阻断。

#### 5. Fork 恢复的特殊路径

恢复 fork 子代理时重建父系统提示（与 spawn 时逻辑相同）：

```ts
if (isResumedFork) {
  forkParentSystemPrompt = toolUseContext.renderedSystemPrompt
    ?? buildEffectiveSystemPrompt(...)
  workerTools = toolUseContext.options.tools  // 精确工具池
  useExactTools = true
}
```

#### 6. 构造 runAgent 参数

```ts
promptMessages = [...resumedMessages, createUserMessage({ content: prompt })]
isAsync = true                        // 恢复始终异步
forkContextMessages = undefined       // transcript 已含历史，不再传
contentReplacementState = resumedReplacementState
worktreePath = resumedWorktreePath
description = meta?.description
```

`forkContextMessages` 为 `undefined`——原始 fork 时已经复制过父上下文，transcript 中已有完整的前置消息。重新传入会导致 `tool_use` ID 重复。

#### 7. 启动异步生命周期

```ts
registerAsyncAgent({ ... })
runWithAgentContext(asyncAgentContext, () =>
  wrapWithCwd(() =>
    runAsyncAgentLifecycle({
      taskId, abortController,
      makeStream: (onCacheSafeParams) => runAgent({ ... }),
      enableSummarization: ...,
      getWorktreeResult: () => resumedWorktreePath ? { worktreePath } : {}
    })
  )
)
```

Analytics 元数据用 `invocationKind: 'resume'` 区分于 `'spawn'`。

---

### 与 spawn 路径的关键区别

| 方面 | Spawn | Resume |
|---|---|---|
| 消息源 | 构建新消息 | 从 transcript 读取过滤后的历史 |
| Agent 类型 | 运行时选择 | 从 metadata 恢复（或降级） |
| forkContextMessages | 传入父代理完整消息 | `undefined`（transcript 已有） |
| Worktree | 创建新的 | 检查已存在并 bump mtime |
| forkContextMessages | 传父历史避免 tool_use 重复 | 不传——transcript 中已有 |
| permission re-check | 经过 `filterDeniedAgents` | 跳过（已在 spawn 时检查过） |
| 始终异步 | 取决于条件 | 是 |
| contentReplacementState | 来自父代理 | 从 transcript.contentReplacements 重建 |

---

### 读完这一站后，你应该抓住的 5 个事实

1. Agent 恢复始终异步——无论原始 agent 是同步还是异步启动的，`resumeAgentBackground` 都注册后台代理并通过 `runAsyncAgentLifecycle` 执行。
2. 恢复时消息经过三层过滤：unresolved tool uses、orphaned thinking-only messages、whitespace-only assistant messages。这是因为持久化的 transcript 可能包含不完整的执行状态，恢复时需要干净的起点。
3. `forkContextMessages` 在恢复时为 `undefined`——原始 fork spawn 已经复制了父代理消息历史到 transcript 中。重新传入会导致 `tool_use` ID 重复，引起 API 错误。
4. 恢复路径跳过 `filterDeniedAgents` 权限重新检查——原始 spawn 时的权限决策应该持久化，不应因会话间新增的 deny rule 而阻断恢复。这是为了会话持久化的一致性。
5. Worktree 恢复时 bump `mtime` 防止被 stale-worktree 清理删除——如果一个 worktree 刚被恢复就很快被清理进程认为是不活跃而删除，会导致 agent 丢失隔离环境。
