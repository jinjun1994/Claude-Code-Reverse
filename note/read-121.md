## 第 121 站：`src/tools/ExitPlanModeTool/`（4 个文件，~20K）

### 这是什么

ExitPlanModeV2Tool 是模型在 plan 模式下完成规划后请求用户批准并进入实现阶段的工具——它把计划呈现给用户、等待批准、然后切换到合适的权限模式。

---

### 输入 Schema

```ts
{
  allowedPrompts?: {
    tool: 'Bash'                        // 只能是 Bash
    prompt: string                      // 语义描述："run tests", "install deps"
  }[]
}
```

### 输出 Schema

```ts
{
  plan: string | null              // 被批准的方案
  isAgent: boolean
  filePath?: string
  hasTaskTool?: boolean
  planWasEdited?: boolean
  awaitingLeaderApproval?: boolean // teammate 等待 leader 批准
  requestId?: string
}
```

---

### 三种批准路径

#### 1. 非 teammate（普通用户）

```ts
async checkPermissions() {
  return { behavior: 'ask', message: 'Exit plan mode?' }
}
requiresUserInteraction() { return true }
```

用户在 TUI 中看到批准对话框——确认或拒绝。批准后恢复到进入 plan 模式前的权限模式（`prePlanMode`）。

#### 2. Teammate 且需要 leader 批准

```ts
if (isTeammate() && isPlanModeRequired()) {
  // 写入 team-lead 的 mailbox
  await writeToMailbox('team-lead', {
    text: jsonStringify({ type: 'plan_approval_request', ... }),
  })
  // 返回等待状态
  return { awaitingLeaderApproval: true, requestId, ... }
}
```

Teammate 的方案通过 mailbox 提交给 team lead——不阻塞，等待异步批准。

#### 3. Teammate 且不需要 leader 批准

```ts
if (isTeammate()) {
  return { behavior: 'allow' }  // bypass 权限 UI
}
```

Voluntary plan mode 的 teammate 直接退出——不需批准。

---

### 模式恢复机制

```ts
context.setAppState(prev => {
  let restoreMode = prev.toolPermissionContext.prePlanMode ?? 'default'
  // Gate-off 回退：auto 模式不可用时回到 default
  if (restoreMode === 'auto' && !isAutoModeGateEnabled()) {
    restoreMode = 'default'
  }
  // 恢复到非 auto 模式时还原危险权限
  if (!restoringToAuto && prev.strippedDangerousRules) {
    baseContext = restoreDangerousPermissions(baseContext)
  }
  return {
    ...prev,
    toolPermissionContext: { mode: restoreMode, prePlanMode: undefined },
  }
})
```

退出 plan 模式时，不是简单地回到 `default`——而是恢复到进入 plan 前的模式：
- 进入前是 `auto`→ 回到 `auto`
- 进入前是 `default`→ 回到 `default`
- 进入前是 `bypass`→ 回到 `bypass`

---

### Circuit Breaker 防御

```ts
if (prePlanRaw === 'auto' && !isAutoModeGateEnabled()) {
  restoreMode = 'default'  // 不回退到 auto
  context.addNotification({
    text: `plan exit → default · auto mode unavailable`,
    priority: 'immediate',
  })
}
```

当 auto 模式的 circuit breaker 被触发后，即使 enter plan 前是 auto 模式，退出时也回到 default 而不是重新激活 auto。

---

### 方案编辑检测

```ts
const inputPlan = 'plan' in input ? input.plan : undefined
const plan = inputPlan ?? getPlan(context.agentId)
const planWasEdited = inputPlan !== undefined

if (inputPlan && filePath) {
  await writeFile(filePath, inputPlan, 'utf-8')
  void persistFileSnapshotIfRemote()
}
```

CCR web UI 可以在批准时编辑方案——通过检测 `input.plan` 是否存在来判断是否编辑过。编辑后的方案写回磁盘。

---

### 工具参考输出

批准后，结果消息包含：

```
User has approved your plan. You can now start coding.
...
## Approved Plan:
{plan content}
```

如果方案被用户编辑过，标签变为 `Approved Plan (edited by user)`——这让模型知道用户改了某些东西。

---

### 动态导入

```ts
const autoModeStateModule = feature('TRANSCRIPT_CLASSIFIER')
  ? require('../../utils/permissions/autoModeState.js')
  : null
const permissionSetupModule = feature('TRANSCRIPT_CLASSIFIER')
  ? require('../../utils/permissions/permissionSetup.js')
  : null
```

仅在 TRANSCRIPT_CLASSIFIER 功能开启时加载——外部构建中整个 auto mode 恢复逻辑被 tree-shake。

---

### 元

问题：**这一站真正想解决的架构问题是什么？**

回答：这一站负责把 plan 模式的产出正式提交出来，并据此恢复到进入前的权限状态。三种审批路径、`prePlanMode` 恢复和 circuit breaker 防御，说明它关心的是“计划被谁批准、批准后回到哪里”。

### 反

问题：**如果把这一站的设计反过来，会发生什么？**

回答：如果退出 plan 只是简单回 default，之前从 auto 或其他模式进入的上下文就会被粗暴抹平。更危险的是，若不经审批直接出 plan，计划阶段就失去了存在意义。

### 空

问题：**跳出当前文件名，这一站背后更大的问题是什么？**

回答：更大的问题是，规划与执行之间怎样完成一次制度化交接。ExitPlanModeTool 把这条交接线画出来，让计划不只是说完，而是被接纳后才转入行动权限。

### 读完后应抓住的 5 个事实

1. **三种批准路径**：非 teammate（TUI 确认）、teammate 需要批准（mailbox 提交）、teammate 不需要批准（直接退出）。ExitPlanMode 根据 `isTeammate()` 和 `isPlanModeRequired()` 决定走哪条路径。

2. **模式恢复到进入前的状态**——不是简单回到 `default`，而是恢复到 `prePlanMode` 中记录的模式。这使得 plan 模式可以从 auto 进入、审批通过后回到 auto 继续工作。

3. **Circuit breaker 防御**——当 auto 模式因 circuit breaker 被禁用时，退出 plan 不回到 auto 而是 default。这防止 ExitPlanMode 绕过 circuit breaker 重新激活 auto。

4. **方案编辑检测**——CCR web UI 可以在批准时编辑方案，通过 `input.plan` 是否存在来检测。编辑后的方案写回磁盘，结果标签标明 `(edited by user)`。

5. **Team-lead mailbox 集成**——Teammate 的方案通过 SendMessageTool 的 mailbox 机制提交给 team lead，包括 `plan_approval_request` 类型消息。Team lead 批准/拒绝后 teammate 的 mailbox 收到响应。
