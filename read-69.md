## 第 69 站：`src/schemas/hooks.ts`

### 这是什么文件

`src/schemas/hooks.ts` 是 hooks 子系统里的 **配置 schema 定义层、持久化 hook 配置验证边界，以及可持久化 hook 类型集合的正式入口**。

上一站 `src/types/hooks.ts` 已经看懂：

- 那一层定义的是 hook 运行时协议
- 重点是 hook 输出 JSON、prompt 子协议、callback/result 类型
- 但还差另一半：
  - settings / plugin / skill 里写的 hooks 配置，究竟允许哪些字段
  - 哪些 hook 类型能持久化到配置文件
  - matcher 配置的结构边界是什么
  - `if` 条件在配置面怎样定义

这一站回答的是：

```text
hooks 在配置文件里到底怎样被声明；
command / prompt / http / agent 这几类持久化 hook 的配置结构是什么；
matcher 与 event-keyed hooks map 又怎样被 Zod 正式约束
```

所以最准确的一句话是：

```text
schemas/hooks.ts = hooks 配置面的 schema 边界：负责定义可持久化 hooks 的配置结构，并为 settings / plugins / skills 提供统一验证入口
```

---

### 先看它的整体定位

位置：`src/schemas/hooks.ts:1`

这个文件不是：

- hook runtime 执行器
- hook JSON 输出协议定义文件
- session hook / callback hook 的运行态实现
- matcher 执行逻辑本身

它的职责是：

1. 定义 hooks 配置字段的 Zod schema
2. 定义可持久化 hook 类型的 discriminated union
3. 定义 matcher 层 `matcher + hooks[]` 结构
4. 定义整份 hooks settings 的 event-keyed map 结构
5. 给 settings/types、plugins/schemas 等模块提供共享 schema，避免循环依赖

所以它本质上是一个：

```text
hook settings schema module
+ persisted hook config boundary
```

也就是说：

- `types/hooks.ts` 定义运行时协议 contract
- `schemas/hooks.ts` 定义配置文件 contract

两者正好构成 hooks 子系统的“运行时协议面”和“配置面”两半。

---

### 第一部分：文件头注释已经明确说明它存在的直接原因不是功能扩展，而是架构解耦——把 hook schema 从 `settings/types.ts` 中抽出来，专门用来打断 import cycle

位置：`src/schemas/hooks.ts:1`

注释写得很清楚：

- 这些 schema 原本在 `src/utils/settings/types.ts`
- 抽出来后打断了 `settings/types.ts` 与 `plugins/schemas.ts` 的循环依赖
- 两边现在都改为从这个共享位置导入

这说明这个文件的架构价值首先是：

```text
shared hook schema dependency hub
```

也就是说它不只是“把 schema 搬了个家”，
而是在模块边界上承担了：

```text
make hook config validation reusable across settings and plugins
without entangling those subsystems
```

这种拆分非常像典型的：

- domain schema 下沉
- 让多个上层模块共享同一份验证边界

---

### 第二部分：`IfConditionSchema` 很关键，说明 hook 配置里的 `if` 字段不是任意条件表达式，而是明确复用 permission rule syntax 的一段字符串 DSL

位置：`src/schemas/hooks.ts:19`

注释已经把语义说透：

- `if` 使用 permission rule syntax
- 示例：`Bash(git *)`、`Read(*.ts)`
- 用于在 spawn 前过滤 hook
- 基于 `tool_name` 和 `tool_input` 求值

这说明 hooks 配置面的 `if` 设计不是另造一门过滤语言，
而是：

```text
reuse the permission rule DSL as the hook precondition language
```

这点特别重要，
因为它把三层东西接到了一起：

- permissions rule parser
- hooks runtime matcher
- hooks settings schema

也就是说这一层已经把“hooks 与 permissions 共用内容匹配 DSL”正式固化在配置边界里了。

---

### 第三部分：`buildHookSchemas()` 很能说明作者的设计意图——不是直接在 `HookCommandSchema` 里写一个大 union，而是先把各个 hook 变体 schema 拆成独立构件，再由统一工厂回收，方便共享与懒加载

位置：`src/schemas/hooks.ts:31`

这里先独立构造四类 schema：

- `BashCommandHookSchema`
- `PromptHookSchema`
- `HttpHookSchema`
- `AgentHookSchema`

然后在后面 `HookCommandSchema` 里再组合成 discriminated union。

这说明这个文件的结构不是：

```text
one monolithic schema blob
```

而是：

```text
build reusable per-hook schema pieces first,
then assemble the persisted hook union
```

这样做的好处是：

- 每种 hook 类型的字段边界更清楚
- 更容易被其他 schema 复用
- 配合 `lazySchema` 能更稳地避开 import cycle / eager-init 问题

---

### 第四部分：`BashCommandHookSchema` 不只是“command + timeout”，而是把 command hook 的产品能力面完整暴露出来：shell 选择、once、async、asyncRewake、statusMessage 都在配置层正式建模

位置：`src/schemas/hooks.ts:32`

字段包括：

- `type: 'command'`
- `command`
- `if`
- `shell`
- `timeout`
- `statusMessage`
- `once`
- `async`
- `asyncRewake`

这说明 command hooks 的配置语义已经远远超出传统 CLI hook：

#### shell 选择
- `bash` / `powershell`

#### 生命周期控制
- `once`

#### UX 可见性
- `statusMessage`

#### 执行模式
- `async`
- `asyncRewake`

尤其 `asyncRewake` 的描述非常关键：

```text
wake the model on exit code 2
```

这说明连“后台跑完后怎样回灌模型”的能力，也已经在配置 schema 层被正式暴露出来了。

---

### 第五部分：`PromptHookSchema` 与 `AgentHookSchema` 很值得并排看，因为它们说明 hooks 子系统已经不再局限于外部 shell 自动化，而是把“LLM verifier hook”和“agentic verifier hook”都纳入同一套配置面

位置：

- prompt `src/schemas/hooks.ts:67`
- agent `src/schemas/hooks.ts:128`

二者都包含：

- `prompt`
- `if`
- `timeout`
- `model`
- `statusMessage`
- `once`

这说明在配置面上：

```text
prompt hooks and agent hooks are peers of command hooks,
not bolt-on exceptions
```

也就是说 hooks 的配置语言里，自动化执行载体本来就是多模态的：

- shell command
- LLM prompt
- agent
- HTTP endpoint

而且 prompt / agent 都允许单独指定 `model`，这进一步说明 hooks 不只是“有没有 hook”，还包括：

```text
what execution substrate should power this automation
```

---

### 第六部分：`AgentHookSchema` 上那段“不要加 transform”的注释非常重要，说明这个 schema 文件不只是描述字段，它还承担持久化安全边界——避免 schema transform 在 parse→stringify roundtrip 里悄悄吞掉用户配置

位置：`src/schemas/hooks.ts:130`

注释说得极具体：

- 这个 schema 会被 `parseSettingsFile` 使用
- `updateSettingsForSource` 会把 parse 结果再 `JSON.stringify`
- 如果这里做 `.transform()` 把字符串 prompt 变成函数值
- 那 stringify 时函数会被静默丢掉
- 结果用户的 prompt 会从 settings.json 消失

这说明作者非常清楚一个常见但危险的坑：

```text
Zod transform can be semantically correct at runtime,
but destructive for persisted config round-trips
```

所以这个文件的职责不仅是“验证配置合法”，
还要保证：

```text
parsed settings remain serializable back to disk without silent data loss
```

这是配置 schema 设计里非常成熟的一点。

---

### 第七部分：`HttpHookSchema` 的 `headers + allowedEnvVars` 设计说明 HTTP hooks 在配置面被视为受控外发能力，而不是任意字符串模板——环境变量插值必须显式白名单化

位置：`src/schemas/hooks.ts:97`

这里字段包括：

- `url`
- `if`
- `timeout`
- `headers`
- `allowedEnvVars`
- `statusMessage`
- `once`

最关键的是描述文案：

- headers 值里可以写 `$VAR` / `${VAR}`
- 但只有 `allowedEnvVars` 里列出的变量才会被解析
- 未列出的变量会被留空

这说明 HTTP hook 的配置层在安全模型上有一条明确原则：

```text
env interpolation is opt-in and allowlisted
```

也就是说系统意识到：

- HTTP hook = 向外部系统发送数据
- header env 注入 = 可能涉及 token / secret

所以这里在 schema 层就先把“允许哪些 env 被用”做成显式配置字段。

---

### 第八部分：`HookCommandSchema` 作为 discriminated union 很关键，因为它定义了“可持久化 hook”的正式闭包——只有 `command` / `prompt` / `agent` / `http` 四类能进 settings；function/callback hooks 明确不属于持久化配置面

位置：`src/schemas/hooks.ts:176`

这里返回的是：

```ts
z.discriminatedUnion('type', [
  BashCommandHookSchema,
  PromptHookSchema,
  AgentHookSchema,
  HttpHookSchema,
])
```

注释也直接写了：

```text
excludes function hooks - they can't be persisted
```

这说明 hooks 系统在架构上明确区分了两层：

#### persisted hook kinds
可来自 settings / plugin manifest / skill config

#### runtime-only hook kinds
例如 function hook / callback hook，只存在于进程内注册体系

这个边界非常重要，
因为它解释了为什么上一站 `types/hooks.ts` 里有 callback 类型，
但这个文件里没有 callback schema：

```text
not every runtime hook type is a user-persistable config type
```

---

### 第九部分：`HookMatcherSchema` 说明 hooks 配置的基本组织单位不是“event -> hook[]”直挂，而是 `matcher + hooks[]` 的两层结构；event 只是第一层分类，matcher 才是第二层分发条件

位置：`src/schemas/hooks.ts:194`

结构是：

```ts
{
  matcher?: string,
  hooks: HookCommand[]
}
```

也就是说配置层的层次是：

```text
hook event
  -> matcher entries
    -> hooks array
```

而不是：

```text
hook event
  -> hooks array directly
```

这样设计的价值是：

- 同一事件可以挂多个 matcher 桶
- 每个 matcher 桶下再挂多个 hook
- hooks runtime 后面就能高效按 matcher 先筛一层，再决定真正执行哪些 hook

所以 `HookMatcherSchema` 本质上定义的是 hooks 配置的“中间索引层”。

---

### 第十部分：`HooksSchema` 把整份 hooks 配置建模成 `Partial<Record<HookEvent, HookMatcher[]>>`，说明 event 维度是顶层命名空间，但并不是所有事件都必须出现

位置：`src/schemas/hooks.ts:211`

这里用的是：

```ts
z.partialRecord(z.enum(HOOK_EVENTS), z.array(HookMatcherSchema()))
```

这个选择非常合理，因为 hooks settings 的真实语义是：

- 只有配置过的 event 才需要存在
- 未配置 event 不应被强行写成空数组
- 顶层 key 仍然必须来自受支持的 `HOOK_EVENTS`

这说明整个 hooks 配置面的正式边界是：

```text
sparse event-keyed config map
with each key constrained to the known hook event vocabulary
```

所以这个 schema 一方面保持灵活，另一方面又保证 event key 不会飘出协议边界。

---

### 第十一部分：文件尾部导出的类型别名也很关键，说明这份文件不只给 Zod parse 用，它还把“配置层类型”作为全项目共享的 TS 类型源头提供出去

位置：`src/schemas/hooks.ts:216`

这里导出：

- `HookCommand`
- `BashCommandHook`
- `PromptHook`
- `AgentHook`
- `HttpHook`
- `HookMatcher`
- `HooksSettings`

这说明这个文件同时承担两件事：

#### 运行时验证
Zod schema

#### 编译时约束
从 schema infer 出的 TS 类型

也就是说它和上一站的 `types/hooks.ts` 形成了一个互补关系：

- `types/hooks.ts` 更偏 runtime protocol types
- `schemas/hooks.ts` 更偏 persisted config types inferred from schema

这种分层很干净：

```text
protocol types
vs
config types inferred from validation schemas
```

---

### 第十二部分：整份文件最核心的架构价值，是把 hooks 的“配置入口”从散乱 JSON 约定提升成了一个能描述不同 hook 载体、共享 `if` DSL、安全限制与持久化边界的正式 schema 系统

如果把整份文件压缩，会发现它其实在做四层事：

#### 1. schema decoupling layer
- 从 `settings/types.ts` 抽离出来
- 打断 settings / plugins 循环依赖

#### 2. shared conditional DSL boundary
- `IfConditionSchema`
- 明确复用 permission rule syntax

#### 3. persisted hook type system
- `command`
- `prompt`
- `http`
- `agent`
- `HookCommandSchema` discriminated union

#### 4. hooks settings container shape
- `HookMatcherSchema`
- `HooksSchema`
- schema-inferred TS types

所以最准确的压缩表达是：

```text
schemas/hooks.ts = hooks 配置面的正式 schema 边界：负责定义可持久化 hook 类型、matcher 组织结构与 event-keyed hooks settings 形态，并通过共享 schema 打通 settings、plugins、skills 对 hooks 配置的统一验证
```

---

### 读完这一站后，你应该抓住的 10 个事实

1. `src/schemas/hooks.ts` 不是 hooks runtime，而是 hooks 配置面的 schema 定义层，用来验证 settings / plugins / skills 里的可持久化 hooks 配置。
2. 这个文件被单独抽出来的直接原因是打断 `settings/types.ts` 与 `plugins/schemas.ts` 的循环依赖，因此它是一个共享 schema hub。
3. `IfConditionSchema` 明确规定 hook 配置里的 `if` 字段使用 permission rule syntax，说明 hooks 与 permissions 在 DSL 层是打通的。
4. `buildHookSchemas()` 先构造各 hook 变体 schema，再由 `HookCommandSchema` 统一组装，体现了可复用的分层 schema 组织方式。
5. `BashCommandHookSchema` 把 command hook 的完整产品能力面暴露到配置层：`shell`、`timeout`、`statusMessage`、`once`、`async`、`asyncRewake` 都是正式配置字段。
6. `PromptHookSchema` 与 `AgentHookSchema` 说明 hooks 配置面原生支持 LLM prompt hook 和 agent hook，它们和 command/http hook 一样是 persisted config 中的一等类型。
7. `HttpHookSchema` 的 `headers` + `allowedEnvVars` 设计说明 HTTP hooks 的环境变量插值是显式白名单化的，这是面向外部请求的安全约束。
8. `AgentHookSchema` 上关于禁止 `.transform()` 的注释说明 schema 设计必须兼顾 settings round-trip 持久化安全，避免 parse→stringify 过程中静默丢失用户配置。
9. `HookCommandSchema` 作为 discriminated union 明确限定了可持久化 hook 的闭包只有 `command` / `prompt` / `agent` / `http` 四类；callback/function hooks 属于 runtime-only 类型，不进入配置文件 schema。
10. `HookMatcherSchema` 与 `HooksSchema` 说明 hooks 配置的整体结构是 `Partial<Record<HookEvent, HookMatcher[]>>`，即以 hook event 为顶层命名空间、以 matcher 为第二层分发单位的稀疏配置 map。

---

### 现在把第 68-69 站串起来

```text
src/types/hooks.ts
  -> 定义 hooks runtime 协议、JSON 输出 schema 与聚合结果类型
src/schemas/hooks.ts
  -> 定义 hooks 持久化配置 schema、hook 类型 union 与 matcher/settings 容器结构
```

所以现在 hooks 子系统可以进一步压缩成：

```text
settings / plugin / skill hook config
  -> schemas/hooks.ts validates persisted hook definitions
runtime hook execution
  -> hooks.ts matches and runs them
hook output
  -> types/hooks.ts validates and types the returned protocol
```

也就是说：

```text
types/hooks.ts 回答“hook 在运行时允许返回什么”
schemas/hooks.ts 回答“hook 在配置文件里允许写成什么样”
```

---

### 下一站建议

下一站最顺的是：

```text
src/utils/hooks/hooksConfigManager.ts
```

因为现在你已经看懂了：

- `hooks.ts` 是 runtime 执行中枢
- `types/hooks.ts` 是运行时协议层
- `schemas/hooks.ts` 是配置 schema 层
- 但还差“配置是怎样从 settings source 读进来、怎样 snapshot、怎样供 runtime 快速消费”的桥接层

而 `hooks.ts` 文件头已经直接依赖：

- `getHooksConfigFromSnapshot`
- 以及 managed-only / disable-all 相关 snapshot 逻辑

所以下一步最自然就是把 hooks 配置治理与 snapshot 层补齐：

**`src/utils/hooks/hooksConfigManager.ts` 到底怎样加载 hooks 配置、构建 snapshot、协调 managed-only / disable-all 等策略，并把可执行 hooks 视图提供给 `hooks.ts` 运行时使用。**
