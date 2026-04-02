## 第 66 站：`src/utils/permissions/permissionsLoader.ts`

### 这是什么文件

`src/utils/permissions/permissionsLoader.ts` 是权限系统里的 **磁盘规则加载器、可编辑 settings 规则读写桥，以及 managed-only 规则门控层**。

上一站 `permissionSetup.ts` 已经看懂：

- `initializeToolPermissionContext(...)` 会调用 `loadAllPermissionRulesFromDisk()`
- auto mode / bypass / plan mode 的上下文装配和护栏都依赖磁盘规则输入
- 但“规则到底怎样从不同 settings source 读出来、怎样写回、managed-only 怎么影响加载与编辑”，还没拆开

这一站回答的是：

```text
权限规则在 settings 文件里到底怎样被加载成 PermissionRule[]；
哪些 source 可编辑、哪些只读；
allowManagedPermissionRulesOnly 又怎样改变加载与 UI/持久化行为
```

所以最准确的一句话是：

```text
permissionsLoader.ts = 权限规则的 settings I/O 层与来源治理层
```

---

### 先看它的整体定位

位置：`src/utils/permissions/permissionsLoader.ts:1`

这个文件不是：

- permission rule parser 本体
- permission runtime evaluator
- ToolPermissionContext assembler
- interactive permission UI

它的职责是：

1. 从 settings source 读取 permission rules
2. 把 settings JSON 转成 `PermissionRule[]`
3. 根据 policy 决定是否只尊重 managed rules
4. 提供可编辑 source 的 add/delete 规则写回能力
5. 在读写过程中处理 legacy tool name 归一化、重复项规整、部分 schema 失效时的 lenient fallback

所以它本质上是一个：

```text
permission rules persistence adapter
+ source policy gate
```

也就是说它站在：

```text
settings files
  <-> PermissionRule[]
  <-> runtime permission setup
```

的中间。

---

### 第一部分：`shouldAllowManagedPermissionRulesOnly()` 和 `shouldShowAlwaysAllowOptions()` 一开场就说明这个文件不只是读文件，还在实现产品级“谁有资格定义 permission rules”的治理策略

位置：

- managed-only `src/utils/permissions/permissionsLoader.ts:31`
- UI option gate `src/utils/permissions/permissionsLoader.ts:42`

这里的逻辑非常直接：

- `policySettings.allowManagedPermissionRulesOnly === true`
- 则只允许 managed permission rules 生效
- 同时 UI 上的“always allow”选项也应隐藏

这说明 `permissionsLoader.ts` 不只是一个被动的数据读取器。

它还承载了一个很明确的组织策略：

```text
when policy says only managed rules count,
user/project/local permission self-service is no longer authoritative
```

这点非常重要，
因为它直接影响：

- runtime 加载哪些规则
- 交互审批 UI 允许不允许持久化 allow
- settings 写回是否应该发生

也就是说“权限规则来源治理”是在这一层统一落地的。

---

### 第二部分：`SUPPORTED_RULE_BEHAVIORS = ['allow', 'deny', 'ask']` 说明磁盘规则层只承认这三类行为，并且把它们当作 settings JSON 到 runtime `PermissionRule` 的固定桥接枚举

位置：`src/utils/permissions/permissionsLoader.ts:46`

这说明 settings 层里的权限结构不是任意键值对，
而是围绕三种核心行为建模：

- allow
- deny
- ask

后面的 `settingsJsonToRules(...)` 也是围绕这三类行为逐项展开。

这表明磁盘规则格式和 runtime decision vocabulary 是直接对齐的：

```text
settings permissions buckets
  -> runtime PermissionRule behaviors
```

所以这层并不是“通用 JSON 配置读写”，
而是权限语言的持久化专用适配器。

---

### 第三部分：`getSettingsForSourceLenient_FOR_EDITING_ONLY_NOT_FOR_READING(...)` 很关键，说明规则写回场景与规则执行场景被刻意分开——编辑时宁可容忍 settings 里别的字段校验失败，也不能因为 unrelated schema 错误就把现有 permission rules 丢掉

位置：`src/utils/permissions/permissionsLoader.ts:61`

这个函数的语义非常明确：

- 只用于 editing
- 不做 schema validation
- 只是把 JSON parse 出来
- 保留原始对象用于读/改/写

注释特别强调：

```text
FOR EDITING ONLY - do not use this for reading settings for execution.
```

这说明作者在解决一个很实际的问题：

```text
settings file may contain unrelated validation errors (e.g. hooks),
but permission-rule edits should still preserve and update the file safely
```

也就是说运行时读取需要严格，
而编辑时需要保守地“尽量不丢数据”。

这是 settings 系统里一个很成熟的双轨策略：

#### for execution
严格验证

#### for editing
宽松读取，最小修改，尽量保留原状

---

### 第四部分：`settingsJsonToRules(...)` 把磁盘上的 `permissions.{allow,deny,ask}` 数组统一转换成 `PermissionRule[]`，说明 settings 文件里的权限配置格式非常简单，但会在这一层被标准化成带 source/provenance 的 runtime 规则对象

位置：`src/utils/permissions/permissionsLoader.ts:91`

这里做的事情很清楚：

- 没 `data.permissions` -> 空数组
- 遍历 `allow/deny/ask`
- 每个 ruleString 用 `permissionRuleValueFromString(...)` 解析
- 包装成：
  - `source`
  - `ruleBehavior`
  - `ruleValue`

这说明 settings JSON 本身只保存：

```text
behavior bucket + rule strings
```

而 runtime 真正要消费的是：

```text
PermissionRule { source, ruleBehavior, ruleValue }
```

所以 `permissionsLoader.ts` 的一个核心作用就是：

```text
attach provenance and structure to raw persisted rule strings
```

---

### 第五部分：`loadAllPermissionRulesFromDisk()` 很重要，因为它把“managed-only”策略真正落到 runtime 加载语义上——一旦开启，就只加载 `policySettings`；否则按 `getEnabledSettingSources()` 全量收集

位置：`src/utils/permissions/permissionsLoader.ts:120`

这里逻辑分成两种模式：

#### managed-only 开启
```ts
return getPermissionRulesForSource('policySettings')
```

#### 否则
```ts
for (const source of getEnabledSettingSources()) {
  rules.push(...getPermissionRulesForSource(source))
}
```

这说明 `allowManagedPermissionRulesOnly` 不是 UI 提示层的小开关，
而是直接改写磁盘规则加载策略的硬 gate。

也就是说：

```text
policy can collapse a multi-source permission universe
into a single managed source of truth
```

这对企业/受管环境非常关键。

---

### 第六部分：`getPermissionRulesForSource(...)` 非常薄，说明这份文件刻意把职责拆开：settings source 读取归 `settings.ts`，JSON→规则结构转换归 `permissionsLoader.ts`，而不是全塞在一起

位置：`src/utils/permissions/permissionsLoader.ts:140`

这个函数只做：

1. `getSettingsForSource(source)`
2. `settingsJsonToRules(settingsData, source)`

这说明作者想让模块边界保持清楚：

#### settings subsystem
负责“从哪里拿到 source 对应的 settings JSON”

#### permission loader
负责“如何把 settings JSON 中的 permission 部分解释成 PermissionRule[]”

所以这里体现的是：

```text
settings storage concerns
vs
permission domain projection
```

的清晰分层。

---

### 第七部分：`EDITABLE_SOURCES` 只包含 `userSettings/projectSettings/localSettings`，说明这份文件明确区分了“规则可生效的来源”与“规则可被用户编辑的来源”；`policySettings`、`flagSettings` 等是只读治理源

位置：`src/utils/permissions/permissionsLoader.ts:151`

这里可编辑来源只有：

- `userSettings`
- `projectSettings`
- `localSettings`

不包括：

- `policySettings`
- `flagSettings`
- `command`
- `cliArg`
- `session`

这说明权限来源在系统里有两层语义：

#### source participates in evaluation
可以影响 runtime 权限结果

#### source is writable by current surface
当前产品表面可以把规则持久化进去

二者不是同一件事。

这正是权限治理系统成熟的表现：

```text
not every authoritative rule source is user-editable
```

---

### 第八部分：`deletePermissionRuleFromSettings(...)` 通过 parse→serialize roundtrip 来做 legacy/canonical 名称归一化匹配，非常说明规则删除不是简单字符串比较，而是以语义等价为准

位置：`src/utils/permissions/permissionsLoader.ts:163`

这里最关键的细节是：

```ts
const normalizeEntry = (raw: string): string =>
  permissionRuleValueToString(permissionRuleValueFromString(raw))
```

然后用这个归一化结果去比对目��� `ruleString`。

这说明作者明确知道 settings 文件里可能还保存着：

- 旧工具名
- 历史 rule string 形态
- 语义等价但表面不完全一致的条目

所以删除逻辑按的是：

```text
semantic normalization
not raw text identity
```

这能避免：

- 界面上看到的是 canonical 名
- 磁盘里存的是 legacy 名
- 结果删不掉

这样的问题。

---

### 第九部分：`addPermissionRulesToSettings(...)` 也做了同样的 roundtrip 去重，说明这份文件把“规则规范化”当成持久化正确性的基础——防止 legacy/canonical 变体重复共存

位置：`src/utils/permissions/permissionsLoader.ts:229`

这里会：

- 把新增 `ruleValues` 转成 `ruleStrings`
- 把已有 raw entries 也统一 parse→serialize 归一化
- 用 `Set` 去重
- 只追加真正新的规则

这说明写入时也不是简单 append，
而是：

```text
normalize existing state first,
then add only semantically new rules
```

因此这份文件在持久化层保证的是：

- legacy 名不会造成重复
- 规则字符串的规范形态更稳定
- UI / runtime / disk 之间更容易一致化

---

### 第十部分：`addPermissionRulesToSettings(...)` 在正常 settings loader 失败时会退回 lenient loader，这说明权限规则编辑被作者视为“尽量不因 unrelated settings 问题而中断”的核心用户操作

位置：`src/utils/permissions/permissionsLoader.ts:249`

这里顺序是：

1. `getSettingsForSource(source)`
2. 失败则 `getSettingsForSourceLenient_FOR_EDITING_ONLY_NOT_FOR_READING(source)`
3. 再不行就 `getEmptyPermissionSettingsJson()`

这说明规则编辑 UX 的设计目标是：

```text
even if the settings file is partially invalid,
permission-rule persistence should still salvage and preserve as much as possible
```

这很合理，
因为 permission rules 往往来自：

- 用户在审批对话框里点“always allow”
- 用户显式编辑安全相关设置

如果这里太脆弱，用户会非常困惑。

所以这层做的是一种“局部可修复式写入”。

---

### 第十一部分：`shouldAllowManagedPermissionRulesOnly()` 还会影响 `addPermissionRulesToSettings(...)`，说明 managed-only 不只是“运行时忽略非托管规则”，而是“连新增持久化规则都不允许写入”

位置：`src/utils/permissions/permissionsLoader.ts:239`

这里一进函数就判断：

```ts
if (shouldAllowManagedPermissionRulesOnly()) {
  return false
}
```

这说明 managed-only 策略的强度比表面更高：

- 不是“你写了也不生效”
- 而是“系统直接拒绝写入新的非托管 permission rules”

这体现的是一种很明确的治理哲学：

```text
if only managed permission rules are allowed,
self-service persistence should be structurally disabled
```

这也解释了为什么上一站 `permissionSetup.ts` 要配合 `shouldShowAlwaysAllowOptions()` 去隐藏 UI。

---

### 第十二部分：整份文件最核心的架构价值，是把“磁盘 settings 文件里的权限规则”从散乱 JSON 字符串数组提升成了受来源治理、兼容归一化、编辑安全与 managed-only 政策共同约束的正式权限数据层

如果把整份文件压缩，会发现它其实在做五层事：

#### 1. source policy gate
- `shouldAllowManagedPermissionRulesOnly`
- `shouldShowAlwaysAllowOptions`

#### 2. persistence read projection
- `settingsJsonToRules`
- `getPermissionRulesForSource`
- `loadAllPermissionRulesFromDisk`

#### 3. editable source boundary
- `EDITABLE_SOURCES`
- `PermissionRuleFromEditableSettings`

#### 4. safe mutation helpers
- `deletePermissionRuleFromSettings`
- `addPermissionRulesToSettings`

#### 5. compatibility normalization
- parse→serialize roundtrip for legacy/canonical equivalence
- lenient editing fallback

所以最准确的压缩表达是：

```text
permissionsLoader.ts = 权限规则的持久化与来源治理层：负责把 settings 中的 permission strings 投影成带 source 的 PermissionRule，并在可编辑来源上提供兼容且保守的规则增删写回能力，同时执行 managed-only 策略
```

---

### 元

问题：**这一站真正想解决的架构问题是什么？**

回答：这一站关心的是，磁盘上的权限配置如何被稳定读成 runtime 规则，又如何被安全写回。它把 settings source、managed-only 治理和 lenient editing 边界收在一起。

### 反

问题：**如果把这一站的设计反过来，会发生什么？**

回答：如果读写规则不经过统一 loader，执行态与编辑态会很快分裂。规则来源、可编辑性和兼容性判断会在不同代码路径里相互冲突。

### 空

问题：**跳出当前文件名，这一站背后更大的问题是什么？**

回答：更大的问题，是“设置文件”怎样被产品化成可治理的数据源。这个文件说明，持久化适配层本身就是权限系统的一部分。

### 读完这一站后，你应该抓住的 10 个事实

1. `permissionsLoader.ts` 不是做权限判定的，而是负责从 settings source 读取/写回 permission rules，并处理来源治理策略。
2. `shouldAllowManagedPermissionRulesOnly()` 说明 policySettings 可以开启 managed-only 模式，此时 runtime 只尊重托管规则。
3. `shouldShowAlwaysAllowOptions()` 表明 managed-only 不只影响加载，还会影响交互 UI 是否应该展示“always allow”类持久化入口。
4. `settingsJsonToRules(...)` 把 settings JSON 里的 `permissions.allow/deny/ask` 字符串数组转换成 runtime `PermissionRule[]`，并补上 `source` 与 `ruleBehavior` provenance。
5. `loadAllPermissionRulesFromDisk()` 在 managed-only 模式下只加载 `policySettings`，否则按 `getEnabledSettingSources()` 聚合所有启用来源。
6. `EDITABLE_SOURCES` 只允许 `userSettings`、`projectSettings`、`localSettings` 被写回，说明不是所有生效来源都可编辑。
7. `deletePermissionRuleFromSettings(...)` 用 parse→serialize roundtrip 归一化 existing entries，因此删除依据是语义等价，而不是原始字符串完全一致。
8. `addPermissionRulesToSettings(...)` 也会做 roundtrip 去重，防止 legacy/canonical 工具名变体导致重复规则并存。
9. 规则编辑时，正常 settings loader 失败会退回 lenient loader，说明 permission rule 持久化被设计成尽量不受 unrelated schema 错误牵连。
10. managed-only 不只是运行时忽略非托管规则，还会直接阻止新增 permission rules 写入，从而在 persistence 层彻底关闭自助授权持久化。

---

### 现在把第 65-66 站串起来

```text
permissionSetup.ts
  -> 组装 ToolPermissionContext，并根据 mode/gate 对规则做运行态护栏变换
permissionsLoader.ts
  -> 从 settings source 读取/写回 PermissionRule，并管理哪些来源可读、可写、可被托管策略屏蔽
```

所以现在权限控制面的数据流可以压缩成：

```text
settings files / policy settings
  -> permissionsLoader.ts loads PermissionRule[]
  -> permissionSetup.ts assembles ToolPermissionContext
  -> permissions.ts evaluates tool use under that context
```

也就是说：

```text
permissionSetup.ts 回答“规则和 mode 在启动/切换时怎样进入当前权限上下文”
permissionsLoader.ts 回答“这些规则在磁盘 settings source 里怎样读取、怎样写回，以及 managed-only 策略怎样改变来源治理”
```

---

### 下一站建议

下一站最顺的是：

```text
src/utils/hooks.ts
```

因为现在权限子系统已经从：

- rule parser
- permissions engine
- permission setup
- permissions loader
- useCanUseTool / coordinator / interactive / swarm handlers

这一整条都基本串起来了。

而在这条链里，下一块最自然的邻接模块就是前面多次出现、但还没系统拆开的：

- `executePermissionRequestHooks(...)`
- `runHooks(...)`
- hook event / matcher / execution model

也就是权限 ask-path 背后的 hooks 基础设施。

下一步最自然就是把 hooks 主实现层补齐：

**`src/utils/hooks.ts` 到底怎样组织 hook 类型、匹配规则、执行入口与事件分发，并让 PermissionRequest / sessionStart / stopHooks / tool loop 等多个运行时切面都能通过同一套 hook 基础设施接入自动化逻辑。**
