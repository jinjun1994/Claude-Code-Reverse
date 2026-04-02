## 第 67 站：`src/utils/hooks.ts`

### 这是什么文件

`src/utils/hooks.ts` 是 Claude Code hook 子系统里的 **统一 hook 匹配器、执行编排器、协议解释层与事件分发中枢**。

上一站权限链已经看懂：

- `permissions.ts` 会在 ask-path、headless path、tool loop 中调用 hook 相关逻辑
- `coordinatorHandler.ts` / `interactiveHandler.ts` 也会把 hooks 当成正式审批源
- 但“hooks 本身到底怎样配置、怎样匹配、怎样执行、怎样把 shell/HTTP/prompt/agent/function/callback 多种 hook 统一成同一套结果协议”，还没拆开

这一站回答的是：

```text
Claude Code 的 hooks 基础设施怎样把：
- 不同来源的 hook 配置
- 不同事件类型
- 不同 hook 载体（command / http / prompt / agent / callback / function）
- 不同输出协议（plain text / JSON / async JSON）
统一收敛成一套可复用的执行与结果分发框架
```

所以最准确的一句话是：

```text
hooks.ts = Claude Code hook 系统的运行时中枢：负责收集匹配 hook、执行它们、解释输出协议，并把结果投影回 tool loop / permission flow / session lifecycle
```

---

### 先看它的整体定位

位置：`src/utils/hooks.ts:1`

这个文件不是：

- hooks schema 定义文件
- settings snapshot 采集器
- 某一种 hook（例如 HTTP hook）的单独实现本体
- 某个具体事件（如 PermissionRequest）专用的小模块

它的职责是：

1. 组装 hook 输入 payload
2. 从 settings / registered hooks / session hooks 中合并可用 hook
3. 按 matcher 与 `if` 条件筛选真正命中的 hook
4. 统一执行 command / http / prompt / agent / callback / function hooks
5. 解析 plain text / structured JSON / async JSON 输出
6. 把 hook 结果折叠成统一 `AggregatedHookResult`
7. 为不同事件暴露 `executeXxxHooks(...)` 入口
8. 在 trust / managed-only / disable-all / telemetry / tracing / env-file 等产品约束下安全运行 hooks

所以它本质上是一个：

```text
hook runtime orchestrator
+ hook protocol interpreter
+ lifecycle event dispatcher
```

也就是说，这个文件不是某个 hook feature 的“边角工具”，
而是整套 hooks 机制真正的 runtime core。

---

### 第一部分：文件头 import 已经说明这不是“跑个 shell 命令”那么简单，而是把 settings、session、permission parser、plugin、telemetry、message attachment、subprocess、prompt bridge 全卷进来的平台层

位置：`src/utils/hooks.ts:1`

从 import 就能看出，这个文件同时依赖：

- settings / hooks snapshot / managed-only policy
- plugin options / plugin root / skill root
- permission rule parser
- Tool / ToolUseContext / input schema
- telemetry / OTEL / tracing / analytics
- async hook registry / message queue / task notification
- prompt request schema / structured JSON schema
- session hooks / registered hooks / callback hooks
- env file / cwd / project root / transcript path

这说明 `hooks.ts` 的真实职责不是：

```text
given a hook, execute it
```

而是：

```text
take a lifecycle event,
resolve all applicable hooks across multiple sources,
execute them under product/security policies,
and normalize their outputs into Claude Code runtime semantics
```

所以它是一个 hooks 平台层，不是单点 helper。

---

### 第二部分：`shouldSkipHookDueToTrust()` 非常关键，说明 hook 安全边界首先不是 matcher 或 event，而是 workspace trust；所有 hooks 在 interactive 模式下都被统一放在“必须先信任工作区”之后

位置：`src/utils/hooks.ts:286`

这里逻辑非常硬：

- 非 interactive / SDK 模式：默认允许
- interactive 模式：所有 hooks 都要求 trust dialog 已接受

注释还特别列出历史问题：

- `SessionEnd` hooks 在用户拒绝 trust 后仍执行
- `SubagentStop` hooks 可能在 trust 建立前触发

这说明 hooks 在 Claude Code 的安全模型里被视为：

```text
arbitrary command execution capability
```

因此信任边界不是某几个危险 hook 单独处理，
而是在统一执行入口 `executeHooks(...)` / `executeHooksOutsideREPL(...)` 前集中拦截。

也就是说这个文件的安全哲学是：

```text
trust gating is global and centralized,
not left to individual hook call sites
```

---

### 第三部分：`createBaseHookInput(...)` 说明 hook 输入协议的核心不是“某个局部事件参数”，而是每种 hook 都自动带上 session / transcript / cwd / permission mode / agent identity 等运行时上下文

位置：`src/utils/hooks.ts:301`

这里统一注入：

- `session_id`
- `transcript_path`
- `cwd`
- `permission_mode`
- `agent_id`
- `agent_type`

这意味着 hooks 不是只收到：

```text
the local event payload
```

而是收到：

```text
event payload
+ runtime provenance
+ execution scope
+ session identity
```

尤其 `agent_type` 的注释很关键：

- subagent 的 type 优先于主线程 `--agent` flag
- hooks 靠 `agent_id` 区分 subagent 与 main-thread 调用

这说明 hooks 协议从一开始就为多 agent 运行时设计了 provenance 语义，而不是后面零散拼补。

---

### 第四部分：`parseHookOutput(...)` / `parseHttpHookOutput(...)` / `processHookJSONOutput(...)` 是整份文件的协议解释核心——说明 hook 不只是“退出码 + 文本”，而是一个支持结构化控制语义的正式 wire protocol

位置：

- parse stdout `src/utils/hooks.ts:399`
- parse HTTP body `src/utils/hooks.ts:453`
- process JSON `src/utils/hooks.ts:489`

这三层共同定义的 hook 输出协议非常清楚：

#### 非 JSON stdout
按 plain text 处理

#### JSON stdout / body
按 schema 校验后解释结构化字段

#### async JSON
走后台异步 hook 协议

而 `processHookJSONOutput(...)` 能提取的不只是 approve / block：

- `continue: false`
- `stopReason`
- `systemMessage`
- `permissionDecision`
- `permissionDecisionReason`
- `updatedInput`
- `additionalContext`
- `updatedMCPToolOutput`
- `permissionRequestResult`
- `watchPaths`
- `elicitationResponse`
- `retry`

这说明 hook JSON 不是简单“返回一个 decision”，而是：

```text
a structured side-channel protocol
for mutating runtime behavior, enriching context, and steering the next step
```

所以 `hooks.ts` 的核心价值之一就是：

```text
turn raw hook process output into first-class Claude Code runtime signals
```

---

### 第五部分：`execCommandHook(...)` 的复杂度非常高，说明 command hook 执行在产品里已经不是朴素 spawn，而是一个完整的跨平台 shell 适配层、plugin variable 注入层、async/prompt 协议执行层

位置：`src/utils/hooks.ts:747`

这段逻辑同时处理了很多产品级细节：

#### shell 选择
- `hook.shell ?? DEFAULT_HOOK_SHELL`
- bash 与 PowerShell 走完全不同 spawn 路径

#### Windows path 语义
- Git Bash 需要 POSIX path
- PowerShell 保持 native path

#### plugin / skill 变量替换
- `${CLAUDE_PLUGIN_ROOT}`
- `${CLAUDE_PLUGIN_DATA}`
- `${user_config.X}`

#### env 注入
- `CLAUDE_PROJECT_DIR`
- `CLAUDE_PLUGIN_ROOT`
- `CLAUDE_PLUGIN_OPTION_*`
- `CLAUDE_ENV_FILE`

#### cwd 容错
- worktree 被删除时 fallback 到 `getOriginalCwd()`

#### stdin/stdout 协议
- 传 JSON input
- 监测 prompt request 行
- 监测 async JSON 首行
- 处理 EPIPE / abort / stream-end race

这说明 command hook 执行的真实语义是：

```text
spawn an external automation endpoint
under a controlled runtime contract,
not merely run a shell snippet
```

也就是说这里已经是 hook runtime 的 mini process platform。

---

### 第六部分：async hook 机制很关键，说明 hooks 系统不要求所有 hook 同步返回；它支持“先声明 async，再后台继续跑”的双阶段协议，避免长耗时 hook 阻塞主流程

位置：

- async backgrounding `src/utils/hooks.ts:184`
- stdout first-line async detection `src/utils/hooks.ts:1112`
- config-based async `src/utils/hooks.ts:995`

这里有两种 async 入口：

#### hook 输出首行是 async JSON
运行时检测后转后台执行

#### hook 配置自带 `async` / `asyncRewake`
直接按后台 hook 处理

而 `asyncRewake` 更进一步：

- 不进普通 registry
- 结束后若 exit code 2，会 enqueue 为 `task-notification`
- 可以在 idle 或 mid-query 时重新唤醒模型

这说明 hooks 在系统里的定位已经超出“阻塞式前后置脚本”，而是具备：

```text
background automation with later re-entry into the conversation runtime
```

这和普通 CLI hooks 相比要成熟得多。

---

### 第七部分：`matchesPattern(...)` + `prepareIfConditionMatcher(...)` 说明 hook 匹配分两层：先按 event-level matcher 选候选，再按 `if` 条件做工具语义级过滤；其中 `if` 直接复用了 permission rule DSL

位置：

- matcher `src/utils/hooks.ts:1346`
- if matcher prepare `src/utils/hooks.ts:1390`

第一层 matcher 支持：

- `*`
- 精确字符串
- `A|B|C`
- regex
- legacy tool name 兼容

第二层 `if` 更有意思：

- 仅 tool-like events 可用
- 先 parse tool input schema
- 如果 tool 提供 `preparePermissionMatcher(...)`
- 再把 `ifCondition` 用 `permissionRuleValueFromString(...)` 解析成 permission rule DSL
- 用 tool-specific matcher 判断 `ruleContent` 是否匹配

这说明 hook `if` 条件不是独立再造一门语法，
而是直接复用权限规则语言：

```text
hook conditional matching reuses the permission rule DSL
```

所以 hooks 与 permissions 并不是松耦合并列系统，
而是共享同一套工具内容匹配语义基础设施。

---

### 第八部分：`getHooksConfig(...)` 非常关键，因为它把 hook 来源治理统一落地——snapshot hooks、registered hooks、session hooks 会在这里合并，同时受 managed-only 策略影响

位置：`src/utils/hooks.ts:1492`

这里依次合并：

1. snapshot hooks
2. registered hooks（SDK callbacks / plugin native hooks）
3. session hooks
4. session function hooks

但中间穿插了非常关键的治理逻辑：

- `shouldAllowManagedHooksOnly()` 为真时
  - plugin registered hooks 会被跳过
  - session hooks 整体跳过

这说明 hooks 的来源治理和权限规则的来源治理是同一哲学：

```text
not every hook source remains authoritative under managed-only policy
```

也就是说这个文件不仅执行 hooks，
还在这里落地“谁有资格参与 hook runtime”的组织策略。

---

### 第九部分：`getMatchingHooks(...)` 是 hooks runtime 的选择器核心，它把“事件 -> matchQuery -> matcher 过滤 -> 去重 -> if 条件过滤 -> 特定事件限制”收敛成统一 matched hook 集

位置：`src/utils/hooks.ts:1603`

这里做了五层事：

#### 1. 为不同 hook event 计算 `matchQuery`
例如：
- `PreToolUse` / `PostToolUse` -> `tool_name`
- `SessionStart` -> `source`
- `Setup` -> `trigger`
- `Notification` -> `notification_type`
- `InstructionsLoaded` -> `load_reason`
- `FileChanged` -> `basename(file_path)`

#### 2. matcher 过滤
`matchesPattern(matchQuery, matcher.matcher)`

#### 3. 附上 plugin / skill / settings provenance
形成 `MatchedHook`

#### 4. 去重
- command hook：按 `shell + command + if`
- prompt / agent：按 `prompt + if`
- http：按 `url + if`
- callback / function 不去重

#### 5. `if` 条件过滤与 event-specific 限制
- `if` 不匹配则跳过
- `SessionStart` / `Setup` 明确禁用 HTTP hooks

所以这层做的本质是：

```text
from all configured hooks,
derive the exact executable hook set for this runtime event
```

而不是简单数组过滤。

---

### 第十部分：`executeHooks(...)` 是整份文件真正的主执行内环——它把多种 hook 载体统一成同一个异步生成器协议，并在内部做并行执行、增量结果产出与聚合优先级处理

位置：`src/utils/hooks.ts:1952`

这个函数的结构非常关键：

#### 前置 gate
- `disableAllHooks`
- `CLAUDE_CODE_SIMPLE`
- trust check
- `getMatchingHooks(...)`

#### fast-path
如果全是 internal callback hooks：
- 直接执行
- 跳过 span / progress / full result loop

#### 普通路径
- 发 progress message
- 懒序列化一次 JSON input
- 针对每个 matched hook 建一个 async generator
- command / prompt / agent / http / callback / function 分支统一进来
- 用 `all(hookPromises)` 并行消费

#### 结果聚合
- 统计 success / blocking / non_blocking_error / cancelled
- 逐条 yield message / blockingError / additionalContext / updatedInput / permissionBehavior / permissionRequestResult / watchPaths / elicitationResponse
- 用 deny > ask > allow 做 permissionBehavior 优先级折叠

这里最重要的是：

```text
executeHooks does not return one final hook verdict only;
it streams hook side effects as they arrive,
while also maintaining aggregate semantics across the batch
```

所以它既是：

- execution engine
- result multiplexer
- precedence resolver
- telemetry collector

四者合一。

---

### 第十一部分：permission 相关聚合逻辑尤其重要，说明 hooks 可以正式参与权限系统，而且多 hook 冲突时有稳定优先级：`deny > ask > allow > passthrough`

位置：`src/utils/hooks.ts:2820`

这里的聚合规则非常清楚：

- `deny` 永远最高
- `ask` 压过 `allow`
- `allow` 只有在没有更强意见时才成立
- `passthrough` 不建立 permission behavior

而且 `updatedInput` 还跟 permission decision 绑定：

- `allow` / `ask` 时可以伴随更新后的 input
- 无 permission decision 时，也能单独返回 `updatedInput`

再加上 `PermissionRequest` hook 可以给 `permissionRequestResult`，这说明 hooks 在权限系统里不是外围观察者，而是：

```text
first-class permission participants
with structured authority to modify inputs and steer approval flow
```

这也正好解释了前面 `permissions.ts` 和 `coordinatorHandler.ts` 为什么会把 hooks 当正式审批源。

---

### 第十二部分：`executeHooksOutsideREPL(...)` 和一批 `executeXxxHooks(...)` 导出函数说明这个文件把 hooks 平台同时暴露给两种运行语境：REPL 内的“对模型可见”执行，以及 REPL 外的“系统生命周期/后台事件”执行

位置：

- outside REPL `src/utils/hooks.ts:3003`
- `executePreToolHooks` `src/utils/hooks.ts:3394`
- `executePostToolHooks` `src/utils/hooks.ts:3450`
- `executePostToolUseFailureHooks` `src/utils/hooks.ts:3492`
- `executePermissionDeniedHooks` `src/utils/hooks.ts:3529`
- `executeNotificationHooks` `src/utils/hooks.ts:3570`
- `executeStopHooks` `src/utils/hooks.ts:3639`
- `executePermissionRequestHooks` `src/utils/hooks.ts:4157`
- `executeConfigChangeHooks` `src/utils/hooks.ts:4214`
- `executeCwdChangedHooks` `src/utils/hooks.ts:4260`
- `executeFileChangedHooks` `src/utils/hooks.ts:4278`
- `executeInstructionsLoadedHooks` `src/utils/hooks.ts:4335`

这里的边界非常清晰：

#### executeHooks
- yield attachment/progress/additionalContext
- 面向 turn / tool loop / permission flow
- 结果可以被模型或当前 query 消费

#### executeHooksOutsideREPL
- 返回 `HookOutsideReplResult[]`
- 不直接往模型消息面注入内容
- 更适合 notification / session end / config change / env change 等系统事件

这说明 hooks 子系统不是只服务工具调用，
而是一个横跨：

```text
tool lifecycle
permission lifecycle
session lifecycle
environment change lifecycle
```

的统一自动化底座。

---

### 第十三部分：几个事件专用导出函数还体现了很强的产品级特殊语义，而不只是机械包装 `executeHooks(...)`

举几个最关键的例子：

#### `executePermissionRequestHooks(...)`
位置：`src/utils/hooks.ts:4157`

- 专门构造 `PermissionRequestHookInput`
- 把 `permissionSuggestions` 一起给 hook
- 这就是 ask-path hooks 能参与审批决策的入口

#### `executeConfigChangeHooks(...)`
位置：`src/utils/hooks.ts:4214`

- policy settings 变化也会触发 hooks
- 但其 blocking 结果会被强制忽略

说明：

```text
enterprise policy changes are auditable by hooks,
but must never be blockable by hooks
```

#### `executeCwdChangedHooks(...)` / `executeFileChangedHooks(...)`
位置：`src/utils/hooks.ts:4260`, `src/utils/hooks.ts:4278`

- 走 `executeEnvHooks(...)`
- 结果会触发 `invalidateSessionEnvCache()`
- 并返回 `watchPaths`

说明 hooks 还能参与环境脚本/动态上下文刷新链路。

#### `executeInstructionsLoadedHooks(...)`
位置：`src/utils/hooks.ts:4335`

- 纯 observability / audit
- 不支持 blocking
- 专门追踪 CLAUDE.md / rules 文件被加载进入上下文

这说明 hook 事件并非同质，
而是按产品语义分层设计的。

---

### 第十四部分：整份文件最核心的架构价值，是把 hooks 从“若干 shell callback”提升成了一套有来源治理、协议语义、并发执行、事件分发、权限耦合与生命周期覆盖能力的正式自动化平台

如果把整份文件压缩，会发现它其实在做六层事：

#### 1. input protocol
- `createBaseHookInput`
- 各类 `HookInput` 构造

#### 2. source governance
- `getHooksConfig`
- managed-only / disable-all / trust gate

#### 3. matching & selection
- `matchesPattern`
- `prepareIfConditionMatcher`
- `getMatchingHooks`

#### 4. execution runtime
- `execCommandHook`
- HTTP / prompt / agent / callback / function 分支
- async/background hooks

#### 5. output protocol interpretation
- `parseHookOutput`
- `parseHttpHookOutput`
- `processHookJSONOutput`
- precedence aggregation

#### 6. event dispatch surface
- `executePreToolHooks`
- `executePermissionRequestHooks`
- `executeStopHooks`
- `executeConfigChangeHooks`
- `executeInstructionsLoadedHooks`
- 其他 session / env / notification 入口

所以最准确的压缩表达是：

```text
hooks.ts = Claude Code hooks 自动化平台的运行时中枢：负责在多种生命周期事件上，从多来源收集并筛选 hooks，统一执行不同载体的 hook，并把它们的文本/JSON/异步输出解释为 Claude Code 可消费的权限、上下文、阻断与观测结果
```

---

### 元

问题：**这一站真正想解决的架构问题是什么？**

回答：这一站真正要回答的是，为什么 Claude Code 的 hooks 不能只是“执行一条命令”，而必须有一层运行时中枢。它负责收集、匹配、执行、解释并聚合不同 hook 载体与输出协议。

### 反

问题：**如果把这一站的设计反过来，会发生什么？**

回答：如果把 command、http、prompt、agent、callback 各自散开实现，hook 系统会立刻失去统一 contract。到最后每种 hook 都像是单独功能，无法共享事件和结果语义。

### 空

问题：**跳出当前文件名，这一站背后更大的问题是什么？**

回答：更大的问题，是一个可扩展自动化系统怎样把“外部扩展点”纳入正式 runtime。`hooks.ts` 的价值，就是把扩展从零散接口提升成一套平台。

### 读完这一站后，你应该抓住的 10 个事实

1. `src/utils/hooks.ts` 不是单个 hook 实现，而是整个 hooks 子系统的运行时中枢，负责匹配、执行、协议解析和事件分发。
2. 所有 hooks 在 interactive 模式下都统一受 workspace trust gate 约束，说明 hooks 被视为任意命令执行能力，而不是普通配置项。
3. `createBaseHookInput(...)` 会为各类 hook 自动补齐 `session_id`、`transcript_path`、`cwd`、`permission_mode`、`agent_id`、`agent_type`，使 hooks 拥有统一的运行时 provenance。
4. hook 输出协议支持 plain text、sync JSON 和 async JSON；`processHookJSONOutput(...)` 会把它们解释成 `permissionDecision`、`updatedInput`、`additionalContext`、`watchPaths`、`updatedMCPToolOutput` 等正式 runtime 信号。
5. `execCommandHook(...)` 不是简单 spawn，而是一个跨平台 shell 适配层，处理 bash/PowerShell 分流、plugin/skill 变量替换、env 注入、prompt request、async hook 检测和 stream race。
6. `matchesPattern(...)` 与 `prepareIfConditionMatcher(...)` 说明 hook 选择分成 matcher 和 `if` 两层，其中 `if` 直接复用了 permission rule DSL 与 tool-specific permission matcher。
7. `getHooksConfig(...)` 会合并 snapshot hooks、registered hooks、session hooks 和 session function hooks，并在这里执行 managed-only 来源治理。
8. `getMatchingHooks(...)` 会把事件 payload 映射成 `matchQuery`，再做 matcher 过滤、source provenance 附着、去重和 `if` 条件过滤，得到真正要跑的 hooks 集合。
9. `executeHooks(...)` 用异步生成器统一承载 hooks 执行与结果流式产出，并在批量 hook 冲突时按 `deny > ask > allow > passthrough` 聚合 permission behavior。
10. 导出的 `executePreToolHooks(...)`、`executePermissionRequestHooks(...)`、`executeStopHooks(...)`、`executeConfigChangeHooks(...)`、`executeInstructionsLoadedHooks(...)` 等函数表明 hooks 覆盖了 tool、permission、session、env、config、instructions 等多条运行时切面。

---

### 现在把第 66-67 站串起来

```text
permissionsLoader.ts
  -> 从 settings source 读取/写回 PermissionRule，并执行 managed-only 等来源治理
hooks.ts
  -> 从 hooks source 收集/筛选/执行 hook，并把结果投影为权限、上下文、阻断、观测等 runtime 信号
```

所以现在这两站可以一起压缩成：

```text
settings / policy / session
  -> permissionsLoader.ts materializes permission rules
  -> permissions.ts / permissionSetup.ts consume those rules
lifecycle event / tool event / permission ask
  -> hooks.ts materializes hook automations
  -> permission flow / tool loop / session lifecycle consume those hook results
```

也就是说：

```text
permissionsLoader.ts 回答“权限规则怎样从配置来源进入系统，并受哪些来源治理约束”
hooks.ts 回答“hooks 怎样从配置来源进入系统，并在运行时被匹配、执行、解释成正式控制信号”
```

---

### 下一站建议

下一站最顺的是：

```text
src/types/hooks.ts
```

因为现在你已经看懂了：

- `hooks.ts` 是 hooks 运行时中枢
- 但里面大量关键语义都依赖类型协议：
  - `HookJSONOutput`
  - `HookCallbackMatcher`
  - `PromptRequest` / `PromptResponse`
  - `PermissionRequestResult`
  - sync / async JSON 输出边界
- 如果不把 `src/types/hooks.ts` 补齐，hooks runtime 的协议层仍然有一块是“会用但未正式拆开”的

下一步最自然就是把 hooks 的类型与协议面补齐：

**`src/types/hooks.ts` 到底怎样定义 hook 输入输出协议、sync/async JSON 结构、permission request 决策载荷与 prompt-request 子协议，并作为 `hooks.ts` 的正式 wire contract。**
