## 第 65 站：`src/utils/permissions/permissionSetup.ts`

### 这是什么文件

`src/utils/permissions/permissionSetup.ts` 是权限系统里的 **初始权限上下文装配器、mode 切换协调器与 auto-mode 安全护栏层**。

上一站 `permissionRuleParser.ts` 已经看懂：

- 权限规则字符串怎样解析成 `PermissionRuleValue`
- legacy tool names 怎样归一化
- 规则 DSL 的转义和 round-trip 怎样成立

这一站回答的是：

```text
这些规则与 permission mode 到了运行时，
到底怎样被装进 ToolPermissionContext；
CLI / settings / disk / session / add-dir / auto mode / plan mode
这些输入又怎样被统一整理成一份可执行的权限上下文
```

所以最准确的一句话是：

```text
permissionSetup.ts = 权限系统的装配层与模式护栏协调中心
```

---

### 先看它的整体定位

位置：`src/utils/permissions/permissionSetup.ts:1`

这个文件不是：

- 底层 permission rule parser
- ask/allow/deny 的运行时判定器
- interactive permission handler
- 单个 tool 的 permission checker

它的职责是：

1. 选择并初始化起始 permission mode
2. 从磁盘、CLI、session 等来源组装 `ToolPermissionContext`
3. 处理 `--allowed-tools` / `--disallowed-tools` / `baseTools` / `--add-dir`
4. 识别并清洗 auto mode 下会绕过 classifier 的危险 allow rules
5. 协调 `default / auto / plan / bypassPermissions` 之间的 mode transition
6. 处理 auto mode gate、plan mode with auto、bypass disable 等产品级护栏

所以它本质上是一个：

```text
permission context bootstrapper
+ permission mode transition coordinator
+ auto-mode safety guardrail layer
```

也就是说 `permissions.ts` 决定“怎么判”，
而 `permissionSetup.ts` 决定“以什么上下文和什么模式去判”。

---

### 第一部分：文件头 import 一眼就能看出这不是纯静态配置文件，而是连接 bootstrap state、settings、GrowthBook、tools、filesystem、permission rules 的装配中枢

位置：`src/utils/permissions/permissionSetup.ts:1`

这份文件同时依赖：

- bootstrap state（plan / auto mode transitions）
- settings 读写与 settings file path
- GrowthBook / Statsig / dynamic config
- tools preset / tool list parsing
- fs path / symlink / workspace validation
- dangerous shell patterns
- permission parser / permission updates / permissionsLoader

这说明权限 setup 不是一个简单的：

```text
read settings -> make object
```

而是：

```text
assemble a runtime-safe permission context
under product gates, model capability checks, tool presets, and filesystem constraints
```

所以这份文件是权限系统真正的“控制面”层。

---

### 第二部分：`isDangerousBashPermission(...)` / `isDangerousPowerShellPermission(...)` / `isDangerousTaskPermission(...)` 很关键，说明 auto mode 的安全性不是只靠 classifier 本身，而是先在上下文装配阶段清洗那些会让 classifier 根本失去介入机会的 allow rules

位置：

- Bash `src/utils/permissions/permissionSetup.ts:94`
- PowerShell `src/utils/permissions/permissionSetup.ts:157`
- Agent `src/utils/permissions/permissionSetup.ts:240`

这些函数识别的不是“危险命令本身”，
而是：

```text
dangerous permission rules
```

也就是那类一旦存在，工具会在 classifier 之前就被直接 allow 的规则。

例如：

- `Bash(*)`
- `Bash(python:*)`
- `PowerShell(iex:*)`
- 任意 `Agent(...)` allow rule

这说明 auto mode 的护栏设计非常成熟：

```text
do not merely classify dangerous actions;
prevent allow-rules from bypassing classification altogether
```

也就是说安全边界不是只放在执行时，
而是前移到了上下文装配阶段。

---

### 第三部分：`isDangerousTaskPermission(...)` 直接把任何 Agent allow rule 都视为危险，这非常说明作者对 sub-agent delegation 攻击面的重视——auto mode 下不能让 agent spawn 绕过 classifier 去隐式放大能力

位置：`src/utils/permissions/permissionSetup.ts:240`

这里逻辑极其直接：

```ts
return normalizeLegacyToolName(toolName) === AGENT_TOOL_NAME
```

注释说得很清楚：

```text
Any Agent allow rule would auto-approve sub-agent spawns before the auto mode classifier
can evaluate the sub-agent's prompt, defeating delegation attack prevention.
```

这说明在 Claude Code 的安全模型里，
Agent tool 不是普通工具：

```text
allowing delegation is itself a high-risk permission edge
```

因为一旦 Agent 被直接 allow，
auto mode classifier 就失去了查看 sub-agent prompt 的机会。

所以这里的语义是：

```text
protect the delegation boundary,
not just the immediate tool boundary
```

---

### 第四部分：`findDangerousClassifierPermissions(...)` 和 `findOverlyBroadBashPermissions(...)` 体现了两个不同层次的风险建模：一种是“会绕过 classifier”，另一种是“其实等价于 shell 版 YOLO”

位置：

- dangerous classifier permissions `src/utils/permissions/permissionSetup.ts:295`
- overly broad bash/powershell `src/utils/permissions/permissionSetup.ts:379`

这两类检测虽然看起来相近，但语义不同：

#### dangerous classifier permissions
重点是：
- 在 auto mode 下会让 classifier 失效
- 因此需要 strip / ignore

#### overly broad shell permissions
重点是：
- `Bash(*)` / `PowerShell(*)`
- 本质上等于 shell 专属 bypass/yolo
- 主要用于检测和告警

这说明作者并没有把“危险”简单做成一个布尔值，
而是区分了：

```text
unsafe because it bypasses classifier
vs
unsafe because it is operationally too broad
```

这是很细的权限产品设计。

---

### 第五部分：`removeDangerousPermissions(...)`、`stripDangerousPermissionsForAutoMode(...)`、`restoreDangerousPermissions(...)` 三件套，是这份文件最关键的架构机制之一——进入 auto mode 时剥离危险 allow rules，退出时再恢复，说明 auto mode 安全护栏不是永久改配置，而是运行态可逆变换

位置：

- remove `src/utils/permissions/permissionSetup.ts:472`
- strip `src/utils/permissions/permissionSetup.ts:510`
- restore `src/utils/permissions/permissionSetup.ts:561`

这套机制的核心语义是：

```text
enter auto mode
  -> temporarily strip dangerous allow rules from the in-memory context
leave auto mode
  -> restore them
```

而且 `stripDangerousPermissionsForAutoMode(...)` 还会把被拿掉的规则 stash 到：

- `strippedDangerousRules`

这说明作者不想：

- 永久改写用户配置
- 一刀删除用户规则
- 让 auto mode 的安全护栏影响 default mode 体验

而是采取：

```text
reversible runtime sanitization
```

这点非常漂亮，
因为它把安全与兼容同时保住了。

---

### 第六部分：`transitionPermissionMode(...)` 很重要，它把 plan/auto mode 的切换副作用集中到一个统一入口，说明 mode 切换在系统里不是简单地改一个字符串，而是一组有状态副作用的事务

位置：`src/utils/permissions/permissionSetup.ts:597`

这个函数会统一处理：

- `handlePlanModeTransition(fromMode, toMode)`
- `handleAutoModeTransition(fromMode, toMode)`
- `setHasExitedPlanMode(true)`
- 进入/退出 auto 时的 `setAutoModeActive(...)`
- strip / restore dangerous permissions
- plan 进入时 `prepareContextForPlanMode(...)`
- 离开 plan 时清 `prePlanMode`

这说明 permission mode 在架构上不是单纯枚举值，
而是：

```text
a stateful runtime regime
with associated side effects, attachments, and context transformations
```

所以这里本质上是在做 mode transition transaction，而不是状态字段赋值。

---

### 第七部分：`initialPermissionModeFromCLI(...)` 说明启动时 mode 的选择不是“CLI 参数覆盖 settings”这么简单，而是要同时考虑 Statsig gate、settings 禁用、auto-mode circuit breaker、CCR 环境限制等多层优先级

位置：`src/utils/permissions/permissionSetup.ts:689`

这个函数会综合：

- `dangerouslySkipPermissions`
- `--permission-mode`
- settings 里的 `permissions.defaultMode`
- bypass disable gate / settings
- auto mode circuit breaker 缓存态
- CCR 只支持 `acceptEdits` / `plan`

最后构造一个 `orderedModes`，再找第一个合法 mode。

这说明“初始 mode 选择”其实是：

```text
candidate mode selection
-> gate filtering
-> environment compatibility filtering
-> first valid survivor wins
```

所以启动时权限模式并不是一个简单常量，
而是一个受产品政策、模型可用性、远程环境能力约束的 negotiated result。

---

### 第八部分：`parseToolListFromCLI(...)` 很细但很关键，它说明 `--allowed-tools` / `--disallowed-tools` 的 CLI 语法本身就支持带括号内容的规则，因此不能简单按空格或逗号 split

位置：`src/utils/permissions/permissionSetup.ts:813`

这里实现了一个小 parser：

- 维护 `isInParens`
- 在括号外，逗号和空格都可作为分隔符
- 在括号内，逗号和空格保留为内容的一部分

这说明 CLI 允许这样的输入：

```text
--allowed-tools "Bash(npm install:*), Agent(Explore)"
```

而不会把括号里的空格、逗号误切断。

也就是说这份文件不仅组装权限上下文，
还承担了 permission rule CLI surface 的语法正确性。

---

### 第九部分：`initializeToolPermissionContext(...)` 是整个权限装配的总入口——它把 CLI rules、disk rules、mode、additional directories、baseTools、dangerous/overly-broad detection 全部汇合成一份初始 `ToolPermissionContext`

位置：`src/utils/permissions/permissionSetup.ts:872`

这个函数做的事情非常多：

1. parse allowed/disallowed tools
2. 归一化 legacy tool names
3. 根据 `baseTools` 自动生成 disallow list
4. 处理 symlink working directory
5. 计算 bypassPermissions 是否可用
6. `loadAllPermissionRulesFromDisk()`
7. 检测 overly broad shell permissions
8. 检测 auto mode dangerous permissions
9. `applyPermissionRulesToPermissionContext(...)`
10. 处理 settings + `--add-dir` 的 additional directories
11. 返回：
   - `toolPermissionContext`
   - `warnings`
   - `dangerousPermissions`
   - `overlyBroadBashPermissions`

这说明它的真实职责不是“初始化一个对象”，
而是：

```text
build a fully operational permission context
and return the diagnostics discovered during assembly
```

这就是整个权限控制面的 bootstrap point。

---

### 第十部分：`baseToolsCli` 的处理方式很有意思——不是生成 allow list，而是用默认全量工具集合反推出“应 deny 的工具”，说明 base tools 的语义更像“能力裁剪基线”，不是普通 allow rule 叠加

位置：`src/utils/permissions/permissionSetup.ts:900`

这里的逻辑是：

- parse `baseTools`
- 转成 `baseToolsSet`
- 取 `getToolsForDefaultPreset()` 的全量工具集
- 用差集得到 `toolsToDisallow`
- 追加进 `parsedDisallowedToolsCli`

这说明 `baseTools` 不是：

```text
just another allow list
```

而是：

```text
define the baseline tool surface,
then explicitly deny everything outside it
```

这在产品语义上更强，
因为它意味着用户是在指定一个“工具子集运行环境”。

---

### 第十一部分：working directory 处理也被纳入 permission setup，说明目录访问边界本身就是权限上下文的一部分，而不只是文件系统层的独立配置

位置：

- symlink handling `src/utils/permissions/permissionSetup.ts:917`
- add-dir validation `src/utils/permissions/permissionSetup.ts:993`

这里做了两件关键事：

#### symlink PWD 兼容
如果 `process.env.PWD` 是指向 `originalCwd` 的 symlink，
会把它也加进 `additionalWorkingDirectories`

#### settings + `--add-dir`
统一跑 `validateDirectoryForWorkspace(...)`，
合法的就 `addDirectories`

这说明权限上下文里不仅有：

- 工具 allow/deny/ask 规则

还有：

```text
which filesystem roots count as permitted workspace scope
```

也就是说目录边界本身就是 permission context 的一部分。

---

### 第十二部分：`verifyAutoModeGateAccess(...)` 非常关键，它说明 auto mode 可用性不是纯同步常量，而是一个异步 gate reconciliation 过程；更重要的是它返回的是 context transform function，而不是预计算后的 context，专门用来避免 await 期间的 stale state 覆盖

位置：`src/utils/permissions/permissionSetup.ts:1078`

注释已经把问题说透了：

- GrowthBook 读取是异步的
- await 期间用户可能 shift-tab 改了 mode
- 如果直接返回一个算好的 context snapshot
- 就可能把用户中途的 mode change 覆盖掉

所以这里返回：

```ts
updateContext: (ctx) => ToolPermissionContext
```

而不是直接返回 context。

这说明作者在处理的是一个非常具体的 race：

```text
async gate check finishes after the user has already changed permission mode
```

所以这里的设计非常成熟：

```text
return a context transformer,
not a stale context snapshot
```

这在整个 mode/gate 系统里是一个高质量并发设计点。

---

### 第十三部分：auto mode gate 里同时区分 `carouselAvailable` 与 `canEnterAuto`，说明“是否在 UI 上展示 auto mode”与“如果用户明确要求，是否允许进入 auto mode”是两个不同问题

位置：`src/utils/permissions/permissionSetup.ts:1103`

这里区分：

#### `carouselAvailable`
- 是否在 mode picker / shift-tab carousel 中展示 auto mode
- 受 opt-in、settings、model、circuit breaker 等影响

#### `canEnterAuto`
- 用户显式通过 CLI / defaultMode 想进 auto 时，是否允许进入
- 是更底层的 entry gate

这说明产品设计上：

```text
surface availability
!=
explicit entry permission
```

例如：

- UI 不展示 auto（没 opt-in）
- 但显式 CLI flag 本身可以构成 opt-in

这是一个很细的产品语义边界，
在 setup 层被认真编码了出来。

---

### 第十四部分：`kickOutOfAutoIfNeeded(...)` 与相关 notification 逻辑说明 auto mode gate 不是仅在进入时校验；当 gate、settings、model 变化导致 auto 不再可用时，系统会把当前正在 auto/plan-auto 的用户踢回更安全模式，并恢复之前被 strip 的危险规则

位置：`src/utils/permissions/permissionSetup.ts:1182`

这里的 transform 会：

- 检查 fresh ctx 里是否还在 auto / plan-with-auto
- 如果在 auto：
  - `setAutoModeActive(false)`
  - `setNeedsAutoModeExitAttachment(true)`
  - `restoreDangerousPermissions(ctx)`
  - `setMode('default')`
- 如果在 plan with auto：
  - 关闭 auto
  - 恢复危险规则
  - 让 `prePlanMode` 从 `auto` 退成 `default`

这说明 auto mode 在系统里不是“进去了就算了”，
而是持续受 gate / model / settings 约束的动态 regime。

所以这里的真正语义是：

```text
auto mode availability is continuously reconciled,
not just checked once at startup
```

---

### 第十五部分：`prepareContextForPlanMode(...)` 和 `transitionPlanAutoMode(...)` 表明 plan mode 不是简单暂停当前模式，而是一个会根据 auto-mode opt-in / gate 状态决定是否延续 classifier 语义的特殊过渡态

位置：

- prepare `src/utils/permissions/permissionSetup.ts:1462`
- transition `src/utils/permissions/permissionSetup.ts:1502`

这里有三种情况：

#### 当前就是 auto，进入 plan
- 若 `shouldPlanUseAutoMode()` 为真：保留 `prePlanMode='auto'`
- 否则：关闭 auto、恢复危险规则、记 exit attachment

#### 当前不是 auto，进入 plan
- 若 plan 应使用 auto 且当前不是 bypass：
  - 激活 auto
  - strip dangerous permissions
  - stash `prePlanMode`

#### settings 在 plan 中变化
- `transitionPlanAutoMode(...)` 会即时重新协调 want/have

这说明 plan mode 的本质更像：

```text
a wrapper mode with optional embedded auto semantics
```

而不是一个完全独立、与其他 mode 隔离的世界。

---

### 第十六部分：`isAutoModeGateEnabled()`、`getAutoModeUnavailableReason()`、`isBypassPermissionsModeDisabled()`、`createDisabledBypassPermissionsContext()` 等 helper 表明这份文件同时也是权限模式可用性查询 API 层，供其他 UI / runtime 表面统一判断当前 mode 是否还能用

位置：

- auto enabled `src/utils/permissions/permissionSetup.ts:1283`
- auto unavailable reason `src/utils/permissions/permissionSetup.ts:1294`
- bypass disabled `src/utils/permissions/permissionSetup.ts:1371`
- disabled bypass context `src/utils/permissions/permissionSetup.ts:1389`

这说明 `permissionSetup.ts` 不只在 startup 阶段使用，
还为后续其他模块提供：

```text
current permission mode capability state
```

也就是说它是：

- bootstrap layer
- transition layer
- query layer

三者合一的控制面模块。

---

### 第十七部分：整份文件最核心的架构价值，是把“权限模式”从一个静态枚举提升成了一个受 settings、gates、model、tool rules、workspace scope 与 runtime transitions 共同约束的动态控制平面

如果把整份文件压缩，会发现它其实在做五层事：

#### 1. rule risk classification
- dangerous classifier permissions
- overly broad shell permissions

#### 2. initial context assembly
- load rules from disk
- merge CLI allow/deny/base tools
- build `ToolPermissionContext`
- attach additional directories

#### 3. runtime safety transformations
- strip / restore dangerous rules for auto mode
- plan mode preparation
- plan-auto reconciliation

#### 4. mode selection / gating
- initial mode from CLI/settings
- auto mode gate verify
- bypass disable gate

#### 5. capability query helpers
- auto available?
- why unavailable?
- bypass disabled?

所以最准确的压缩表达是：

```text
permissionSetup.ts = Claude Code 权限控制面的装配与过渡中枢：负责构造初始 ToolPermissionContext、协调 permission mode 生命周期，并在 auto/plan 等模式下施加可逆的安全护栏变换
```

---

### 元

问题：**这一站真正想解决的架构问题是什么？**

回答：这一站的元问题，是规则和 mode 到了运行时后，为什么还需要一层专门装配器。它真正做的是把 CLI、disk、session、auto mode、plan mode 等输入整理成可执行的权限上下文。

### 反

问题：**如果把这一站的设计反过来，会发生什么？**

回答：如果上下文装配散在初始化链路各处，危险 allow rule 清洗和 mode 护栏就无法统一生效。那时看起来是配置更多，实际上是安全边界更模糊。

### 空

问题：**跳出当前文件名，这一站背后更大的问题是什么？**

回答：更大的问题，是权限系统如何在“可配置”与“不可失控”之间维持平衡。`permissionSetup.ts` 体现的正是运行前的治理，而不是运行中的补救。

### 读完这一站后，你应该抓住的 10 个事实

1. `permissionSetup.ts` 不是做单次权限判定的，而是负责初始 `ToolPermissionContext` 装配、mode 生命周期协调和 auto-mode 安全护栏。
2. 它会在进入 auto mode 前识别危险 allow rules（如 `Bash(*)`、`Bash(python:*)`、`PowerShell(iex:*)`、任意 `Agent(...)` allow），因为这些规则会在 classifier 之前直接放行，破坏 auto-mode 安全边界。
3. 危险规则不会被永久删掉，而是通过 `stripDangerousPermissionsForAutoMode()` 暂时从内存 context 中剥离，并在退出 auto 时通过 `restoreDangerousPermissions()` 恢复。
4. `transitionPermissionMode(...)` 说明 permission mode 切换不是改字符串，而是一组有副作用的事务：plan/auto 状态、attachment 标记、dangerous rule strip/restore 都在这里统一协调。
5. `initialPermissionModeFromCLI(...)` 会综合 CLI、settings、Statsig/GrowthBook gate、CCR 环境限制和 auto-mode circuit breaker 选择起始 mode，因此启动 mode 是协商结果而不是简单优先级覆盖。
6. `parseToolListFromCLI(...)` 说明 CLI 的工具规则列表支持 `Tool(content)` 语法，括号内的空格和逗号不会被误切分。
7. `initializeToolPermissionContext(...)` 是权限装配总入口：它把磁盘规则、CLI allow/deny/base tools、working directory、additional directories 和风险诊断统一汇总成可执行的初始上下文。
8. `verifyAutoModeGateAccess(...)` 返回的是 `updateContext(ctx)` 变换函数，而不是预计算 context，专门用来避免异步 gate 检查完成后覆盖用户在等待期间做出的 mode 切换。
9. auto mode 的可用性被拆成 `carouselAvailable` 与 `canEnterAuto` 两层语义，说明“是否在 UI 上展示 auto”与“显式请求时是否允许进入 auto”是两个不同问题。
10. `prepareContextForPlanMode()` 与 `transitionPlanAutoMode()` 表明 plan mode 可以在特定条件下继续使用 auto semantics，因此 plan 不是纯静态模式，而是一个可嵌入 classifier 语义的过渡态。

---

### 现在把第 64-65 站串起来

```text
permissionRuleParser.ts
  -> 规则字符串 DSL 的编解码与 legacy 归一化层
permissionSetup.ts
  -> 把这些规则与 mode/gate/CLI/settings/workspace 信息装配成实际运行时的 ToolPermissionContext
```

所以现在 permission 控制面可以压缩成：

```text
settings / CLI / disk rule strings
  -> permissionRuleParser.ts parses & normalizes
  -> permissionSetup.ts assembles ToolPermissionContext
  -> permissions.ts evaluates tool use under that context
  -> useCanUseTool / handlers route the resulting decision
```

也就是说：

```text
permissionRuleParser.ts 回答“规则字符串怎么表达、怎么解析、怎么兼容历史命名”
permissionSetup.ts 回答“这些规则和 mode 在启动/切换时怎么被装进当前 session 的权限上下文，并怎样施加 auto/plan 等模式护栏”
```

---

### 下一站建议

下一站最顺的是：

```text
src/utils/permissions/permissionsLoader.ts
```

因为现在你已经看懂了：

- `permissionRuleParser.ts` 负责规则 DSL
- `permissionSetup.ts` 负责上下文装配与模式护栏
- 但“磁盘上的 permission rules 到底怎样被加载、按 source 组织、过滤 read-only / managed-only 约束”这条链本身还没拆开
- 而 `permissionSetup.ts` 正在直接依赖：
  - `loadAllPermissionRulesFromDisk()`
  - `deletePermissionRuleFromSettings(...)`
  - `shouldAllowManagedPermissionRulesOnly()`

下一步最自然就是把磁盘规则加载与来源管理层补齐：

**`permissionsLoader.ts` 到底怎样从不同 settings source 读取权限规则、构造 `PermissionRule[]`、处理 managed-only 限制与可编辑/只读来源边界，并把这些规则提供给 `permissionSetup.ts` 与 `permissions.ts` 使用。**
