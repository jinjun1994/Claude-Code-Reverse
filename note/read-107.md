## 第 107 站：`src/tools/AgentTool/`（14 个文件，~572K）

### 这是什么

AgentTool 是 Claude Code 的子代理（sub-agent）生成工具——允许模型在执行过程中启动独立的 agent 来处理子任务。支持同步和异步执行，有 fork 实验路径、worktree 隔离、远程 agent 等高级特性。

这是所有工具中最大的目录（~572K，234K 的 AgentTool.tsx 核心文件）。

---

### 输入 Schema

```ts
{
  description: string           // 必填：3-5 字任务描述
  prompt: string                // 必填：具体任务
  subagent_type?: string        // 可选：专用 agent 类型
  model?: 'sonnet' | 'opus' | 'haiku'  // 可选：模型覆盖
  run_in_background?: boolean   // 可选：后台执行
  name?: string                 // 可选：agent 名称（可用于 SendMessage 路由）
  team_name?: string            // 可选：团队名称（Agent Teams）
  mode?: permissionModeSchema   // 可选：权限模式
  isolation?: 'worktree' | 'remote'  // 可选：隔离模式
  cwd?: string                  // 可选：工作目录覆盖
}
```

---

### Agent 启动流程

```
1. 解析 team_name + name 参数
2. 验证 Agent Swarms 权限
3. 校验 teammate 的嵌套限制（flat roster）
4. 选择 agent：fork path 或 normal path
   - Fork path: selectedAgent = FORK_AGENT
   - Normal path: 按 subagent_type 查找 agent 定义
5. 检查 MCP 依赖（等待 pending 的 server，最多 30s）
6. 设置颜色（UI 显示）
7. 解析隔离模式（worktree/remote/无）
8. 构建 prompt messages（fork path 用 buildForkedMessages，normal path 用 createUserMessage）
9. 组装 worker tool pool（独立权限上下文）
10. 创建 worktree（如 isolation=worktree）
11. 决定同步或异步执行
12. 执行 agent 循环（runAgent）
13. 清理 worktree（如没有变更则删除）
```

---

### Agent 选择机制

#### Fork Path（`isForkSubagentEnabled()` gate）
当 fork gate 开启且未指定 `subagent_type` 时，使用 `FORK_AGENT`：
- 子 agent **继承父的系统 prompt**（不是 FORK_AGENT 自身的）——保持 API 缓存一致性
- 消息通过 `buildForkedMessages()` 构建（克隆父的所有 assistant message + 占位 tool_result + 子 agent 指令）
- `useExactTools: true`——继承父的 thinkingConfig

#### Normal Path
按 `subagent_type` 查找 agent 定义，支持：
- 内建 agents（`built-in/` 目录）
- 自定义 agents（`loadAgentsDir.js` 加载）
- MCP 依赖检查（agent 定义中的 `requiredMcpServers`）

---

### 递归 Fork 防护

```ts
if (toolUseContext.options.querySource === `agent:builtin:${FORK_AGENT.agentType}` ||
    isInForkChild(toolUseContext.messages)) {
  throw new Error('Fork is not available inside a forked worker.')
}
```

Fork 子 agent 不能再次 fork——因为 fork child 保留 Agent tool 在 tool pool 中用于缓存一致性，但这可能导致无限嵌套。

---

### Telemate Spawn 路由

当 `team_name + name` 同时设置时，走 `spawnTeammate()` 路径而非 `runAgent()`：
- 这启动一个 **独立的 teammate**（tmux 分割窗格或进程内）
- Agent Teams 需要 `isAgentSwarmsEnabled()` 开启
- In-process teammate 不能 spawn 后台 agent（生命周期绑定到 leader）
- Telemate 的 roster 是 flat 的——禁止嵌套 teammate

---

### MCP 依赖检查

当 agent 定义要求特定 MCP servers 时：
```ts
// 如果任何 required server 仍在 pending，轮询等待（最多 30s）
// 格式: mcp__serverName__toolName
// 检查 authenticated + connected = 有 tools 可用
```

这解决了 race condition：agent 被调用时 MCP server 可能还在认证中。

---

### 执行模式决策

```ts
const forceAsync = isForkSubagentEnabled()           // Fork experiment: 全部异步
const assistantForceAsync = appState.kairosEnabled   // Assistant mode: 全部异步
const shouldRunAsync =
  run_in_background === true ||          // 显式后台
  selectedAgent.background === true ||   // agent 定义要求
  isCoordinator ||                       // 协调器模式
  forceAsync ||                          // Fork gate
  assistantForceAsync ||                 // Assistant mode
  proactiveModule?.isProactiveActive()   // Proactive 模式
```

---

### 同步执行路径

```
1. 注册为 foreground task（registerAgentForeground）
   - 支持 auto-background: 超时后自动变后台
2. 迭代 runAgent() async iterator
3. 每次迭代:
   - 检查 background signal（Promise.race）
   - 如果被后台化: 释放当前 iterator，创建新的后台 closure
   - 转发 bash_progress 事件
   - 转发 agent_progress 事件
   - 统计 token 使用
4. finalizeAgentTool() 返回结果
5. classifyHandoffIfNeeded() 检查交接警告
6. cleanupWorktreeIfNeeded() 清理 worktree
```

---

### 异步执行路径

```
1. registerAsyncAgent() 注册后台任务
2. 注册 name → agentId 路由（用于 SendMessage）
3. void runWithAgentContext(asyncAgentContext, () => runAsyncAgentLifecycle(...))
4. 立即返回 async_launched 结果
5. 后台完成后通过 enqueueAgentNotification() 通知
```

---

### Worktree 隔离

当 `isolation: 'worktree'` 时：
```ts
const slug = `agent-${earlyAgentId.slice(0, 8)}`
worktreeInfo = await createAgentWorktree(slug)
```

完成后清理：
- 检查 worktree 是否有变更（比较 head commit）
- 无变更 → `removeAgentWorktree()`
- 有变更 → 保留
- hook-based worktrees 始终保留（无法检测 VCS 变更）

---

### 输出 Schema

```ts
// 同步完成
{ status: 'completed', prompt, content, ... }

// 异步启动
{ status: 'async_launched', agentId, description, prompt, outputFile, canReadOutputFile }

// Teammate spawn
{ status: 'teammate_spawned', teammate_id, agent_id, tmux_session_name, ... }

// 远程启动（ant only）
{ status: 'remote_launched', taskId, sessionUrl, description, ... }
```

---

### 权限检查

```ts
isReadOnly() { return true }  // 权限委托给底层工具
```

AgentTool 本身是只读的——它只是启动一个独立的 agent 上下文，agent 内部的工具调用各自走权限检查。

---

### 后台提示机制

```ts
const PROGRESS_THRESHOLD_MS = 2000  // 2 秒后显示后台提示
```

同步 agent 运行超过 2 秒时显示 "Background hint"，告知用户任务仍在后台运行，可以稍后检查进度。

---

### Agent 色彩管理

每个 agent 类型可以定义自己的颜色，用于 UI 中的分组显示：
```ts
setAgentColor(agentType, color)
```

颜色从 agent 定义中继承或显式设置。

---

### 辅助文件概览

| 文件 | 用途 |
|---|---|
| `agentColorManager.ts` | Agent 颜色注册和管理 |
| `agentDisplay.ts` | Agent UI 显示逻辑 |
| `agentMemory.ts` | Agent 内存注入（scope: user/project/session） |
| `agentMemorySnapshot.ts` | Agent 内存快照管理 |
| `agentToolUtils.ts` | Agent 结果提取、进度追踪、转移分类 |
| `constants.ts` | AgentTool 名称常量 |
| `forkSubagent.ts` | Fork 子代理的消息构建和上下文管理 |
| `loadAgentsDir.ts` | Agent 目录发现、过滤、定义加载 |
| `prompt.ts` | Agent 选择的 prompt 生成 |
| `resumeAgent.ts` | Agent 恢复机制 |
| `runAgent.ts` | 核心 agent 循环执行，含 stream 控制 |
| `UI.tsx` | 大规模 UI 渲染（125K） |
| `built-in/` | 内建 agent 定义 |
| `builtInAgents.ts` | 内建 agent 注册 |

---

### 元

问题：**这一站真正想解决的架构问题是什么？**

回答：这一站的核心不是“再开一个 agent”，而是把子代理变成可选同步、异步、远程、worktree 隔离的标准执行单元。它把 agent 选择、上下文构造、MCP 依赖等待和后台生命周期都收进同一条启动链里。

### 反

问题：**如果把这一站的设计反过来，会发生什么？**

回答：如果子代理只是随手 fork 出去，最先失控的会是递归、权限、清理和上下文污染。Agent 越多，系统越像一群能动但没人统筹的进程，而不是可管理的协作网络。

### 空

问题：**跳出当前文件名，这一站背后更大的问题是什么？**

回答：再高一层看，这一站在回答“AI 系统如何把并行思考组织成制度，而不是偶发技巧”。AgentTool 不是附属能力，它是在给多智能体协作提供基础宪法。

### 读完后应抓住的 6 个事实

1. **AgentTool 是子代理生成的总入口**——无论是 fork path、normal path、teammate spawn，还是 remote agent，都经过这个工具。它管理完整的生命周期：agent 选择、MCP 检查、权限验证、执行、进度追踪、资源清理（worktree）、结果返回、通知。

2. **Fork Path 保持 API 缓存一致性**——这是关键的 cache 优化。Fork child 继承父的系统 prompt 和完整工具数组（`useExactTools: true`），确保 API 请求的前缀 token 完全相同。如果不这样做，系统 prompt 或工具定义的任何差异都会破坏缓存、增加 API 成本。

3. **同步→异步的转换是核心 UX 机制**——同步 agent 通过 `registerAgentForeground` 开始执行，同时设置 auto-background timer。如果用户切换到其他工作或 agent 超时，agent 自动被后台化。这防止长时间运行的 agent 阻塞主 REPL 循环。

4. **Worker tool pool 是独立组装的**——agent 的权限上下文从 selectedAgent.permissionMode 或默认 'acceptEdits' 获取，与父的权限模式无关。这意味着 agent 可以拥有与父完全不同的工具集和权限（比如更宽松的权限模式）。

5. **MCP 依赖等待解决了 race condition**——如果 agent 要求某些 MCP servers，工具在调用 runAgent 前最多等待 30 秒让这些 servers 完成认证。如果 server 在等待过程中失败，agent 启动被直接拒绝并给出明确的错误。

6. **Worktree 创建与清理是幂等的**——创建时用 `agent-{id[:8]}` 作为 slug 生成唯一路径。完成时 `cleanupWorktreeIfNeeded()` 检查是否有 git 变更：如果无变更则删除 worktree，如果有变更保留。Hook-based worktrees 总是保留，因为无法检测变更。
