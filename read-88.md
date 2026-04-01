## 第 88 站：`src/tools.ts`

### 这是什么文件

`src/tools.ts` 是工具子系统的 **注册/组装中枢：汇集所有内置工具实例（含条件加载和 feature flag 控制），提供 preset 解析、deny rule 过滤、REPL/special mode 处理，以及与 MCP 工具的合并去重逻辑。**

---

### `getAllBaseTools()`

位置：`src/tools.ts:193`

这是最权威的内置工具来源。返回的列表包含：

| 工具 | 说明 |
|---|---|
| AgentTool | Agent（子代理）工具 |
| TaskOutputTool | 任务输出读取 |
| BashTool | Shell 命令执行 |
| GlobTool / GrepTool | 文件搜索（如无嵌入式搜索时加载） |
| ExitPlanModeV2Tool | 退出计划模式 |
| FileReadTool / FileEditTool / FileWriteTool | 文件操作 |
| NotebookEditTool | Notebook 编辑 |
| WebFetchTool | Web 内容获取 |
| TodoWriteTool | 待办事项 |
| WebSearchTool | Web 搜索 |
| TaskStopTool | 任务停止 |
| AskUserQuestionTool | 提问用户 |
| SkillTool | 技能执行 |
| EnterPlanModeTool | 进入计划模式 |
| ConfigTool | 配置（ant-only） |
| TungstenTool | Tungsten（ant-only） |
| SuggestBackgroundPRTool | 背景 PR 建议（ant-only） |
| WebBrowserTool | Web 浏览器 |
| TaskCreate / TaskGet / TaskUpdate / TaskList | 任务管理（todoV2） |
| AskUserQuestionTool | 提问 |
| OverflowTestTool / CtxInspectTool | 测试/上下文工具 |
| TerminalCaptureTool | 终端捕获 |
| LSPTool | LSP 语言服务器 |
| EnterWorktreeTool / ExitWorktreeTool | 工作树 |
| SendMessageTool | 发送消息 |
| ListPeersTool | 列出对等端 |
| TeamCreateTool / TeamDeleteTool | 团队（agent swarms） |
| VerifyPlanExecutionTool | 验证计划执行 |
| REPLTool | REPL（ant-only） |
| WorkflowTool | 工作流脚本 |
| SleepTool | 休眠 |
| ScheduleCronTool: CronCreate/Delete/List | 定时任务 |
| RemoteTriggerTool | 远程触发 |
| MonitorTool | 监控 |
| BriefTool | 简报 |
| SendUserFileTool / PushNotificationTool / SubscribePRTool | 文件/通知/GitHub |
| PowerShellTool | PowerShell |
| SnipTool | 剪贴 |
| TestingPermissionTool | 测试专用 |
| ListMcpResourcesTool / ReadMcpResourceTool | MCP 资源 |
| ToolSearchTool | 工具搜索 |

注意这里使用了**大量 feature flag 和 `require()` 延迟加载**来：
1. 实现 tree-shaking（bun 编译时不导入未启用模块）
2. 打破循环依赖
3. 支持不同部署环境的功能差异

---

### `getTools(permissionContext)`

位置：`src/tools.ts:271`

这个函数根据 permission context 返回可用的内置工具集：

#### CLAUDE_CODE_SIMPLE 模式
只返回 `BashTool + FileReadTool + FileEditTool (+ AgentTool + TaskStopTool + SendMessageTool 如 coordinator 模式)`。

如果 REPL mode 启用，则只返回 `REPLTool`。

#### 正常模式
1. 从 `getAllBaseTools()` 获取
2. 排除特殊工具：`ListMcpResourcesTool, ReadMcpResourceTool, SYNTHETIC_OUTPUT_TOOL`
3. 应用 deny rules 过滤
4. REPL 模式下隐藏 REPL_ONLY_TOOLS（原始工具由 REPL 包装）
5. 只保留 `isEnabled() === true` 的工具

---

### `assembleToolPool(permissionContext, mcpTools)`

位置：`src/tools.ts:345`

这是内置工具与 MCP 工具合并的标准化入口：

```ts
assembleToolPool() =
  uniqBy(
    [builtInTools].sort(byName)
      .concat([allowedMcpTools].sort(byName)),
    'name'
  )
```

- 按名称排序两个分区（内置在前，MCP 在后）
- 名称冲突时内置工具优先
- 排序用于 prompt cache 稳定性——服务器在最后一个内置工具后设置 cache breakpoint

---

### `getMergedTools(permissionContext, mcpTools)`

更简单的合并不去重——直接拼接两个数组。

---

### `filterToolsByDenyRules(tools, permissionContext)`

位置：`src/tools.ts:262`

过滤掉在权限上下文中被 deny rule 屏蔽的工具：

```ts
tools.filter(tool => !getDenyRuleForTool(permissionContext, tool))
```

使用与运行时权限检查相同的匹配器——MCP 服务器前缀规则（如 `mcp__server`）在模型看见之前就屏蔽该服务器的所有工具。

---

### `TOOL_PRESETS`

位置：`src/tools.ts:161`

当前只支持 `'default'` preset。

---

### 条件加载模式

这个文件使用的加载模式很有意思：

#### `process.env.USER_TYPE === 'ant'`
- REPLTool, ConfigTool, TungstenTool, SuggestBackgroundPRTool

#### `feature('...')` (Bun feature flags)
- PROACTIVE -> SleepTool
- KAIROS -> SleepTool, SendUserFileTool, PushNotificationTool
- AGENT_TRIGGERS -> cronTools
- AGENT_TRIGGERS_REMOTE -> RemoteTriggerTool
- MONITOR_TOOL -> MonitorTool
- KAIROS_PUSH_NOTIFICATION -> PushNotificationTool
- KAIROS_GITHUB_WEBHOOKS -> SubscribePRTool
- COORDINATOR_MODE -> coordinatorModeModule
- OVERFLOW_TEST_TOOL -> OverflowTestTool
- CONTEXT_COLLAPSE -> CtxInspectTool
- TERMINAL_PANEL -> TerminalCaptureTool
- WEB_BROWSER_TOOL -> WebBrowserTool
- HISTORY_SNIP -> SnipTool
- UDS_INBOX -> ListPeersTool
- WORKFLOW_SCRIPTS -> WorkflowTool

#### `feature('...') + env vars`
- CLAUDE_CODE_VERIFY_PLAN -> VerifyPlanExecutionTool
- ENABLE_LSP_TOOL -> LSPTool
- NODE_ENV === 'test' -> TestingPermissionTool

#### 运行时函数
- `hasEmbeddedSearchTools()` -> 有嵌入式搜索则不需要 Glob/Grep
- `isTodoV2Enabled()` -> Task 工具组
- `isWorktreeModeEnabled()` -> Worktree 工具
- `isAgentSwarmsEnabled()` -> Team 工具
- `isPowerShellToolEnabled()` -> PowerShellTool
- `isToolSearchEnabledOptimistic()` -> ToolSearchTool

---

### 读完这一站后，你应该抓住的 6 个事实

1. `src/tools.ts` 是工具池组装的唯一真实来源——所有内置工具通过 `getAllBaseTools()` 定义，通过 `getTools()` 过滤，通过 `assembleToolPool()` 与 MCP 合并。
2. 50+ 个工具的加载受 feature flags 控制，支持不同部署环境和实验功能的渐进式发布。
3. `getTools()` 在 `CLAUDE_CODE_SIMPLE` 模式下只返回 3 个基本工具（Bash/Read/Edit），用于简化模型。
4. `assembleToolPool()` 对内置工具和 MCP 工具分开排序后合并，目的是保持 prompt 缓存前缀的稳定性——MCP 工具插在内置工具后面。
5. Deny rules 在工具到达模型前就过滤工具，不仅阻止调用时检查，也阻止 schema 可见性。
6. REPL 模式下，原始工具（Bash/Read/Edit 等）被 REPLTool 替代，原始工具只在 REPL 内部通过 VM 上下文访问。
