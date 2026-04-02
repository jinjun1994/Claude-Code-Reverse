## 第 109 站：`src/tools/SkillTool/`（4 个文件，~76K）

### 这是什么

SkillTool 是斜杠命令（slash command）的工具封装——允许模型执行已注册的技能（skill），如 `commit`、`review-pr`、`pdf` 等。技能分为两种执行路径：**inline 模式**（返回新的消息和 contextModifier）和 **fork 模式**（在隔离的子代理中执行）。

---

### 输入 Schema
```ts
{
  skill: string          // 技能名称（如 "commit"）
  args?: string          // 可选参数
}
```

### 输出 Schema
```ts
// Inline 模式
{
  success: boolean
  commandName: string
  allowedTools?: string[]
  model?: string
  status?: 'inline'
}

// Fork 模式
{
  success: boolean
  commandName: string
  status: 'forked'
  agentId: string
  result: string
}
```

---

### 执行路径分叉

#### Inline 模式（默认）
1. `findCommand()` 查找命令
2. `processPromptSlashCommand()` 处理命令并生成消息
3. 返回 `{ data, newMessages, contextModifier }`
   - `newMessages`：命令展开的用户消息
   - `contextModifier`：修改后的 ToolUseContext（如 allowedTools、model 覆盖）

#### Fork 模式（`command.context === 'fork'`）
1. `prepareForkedCommandContext()` 准备上下文
2. `runAgent()` 启动子代理执行
3. 收集消息并提取结果
4. 返回 `{ success, commandName, status: 'forked', agentId, result }`

---

### 远程技能系统（ant-only）

```ts
feature('EXPERIMENTAL_SKILL_SEARCH') && process.env.USER_TYPE === 'ant'
```

远程技能从云端（AKI/GCS）拉取，格式是纯 markdown（`SKILL.md`）——不需要 `!command` / `$ARGUMENTS` 扩展，因为它们是声明式 markdown 而不是可执行命令。

远程技能使用 `_canonical_<slug>` 格式的 canonical 名称——在 validateInput 和 call() 前被拦截处理。

---

### Skill 权限系统

#### 白名单属性检查
```ts
const SAFE_SKILL_PROPERTIES = new Set([
  'type', 'progressMessage', 'contentLength', 'argNames', 'model',
  'effort', 'source', 'pluginInfo', 'disableNonInteractive', ...
])
```

如果技能有任何不在白名单上的属性有值，权限就会变 `ask` 而非 auto-allow。这是安全的设计——新加入的属性默认需要权限，直到显式审查后被加入白名单。

#### 规则匹配
1. Deny rules 最优先（`Skill(deny_rule)`）
2. Remote canonical skills 自动允许
3. Allow rules 匹配
4. Safe properties 检查
5. 无匹配 → ask

#### 规则格式
- `Skill(commit)` — 精确匹配
- `Skill(review:*)` — 前缀匹配

---

### 命令发现

```ts
async function getAllCommands(context) {
  // 合并本地命令 + MCP skills
  const mcpSkills = context.getAppState().mcp.commands
    .filter(cmd => cmd.type === 'prompt' && cmd.loadedFrom === 'mcp')
  const localCommands = await getCommands(getProjectRoot())
  return uniqBy([...localCommands, ...mcpSkills], 'name')
}
```

只包括 MCP 的 **skills**（`loadedFrom === 'mcp'`），不是普通的 MCP prompts。这防止模型猜测未发现的 MCP prompt 名称并调用。

---

### Context Modifier 机制

Skill 可以在输出中返回 `contextModifier(ctx)` 函数来修改后续工具调用的上下文：

#### allowedTools 覆盖
```ts
contextModifier(ctx) {
  // 在 permissionContext 中添加 alwaysAllowRules.command
  return {
    ...ctx,
    getAppState() {
      return {
        ...appState,
        toolPermissionContext: {
          ...appState.toolPermissionContext,
          alwaysAllowRules: {
            command: [...allowedTools]
          }
        }
      }
    }
  }
}
```

#### Model override
```ts
contextModifier(ctx) {
  return {
    ...ctx,
    options: {
      mainLoopModel: resolveSkillModelOverride(model, ctx.options.mainLoopModel)
    }
  }
}
```

#### Effort override
```ts
contextModifier(ctx) {
  return { ...ctx, getAppState() { return { ...appState, effortValue: effort } } }
}
```

---

### 消息处理

#### tagMessagesWithToolUseID
将技能展开的用户消息标记为 transient——它们保留在上下文中直到此工具结果返回，确保不会在压缩时被意外删除。

#### COMMAND_MESSAGE_TAG 过滤
过滤掉包含 `<command_message>` 标签的消息——这些消息已经在 SkillTool 的 UI 中处理，不应重复注入到消息流。

---

### 遥测系统

```ts
logEvent('tengu_skill_tool_invocation', {
  command_name: sanitizedCommandName,   // 内置技能有真实名字
  _PROTO_skill_name: commandName,       // PII-tagged 列
  execution_context: 'inline' | 'fork', // 执行路径
  invocation_trigger: 'nested-skill' | 'claude-proactive',
  query_depth,                          // 嵌套深度
  // Ant-specific fields
  skill_source, skill_loaded_from, skill_kind,
  plugin_name, plugin_repository, ...
})
```

---

### 并发约束

```ts
// Only one skill/command should run at a time
```

Skill 工具是并发安全的，**但不应并调用多个**。因为每个 Skill 调用会把命令展开为完整 prompt，模型需要处理这些消息再继续——并发的 Skill 调用会导致上下文膨胀和竞争。

---

### 辅助文件

| 文件 | 用途 |
|---|---|
| `constants.ts` | SKILL_TOOL_NAME 常量 |
| `prompt.ts`（8.2K） | SkillTool 的 prompt 描述，包括技能列表和使用场景 |
| `UI.tsx`（19K） | Skill 执行的 UI 渲染组件 |

---

### 读完后应抓住的 5 个事实

1. **SkillTool 有两种执行路径**：Inline 路径展开命令为消息序列 + contextModifier（大多数内置技能），Fork 路径启动隔离子代理执行（有安全约束或大量操作的技能如 commit、pdf）。Fork 路径返回 `status: 'forked'` 和完整执行结果，Inline 路径返回 `status: 'inline'` 和新消息。

2. **Context Modifier 是 Skill 的核心输出机制**——它不只是返回结果，还可以修改模型执行环境的权限（allowedTools）、模型选择（model override）、努力级别（effort）。这让 Skill 可以临时授权额外工具或切换模型来处理复杂任务，而无需全局改变。

3. **Safe Properties 白名单是防御性设计**——`SAFE_SKILL_PROPERTIES` 列出了所有不会触发用户权限询问的属性。如果 Skill 有任何不在此白名单中的属性有值，会要求用户确认。这意味着新属性默认是"需要权限"的，直到被审查并添加到白名单中。

4. **远程技能是声明式的**——与本地技能通过 `processPromptSlashCommand()` 展开（支持 `!command` 和 `$ARGUMENTS` 替换），远程 `SKILL.md` 是纯 markdown 内容，直接作为用户消息注入。这是因为远程技能来自云端，不需要本地命令展开。它们使用 `_canonical_<slug>` 名称格式来区分。

5. **Skill 使用量被跟踪用于排名**——`recordSkillUsage(commandName)` 记录技能使用率，系统可以根据使用频率对技能进行排名。这让 Claude 可以在未来的交互中更智能地推荐常用的技能。
