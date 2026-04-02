## 第 120 站：`src/tools/TodoWriteTool/`（3 个文件，~16K）

### 这是什么

TodoWriteTool 是会话级任务清单管理工具——模型用它来跟踪复杂任务的进度。与 Task 工具系统不同，TodoWriteTool 是面向会话的、轻量级的 TODO 列表，不涉及工具级权限或执行链。

---

### 输入 Schema

```ts
{
  todos: {
    content: string       // 任务描述（命令式："Fix bug"）
    status: 'pending' | 'in_progress' | 'completed'
    activeForm?: string   // 进行中的形式："Fixing bug"
  }[]
}
```

### 输出 Schema

```ts
{
  oldTodos: TodoItem[]
  newTodos: TodoItem[]          // 更新后的列表
  verificationNudgeNeeded?: boolean
}
```

---

### 关键行为

#### 全部完成时清空

```ts
const allDone = todos.every(_ => _.status === 'completed')
const newTodos = allDone ? [] : todos
```

当所有任务标记为完成时，todo 列表自动清空——这是为了防止列表无限增长。

#### 代理隔离

```ts
const todoKey = context.agentId ?? getSessionId()
```

TODO 按 `agentId` 分隔——主会话和每个子代理有自己独立的 TODO 列表。

#### 权限始终 allow

```ts
async checkPermissions() {
  return { behavior: 'allow', updatedInput: input }
}
```

TODO 操作不需要用户权限——这是模型的自我管理工具。

---

### Verification Nudge

```ts
if (
  feature('VERIFICATION_AGENT') &&
  getFeatureValue_CACHED_MAY_BE_STALE('tengu_hive_evidence', false) &&
  !context.agentId &&           // 只针对主线程
  allDone &&
  todos.length >= 3 &&          // 3+ 任务
  !todos.some(t => /verif/i.test(t.content))  // 没有 verification 任务
) {
  verificationNudgeNeeded = true
}
```

当主线程代理完成了 3+ 个任务且没有验证步骤时，工具结果中追加提醒：

```
NOTE: You just closed out 3+ tasks and none of them was a verification step.
Before writing your final summary, spawn the verification agent.
You cannot self-assign PARTIAL by listing caveats in your summary —
only the verifier issues a verdict.
```

这是引导模型使用 verification agent 的机制——模型不应该自己在总结中列出"部分完成"的警告，而应该生成 verification agent 来获取独立验证判断。

---

### TodoListSchema

```ts
import { TodoListSchema } from '../../utils/todo/types.js'
```

TODO schema 定义在共享类型中——与 Task 系统的 `TaskCreate`/`TaskUpdate` 等不同，TodoWriteTool 使用更简单的列表格式。

---

### Todo V2

```ts
isEnabled() {
  return !isTodoV2Enabled()
}
```

TodoWriteTool 在 Todo V2 启用时被禁用——V2 版本有不同的 TODO 机制。Todo V2 是一个实验性功能，通过 GrowthBook 控制。

---

### 读完后应抓住的 4 个事实

1. **会话级 vs 任务级**——TodoWriteTool 是轻量级的会话内管理工具，与 TaskCreate/TaskList/TaskUpdate 的任务执行系统不同。TODO 按 agentId 分隔，每个代理有独立的列表。

2. **全部完成时自动清空**——当所有任务完成，列表自动清空。这是为了防止列表无限膨胀。

3. **Verification Nudge**——当主线程完成 3+ 任务且没有验证步骤时，工具结果中追加提醒引导使用 verification agent。模型不应该自己在总结中列出 caveats，应该生成 verification agent 来获取独立判断。

4. **Todo V2 替代**——当 Todo V2 启用时，TodoWriteTool 被禁用。V2 版本有不同的架构和 schema。

---

## 下一步

剩余工具目录：
- ExitPlanModeTool
- EnterWorktreeTool / ExitWorktreeTool
- TeamCreateTool / TeamDeleteTool
- ConfigTool (已读，待写)
- GlobTool
- REPLTool
- SleepTool
- SyntheticOutputTool
- RemoteTriggerTool
- McpAuthTool / ListMcpResourcesTool / ReadMcpResourceTool
- PowerShellTool
- shared/
