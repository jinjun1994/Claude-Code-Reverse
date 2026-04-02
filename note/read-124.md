## 第 127 站：`src/tools/TeamCreateTool/`（4 个文件）

### 这是什么

TeamCreateTool 是 agent swarm 团队创建工具——创建一个多代理团队的 team lead 角色并写入 team file。与 AgentTool 的单个子代理不同，TeamCreateTool 建立的是多代理协作框架。

---

### 输入 Schema

```ts
{
  team_name: string        // 团队名
  description?: string     // 团队描述
  agent_type?: string      // team lead 角色（默认 "team-lead"）
}
```

### 输出 Schema

```ts
{
  team_name: string
  team_file_path: string
  lead_agent_id: string
}
```

---

### 创建流程

```ts
// 1. 检查不能同时领导两个团队
if (appState.teamContext?.teamName) → throw 'Already leading a team'

// 2. 生成唯一团队名（冲突时用 generateWordSlug()）
const finalTeamName = generateUniqueTeamName(team_name)

// 3. 构建 team file
const teamFile: TeamFile = {
  name: finalTeamName,
  leadAgentId: formatAgentId('team-lead', teamName),
  leadSessionId: getSessionId(),
  members: [{ agentId, name: 'team-lead', model, cwd, ... }],
}

// 4. 写入磁盘
await writeTeamFileAsync(finalTeamName, teamFile)
registerTeamForSessionCleanup(finalTeamName)  // 跟踪清理

// 5. 重置 task list
await resetTaskList(sanitizeName(finalTeamName))
setLeaderTeamName(sanitizeName(finalTeamName))

// 6. 更新 AppState
setAppState({ teamContext: { teamName, teamFilePath, leadAgentId, ... } })
```

---

### Team Lead 不是 Teammate

```
Note: We intentionally don't set CLAUDE_CODE_AGENT_ID for the team lead:
1. The lead is not a "teammate" — isTeammate() should return false
2. Their ID is deterministic (team-lead@teamName)
3. Setting it would cause isTeammate() to return true, breaking inbox polling
```

Team lead 不被视为 teammate——这是微妙的身份设计。`isTeammate()` 用于检查代理是否是团队中的子角色。如果 team lead 的 `CLAUDE_CODE_AGENT_ID` 被设置，`isTeammate()` 会错误地返回 true，破坏 inbox 轮询。

---

### 会话清理注册

```ts
registerTeamForSessionCleanup(finalTeamName)
```

团队被跟踪为 session cleanup 项——如果用户退出会话而没有显式 TeamDelete，团队目录和 worktree 会自动清理（gh-32730 bug 修复）。

---

### 元

问题：**这一站真正想解决的架构问题是什么？**

回答：这一站处理的是 agent swarm 的“建制时刻”，即团队如何拥有 lead、team file、task list 与会话清理挂钩。特别是 team lead 故意不被当作 teammate，说明身份划分在这里比创建动作本身更关键。

### 反

问题：**如果把这一站的设计反过来，会发生什么？**

回答：如果 team lead 也被当成普通 teammate，inbox 轮询与角色判断会立即混乱，整个团队结构会从根上失去层次。那时创建团队就不再是建制，而只是多起了几个 agent。

### 空

问题：**跳出当前文件名，这一站背后更大的问题是什么？**

回答：更大的问题是，多代理系统怎样从一开始就建立正确的组织身份。TeamCreateTool 的价值，不只是生成配置，而是在宣告谁是协调者、谁是被协调者。

### 读完后应抓住的 2 个事实

1. **Team lead 不是 teammate**——Team lead 不被设置 `CLAUDE_CODE_AGENT_ID`，`isTeammate()` 对领导返回 false。这是身份区分的设计——领导管理代理，不是代理本身。

2. **团队名冲突自动解决**——如果团队名已存在，`generateUniqueTeamName()` 生成随机的 word slug（如 "glistening-kindling-thimble"）而不是失败。

---

## 第 128 站：`src/tools/TeamDeleteTool/`（4 个文件）

### 这是什么

TeamDeleteTool 是 agent swarm 团队清理工具——解散团队、清理目录、重置状态。

---

### 输入 Schema

```ts
{}  // 无参数
```

### 输出 Schema

```ts
{
  success: boolean
  message: string
  team_name?: string
}
```

---

### 安全守卫：Active Member 检查

```ts
const teamFile = readTeamFile(teamName)
const activeMembers = teamFile.members.filter(m =>
  m.name !== TEAM_LEAD_NAME && m.isActive !== false
)
if (activeMembers.length > 0) {
  return { success: false, message: `${activeMembers.length} active member(s)...` }
}
```

如果有活动的 teammate 存在，团队删除被拒绝。模型需要先用 shutdown request 终止 teammates。

### 清理链

```ts
await cleanupTeamDirectories(teamName)     // 清理 team 目录和 worktree
unregisterTeamForSessionCleanup(teamName)  // 移除清理跟踪
clearTeammateColors()                      // 清除颜色分配
clearLeaderTeamName()                      // 清除 leader 的 team name
setAppState({ teamContext: undefined, inbox: { messages: [] } })
```

Unregister 清理跟踪是关键的——团队已被清理后，graceful shutdown 不应尝试再次清理。

---

### 元

问题：**这一站真正想解决的架构问题是什么？**

回答：这一站不是简单删目录，而是要求团队解散前先确认没有活跃 teammate 悬在外面。active member 检查、cleanup unregister 和状态清空，都说明“删除团队”实际上是在做一次组织收尾。

### 反

问题：**如果把这一站的设计反过来，会发生什么？**

回答：如果删除时不看活跃成员，最先留下来的就是孤儿 agent、脏状态和重复清理问题。那种“删掉了但又没删干净”的系统尾巴，往往比创建时的错误更难补救。

### 空

问题：**跳出当前文件名，这一站背后更大的问题是什么？**

回答：更大的问题是，多代理编排在结束时能否像开始时一样讲秩序。TeamDeleteTool 让解散也成为正式流程，而不是临时打扫战场。

### 读完后应抓住的 2 个事实

1. **Active member 保护**——TeamDelete 不删除有活动 teammate 的团队。模型必须先通过 shutdown request 终止它们。这是防止代理泄露的安全机制。

2. **Unregister 防止重复清理**——团队清理后调用 `unregisterTeamForSessionCleanup()`——这防止 graceful shutdown 再次清理已清理的团队。

---

## 第 129 站：`src/tools/RemoteTriggerTool/`（3 个文件）

### 这是什么

RemoteTriggerTool 管理远端（claude.ai）的定时 agent 触发器——这是一个 HTTP 封装工具，直接调用 Anthropic API 来创建、更新、运行远程触发器。

---

### 输入 Schema

```ts
{
  action: 'list' | 'get' | 'create' | 'update' | 'run'
  trigger_id?: string     // get/update/run 时需要
  body?: Record<string, unknown>  // create/update 时需要
}
```

### 输出 Schema

```ts
{
  status: number    // HTTP 状态码
  json: string      // API 响应
}
```

---

### API 端点

```
GET    /v1/code/triggers                    # list
GET    /v1/code/triggers/{id}               # get
POST   /v1/code/triggers                    # create
POST   /v1/code/triggers/{id}               # update
POST   /v1/code/triggers/{id}/run           # run
```

Beta header: `ccr-triggers-2026-01-30`

### OAuth 认证

```ts
await checkAndRefreshOAuthTokenIfNeeded()
const accessToken = getClaudeAIOAuthTokens()?.accessToken
const orgUUID = getOrganizationUUID()
```

需要 claude.ai 账户 OAuth token——不是 API key。通过 `/login` 认证。

### 启用条件

```ts
isEnabled() {
  return getFeatureValue_CACHED_MAY_BE_STALE('tengu_surreal_dali', false) &&
    isPolicyAllowed('allow_remote_sessions')
}
```

两个 gate 都必须满足：
1. GrowthBook feature `tengu_surreal_dali`（实验性功能）
2. Policy limit `allow_remote_sessions`（组织允许远程会话）

---

### 元

问题：**这一站真正想解决的架构问题是什么？**

回答：这一站把远端 claude.ai 触发器封装成一个很薄的 HTTP 工具层，核心在 OAuth、beta header 和策略 gate。它并不执行远程任务本身，而是承担本地到服务端触发器管理面的连接职责。

### 反

问题：**如果把这一站的设计反过来，会发生什么？**

回答：如果把远端触发器逻辑揉进本地调度系统，认证、策略和云端运行边界都会被搅混。那样用户很难分清自己是在安排本机会话，还是在操作一个托管服务。

### 空

问题：**跳出当前文件名，这一站背后更大的问题是什么？**

回答：更大的问题是，Claude Code 如何把“本地助理”与“云端代理”纳入同一心智模型。RemoteTriggerTool 的薄，恰恰说明这两者之间需要一条清晰边界。

### 读完后应抓住的 2 个事实

1. **RemoteTriggerTool 是 HTTP 薄层**——不执行实际功能，只做 OAuth 认证的 API 调用。触发器的实际执行在 claude.ai 服务端完成。

2. **Beta 功能**——`tengu_surreal_dali` 默认为 false——需要 GrowthBook enrollment 加上组织策略允许 `allow_remote_sessions`。

---

## 第 130 站：`src/tools/SyntheticOutputTool/`（1 个文件）

### 这是什么

SyntheticOutputTool（`StructuredOutput`）是结构化输出工具——在非交互式场景（SDK/脚本）中让模型以特定 JSON 格式返回结果。输入 schema 动态配置，不固定。

---

### 输入 Schema：动态

```ts
// 通过 createSyntheticOutputTool(jsonSchema) 动态配置
// 输入是任意 JSON Schema
z.object({}).passthrough()  // 模板接受任何输入
```

### 输出 Schema

```ts
{
  data: string              // "Structured output provided successfully"
  structured_output: any    // 模型返回的完整输入
}
```

---

### 弱缓存

```ts
const toolCache = new WeakMap<object, CreateResult>()

export function createSyntheticOutputTool(jsonSchema) {
  const cached = toolCache.get(jsonSchema)
  if (cached) return cached
  const result = buildSyntheticOutputTool(jsonSchema)
  toolCache.set(jsonSchema, result)
  return result
}
```

用 `WeakMap` 按 schema 对象引用缓存。工作流脚本经常用同一个 schema 调用 30-80 次——缓存把 Ajv 编译开销从 110ms 降到 4ms。

---

### Schema 验证

```ts
const ajv = new Ajv({ allErrors: true })
const validateSchema = ajv.compile(jsonSchema)

// call() 中：
const isValid = validateSchema(input)
if (!isValid) throw new TelemetrySafeError('Output does not match required schema')
```

Ajv 编译 schema 做验证——模型返回的结果必须符合指定的 JSON Schema，否则报错。

---

### 启用条件

```ts
isSyntheticOutputToolEnabled({ isNonInteractiveSession }) {
  return isNonInteractiveSession
}
```

只在非交互式会话中启用（`--output-format json`、SDK 调用等）。交互式 TUI 不需要。

---

### 元

问题：**这一站真正想解决的架构问题是什么？**

回答：这一站关注的是在非交互环境里，如何强制模型把答案装进指定 JSON Schema。WeakMap 缓存 Ajv 编译结果，说明它不仅要校验输出正确，还要把结构化输出做成高频可复用能力。

### 反

问题：**如果把这一站的设计反过来，会发生什么？**

回答：如果没有专门工具承接 schema 约束，结构化输出只能靠 prompt 暗示，最终一定会在边角字段和类型一致性上漂移。那时看似少了工具，实则把校验责任推给了调用方。

### 空

问题：**跳出当前文件名，这一站背后更大的问题是什么？**

回答：更大的问题是，模型生成何时能真正像接口返回，而不是像自由文本。SyntheticOutputTool 给出的答案是：结构必须由工具层兜底，而不是由语言习惯兜底。

### 读完后应抓住的 2 个事实

1. **动态 schema + WeakMap 缓存**——SyntheticOutputTool 的输入 schema 由调用方指定（非固定）。WeakMap 按对象引用缓存——同一 schema 对象引用只编译一次 Ajv。

2. **只在非交互式场景**——在 `--output-format json` 或 SDK 自动化场景中使用。交互式 TUI 不需要结构化输出工具，因为它直接与用户对话。

---

## 状态总结

已完成文档数量：read-28 到 read-130（约 62 个站点文档）

剩余未处理工具目录：
- REPLTool
- McpAuthTool
- ListMcpResourcesTool / ReadMcpResourceTool
- ConfigTool（已读代码，文档需要合并）
- shared/
