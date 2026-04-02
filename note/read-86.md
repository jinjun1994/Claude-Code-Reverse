## 第 86 站：`src/Tool.ts`

### 这是什么文件

`src/Tool.ts` 是整个工具执行子系统的 **类型协议中枢：定义 `Tool<Input, Output, Progress>` 接口的完整签名（50+ 属性），`ToolUseContext` 执行上下文（290 多行的状态/回调/UI 句柄），`ToolPermissionContext` 权限上下文结构，`ToolResult<T>` 返回值，以及一组查找辅助函数。**

这个文件是工具层的核心合同——每个内置工具（Bash、Read、Edit、Agent 等）和 MCP 工具都必须遵守这个接口。

---

### `Tool` 接口的关键属性分类

#### 身份与可见性
| 属性 | 说明 |
|---|---|
| `name` | 工具主名（必填） |
| `aliases?` | 向后兼容别名 |
| `searchHint?` | 3-10 字能力短语，供 ToolSearch 关键词匹配 |
| `userFacingName(input)` | 面向用户的显示名 |
| `prompt(options)` | 工具描述 prompt（给 model 看的说明） |

#### 输入/输出验证
| 属性 | 说明 |
|---|---|
| `inputSchema` | Zod 输入 schema（必填） |
| `inputJSONSchema?` | MCP 工具的 JSON Schema 直接格式 |
| `outputSchema?` | Zod 输出 schema |
| `validateInput?(input, context)` | 输入校验，失败返回 ValidationResult |

#### 执行核心
| 属性 | 说明 |
|---|---|
| `call(args, context, canUseTool, parentMessage, onProgress)` | 执行入口（必填） |
| `checkPermissions(input, context)` | 工具特定权限检查 |
| `preparePermissionMatcher?(input)` | 为 hook `if` 条件准备 matcher |
| `inputsEquivalent?(a, b)` | 判断两个输入是否等价 |

#### UI 与展示
| 属性 | 说明 |
|---|---|
| `renderToolResultMessage?(content, progressMessages, options)` | 渲染结果 UI |
| `extractSearchText?(out)` | 转录搜索索引用纯文本提取 |
| `getToolUseSummary?(input)` | 紧凑视图短描述 |
| `getActivityDescription?(input)` | Spinner 活动描述（"Reading src/foo.ts"） |
| `userFacingNameBackgroundColor?(input)` | 名称背景色 |
| `isSearchOrReadCommand?(input)` | 是否可折叠的搜索/读取操作 |
| `isTransparentWrapper?()` | 透明包装器（不自己渲染） |

#### 安全与控制
| 属性 | 说明 |
|---|---|
| `isConcurrencySafe(input)` | 是否可并发执行 |
| `isEnabled()` | 是否启用 |
| `isReadOnly(input)` | 是否只读操作 |
| `isDestructive?(input)` | 是否破坏性操作（删除/覆盖/发送） |
| `interruptBehavior?()` | 用户中断时的行为：`cancel` 或 `block` |
| `isOpenWorld?(input)` | 是否开放世界（不受限）操作 |
| `requiresUserInteraction?()` | 是否需要用户交互 |
| `toAutoClassifierInput(input)` | 自动模式安全分类器输入 |

#### 工具加载策略
| 属性 | 说明 |
|---|---|
| `shouldDefer?` | 延迟加载（需 ToolSearch 先启用） |
| `alwaysLoad?` | 永不延迟，首轮必须加载 |
| `strict?` | 严格模式（API 更严格遵循指令/schemas） |
| `isMcp?` | MCP 协议工具标记 |
| `isLsp?` | LSP 协议工具标记 |
| `mcpInfo?` | MCP 服务名/工具名（未经标准化） |

#### 其他
| 属性 | 说明 |
|---|---|
| `maxResultSizeChars` | 工具结果大小限制（超则存入文件） |
| `backfillObservableInput?(input)` | 在 observers 看到前注入衍生字段 |
| `getPath?(input)`| 获取操作的文件路径（文件操作工具用） |
| `mapToolResultToToolResultBlockParam(content, toolUseID)` | 结果转换为 API block 参数 |

---

### `ToolUseContext`

这是执行时最丰富的上下文结构，包含近 300 行的字段：

#### options（工具池与配置）
- `commands` - 可用命令列表
- `tools` - 当前工具池
- `mainLoopModel` - 主循环模型
- `debug/verbose` - 调试模式
- `thinkingConfig` - 思考模式配置
- `mcpClients` / `mcpResources` - MCP 客户端和资源
- `isNonInteractiveSession` - 非交互模式
- `agentDefinitions` - Agent 定义
- `customSystemPrompt?` / `appendSystemPrompt?` - 系统提示覆盖
- `querySource?` - 分析来源
- `refreshTools?()` - 获取最新工具的回调

#### 执行控制
- `abortController` - 中止控制器
- `messages` - 消息历史
- `getAppState()` / `setAppState(f)` - 状态访问/更新
- `setAppStateForTasks?` - 不受 agent 隔离限制的状态更新

#### 文件/状态缓存
- `readFileState` - 文件状态缓存
- `loadedNestedMemoryPaths?` - 已注入的 CLAUDE.md 路径（去重用）
- `updateFileHistoryState` - 文件历史更新
- `updateAttributionState` - 代码归属状态更新
- `contentReplacementState?` - 每线程内容替换状态

#### UI 回调
- `setToolJSX?` - 设置 JSX UI
- `addNotification?` - 添加通知
- `sendOSNotification?` - 发送系统通知
- `setStreamMode?` - 设置 Spinner 模式
- `setSDKStatus?` - 设置 SDK 状态
- `openMessageSelector?` - 打开消息选择器

#### Agent/任务
- `agentId?` - Agent ID
- `agentType?` - Agent 类型名称
- `queryTracking?` - 查询链跟踪（chainId, depth）
- `toolDecisions?` - 工具决定记录（accept/reject）

#### 权限与验证
- `toolPermissionContext` (通过 getAppState) - 权限上下文
- `localDenialTracking?` - 本地拒绝计数（异步 subagent 用）
- `requireCanUseTool?` - 是否必须调用 canUseTool

#### 通信
- `requestPrompt?` - 交互提示回调
- `toolUseId?` - 当前工具使用 ID
- `handleElicitation?` - URL elicitation 处理

#### 性能与内存
- `fileReadingLimits?` - 文件读取大小限制
- `globLimits?` - glob 最大结果数
- `setHasInterruptibleToolInProgress?` - 可中断工具进度标记
- `setResponseLength` - 响应长度跟踪（spinner 显示用）
- `pushApiMetricsEntry?` - API 指标推送（ant OTPS 用）

#### MCP 与技能
- `nestedMemoryAttachmentTriggers?` - 嵌套内存附件触发器
- `dynamicSkillDirTriggers?` - 动态技能目录触发器
- `discoveredSkillNames?` - 已发现技能名称
- `renderedSystemPrompt?` - 父级渲染的系统提示（fork agent prompt 缓存共享用）

---

### `ToolPermissionContext`

权限上下文的完整结构：

```ts
{
  mode: PermissionMode
  additionalWorkingDirectories: Map<string, AdditionalWorkingDirectory>
  alwaysAllowRules: ToolPermissionRulesBySource
  alwaysDenyRules: ToolPermissionRulesBySource
  alwaysAskRules: ToolPermissionRulesBySource
  isBypassPermissionsModeAvailable: boolean
  isAutoModeAvailable?: boolean
  strippedDangerousRules?: ToolPermissionRulesBySource
  shouldAvoidPermissionPrompts?: boolean        // 后台 agent 需要自动拒绝
  awaitAutomatedChecksBeforeDialog?: boolean    // coordinator worker 等自动检查
  prePlanMode?: PermissionMode                  // plan mode 进入前保存旧模式
}
```

---

### `ToolResult<T>`

工具返回值包括：

- `data: T` - 核心返回数据
- `newMessages?` - 附加消息（user/assistant/attachment/system）
- `contextModifier?` - 修改 ToolUseContext（非并发安全工具用）
- `mcpMeta?` - MCP 元数据（structuredContent, _meta）

---

### `ToolInputJSONSchema`

MCP 工具可以直接提供 JSON Schema 而不经由 Zod 转换：

```ts
{
  type: 'object'
  properties?: { [x: string]: unknown }
}
```

---

### `ValidationResult`

输入校验结果：

```ts
| { result: true }
| { result: false, message: string, errorCode: number }
```

---

### `toolMatchesName(tool, name)` 与 `findToolByName(tools, name)`

支持通过主名或别名查找工具：

```ts
tool.name === name || tool.aliases?.includes(name)
```

---

### 关键观察

1. **Tool 接口极其丰富**（50+ 属性），不仅定义执行逻辑，还覆盖 UI、安全、性能、analytics、搜索。
2. `ToolUseContext` 包含 40+ 个字段，反映一个工具执行时需要访问的几乎所有系统状态。
3. 每个属性都有明确的设计意图注释，很少有随意添加的字段。
4. 工具不是简单的执行函数，它们是一个完整的组件接口——既有后端执行逻辑，也有前端渲染和搜索索引。
5. `preparePermissionMatcher` 让工具自己决定 hook `if` 条件的匹配逻辑（如 Bash 解析命令行的 pattern）。
6. `backfillObservableInput` 让工具在 observers/streams 看到之前注入衍生字段。
7. `contextModifier` 允许非并发安全工具在执行时安全地修改执行上下文。
8. `shouldDefer` / `alwaysLoad` 控制工具 schema 是立即发送还是延迟按需加载。

---

### 元

问题：**这一站真正想解决的架构问题是什么？**

回答：这一站的元问题，是工具子系统为什么必须先有一份超明确的类型协议，而不是先写工具实现。`Tool`、`ToolUseContext`、`ToolPermissionContext` 和 `ToolResult` 共同定义了工具被系统理解的方式。

### 反

问题：**如果把这一站的设计反过来，会发生什么？**

回答：如果没有这份统一合同，每个工具都会对输入、权限、UI、进度和结果各说各话。到最后工具数量越多，宿主越难知道“一个工具到底是什么”。

### 空

问题：**跳出当前文件名，这一站背后更大的问题是什么？**

回答：更大的问题，是平台怎样把异构能力都纳入同一执行接口。`Tool.ts` 的价值，就是把“工具”从实现类提升成一套共享制度。
