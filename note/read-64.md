## 第 64 站：`src/utils/permissions/permissionRuleParser.ts`

### 这是什么文件

`src/utils/permissions/permissionRuleParser.ts` 是权限规则系统里的 **规则字符串语法解析器、序列化器与历史别名归一化层**。

上一站 `permissions.ts` 已经看懂：

- 整个权限引擎到处在消费 `PermissionRuleValue`
- allow / deny / ask rules 最终都会变成结构化的 `{ toolName, ruleContent? }`
- 但规则字符串本身到底怎样从配置/磁盘里解析出来，还没拆开

这一站回答的是：

```text
像：
- Bash
- Bash(npm publish:*)
- Agent(Explore)
- mcp__server__tool
- 含括号和反斜杠的复杂内容

这些权限规则字符串，
到底怎样被解析成结构化 PermissionRuleValue，
又怎样在写回配置时被稳定地转回字符串
```

所以最准确的一句话是：

```text
permissionRuleParser.ts = 权限规则语言的语法边界与双向编解码器
```

---

### 先看它的整体定位

位置：`src/utils/permissions/permissionRuleParser.ts:1`

这个文件不是：

- permission rule evaluator
- tool permission orchestrator
- settings loader
- 某个 tool 的具体 rule matcher

它的职责是：

1. 解析 rule string -> `PermissionRuleValue`
2. 序列化 `PermissionRuleValue` -> rule string
3. 处理内容里的转义字符
4. 归一化 legacy tool names 到 canonical tool names
5. 为旧规则、旧持久化数据、旧 hooks/wire names 提供兼容桥

所以它本质上是一个：

```text
permission rule syntax codec
+ legacy name normalizer
```

也就是说它提供的不是“权限决定”，
而是权限系统共同使用的那门小型规则语言。

---

### 第一部分：文件头的 `LEGACY_TOOL_NAME_ALIASES` 很关键，说明这份文件不仅处理语法，还承担规则语言的历史兼容层；旧工具名会在解析阶段就被折叠到当前 canonical name

位置：`src/utils/permissions/permissionRuleParser.ts:18`

这里把旧名映射成新名：

- `Task -> Agent`
- `KillShell -> TaskStop`
- `AgentOutputTool -> TaskOutput`
- `BashOutputTool -> TaskOutput`
- 某些 feature-gated 工具也会加 alias

注释写得非常明确：

```text
When a tool is renamed, add old → new here so permission rules,
hooks, and persisted wire names resolve to the canonical name.
```

这说明权限规则语言不是纯粹“当前版本语法”，
而是要兼容：

- 旧配置文件中的规则
- 历史持久化状态
- 旧 hooks / wire names

所以 parser 的职责之一是：

```text
make historical rule strings continue to mean the same thing
```

这对于长期使用中的 permission settings 非常重要。

---

### 第二部分：`normalizeLegacyToolName(...)` 和 `getLegacyToolNames(...)` 一进一出，说明这层不仅要在读取时做归一化，还要保留“从 canonical name 反查历史名”的能力，供兼容展示/匹配使用

位置：

- normalize `src/utils/permissions/permissionRuleParser.ts:31`
- reverse lookup `src/utils/permissions/permissionRuleParser.ts:35`

`normalizeLegacyToolName(name)` 很简单：

- 有 alias -> 返回 canonical
- 没 alias -> 原样返回

而 `getLegacyToolNames(canonicalName)` 则反向扫一遍映射表。

这说明这个文件的兼容语义不是单向的“读旧写新”而已，
还要支持：

```text
what historical names should still be recognized as this canonical tool?
```

这对：

- migration
- settings cleanup
- 规则展示
- 兼容匹配

都很有价值。

---

### 第三部分：`escapeRuleContent(...)` / `unescapeRuleContent(...)` 暴露了这门规则语言最核心的语法难点：规则使用 `Tool(content)` 结构，所以 content 本身如果含括号或反斜杠，必须稳定转义

位置：

- escape `src/utils/permissions/permissionRuleParser.ts:55`
- unescape `src/utils/permissions/permissionRuleParser.ts:74`

这里的规则非常明确：

#### escape 顺序
1. `\ -> \\`
2. `( -> \(`
3. `) -> \)`

#### unescape 顺序
1. `\( -> (`
2. `\) -> )`
3. `\\ -> \`

注释反复强调：

```text
order matters
```

这说明作者非常清楚这个 parser 的脆弱点不是“字符串切一刀”，
而是：

```text
content itself may contain syntax-like characters
```

如果顺序错了，就会：

- 双重转义错乱
- 反斜杠被提前吃掉
- 括号边界被误判

所以这里本质上是在保证这门小语法能处理真实命令内容，而不是只处理玩具示例。

---

### 第四部分：`permissionRuleValueFromString(...)` 的设计很重要——它不是用正则一把梭，而是用“找第一个未转义左括号 + 最后一个未转义右括号”的方式做结构边界识别，说明作者在尽量容忍 content 内部的复杂字符

位置：`src/utils/permissions/permissionRuleParser.ts:93`

核心逻辑是：

1. 找第一个未转义 `(`
2. 找最后一个未转义 `)`
3. 要求 `)` 必须在字符串末尾
4. 中间部分视为 rawContent
5. 再 unescape

这说明 parser 的结构假设是：

```text
toolName(content)
```

但 content 允许包含：

- 转义括号
- 反斜杠
- 其他任意字符

作者没有用贪婪/非贪婪正则，
而是手写边界扫描，说明他们更在意：

```text
predictable handling of escapes and malformed input
```

这是很稳妥的 parser 设计。

---

### 第五部分：当规则字符串不合法时，这个 parser 基本都选择“退化成整串 tool name”，而不是抛错，说明它的首要目标是兼容和容错，而不是严格拒绝非规范输入

位置：`src/utils/permissions/permissionRuleParser.ts:96`

这些情况都会退回：

- 没有未转义左括号
- 没有合法右括号
- 右括号不在结尾
- toolName 缺失，例如 `(foo)`

返回的都是：

```ts
{ toolName: normalizeLegacyToolName(ruleString) }
```

这说明这份文件的设计哲学是：

```text
best-effort parse,
not strict validation
```

也就是说 parser 假定：

- 规则字符串可能来自历史配置
- 可能有轻微 malformed 情况
- 比起直接炸掉，保守地把它当作 tool-wide rule 更安全/更兼容

这也说明“语法验证”与“语法解析”在这里不是同一层责任。

---

### 第六部分：`Bash()` 和 `Bash(*)` 都会被当成纯 `Bash`，说明这门规则语言把“空内容”与“整体 wildcard 内容”归一化为 tool-wide rule，而不是保留一个表面不同但语义相同的结构

位置：`src/utils/permissions/permissionRuleParser.ts:124`

这里有一条非常关键的归一化：

- `rawContent === ''`
- `rawContent === '*'`

都会返回：

```ts
{ toolName: normalizeLegacyToolName(toolName) }
```

而不是：

```ts
{ toolName: 'Bash', ruleContent: '*' }
```

这说明在这门规则语言里：

```text
Tool()
Tool(*)
Tool
```

被视为同一语义：

```text
tool-wide rule
```

这能避免无意义的语法分叉，
也让后面的 matcher 更简单。

---

### 第七部分：`permissionRuleValueToString(...)` 与 `permissionRuleValueFromString(...)` 是一对稳定的双向编解码器，它们保证 permission rules 可持久化、可 round-trip，而不是一次性 parse helper

位置：`src/utils/permissions/permissionRuleParser.ts:144`

序列化逻辑很直接：

- 没 `ruleContent` -> 输出 `toolName`
- 有 `ruleContent` -> 先 escape，再输出 `toolName(content)`

这说明这个文件不是只为 runtime parse 服务，
还负责：

```text
structured in memory representation
<->
persisted string representation
```

也就是说它是权限规则的“线协议/磁盘协议”编解码器。

这一点很重要，
因为后面 `permissions.ts`、settings loader、UI 编辑规则时都依赖这套 round-trip 稳定性。

---

### 第八部分：`findFirstUnescapedChar(...)` / `findLastUnescapedChar(...)` 用“前面有奇数个反斜杠就算 escaped”的规则来判断边界，这说明作者显式支持了嵌套式转义情形，而不是只处理单个 `\(`

位置：

- first `src/utils/permissions/permissionRuleParser.ts:158`
- last `src/utils/permissions/permissionRuleParser.ts:181`

它们的逻辑是：

- 遇到目标字符
- 向前数连续反斜杠个数
- 偶数 -> 未转义
- 奇数 -> 已转义

这说明 parser 正式支持这样的语义：

```text
\(   -> escaped paren
\\(  -> unescaped paren, because the second slash escapes the first slash
```

这比简单地看“前一个字符是不是反斜杠”更正确。

所以这里不是小工具函数，
而是整个规则语法正确性的核心基础件。

---

### 第九部分：`findFirstUnescapedChar` 取左边界、`findLastUnescapedChar` 取右边界，这种不对称设计很妙，说明 content 本身允许包含中间未转义的括号文本，只要最外层结构仍然能被唯一定位

位置：`src/utils/permissions/permissionRuleParser.ts:96`

解析时不是找“第一个左括号后对应的第一个右括号”，
而是：

- 第一个未转义 `(`
- 最后一个未转义 `)`

这意味着像这样的内容：

```text
Bash(python -c "print(1)")
```

如果内部括号已被正确转义，就没问题；
甚至在一些更复杂但仍能唯一定位外层边界的情形下，也能保持稳健。

说明作者的目标不是最简 parser，
而是：

```text
robust outer-boundary detection for a single-layer wrapper syntax
```

---

### 第十部分：feature-gated 的 `BRIEF_TOOL_NAME` 采用动态 require，而不是静态 import，说明这份 parser 还承担“不要把某些 ant-only tool names 泄漏到外部 build”的打包边界职责

位置：`src/utils/permissions/permissionRuleParser.ts:7`

注释非常明确：

```text
Dead code elimination: ant-only tool names are conditionally required so
their strings don't leak into external builds. Static imports always bundle.
```

这说明 permission rule parser 不只是语法文件，
它还位于一个很敏感的边界：

```text
tool names appear in persisted / configured permission rules,
so parser-level imports can leak product surface area into builds
```

因此这里要格外小心：

- 哪些工具名会被打进 bundle
- 哪些 alias 会暴露给外部版本

这是一种很容易忽略，但很真实的产品打包约束。

---

### 第十一部分：这份文件与 `permissions.ts` 的关系非常清楚——前者定义权限规则的“语言”，后者定义权限规则的“执行语义”

把上一站和这一站并排看：

#### `permissionRuleParser.ts`
负责：
- `string -> PermissionRuleValue`
- `PermissionRuleValue -> string`
- legacy name normalization
- escape / unescape

#### `permissions.ts`
负责：
- 根据 `PermissionRuleValue` 做 allow / deny / ask 匹配
- 和 tool.checkPermissions / mode / classifier 等整合
- 生成最终 `PermissionDecision`

所以两者关系可以压缩成：

```text
permissionRuleParser.ts = rule language layer
permissions.ts = rule execution layer
```

也就是说：

- parser 决定“规则字符串长什么样、怎样转成结构化值”
- permissions 决定“这个结构化值在权限流水线里意味着什么”

这两个层次分得很干净。

---

### 第十二部分：整份文件最核心的架构价值，是把 permissions 子系统里的“规则字符串”从脆弱的 ad-hoc 文本约定提升成了一个有转义、有兼容层、有 round-trip 语义的小型 DSL

如果把整份文件压缩，会发现它其实在做四件事：

#### 1. 语法兼容
- 旧工具名归一化
- canonical / legacy 名双向桥接

#### 2. 语法安全
- 内容里的括号 / 反斜杠转义与反转义

#### 3. 容错解析
- malformed 输入保守退化
- 不轻易抛错

#### 4. 稳定序列化
- `PermissionRuleValue <-> string` round-trip

所以最准确的压缩表达是：

```text
permissionRuleParser.ts = 权限规则 DSL 的编解码层：既定义规则字符串的外层语法与转义规则，又负责把历史工具名与持久化规则稳定归一化到当前权限系统可消费的结构化表示
```

---

### 读完这一站后，你应该抓住的 10 个事实

1. `permissionRuleParser.ts` 不是做权限判定的，而是做权限规则语言本身的解析、序列化和历史兼容归一化。
2. `LEGACY_TOOL_NAME_ALIASES` 说明工具一旦改名，旧规则、旧 hooks、旧 wire names 仍能在 parser 层被折叠到 canonical tool name。
3. `normalizeLegacyToolName(...)` 和 `getLegacyToolNames(...)` 一正一反，说明这层不仅处理输入兼容，也保留 canonical→legacy 的反查能力。
4. 规则内容采用 `Tool(content)` 语法，因此括号和反斜杠必须显式 escape/unescape，而且顺序非常重要。
5. `permissionRuleValueFromString(...)` 通过“第一个未转义左括号 + 最后一个未转义右括号”来识别外层结构，避免简单正则在复杂 content 上失真。
6. 对 malformed 规则字符串，这个 parser 基本采取 best-effort 策略：尽量退化成整串 tool name，而不是抛错中断系统。
7. `Tool()` 与 `Tool(*)` 都会被归一化成纯 `Tool`，说明空内容和整体 wildcard 在语义上都等价于 tool-wide rule。
8. `permissionRuleValueToString(...)` 与 `permissionRuleValueFromString(...)` 共同构成稳定的 round-trip codec，用于 settings / disk / runtime 间往返持久化。
9. `findFirstUnescapedChar(...)` / `findLastUnescapedChar(...)` 用“奇数个反斜杠表示 escaped”来处理边界，说明它支持比“前一个字符是否是反斜杠”更正确的转义语义。
10. feature-gated 动态 require 说明这份 parser 还承担了打包边界职责，避免某些 ant-only tool names 泄漏到外部 build 中。

---

### 现在把第 63-64 站串起来

```text
permissions.ts
  -> 定义 PermissionRuleValue 在权限流水线里的执行语义
permissionRuleParser.ts
  -> 定义 PermissionRuleValue 与规则字符串之间的编解码与历史兼容语义
```

所以现在 rule 子系统可以压缩成：

```text
rule string from settings / session / CLI
  -> permissionRuleParser.ts parses into PermissionRuleValue
  -> permissions.ts matches/explains/applies it inside the full permission pipeline
```

也就是说：

```text
permissions.ts 回答“规则在运行时怎么生效”
permissionRuleParser.ts 回答“规则字符串本身长什么样、怎样解析、怎样安全持久化以及怎样兼容历史命名”
```

---

### 下一站建议

下一站最顺的是：

```text
src/utils/permissions/permissionSetup.ts
```

因为现在你已经看懂了：

- `permissions.ts` 是核心权限引擎
- `permissionRuleParser.ts` 是规则 DSL 编解码层
- 但还没看规则和 mode 是怎样在 session / startup 阶段被装配进 `ToolPermissionContext` 的：
  - 初始 permission mode 怎么定
  - settings / CLI arg / session 规则怎么合并
  - 哪些危险规则会被修正或剔除
  - PowerShell / auto mode 的保护性规则清洗怎么落地

下一步最自然就是回到权限系统的装配入口：

**`permissionSetup.ts` 到底怎样在启动或上下文切换时构造初始 `ToolPermissionContext`、合并多来源权限规则，并对危险/过宽的 permission 配置做预处理与护栏修正。**
