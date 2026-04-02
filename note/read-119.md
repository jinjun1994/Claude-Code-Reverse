## 第 120 站：`src/tools/EnterPlanModeTool/`（4 个文件，~28K）

### 这是什么

EnterPlanModeTool 是模型请求切换到 plan 模式的工具——用户通过 TUI 的 `/plan` 斜杠命令进入，而这是模型在遇到复杂任务时主动规划的方式。

---

### 输入 Schema

```ts
{}  // 无参数
```

### 输出 Schema

```ts
{ message: string }  // 确认消息
```

---

### 关键特征

#### 权限始终 ask

```ts
async checkPermissions() {
  return { behavior: 'ask', message: 'Enter plan mode?' }
}
```

没有 auto-allow 路径——用户必须明确批准才能进入 plan 模式。

#### Channel 模式自动禁用

```ts
isEnabled() {
  if ((feature('KAIROS') || feature('KAIROS_CHANNELS')) && getAllowedChannels().length > 0) {
    return false
  }
  return true
}
```

当通过 Telegram/Discord 等频道运行时工具直接被禁用——Channel 模式下 ExitPlanMode 审批对话框需要终端。

---

### Plan Mode 转换

```ts
handlePlanModeTransition(appState.toolPermissionContext.mode, 'plan')
context.setAppState(prev => ({
  ...prev,
  toolPermissionContext: applyPermissionUpdate(
    prepareContextForPlanMode(prev.toolPermissionContext),
    { type: 'setMode', mode: 'plan', destination: 'session' },
  ),
}))
```

转换涉及两个操作：
1. `handlePlanModeTransition`——处理计划模式转换
2. `applyPermissionUpdate`——更新 `toolPermissionContext.mode`

---

### 结果消息：指导内容

Plan mode 进入后，结果消息包含详细指导：

```
In plan mode, you should:
1. Thoroughly explore the codebase
2. Identify similar features and approaches
3. Consider multiple approaches and their trade-offs
4. Use AskUserQuestion if you need to clarify
5. Design a concrete implementation strategy
6. When ready, use ExitPlanMode to present plan for approval
```

在 plan mode interview phase 中，指导更简短。这告诉模型在 plan 阶段做什么。

---

### Plan Mode 状态

`prepareContextForPlanMode` 运行分类器激活副作用——当用户的 defaultMode 是 'auto' 时，`permissionSetup.ts` 中有完整的生命周期处理。

---

### 读完后应抓住的 3 个事实

1. **Plan model 是权限转换**——EnterPlanModeTool 不仅仅是状态改变，它通过 `applyPermissionUpdate` 和 `prepareContextForPlanMode` 更新整个 `toolPermissionContext`。Plan 模式有不同工具（主要是只读）和权限规则。

2. **权限永远 ask**——没有 auto-allow 路径。这是有意的——用户在模型进入 plan 模式时必须明确批准。

3. **Channel 模式下禁用**——EnterPlanMode 和 ExitPlanMode 在频道模式下都禁用。这意味着模型不能在没有终端的 Telegram/Discord 频道中进入 plan 模式——这是防死锁设计。
