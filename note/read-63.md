## 第 63 站：`src/utils/permissions/permissions.ts`

### 这是什么文件

`src/utils/permissions/permissions.ts` 是工具权限系统里的 **核心权限判定器、mode 变换器与规则上下文编辑层**。

上一站 `coordinatorHandler.ts` 已经看懂：

- ask-path 在某些模式下会先跑 hooks / classifier
- unresolved 才会继续进入 interactive dialog
- 但那一层只是 ask 之后的编排
- 还没真正拆开 ask / allow / deny 最底层到底是怎样算出来的

这一站回答的是：

```text
一次 tool use 在进入 coordinator / swarm / interactive handlers 之前，
底层到底怎样综合：
- permission rules
- tool.checkPermissions(...)
- permission mode
- auto/dontAsk/headless 变换
- classifier / denial tracking
- hook fallback
最终产出 allow / deny / ask 的 PermissionDecision
```

所以最准确的一句话是：

```text
permissions.ts = 工具权限系统的核心判定引擎与 ask/allow/deny 结果变换中枢
```

---

### 先看它的整体定位

位置：`src/utils/permissions/permissions.ts:1`

这个文件不是：

- interactive permission UI
- queue / dialog handler
- permission rule parser 本体
- 单个 tool 的具体权限逻辑实现

它的职责是：

1. 从 `ToolPermissionContext` 中取出 allow / deny / ask rules
2. 统一做整工具级 rule match 与内容级 rule lookup
3. 驱动 `tool.checkPermissions(...)` 进入每个工具自己的权限检查
4. 把底层结果再叠加 mode 语义：
   - `bypassPermissions`
   - `dontAsk`
   - `auto`
   - headless / async-agent
5. 在 auto mode 下接入 classifier、allowlist、acceptEdits fast-path、denial tracking
6. 提供 permission rule 的删除、同步与 context 更新辅助函数

所以它本质上是一个：

```text
permission decision engine
+ mode transformer
+ rule-context utility layer
```

也就是说前面各个 handler 都是在消费它的输出，
而这份文件负责生成那个输出。

---

### 第一部分：顶部 import 已经暴露了这份文件的真正复杂度——它不是只看几条规则，而是把 MCP、sandbox、hooks、classifier、analytics、settings、denial tracking 全部卷进同一个权限判定面

位置：`src/utils/permissions/permissions.ts:1`

从 import 就能看出这份文件不只是“permission rules.ts”：

它同时依赖：

- `getToolNameForPermissionCheck` / `mcpInfoFromString`
- Bash / PowerShell / REPL / Agent tool 常量
- `SandboxManager` / `shouldUseSandbox`
- `executePermissionRequestHooks`
- classifier state 与 classifier modules
- denial tracking
- analytics / token / cost telemetry
- permission update apply / persist
- editable settings 删除逻辑

这说明权限系统在 Claude Code 里不是一层薄 wrapper，
而是：

```text
a policy engine that sits at the intersection of
rules, tools, runtime mode, automation, analytics, and persistence
```

所以这份文件是整个工具权限子系统最“底盘级”的一层之一。

---

### 第二部分：`PERMISSION_RULE_SOURCES` 把规则来源线性化，说明权限系统不仅关心规则内容，还关心规则 provenance，并且要按来源统一展开

位置：`src/utils/permissions/permissions.ts:109`

这里把规则来源固定成：

- `SETTING_SOURCES`
- `cliArg`
- `command`
- `session`

然后 `getAllowRules/getDenyRules/getAskRules` 都用同一套路：

```ts
return PERMISSION_RULE_SOURCES.flatMap(source => ...)
```

这说明 `ToolPermissionContext` 内部是按来源分桶存规则，
而这份文件负责把它们摊平成统一的 `PermissionRule[]` 视图。

也就是说权限系统关心的不是只有：

- allow / deny / ask

还关心：

```text
which source produced this rule,
so later decisions can explain themselves correctly
```

这和前面 PermissionContext 对 decision source 的重视完全一致。

---

### 第三部分：`createPermissionRequestMessage(...)` 很关键，因为它把底层 decisionReason 翻译成人类可读的 ask 原因文本，说明权限系统不是只返回结构化结果，还负责生成审批解释文案

位置：`src/utils/permissions/permissions.ts:137`

这个函数会根据 `decisionReason.type` 分情况生成文案：

- `classifier`
- `hook`
- `rule`
- `subcommandResults`
- `permissionPromptTool`
- `sandboxOverride`
- `workingDir`
- `safetyCheck`
- `other`
- `mode`
- `asyncAgent`

尤其值得注意的是：

#### rule
会把 rule value 和 source 一起翻译成解释文本

#### subcommandResults
会把多子命令里真正需要审批的片段摘出来

#### sandboxOverride
直接把 ask 理由翻译成 “Run outside of the sandbox”

这说明 `PermissionDecisionReason` 在这里不仅服务内部控制流，
还承担：

```text
human-facing permission prompt explanation
```

也就是说 ask 结果不是简单一个 `behavior:'ask'`，
而是会被包装成可展示、可审批、可远程转发的解释文本。

---

### 第四部分：`toolMatchesRule(...)` 的 MCP 处理很关键，说明“工具名匹配”在这里并不只是字符串相等，而是已经内建了 MCP fully-qualified name 与 server-level wildcard 语义

位置：`src/utils/permissions/permissions.ts:238`

这里处理了两类匹配：

#### 普通工具
直接比较 `rule.ruleValue.toolName === nameForRuleMatch`

#### MCP 工具
允许：

- `mcp__server1`
- `mcp__server1__*`
- `mcp__server1__tool1`

并且还专门处理了 skip-prefix 模式下的 MCP 名字冲突问题。

这说明权限系统不是只认识 builtin tool names，
而是把 MCP 工具也纳入统一规则语义里。

也就是说规则匹配层已经理解：

```text
builtin tool identity
vs
MCP server/tool identity
```

这是整个扩展工具系统能和权限规则统一工作的关键。

---

### 第五部分：`toolAlwaysAllowedRule` / `getDenyRuleForTool` / `getAskRuleForTool` 只处理“整个工具级”规则，而内容级规则故意交给 tool 自己的 `checkPermissions(...)`，说明权限系统在这里采用了“通用外壳 + tool-specific 内核”的分层

位置：

- allow `src/utils/permissions/permissions.ts:275`
- deny `src/utils/permissions/permissions.ts:287`
- ask `src/utils/permissions/permissions.ts:297`

这三组 helper 都只做一件事：

```text
does the whole tool match a top-level rule?
```

而像：

- `Bash(npm publish:*)`
- agent subtype / content rule
- 更细粒度输入相关规则

则不在这里直接处理，
而是靠后面的 `tool.checkPermissions(...)`。

这说明作者做了一个很清楚的边界划分：

#### permissions.ts
负责统一外层规则框架与 mode 变换

#### tool.checkPermissions(...)
负责具体工具的内容级权限语义

所以这里体现的是：

```text
global permission orchestration shell
+ per-tool semantic permission core
```

---

### 第六部分：`filterDeniedAgents(...)` 很能说明 Agent 工具在权限系统里不是普通黑盒工具；这里专门支持 `Agent(agentType)` 这种 deny 语法，直接在工具目录层过滤子 agent 类型

位置：`src/utils/permissions/permissions.ts:325`

这里会：

- 遍历 deny rules
- 找出 `toolName === agentToolName` 且 `ruleContent !== undefined`
- 收集成 `deniedAgentTypes`
- 过滤 agent 列表

注释还特别说明之前是按 agent 反复 parse rules，
现在优化成了一次性预解析。

这说明权限系统对 Agent tool 的控制不是只停留在“允许不允许用 Agent”，
而是已经细化到：

```text
which agent subtypes are allowed to be spawned
```

这在整个多 agent 架构里非常关键，
因为它表示权限系统能直接裁剪 agent 能力面，而不只是挡住整个工具入口。

---

### 第七部分：`runPermissionRequestHooksForHeadlessAgent(...)` 非常重要，它说明在 headless / async-agent 场景下，虽然不能弹框，但系统仍然会先给 hooks 一个最后的自动化决策机会，而不是直接一刀 deny

位置：`src/utils/permissions/permissions.ts:400`

这个函数的语义是：

```text
if prompts are unavailable,
run PermissionRequest hooks first,
then auto-deny only if hooks stay silent
```

内部逻辑和 `PermissionContext.runHooks(...)` 很相似：

- hook allow -> 可持久化 updates 并返回 `behavior:'allow'`
- hook deny -> 返回 `behavior:'deny'`
- `interrupt` -> 直接 abort controller
- hook 异常 -> 记录日志，但继续走 auto-deny fallback

这说明 headless 场景不是“没有 UI = 没有审批”。

真正的语义是：

```text
replace human approval with hook-only approval opportunity,
then fail closed if no automated decision appears
```

这对于后台 agent / async subagent 非常关键。

---

### 第八部分：`hasPermissionsToUseTool(...)` 不是简单调用 inner 再返回，它还负责做“后处理变换层”，包括 dontAsk、auto mode classifier、headless hook fallback、denial tracking reset 等

位置：`src/utils/permissions/permissions.ts:473`

这个公开入口先调：

```ts
const result = await hasPermissionsToUseToolInner(tool, input, context)
```

然后在 outer layer 再做一大堆 transformation。

这说明作者把权限判定分成了两层：

#### inner
规则 / tool.checkPermissions / bypass / allow rule / ask 归一化

#### outer
mode-based post-processing 与 runtime-specific transformations

也就是说 `hasPermissionsToUseTool` 的真实含义不是：

```text
compute raw permission result
```

而是：

```text
compute base result
-> then adapt it to current execution mode and runtime constraints
```

这个分层非常关键，因为前面各 handler 消费的是变换后的最终结果，而不是裸规则结果。

---

### 第九部分：`dontAsk` 是在 outer layer 最后把 ask 直接改写成 deny，说明 `dontAsk` 不是底层 rule 语义，而是运行模式对 ask 的统一后处理策略

位置：`src/utils/permissions/permissions.ts:503`

这里逻辑很直接：

```ts
if (result.behavior === 'ask') {
  if (appState.toolPermissionContext.mode === 'dontAsk') {
    return {
      behavior: 'deny',
      decisionReason: { type: 'mode', mode: 'dontAsk' },
      message: DONT_ASK_REJECT_MESSAGE(tool.name),
    }
  }
}
```

注释还专门强调：

- 这是在 end 阶段做
- 这样不能被 earlier return 绕过

这说明 `dontAsk` 的语义不是“重新计算 permissions”，
而是：

```text
whatever ended up requiring approval,
convert it to a hard deny because prompting is disallowed by mode
```

所以 `dontAsk` 是一个很纯粹的 mode transform，
不是 permission rule 的一部分。

---

### 第十部分：auto mode 分支极其关键——它说明 ask 在 auto 模式下并不等于人工审批，而是进入一套更复杂的 classifier decision pipeline，其中还有多层 fast-path 绕开 classifier API 调用

位置：`src/utils/permissions/permissions.ts:520`

当：

- `feature('TRANSCRIPT_CLASSIFIER')`
- mode 是 `auto`
- 或 `plan` 但 auto mode active

时，ask 会进入 auto mode pipeline。

但这套 pipeline 不是直接 classifier call，
而是先做很多前置判断：

1. 非 classifier-approvable safetyCheck 直接保持 ask/deny 语义
2. `requiresUserInteraction()` 工具不进入 auto approve
3. PowerShell 在特定 flag 下禁止 auto mode classifier
4. `acceptEdits` fast-path
5. safe-tool allowlist fast-path
6. 最后才真正调用 classifier

这说明 auto mode 不是“所有 ask 都丢给 AI 决策”，
而是：

```text
a layered permission automation pipeline
with several deterministic skip-paths before inference
```

这也解释了为什么这份文件会这么大——因为 auto mode 本身就是一个二级权限系统。

---

### 第十一部分：`acceptEdits` fast-path 很漂亮——它通过伪造一个 `mode:'acceptEdits'` 的临时 context 重新调用 `tool.checkPermissions(...)`，来判断当前 ask 是否其实属于安全编辑操作，从而避免昂贵 classifier 调用

位置：`src/utils/permissions/permissions.ts:593`

这里的关键不是硬编码某些工具，
而是：

- 重新 parse input
- 再调一次 `tool.checkPermissions(...)`
- 但把 `getAppState()` 改写成返回 `mode:'acceptEdits'`

如果这个模拟结果是 allow，
就直接：

- 记 success 到 denial tracking
- 打 analytics fastPath=`acceptEdits`
- 返回 auto-mode allow

这说明作者在这里用了一个很优雅的技巧：

```text
reuse the tool's own permission semantics
under a different mode,
instead of re-implementing a parallel safe-edit checker
```

这是很强的语义复用。

---

### 第十二部分：allowlisted tool fast-path 说明 auto mode 不只是 classifier 驱动，它还内建一组“足够安全，不值得花 classifier token”的工具白名单

位置：`src/utils/permissions/permissions.ts:658`

如果：

```ts
classifierDecisionModule!.isAutoModeAllowlistedTool(tool.name)
```

则直接：

- recordSuccess
- logEvent fastPath=`allowlist`
- return allow

这说明 auto mode classifier 的目标不是“管一切”，
而是：

```text
spend inference budget only where classification adds value
```

所以权限系统在这里已经开始显式做成本优化：

- 安全工具直接放行
- 安全编辑路径直接放行
- 只有模糊/危险动作才真正进 classifier

这是很成熟的生产化设计。

---

### 第十三部分：真正的 classifier call 不只是算 allow/deny，还做了大量遥测与失败分流，说明 auto mode classifier 在系统里是一个被严密观测的高成本审批子系统

位置：`src/utils/permissions/permissions.ts:688`

classifier 调用前后做了很多事：

- `setClassifierChecking(toolUseID)` / `clearClassifierChecking(toolUseID)`
- `classifyYoloAction(...)`
- 记录 prompt lengths
- 记录 token usage
- 记录 stage1/stage2 usage
- 记录 request ids / msg ids
- 计算 classifier cost USD
- `addToTurnClassifierDuration(...)`

这说明 classifier 在产品语义里不是一个小 helper，
而是：

```text
a separately observed and cost-accounted approval subsystem
```

也就是说 Claude Code 不只是用 classifier，
还在认真量化：

- 它花了多少 token
- 延迟多少
- 命中率如何
- overhead 占比多少

这在权限系统里非常少见，
但对 auto mode 非常合理。

---

### 第十四部分：classifier blocked 后并不是单一路径，而是分成 transcript too long、classifier unavailable、普通 blocked、denial limit exceeded 几种不同终局，说明 auto mode 的失败语义被拆得非常细

位置：`src/utils/permissions/permissions.ts:818`

当 `classifierResult.shouldBlock` 时，后面有几层分流：

#### transcriptTooLong
- headless -> 直接 abort
- interactive -> fall back 到正常手动审批

#### unavailable
- `tengu_iron_gate_closed` 开启 -> fail closed，直接 deny
- 否则 fail open -> 回退到正常 permission handling

#### 普通 blocked
- 更新 denial tracking
- 检查 denial limit
- limit hit -> 回退 prompting / 或 headless abort
- 否则正常返回 classifier deny

这说明 auto mode 不是“classifier 说不行就不行”。

真正结构是：

```text
classifier block result
-> interpret why it blocked
-> choose fail-open / fail-closed / prompt-fallback / hard deny / abort
```

这是一整套 resilience policy。

---

### 第十五部分：`handleDenialLimitExceeded(...)` 很关键，它说明 auto mode 在连续/累计被 classifier 拦太多次后，不会无限 deny，而是主动要求用户回来看 transcript；headless 场景则直接 abort

位置：`src/utils/permissions/permissions.ts:984`

这里会：

- 用 `shouldFallbackToPrompting(denialState)` 判断是否超限
- 区分 total / consecutive 两种 limit
- 打 `tengu_auto_mode_denial_limit_exceeded` analytics
- headless -> `throw new AbortError(...)`
- CLI -> 返回一个修改过的 ask result，把 warning + latest blocked action 填进 `decisionReason`

这说明 auto mode 的产品哲学不是盲目自动化，
而是：

```text
if automation keeps disagreeing with the agent,
escalate back to human oversight
```

这点非常关键，
因为它防止模型在 auto mode 下反复撞同一堵权限墙。

---

### 第十六部分：`checkRuleBasedPermissions(...)` 把“bypassPermissions 仍必须尊重的前半段规则检查”单独抽出来，说明系统明确区分了哪些权限约束可被 bypass，哪些是 bypass-immune

位置：`src/utils/permissions/permissions.ts:1071`

注释直接说：

- 这是 bypass mode 仍然尊重的 subset
- 不包含 auto mode classifier / dontAsk / asyncAgent / bypass allow transform
- 调用方自己要先处理 `requiresUserInteraction()`

它检查的内容包括：

1. entire tool deny rule
2. entire tool ask rule
3. `tool.checkPermissions(...)`
4. tool deny
5. content-specific ask rule
6. safetyCheck

这说明权限系统的底层并不是“bypass 就一切放过”。

而是：

```text
some objections are structural and must still surface even in bypass-related flows
```

尤其：

- deny rules
- ask rules
- safety checks

都属于更硬的约束层。

---

### 第十七部分：`hasPermissionsToUseToolInner(...)` 是真正的基础判定流水线，它把权限判断拆成非常清楚的编号阶段：rule deny -> ask rule -> tool.checkPermissions -> bypass -> allow rule -> passthrough→ask

位置：`src/utils/permissions/permissions.ts:1158`

这个 inner pipeline 的主线非常清晰：

#### 1. rule/tool objections
- 1a deny rule
- 1b ask rule
- 1c tool.checkPermissions
- 1d tool deny
- 1e requiresUserInteraction
- 1f content ask rule
- 1g safetyCheck

#### 2. positive overrides
- 2a bypassPermissions / plan+bypass
- 2b always allow rule

#### 3. normalization
- `passthrough` -> `ask`

这说明整个权限系统的底层哲学是：

```text
first honor hard objections,
then apply broad allow/bypass powers,
then normalize unresolved passthrough into ask
```

这个顺序非常重要，
因为它定义了各种 rule/mode/safety 之间的优先级关系。

---

### 第十八部分：`requiresUserInteraction()`、content-specific ask rule、`safetyCheck` 都被明确放在 bypass/alwaysAllow 之前，说明这些 ask 来源具有更高优先级，不会被一般性的 mode/allow rule 轻易覆盖

位置：`src/utils/permissions/permissions.ts:1230`

这几段都出现在：

- bypassPermissions 之前
- alwaysAllowedRule 之前

这说明它们被作者视为：

```text
higher-precedence objections
```

尤其 `safetyCheck` 注释直接说：

- bypass-immune
- 涉及 `.git/`, `.claude/`, `.vscode/`, shell configs 等

也就是说 Claude Code 的权限系统不是简单的 allowlist 优先，
而是保留了一层更硬的安全边界。

这对理解后续 interactive / coordinator handlers 非常关键，
因为它解释了为什么某些 ask 即使在更宽松模式下仍然会被保留下来。

---

### 第十九部分：`passthrough -> ask` 的归一化特别关键——说明 tool 实现可以只表达“我没有明确 allow/deny”，然后由 permissions.ts 统一把这种开放状态收束成真正的 ask 结果

位置：`src/utils/permissions/permissions.ts:1299`

最后这段逻辑是：

```ts
const result: PermissionDecision =
  toolPermissionResult.behavior === 'passthrough'
    ? {
        ...toolPermissionResult,
        behavior: 'ask',
        message: createPermissionRequestMessage(...),
      }
    : toolPermissionResult
```

这说明工具的 `checkPermissions(...)` 不需要总是自己决定最终 ask 文案。

它可以只说：

```text
I am not auto-allowing or auto-denying this
```

然后由统一权限层来把它变成：

```text
formal permission prompt request
```

这是一种非常整洁的职责划分。

---

### 第二十部分：文件尾部的 `deletePermissionRule`、`applyPermissionRulesToPermissionContext`、`syncPermissionRulesFromDisk` 说明这份文件不只是做 runtime 判定，还负责 rule editing 与 context 同步，是权限“运行 + 编辑”双栈的中心层

位置：

- delete `src/utils/permissions/permissions.ts:1329`
- apply `src/utils/permissions/permissions.ts:1408`
- sync `src/utils/permissions/permissions.ts:1419`

这里可以看到三类操作：

#### 删除单条规则
- 按 source/destination 删除
- 只允许 editable settings / in-memory source
- read-only source 直接报错

#### 初始叠加规则
- `convertRulesToUpdates(..., 'addRules')`
- 用于 setup 阶段

#### 从磁盘同步规则
- `replaceRules`
- 先清空 disk-based source:behavior 组合
- 再写入新规则
- 还处理 `allowManagedPermissionRulesOnly`

这说明 `permissions.ts` 不是纯 evaluator，
它还是：

```text
permission context mutation utility layer
```

也就是说 runtime permission semantics 和 rule persistence semantics 在这里汇合了。

---

### 第二十一部分：`syncPermissionRulesFromDisk(...)` 里先清空所有 disk source:behavior combos 再 apply 新规则，非常说明作者在避免 stale rule 残留；这体现了权限上下文同步对 correctness 的高度敏感

位置：`src/utils/permissions/permissions.ts:1448`

注释解释得很清楚：

- 如果只对“有规则的组合”生成 replaceRules
- 那么从磁盘删掉某条规则时
- context 里旧规则会残留

所以这里选择：

1. 先对 `user/project/local` × `allow/deny/ask` 全部 replace 空数组
2. 再 apply 新 rules

这说明权限规则同步不能接受“可能残留旧权限”。

因为 stale rule 直接意味着：

- 误 allow
- 误 deny
- 误 ask

所以这里体现的是很强的权限系统 correctness mindset。

---

### 第二十二部分：整份文件最核心的架构价值，是把“规则、工具自检、模式策略、自动化审批、失败回退、规则编辑”统一收束成一个真正的权限引擎，而不是一堆散落 helper

如果把整份文件压缩，可以看到它实际上同时承担六层工作：

#### 1. rule materialization
- `getAllowRules/getDenyRules/getAskRules`
- `toolMatchesRule`

#### 2. human-readable prompt reasoning
- `createPermissionRequestMessage`

#### 3. base permission pipeline
- `hasPermissionsToUseToolInner`
- `checkRuleBasedPermissions`

#### 4. mode-specific transformation
- `dontAsk`
- `auto`
- headless/async-agent fallback

#### 5. auto-mode classifier control plane
- fast paths
- classifier call
- telemetry
- denial tracking
- fail-open/fail-closed
- denial-limit escalation

#### 6. rule editing/sync helpers
- `deletePermissionRule`
- `applyPermissionRulesToPermissionContext`
- `syncPermissionRulesFromDisk`

所以最准确的压缩表达是：

```text
permissions.ts = Claude Code 工具权限系统的核心引擎：先做 rule/tool 级基础判定，再叠加 mode 与 classifier 语义，并负责把规则上下文的编辑与磁盘同步统一收口
```

---

### 元

问题：**这一站真正想解决的架构问题是什么？**

回答：这一站真正回答的是，ask/allow/deny 最底层究竟怎样被综合算出来。它不是单看几条规则，而是把 rule、tool checker、mode、classifier、hooks、denial tracking 全部压进一个核心判定面。

### 反

问题：**如果把这一站的设计反过来，会发生什么？**

回答：如果没有统一判定引擎，各类 mode 和工具权限逻辑就会各自演化。最终最难维护的不是某条规则，而是不同来源的结论根本不在同一语义平面上。

### 空

问题：**跳出当前文件名，这一站背后更大的问题是什么？**

回答：更大的问题，是权限系统怎样同时做到可解释、可扩展、可组合。`permissions.ts` 的意义，是把“权限决定”提升成产品级决策引擎，而不是若干 if/else。

### 读完这一站后，你应该抓住的 10 个事实

1. `permissions.ts` 不是单纯的 rule helper，而是整个工具权限系统的核心判定引擎、mode 变换层和规则上下文编辑层。
2. 它先把 `ToolPermissionContext` 中按来源分桶的 allow/deny/ask rules 展平成统一 `PermissionRule[]`，并保留 provenance 以支撑后续解释与日志。
3. `createPermissionRequestMessage(...)` 说明 `PermissionDecisionReason` 不只是内部结构体，还会被翻译成人类可读的 ask 原因文本，用于 dialog/bridge/channel 等审批面。
4. `toolMatchesRule(...)` 已经内建 MCP fully-qualified name、server-level wildcard 和 skip-prefix MCP 冲突处理，说明扩展工具与内建工具共享同一权限规则系统。
5. `hasPermissionsToUseToolInner(...)` 的基础流水线顺序是：deny rule -> ask rule -> tool.checkPermissions -> requiresUserInteraction/content ask/safetyCheck -> bypass -> alwaysAllow -> passthrough→ask。
6. `dontAsk` 不是底层规则，而是 outer layer 对 ask 结果施加的统一 mode transform：任何需要审批的动作都会被直接改写成 deny。
7. auto mode 不是直接“问 classifier”，而是一个分层 pipeline：先处理不可 classifier-approve 的安全检查，再试 acceptEdits fast-path、safe allowlist，最后才真正调用 classifier。
8. classifier blocked 后还会继续区分 transcript too long、classifier unavailable、普通 blocked、denial limit exceeded 等不同终局，说明 auto mode 具备完整的 fail-open/fail-closed/fallback policy。
9. headless / async-agent 场景下，系统仍会先运行 `PermissionRequest` hooks，再在无结果时 auto-deny，说明“无 prompt”不等于“无审批机会”。
10. 文件尾部的 rule delete/apply/sync helpers 表明这份文件不仅生成权限判定，也负责规则编辑与 `ToolPermissionContext` 同步，是权限运行时与配置持久化的汇合点。

---

### 现在把第 62-63 站串起来

```text
coordinatorHandler.ts
  -> ask-path 的前置自动化审批协调器：hooks first, classifier second
permissions.ts
  -> 生成 allow / deny / ask 的底层权限结果，并定义 mode/classifier/headless 等运行时变换语义
```

所以现在 permission 子系统可以压缩成：

```text
tool invocation
  -> permissions.ts computes base PermissionDecision
  -> if ask:
       useCanUseTool routes into
         coordinatorHandler (optional pre-dialog automation)
         swarmWorkerHandler (worker->leader escalation)
         interactiveHandler (final interactive race coordinator)
```

也就是说：

```text
coordinatorHandler.ts 回答“ask 之后，dialog 前要不要先跑自动化检查”
permissions.ts 回答“这次 tool use 为什么会变成 allow / deny / ask，以及 auto/dontAsk/headless 等模式怎样进一步改写这个结果”
```

---

### 下一站建议

下一站最顺的是：

```text
src/utils/permissions/permissionRuleParser.ts
```

因为现在你已经看懂了：

- `permissions.ts` 怎样消费 `PermissionRuleValue`
- allow / deny / ask rules 怎样参与整条权限判定流水线
- MCP / Agent / Bash content rules 怎样被解释使用
- 但规则字符串本身到底怎样解析成 `PermissionRuleValue`，还没拆开：
  - `Bash(prefix:*)`
  - `Agent(Explore)`
  - `mcp__server__tool`
  - 这些规则的语法边界和规范化细节是什么

下一步最自然就是把权限规则语言本身补齐：

**`permissionRuleParser.ts` 到底怎样把用户/配置里的权限规则字符串解析成结构化 `PermissionRuleValue`，并为 `permissions.ts` 的整工具级匹配、内容级匹配与规则展示提供统一语法基础。**
