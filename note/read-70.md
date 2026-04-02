## 第 70 站：`src/utils/hooks/hooksConfigManager.ts`

### 这是什么文件

`src/utils/hooks/hooksConfigManager.ts` 是 hooks 子系统里的 **配置展示元数据层、event/matcher 分组整理器，以及 hooks settings / registered hooks 到 UI 可消费视图的桥接层**。

上一站 `src/schemas/hooks.ts` 已经看懂：

- 那一层定义的是 hooks 持久化配置 schema
- 说明了哪些 hook 类型能写进 settings / plugin / skill config
- 也说明了 hooks settings 的顶层结构是 `event -> matcher[] -> hooks[]`
- 但还差一层：
  - UI 或配置管理界面怎样给每个 hook event 提供“人能看懂”的说明
  - 哪些 event 支持 matcher，matcher 匹配的是哪个字段
  - 已加载 hooks 怎样按 event + matcher 整理成稳定展示结构
  - settings hooks、plugin hooks、builtin/internal hooks 又怎样一起被投影成统一视图

这一站回答的是：

```text
hooks 配置在“管理/展示层”怎样被组织；
每个 hook event 有哪些人类可读元数据；
以及所有 hooks 怎样被分组为 event-keyed / matcher-keyed 的可消费结构
```

所以最准确的一句话是：

```text
hooksConfigManager.ts = hooks 配置管理视图层：负责提供 hook event 元数据、matcher 元信息，以及把多来源 hooks 整理成按 event 和 matcher 分组的统一展示结构
```

---

### 先看它的整体定位

位置：`src/utils/hooks/hooksConfigManager.ts:1`

这个文件不是：

- hook runtime 执行器
- hook schema 定义文件
- hook settings 解析/持久化文件
- matcher 实际求值逻辑本身

它的职责是：

1. 定义 hook event 的展示元数据
2. 定义 matcher 的展示元信息（匹配哪个字段、候选值有哪些）
3. 把所有 hooks 按 `event -> matcher -> hook[]` 重新整理
4. 合并 settings hooks 与 registered hooks 到同一展示结构
5. 提供读取某个 event 的 matcher 列表、某个 matcher 下 hooks 列表的便捷 API

所以它本质上是一个：

```text
hook config presentation adapter
+ event metadata registry
+ grouped-view builder
```

也就是说：

- `schemas/hooks.ts` 定义“配置允许长什么样”
- `hooksConfigManager.ts` 定义“这些配置在管理层/展示层怎样被解释和组织”

---

### 第一部分：文件一开头先定义 `MatcherMetadata` / `HookEventMetadata`，说明这里关心的不是 runtime 载荷，而是“给人看”的管理信息

位置：`src/utils/hooks/hooksConfigManager.ts:11`

这里两类元数据非常直接：

#### `MatcherMetadata`
```ts
{
  fieldToMatch: string,
  values: string[]
}
```

#### `HookEventMetadata`
```ts
{
  summary: string,
  description: string,
  matcherMetadata?: MatcherMetadata
}
```

这说明这个文件首先处理的是：

```text
configuration UX metadata
```

而不是：

```text
runtime execution semantics in code form
```

也就是说它要回答的是：

- 这个 event 是干什么的
- 它在什么时候触发
- exit code / stdout / stderr 在产品上是什么意思
- 如果支持 matcher，到底匹配哪一个输入字段
- 用户可以从哪些候选值里选 matcher

所以这一层本质是 hooks 子系统的“配置说明词典”。

---

### 第二部分：`getHookEventMetadata(...)` 是整份文件的核心，它把每个 HookEvent 对应的产品语义、输入说明、退出码语义和 matcher 域统一编码成一份静态元数据表

位置：`src/utils/hooks/hooksConfigManager.ts:26`

这个函数返回：

```ts
Record<HookEvent, HookEventMetadata>
```

也就是说每个 hook event 在这里都有一份正式元数据档案。

里面不只是简短标题，
还包含了大量产品级语义：

- 输入 JSON 长什么样
- stdout / stderr 会被谁看到
- exit code 0 / 2 / other 分别意味着什么
- 哪些事件支持 blocking，哪些不支持
- 哪些事件允许 JSON hookSpecificOutput 覆盖行为

这说明这个文件不是随便做点 UI label，
而是在集中表达：

```text
how hook events should be understood and configured by humans
```

所以它其实是 hook event 的“产品语义目录”。

---

### 第三部分：`memoize(..., toolNames => toolNames.slice().sort().join(','))` 很关键，说明作者明确在防御一个 UI/render 层常见问题：每次传入新数组对象都会把 memoize cache 打爆

位置：`src/utils/hooks/hooksConfigManager.ts:22`

注释写得非常清楚：

- 调用方可能每次 render 都传一个新的 `toolNames` 数组
- 如果直接按对象 identity 做 cache key
- 就会每次 miss，导致 cache entry 不断增长

所以这里改成：

```ts
toolNames => toolNames.slice().sort().join(',')
```

这说明作者要缓存的不是：

```text
this exact array instance
```

而是：

```text
this semantic set of tool names
```

这种设计非常像典型的展示层/selector 层优化：

```text
stable semantic cache key
instead of unstable render-time object identity
```

说明这个文件虽然小，但确实是面向频繁调用的配置 UI 辅助层。

---

### 第四部分：很多 event 的 `description` 里直接写入 exit code 语义，说明这里不只是“展示标题”，而是在把 runtime contract 翻译成配置者可理解的操作说明

位置：例如：

- `PreToolUse` `src/utils/hooks/hooksConfigManager.ts:29`
- `PostToolUse` `src/utils/hooks/hooksConfigManager.ts:38`
- `PermissionRequest` `src/utils/hooks/hooksConfigManager.ts:163`
- `Elicitation` `src/utils/hooks/hooksConfigManager.ts:196`

你会发现说明文案里反复出现：

- `Exit code 0`
- `Exit code 2`
- `Other exit codes`
- stdout/stderr 是否 shown to model / user / transcript
- 某些事件可以 `block`
- 某些事件只是 observability-only

这说明这个文件在做一层重要翻译：

```text
turn hook runtime behavior into author-facing configuration guidance
```

也就是说 `hooks.ts` 里的执行协议，
到了这里被压缩成可写配置时需要理解的“产品文案形式”。

所以它在系统里的价值不是逻辑控制，
而是：

```text
make the runtime contract legible at config-management time
```

---

### 第五部分：不同 event 的 `matcherMetadata` 非常值得看，因为它说明 matcher 在配置面不是统一字符串，而是“事件相关字段上的条件桶”

位置：贯穿 `getHookEventMetadata(...)`

例如：

#### 工具类事件
- `PreToolUse`
- `PostToolUse`
- `PostToolUseFailure`
- `PermissionDenied`
- `PermissionRequest`

匹配字段都是：

```ts
fieldToMatch: 'tool_name'
```

#### 通知事件
```ts
fieldToMatch: 'notification_type'
```

#### SessionStart
```ts
fieldToMatch: 'source'
```

#### PreCompact / PostCompact / Setup
```ts
fieldToMatch: 'trigger'
```

#### SessionEnd
```ts
fieldToMatch: 'reason'
```

#### ConfigChange
```ts
fieldToMatch: 'source'
```

#### InstructionsLoaded
```ts
fieldToMatch: 'load_reason'
```

这说明 matcher 的真实语义不是：

```text
arbitrary regex over some unspecified thing
```

而是：

```text
for each event, matcher is scoped to a specific event field
```

也就是说这个文件把“matcher 匹配的到底是什么”正式说明出来了。

这对配置体验特别重要，
因为否则用户只会看到一个 `matcher?: string`，
却不知道这个字符串到底在匹配：

- tool name
- source
- trigger
- notification type
- load reason
- 还是别的东西

所以这里是在补上 matcher 的语义上下文。

---

### 第六部分：`values` 字段进一步说明这个文件还承担“候选值提供器”角色——不仅告诉你 matcher 匹配哪个字段，还尽量给出这个字段的合法/常见值集合

位置：同样贯穿 `getHookEventMetadata(...)`

例如：

- 工具类事件用 `toolNames`
- `Notification` 给出若干 notification types
- `SessionStart` 给出 `startup/resume/clear/compact`
- `PreCompact/PostCompact` 给出 `manual/auto`
- `SessionEnd` 给出 `clear/logout/prompt_input_exit/other`
- `ConfigChange` 给出 settings source 列表
- `InstructionsLoaded` 给出 load reason 列表

这说明 `matcherMetadata.values` 的作用不是 runtime 判定，
而是：

```text
UI hints / picker candidates / config authoring suggestions
```

特别是 `toolNames` 作为参数传入，
说明这份元数据不是完全静态死表，
而是会根据当前环境里实际可用工具动态补全候选值。

所以这个文件实际上站在：

```text
runtime-discovered capabilities
-> configuration UI hints
```

的中间。

---

### 第七部分：像 `SubagentStart`、`SubagentStop`、`Elicitation`、`ElicitationResult` 这些 `values: []` 特别有意思，说明这个文件允许“字段已知，但候选值暂时拿不到”的半结构化 matcher 描述

位置：

- `SubagentStart` `src/utils/hooks/hooksConfigManager.ts:117`
- `SubagentStop` `src/utils/hooks/hooksConfigManager.ts:126`
- `Elicitation` `src/utils/hooks/hooksConfigManager.ts:196`
- `ElicitationResult` `src/utils/hooks/hooksConfigManager.ts:205`

比如：

- subagent 事件知道 matcher 匹配 `agent_type`
- 但当前文件没有直接提供 agent type 列表
- elicitation 事件知道 matcher 匹配 `mcp_server_name`
- 但这里也没有把 MCP server names 注入进来

这说明作者的设计不是“没有候选值就不支持 matcher”，
而是：

```text
matcher domain can be known
even when the candidate set is not available here
```

所以 `MatcherMetadata` 其实表达了两层信息：

1. 匹配字段名
2. 可选的候选值集合

候选值集合可以为空，
但字段语义仍然成立。

这是一种挺成熟的配置建模方式。

---

### 第八部分：有些 event 根本没有 `matcherMetadata`，说明这个文件明确区分了“支持 matcher 的事件”和“不支持 matcher 的事件”，而不是所有事件都硬塞一个 matcher 桶

位置：例如：

- `UserPromptSubmit` `src/utils/hooks/hooksConfigManager.ts:81`
- `Stop` `src/utils/hooks/hooksConfigManager.ts:95`
- `TeammateIdle` `src/utils/hooks/hooksConfigManager.ts:181`
- `TaskCreated` `src/utils/hooks/hooksConfigManager.ts:186`
- `TaskCompleted` `src/utils/hooks/hooksConfigManager.ts:191`
- `WorktreeCreate` `src/utils/hooks/hooksConfigManager.ts:244`
- `WorktreeRemove` `src/utils/hooks/hooksConfigManager.ts:249`
- `CwdChanged` `src/utils/hooks/hooksConfigManager.ts:254`
- `FileChanged` 这里虽然描述里提到 matcher 语义，但 metadata 本身没提供 matcherMetadata

这说明这个文件不仅定义文案，
还在隐式定义一条展示规则：

```text
whether this event should be grouped/configured by matcher at all
```

后面 `groupHooksByEventAndMatcher(...)` 正是依据：

```ts
metadata[hook.event].matcherMetadata !== undefined
```

来决定 key 用 `hook.matcher` 还是空字符串。

所以这里的 `matcherMetadata` 不只是提示信息，
还是：

```text
a capability flag for matcher-aware grouping
```

---

### 第九部分：`groupHooksByEventAndMatcher(...)` 是这个文件的第二个核心，它把所有 hooks 统一投影成 `Record<HookEvent, Record<string, IndividualHookConfig[]>>`

位置：`src/utils/hooks/hooksConfigManager.ts:269`

这个函数先预建一个完整的 event map：

```ts
Record<HookEvent, Record<string, IndividualHookConfig[]>>
```

也就是说输出视图是：

```text
event
  -> matcher key
    -> hooks[]
```

这里最关键的是：

- 顶层先按 event 分桶
- 第二层再按 matcher 分桶
- 桶里装的是统一的 `IndividualHookConfig[]`

这说明这个文件真正提供的是：

```text
a normalized management view over all hooks
```

而不是只是返回 raw settings 结构。

因为 raw settings 原本是：

- 来自多个 source
- 结构可能不同
- registered hooks 不一定天然长成 settings 那个形状

这里把它们统一折叠成一个管理层可消费的分层 map。

---

### 第十部分：分组时先走 `getAllHooks(appState)`，说明 settings / skill / 其他配置来源里的 hooks 已经先被另一个模块统一摊平成 `IndividualHookConfig`，而这里专注做“再分组”

位置：`src/utils/hooks/hooksConfigManager.ts:306`

这里没有自己解析 settings，
而是直接调用：

```ts
getAllHooks(appState)
```

然后每个 hook 都具有：

- `event`
- `config`
- `matcher`
- `source`
- 可能还有 plugin / skill 相关 provenance

这说明职责分层很清楚：

#### hooksSettings.ts
负责把多来源 hooks 摊平、标准化

#### hooksConfigManager.ts
负责基于这些标准化结果，构造管理层分组视图和元数据

也就是说这个文件不是 source loader，
而是：

```text
post-normalization grouping layer
```

---

### 第十一部分：`metadata[hook.event].matcherMetadata !== undefined ? hook.matcher || '' : ''` 很关键，说明“是否按 matcher 分桶”不是看 hook 自己有没有 matcher 字符串，而是看该事件是否被声明为 matcher-aware

位置：`src/utils/hooks/hooksConfigManager.ts:310`

这点很重要。

因为如果只看 `hook.matcher` 是否存在，
会得到一种很脆弱的逻辑：

```text
whoever happened to have a matcher string gets its own bucket
```

而这里真正做的是：

```text
only events declared matcher-aware in metadata use matcher as a grouping axis
```

否则统一放进 `''` 这个空 key 桶里。

这说明 event metadata 在这里不只是说明文案，
而是参与了分组语义本身。

所以这份文件在架构上是：

```text
metadata-driven grouping
```

而不是简单数据整理。

---

### 第十二部分：registered hooks 的处理特别值得看，因为它说明这个文件要面对的不只是 settings hooks，还要把 plugin/native 注册钩子投影进同一个管理视图里

位置：`src/utils/hooks/hooksConfigManager.ts:322`

它会：

1. `getRegisteredHooks()`
2. 遍历 event -> matchers
3. 把 registered hooks 也塞进对应的 `eventGroup[matcherKey]`

这说明管理界面/配置视图想看到的不是：

```text
only persisted hooks from settings
```

而是：

```text
all effective hooks visible from the current app state and registry
```

也就是说这里的目标是“全景视图”，
不是“只看某一种来源的原始配置”。

---

### 第十三部分：`'pluginRoot' in matcher` 这个分支非常关键，说明 registered hooks 里混着不同 runtime-only 载体；这里只有 plugin matcher 才能被完整映射成展示用 `IndividualHookConfig`

位置：`src/utils/hooks/hooksConfigManager.ts:333`

注释已经说透：

- `PluginHookMatcher` 有 `pluginRoot`
- `HookCallbackMatcher` 没有
- 后者典型例子是 internal callback hooks

所以这里实际上在做来源判别：

#### plugin registered hook
可显示为真实插件 hook

#### internal callback hook
默认不显示成普通可配置 hook

这说明这个文件明确知道：

```text
not every runtime hook source maps cleanly to end-user config UI
```

也就是说 runtime registry 里存在很多“平台内部 hook”，
但配置视图只投影其中一部分。

---

### 第十四部分：`pluginRoot` 分支把 registered plugin hooks 重新包装成 `IndividualHookConfig`，说明这个文件还承担 provenance 翻译职责

位置：`src/utils/hooks/hooksConfigManager.ts:335`

这里包装出的对象包括：

- `event: hookEvent`
- `config: hook`
- `matcher: matcher.matcher`
- `source: 'pluginHook'`
- `pluginName: matcher.pluginId`

这说明 registered plugin hooks 原始结构和 settings hooks 的结构并不完全一致，
需要在这里被翻译成统一视图对象。

所以这个文件又承担了一层：

```text
registered-hook -> IndividualHookConfig projection
```

而且它还把来源语义显式写出来：

```text
source: 'pluginHook'
```

这样 UI/管理层后面就能区分：

- 来自 settings
- 来自 plugin
- 来自 builtin/internal 显示替身

---

### 第十五部分：`process.env.USER_TYPE === 'ant'` 这个分支很有意思，说明内部环境下 builtin/internal callback hooks 也会被投影成一个占位型配置条目，但对普通用户默认隐藏

位置：`src/utils/hooks/hooksConfigManager.ts:346`

这里如果 matcher 不是 plugin matcher，
又处在 `USER_TYPE === 'ant'`，
就会生成：

```ts
config: {
  type: 'command',
  command: '[ANT-ONLY] Built-in Hook',
}
source: 'builtinHook'
```

这说明：

1. 内部环境需要可观测 built-in hooks
2. 这些 hook 并不真的对应某条可持久化 command hook 配置
3. 所以这里只能构造一个“占位型视图对象”来代表它们

这非常说明该文件的定位：

```text
presentation first, literal config fidelity second
```

因为这里投影出来的并不是 hook 的真实执行体，
而是一个“供管理界面展示的替身表示”。

这也说明 internal callback/builtin hooks 并不是配置系统的一等 persisted type，
只是必要时可被管理层观察到。

---

### 第十六部分：`getSortedMatchersForEvent(...)` 说明这个文件不仅负责分组，还负责提供稳定、有优先级的 matcher 展示顺序

位置：`src/utils/hooks/hooksConfigManager.ts:367`

这里做的事情很简单：

1. 取出某个 event 下所有 matcher key
2. 调 `sortMatchersByPriority(...)`

这说明 matcher 在配置/展示层不是无序 map key，
而是：

```text
there is a meaningful presentation order
```

也就是说不同 matcher 可能需要按某种优先级稳定排列，
这样管理界面、列表展示、配置面板才能更可预期。

虽然排序规则不在本文件里，
但这个文件承担的是“把排序后的 matcher 列表暴露出去”的桥接职责。

---

### 第十七部分：`getHooksForMatcher(...)` 看起来很小，但它揭示了这个 grouped-view 的一个重要实现约定：不支持 matcher 的事件，统一用空字符串 key 存储

位置：`src/utils/hooks/hooksConfigManager.ts:379`

这里逻辑是：

```ts
const matcherKey = matcher ?? ''
return hooksByEventAndMatcher[event]?.[matcherKey] ?? []
```

说明内部统一约定：

- 有 matcher 的桶：key 就是 matcher 字符串
- 无 matcher 的桶：key 是 `''`

这是因为 record key 必须是 string。

所以这个文件不只是输出 grouped view，
还定义了 grouped view 的内部编码规则：

```text
no-matcher bucket = empty-string key
```

这类约定如果不在这里集中收敛，
调用方会很容易写出不一致逻辑。

---

### 第十八部分：`getMatcherMetadata(...)` 进一步说明这个文件的 API 设计是围绕“配置管理/配置 UI”来的：你可以先拿 event 元数据，再单独拿 matcher 元数据

位置：`src/utils/hooks/hooksConfigManager.ts:394`

它本质上是：

```ts
return getHookEventMetadata(toolNames)[event].matcherMetadata
```

这说明调用方的消费方式大概是：

- 列出所有 event
- 对某个 event 读 summary/description
- 如果支持 matcher，再读 matcher field / candidate values
- 再用 grouped hooks 渲染具体 hooks 列表

也就是说整个文件其实提供了一整套：

```text
metadata + grouping + lookup helpers
```

正好够一个 hooks config manager / menu / inspector 使用。

---

### 第十九部分：整份文件最核心的架构价值，是把 hooks 从“原始配置结构 + 运行时 registry”提升成了一套带语义说明、分组约定、来源投影和展示排序的管理视图模型

如果把整份文件压缩，会发现它其实在做四层事：

#### 1. event metadata registry
- `summary`
- `description`
- `matcherMetadata`

#### 2. matcher-domain metadata
- 匹配哪个字段
- 候选值有哪些
- 哪些 event 支持 matcher

#### 3. grouped management view
- `groupHooksByEventAndMatcher(...)`
- event -> matcher -> hooks[]

#### 4. view helper API
- `getSortedMatchersForEvent(...)`
- `getHooksForMatcher(...)`
- `getMatcherMetadata(...)`

所以最准确的压缩表达是：

```text
hooksConfigManager.ts = hooks 配置管理视图层：负责维护 hook event 的人类可读元数据、matcher 语义与候选值，并把 settings/registered hooks 统一整理成按 event 与 matcher 分组的可展示视图
```

---

### 元

问题：**这一站真正想解决的架构问题是什么？**

回答：这一站的元问题，是 hooks 配置为什么还需要一个面向“人”的管理视图层。schema 只能说明配置合法，真正让 UI 和管理功能可用的，是 event 元数据、matcher 说明和分组后的统一视图。

### 反

问题：**如果把这一站的设计反过来，会发生什么？**

回答：如果没有这层，hook 管理界面只能直接啃底层配置结构。那样配置虽然存在，但使用者看不懂 event 语义，也很难稳定浏览和整理多来源 hooks。

### 空

问题：**跳出当前文件名，这一站背后更大的问题是什么？**

回答：更大的问题，是复杂配置系统怎样被解释成可操作的产品界面。这个文件把 runtime contract 翻译成了配置管理语言。

### 读完这一站后，你应该抓住的 10 个事实

1. `src/utils/hooks/hooksConfigManager.ts` 不是 hook 执行器，也不是 schema 文件，而是 hooks 的配置管理/展示层。
2. `MatcherMetadata` 与 `HookEventMetadata` 说明这个文件的核心对象是“给人看的配置元数据”，包括 summary、description、matcher field 和候选值。
3. `getHookEventMetadata(...)` 为每个 `HookEvent` 提供统一元数据，里面不仅有标题，还有输入 JSON、stdout/stderr 行为、exit code 语义和 blocking/override 规则说明。
4. `getHookEventMetadata(...)` 使用带 resolver 的 `memoize`，按排序后的 `toolNames` 生成稳定 cache key，避免 UI 层每次传新数组导致缓存泄漏。
5. `matcherMetadata` 说明 matcher 不是抽象字符串，而是针对特定 event 输入字段的过滤条件，例如 `tool_name`、`source`、`trigger`、`reason`、`notification_type`、`load_reason`。
6. `matcherMetadata.values` 不是 runtime 判定所需，而是配置编写/展示时的候选值提示；有些事件字段已知但候选值暂时为空，例如 `agent_type`、`mcp_server_name`。
7. `groupHooksByEventAndMatcher(...)` 会把 hooks 统一整理成 `Record<HookEvent, Record<string, IndividualHookConfig[]>>`，也就是 `event -> matcher -> hooks[]` 的管理视图。
8. 该分组逻辑是否按 matcher 分桶，不是看某条 hook 有没有 matcher，而是看该 event 的 metadata 是否声明了 `matcherMetadata`，体现了 metadata-driven grouping。
9. `groupHooksByEventAndMatcher(...)` 不只合并 settings hooks，还会把 registered plugin hooks 投影成统一的 `IndividualHookConfig`；internal callback hooks 默认不作为普通配置项展示，只在 ant 内部环境下显示为 builtin hook 占位项。
10. `getSortedMatchersForEvent(...)`、`getHooksForMatcher(...)`、`getMatcherMetadata(...)` 表明这个文件最终暴露的是一组面向配置 UI / config manager 的 helper API，而不是 runtime 执行 API。

---

### 现在把第 69-70 站串起来

```text
src/schemas/hooks.ts
  -> 定义 hooks 持久化配置 schema、hook 类型 union 与 matcher/settings 容器结构
src/utils/hooks/hooksConfigManager.ts
  -> 定义 hooks 配置管理视图、event 元数据、matcher 元信息与分组后的展示结构
```

所以现在 hooks 子系统可以进一步压缩成：

```text
settings / plugin / skill hook config
  -> schemas/hooks.ts validates persisted hook definitions
normalized hook configs / registered hooks
  -> hooksConfigManager.ts groups them for management UI and config inspection
runtime execution
  -> hooks.ts matches and runs them
runtime output contract
  -> types/hooks.ts validates returned protocol
```

也就是说：

```text
schemas/hooks.ts 回答“hooks 配置文件允许写成什么样”
hooksConfigManager.ts 回答“这些 hooks 在配置管理层怎样被说明、分组和展示”
```

---

### 下一站建议

下一站最顺的是：

```text
src/utils/hooks/hooksSettings.ts
```

因为现在你已经看懂了：

- `hooks.ts` 是 runtime 执行中枢
- `types/hooks.ts` 是运行时协议层
- `schemas/hooks.ts` 是配置 schema 层
- `hooksConfigManager.ts` 是配置展示/分组层
- 但还差“多来源 hooks 到底怎样被扁平化成 `IndividualHookConfig`、怎样排序、怎样供 configManager/runtime 共享”的标准化数据层

而 `hooksConfigManager.ts` 已经直接依赖：

- `getAllHooks(...)`
- `sortMatchersByPriority(...)`
- `IndividualHookConfig`

所以下一步最自然就是把 hooks settings 的标准化与聚合层补齐：

**`src/utils/hooks/hooksSettings.ts` 到底怎样定义 `IndividualHookConfig`、怎样把 settings / plugin / skill 等来源的 hooks 摊平、排序并提供给 `hooksConfigManager.ts` 和 hooks 运行时共同消费。**
