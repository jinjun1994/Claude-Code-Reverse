## 第 89 站：`src/tools/AgentTool/AgentTool.tsx`

### 这是什么文件

`AgentTool.tsx` 是整个工具子系统中最复杂的单个工具实现（1300+ 行）：子代理（subagent）的 spawn/执行/生命周期管理中枢。它承担了同步执行、异步后台、worktree 隔离、fork 子代理、teammate 分片、远程代理等六条执行路径的路由。

---

### 输入 Schema 设计

```ts
baseInputSchema: { description, prompt, subagent_type?, model?, run_in_background? }
fullInputSchema = baseInputSchema + { name?, team_name?, mode?, isolation?, cwd? }
inputSchema = 按 feature flag 裁剪：
  - KAIROS 关闭 → omit cwd
  - background disabled / fork enabled → omit run_in_background
```

Schema 通过 `.omit()` 而非条件 spread 来移除字段——因为 spread 会破坏 Zod 类型推断（字段类型坍缩为 `unknown`）。

---

### 六条执行路径

AgentTool 的 `call()` 入口有六条分支：

#### 1. Teammate 分片（agent swarms）
```
if (teamName && name) → spawnTeammate()
返回 teammate_spawned 状态（tmux session/pane info）
```
仅 agent swarms 启用时。`isInProcessTeammate()` 禁止后台 agent 和嵌套 teammate spawn。

#### 2. 远程隔离代理（remote isolation）
```
if (external === 'ant' && effectiveIsolation === 'remote') → teleportToRemote()
返回 remote_launched 状态（taskId + sessionUrl）
```
ant 内部部署专属。经过 `checkRemoteAgentEligibility()` 前置检查。

#### 3. Fork 子代理实验
```
if (isForkSubagentEnabled() && !subagent_type) → FORK_AGENT
```
Fork 路径的关键特征：
- **系统提示复用**：`forkParentSystemPrompt = toolUseContext.renderedSystemPrompt`，子代理使用父代理的完全相同的系统提示
- **消息克隆**：`buildForkedMessages(prompt, assistantMessage)` 复制父代理完整 assistant message（所有 tool_use blocks）+ 占位 tool_result + 子代理指令
- **精确工具数组传递**：`availableTools: toolUseContext.options.tools`（不是重建的工具池）
- 目标：API 请求前缀缓存一致性（cache-identical）
- 递归保护：fork 子代理不能再 fork 另一个子代理

#### 4. 同步代理（sync agent）
```
shouldRunAsync = false → 直接 await runAgent() async iterator
```
内部有一个精巧的 **竞态路由机制**：
```ts
const raceResult = await Promise.race([
  agentIterator.next(),           // 下一条消息
  backgroundPromise               // 自动后台化信号
])
```
后台化后：
1. 关闭 `agentIterator.return()`（释放 MCP 连接、session hooks、prompt cache tracking）
2. 用 `run_in_background: true` 重启 `runAgent()` 迭代
3. 立即返回 `async_launched` 给父代理
4. 完成后通过 `enqueueAgentNotification` 推送通知

自动后台化由 `getAutoBackgroundMs()` 控制（默认 120 秒），也可以通过 AgentTool 的 `run_in_background: true` 或 agent 定义的 `background: true` 触发。

#### 5. 异步代理（async agent）
```
shouldRunAsync = true → registerAsyncAgent() + void runAsyncAgentLifecycle()
```
`void` 启动后不等待——通过 AsyncLocalStorage（工作负载传播）继承父级 workload 上下文。完成后通知。

#### 6. 特殊条件强制异步
```ts
forceAsync = isForkSubagentEnabled()          // fork 实验统一异步
assistantForceAsync = kairosEnabled (KAIROS)  // assistant 模式
proactiveActive = proactiveModule.isProactiveActive()
```

---

### Worktree 隔离机制

当 `effectiveIsolation === 'worktree'` 时：
1. `createAgentWorktree(slug)` 在启动前创建临时 git worktree
2. `runWithCwdOverride(path, fn)` 在执行期间覆盖 cwd
3. 完成后 `cleanupWorktreeIfNeeded()`：
   - hook-based 的 worktree 始终保留
   - 无变更的自动删除
   - 有变更的保留供用户检查

Fork 路径中还会注入 `buildWorktreeNotice()` 消息告知子代理路径映射关系。

---

### 工具池过滤（agentToolUtils.ts）

`filterToolsForAgent()` 按规则过滤子代理可用工具：

| 规则 | 说明 |
|---|---|
| MCP 工具 | 全部允许（`mcp__*`） |
| ALL_AGENT_DISALLOWED_TOOLS | 所有子代理禁止 |
| CUSTOM_AGENT_DISALLOWED_TOOLS | 非内置 agent 禁止 |
| ASYNC_AGENT_ALLOWED_TOOLS | 异步 agent 仅允许白名单内 |
| ExitPlanMode + plan mode | 特殊放行 |
| AgentTool + in-process teammate | 允许同步子代理 spawn |

`resolveAgentTools()` 支持通配符（`['*']` → 允许全部），以及 `Agent(worker,researcher)` 形式的类型限定。

---

### 安全分类器

`classifyHandoffIfNeeded()` 在子代理完成后：
1. 仅在 `auto` 模式下运行
2. 构建 `buildTranscriptForClassifier(agentMessages, tools)`
3. 调用 `classifyYoloAction()` 审查子代理操作的潜在风险
4. 如果 `shouldBlock` → 在返回结果前附加安全警告
5. 不可用时允许输出但添加验证提示而非完全阻断

---

### 输出 Schema

```ts
输出 = sync | async | teammate_spawned | remote_launched

Sync: { status: 'completed', content, totalToolUseCount, totalTokens, usage }
Async: { status: 'async_launched', agentId, outputFile, canReadOutputFile }
Teammate: { status: 'teammate_spawned', teammate_id, tmux_session_name, ... }
Remote: { status: 'remote_launched', taskId, sessionUrl }
```

`finalizeAgentTool()` 从 agent 消息中提取最后一条 assistant message 的文本内容和 usage 统计。

---

### 关键属性

| 属性 | 值 |
|---|---|
| `name` | `"Agent"` (legacy: `"Task"`) |
| `isReadOnly` | `true`（把权限检查委托给底层工具） |
| `isConcurrencySafe` | `true` |
| `maxResultSizeChars` | 100,000 |
| `checkPermissions` | auto 模式下 passthrough（需要模型权限），其他模式自动允许 |

---

### 元

问题：**这一站真正想解决的架构问题是什么？**

回答：这一站真正要回答的是，子代理为什么会成为整个工具系统里最复杂的单体工具。因为它不是单一路径执行器，而是同步、异步、worktree、fork、teammate 和 remote 等多条路线的总路由器。

### 反

问题：**如果把这一站的设计反过来，会发生什么？**

回答：如果这些路线没有被收束在 AgentTool，一个“调用子代理”动作就会在不同模式下表现成完全不同的工具。那样表面上像拆分复杂度，实际是把复杂度扔给整个平台。

### 空

问题：**跳出当前文件名，这一站背后更大的问题是什么？**

回答：更大的问题，是宿主怎样把“代理调用”当作一个工具，同时又允许它内部拥有自己的世界。AgentTool 展示的是工具平台向 agent 平台过渡时的边界形态。

### 读完这一站后，你应该抓住的 6 个事实

1. AgentTool 是工具子系统中最复杂的工具（1300+ 行），管理六条不同的执行路径：teammate spawn、remote isolation、fork sync、normal sync backgrounded、async-from-start、force-async。
2. Fork 子代理实验的核心设计是 cache identity——复用父代理的系统提示、精确工具数组和完整消息历史，确保 API 请求前缀字节相同。
3. 同步代理有一个精巧的竞态机制：`Promise.race([next message, background signal])`，允许用户在代理运行超过 120 秒时自动后台化，同时立即返回 async_launched 状态。
4. Agent 工具标记 `isReadOnly: true`——它自己不执行任何文件系统操作，安全性委托给子代理实际使用的工具。权限检查也委托给底层。
5. Worktree 隔离在启动前创建、执行中覆盖 cwd、完成后自动清理无变更的 worktree——这是一个完整的沙箱机制。
6. 工具过滤链有三层：ALL_AGENT_DISALLOWED_TOOLS（全局禁止）→ CUSTOM_AGENT_DISALLOWED_TOOLS（自定义 agent 禁止）→ ASYNC_AGENT_ALLOWED_TOOLS（异步模式白名单）→ 按 agent 定义的 disallowedTools 个体禁止。
