## 第 68 站：`src/types/hooks.ts`

### 这是什么文件

`src/types/hooks.ts` 是 hooks 子系统里的 **类型契约层、JSON 协议 schema 层与 sync/async hook 输出判别层**。

上一站 `src/utils/hooks.ts` 已经看懂：

- 它是 hooks runtime 中枢
- 负责匹配、执行、聚合、分发 hooks
- 但它反复依赖一些“协议级别”的概念：
  - 什么算合法 hook JSON 输出
  - `PermissionRequest` hook 到底能返回什么
  - `prompt request` 子协议长什么样
  - sync / async hook 怎样区分
  - callback hook / aggregated result 的正式类型边界是什么

这一站回答的是：

```text
hooks runtime 所依赖的正式协议，
在类型层到底怎样被定义、验证与约束；
以及 hook 输出怎样从“任意 JSON”提升成 Claude Code 可依赖的结构化 contract
```

所以最准确的一句话是：

```text
types/hooks.ts = hooks 子系统的协议契约层：定义 hook 的输入输出结构、JSON schema、prompt 子协议、权限决策载荷与聚合结果类型
```

---

### 先看它的整体定位

位置：`src/types/hooks.ts:1`

这个文件不是：

- hook 执行器
- hook matcher
- settings 合并器
- 某个具体 hook event 的 runtime handler

它的职责是：

1. 定义 hook event 判别辅助
2. 定义 prompt request / response 子协议
3. 定义 sync hook JSON 输出 schema
4. 定义 async hook JSON 输出 schema
5. 定义统一 `hookJSONOutputSchema`
6. 提供 sync / async type guard
7. 定义 callback hook / matcher / progress / blocking error 等运行时类型
8. 定义 `PermissionRequestResult`、`HookResult`、`AggregatedHookResult` 等 hooks runtime 核心数据结构

所以它本质上是一个：

```text
hook wire-contract module
+ zod validation boundary
+ runtime type bridge
```

也就是说 `hooks.ts` 负责“怎么跑”，
而 `types/hooks.ts` 负责“跑出来的数据在类型上必须长什么样”。

---

### 第一部分：文件头 import 一眼就能看出，这个文件的重点不是本地小 type alias，而是把 SDK 类型、Zod schema、权限 schema 与 runtime 结果类型对齐成同一份正式协议

位置：`src/types/hooks.ts:1`

这里同时依赖：

- `HookEvent` / `HOOK_EVENTS` / `HookInput` / `PermissionUpdate`
- SDK 侧的 `HookJSONOutput` / `AsyncHookJSONOutput` / `SyncHookJSONOutput`
- `permissionBehaviorSchema()`
- `permissionUpdateSchema()`
- `PermissionResult`
- `Message`

这说明这份文件不是“只给本地模块看的内部类型”。

它真正承担的是：

```text
align the agent SDK contract,
runtime validation schema,
and internal hook execution semantics
```

也就是说这里是 hooks 协议的对齐层：

```text
SDK-facing types
<-> Zod validation
<-> runtime TypeScript types
```

---

### 第二部分：`isHookEvent(...)` 很小但很关键，说明 hook event 名称不是松散字符串，而是被 `HOOK_EVENTS` 这套枚举边界正式约束；运行时需要一个显式 type guard 来把外部字符串收束成受支持事件

位置：`src/types/hooks.ts:22`

逻辑非常直接：

```ts
export function isHookEvent(value: string): value is HookEvent {
  return HOOK_EVENTS.includes(value as HookEvent)
}
```

这说明 hook 事件在系统里不是“只要字符串拼对了就行”，
而是：

```text
there is a closed event vocabulary
```

因此这份文件的一层价值是：

```text
turn hook event names from ad-hoc strings into a validated protocol enum
```

这对 settings 解析、SDK 边界和 runtime dispatch 都很关键。

---

### 第三部分：`promptRequestSchema` / `PromptRequest` / `PromptResponse` 定义了 hook 与 Claude Code 之间一个独立的“交互式 prompt 子协议”，说明 hooks 不只是单向回调，还能在执行中向宿主要选择题式询问

位置：

- request schema `src/types/hooks.ts:28`
- request type `src/types/hooks.ts:42`
- response type `src/types/hooks.ts:44`

这个协议长这样：

#### hook 发起请求
```ts
{
  prompt: string,
  message: string,
  options: [{ key, label, description? }]
}
```

#### 宿主返回响应
```ts
{
  prompt_response: string,
  selected: string
}
```

注释说得也很明白：

- `prompt` 字段本身就是 discriminator
- 值是 request id
- 这套模式和 `{ async: true }` 类似

这说明 hooks 协议中已经内建了：

```text
hook -> host interactive elicitation during execution
```

也就是说 command hook 并不只是：

```text
receive input once, return result once
```

而是可以进入一种受控的问答回路。

---

### 第四部分：`syncHookResponseSchema` 是这份文件的核心，说明“同步 hook JSON 输出”不是一个平面对象，而是“通用控制字段 + 事件专属 payload”的两层协议结构

位置：`src/types/hooks.ts:50`

顶层通用字段包括：

- `continue`
- `suppressOutput`
- `stopReason`
- `decision`
- `reason`
- `systemMessage`
- `hookSpecificOutput`

这说明 sync hook JSON 输出本身就分成两层：

#### 通用层
表达：
- 要不要继续
- 要不要 suppress 输出
- approve / block 决策
- system message

#### 事件专属层
由 `hookSpecificOutput` 表达：
- 哪个 hook event
- 该事件特有的附加载荷

这是一种很成熟的协议设计：

```text
common control envelope
+ event-specific typed payload
```

而不是给每个 event 搞一套完全不同的 JSON 根对象。

---

### 第五部分：`hookSpecificOutput` 的 union 非常重要，说明 hook 协议并不是“有个 eventName 字段就行”，而是为不同事件定义了严格不同的结构化能力面

位置：`src/types/hooks.ts:70`

这里按事件把 payload 明确拆开：

#### `PreToolUse`
- `permissionDecision`
- `permissionDecisionReason`
- `updatedInput`
- `additionalContext`

#### `SessionStart`
- `additionalContext`
- `initialUserMessage`
- `watchPaths`

#### `PostToolUse`
- `additionalContext`
- `updatedMCPToolOutput`

#### `PermissionDenied`
- `retry`

#### `PermissionRequest`
- `decision`，且进一步分：
  - `allow` + `updatedInput?` + `updatedPermissions?`
  - `deny` + `message?` + `interrupt?`

#### `Elicitation` / `ElicitationResult`
- `action`
- `content`

#### `CwdChanged` / `FileChanged`
- `watchPaths`

#### `WorktreeCreate`
- `worktreePath`

这说明 hook 协议是强类型的：

```text
what a hook may say depends on which event it claims to be handling
```

所以这层不是“event name + bag of optional fields”，
而是带 discriminator 的正式 tagged union。

---

### 第六部分：`PermissionRequest` 那个分支尤其关键，说明 permission hooks 的输出被正式建模成一个二元批准协议，而不是复用顶层 `decision: approve|block` 草草带过

位置：`src/types/hooks.ts:120`

这里专门定义：

#### allow
```ts
{
  behavior: 'allow',
  updatedInput?: Record<string, unknown>,
  updatedPermissions?: PermissionUpdate[]
}
```

#### deny
```ts
{
  behavior: 'deny',
  message?: string,
  interrupt?: boolean
}
```

这说明 PermissionRequest hooks 的设计目标不是：

```text
just approve or reject the request
```

而是：

```text
act as a structured permission authority
that may also rewrite tool input or emit permission persistence updates
```

这也解释了为什么上一站 `hooks.ts` 会把 `permissionRequestResult` 单独抽出来：

- 它不是普通 hook side effect
- 它是 permission runtime 的正式决策载荷

---

### 第七部分：`hookJSONOutputSchema` 把 sync 与 async 两种 hook 响应统一成一个 union，说明 async hook 不是“执行方式上的特例”，而是 hook 协议的一等响应形态

位置：`src/types/hooks.ts:169`

这里定义：

```ts
const asyncHookResponseSchema = z.object({
  async: z.literal(true),
  asyncTimeout: z.number().optional(),
})

return z.union([asyncHookResponseSchema, syncHookResponseSchema()])
```

也就是说 hook 输出协议从顶层就明确承认两种合法形态：

#### 同步结果
直接携带控制语义与专属 payload

#### 异步结果
只声明：
- 我是 async hook
- 可选 timeout

这说明异步 hook 不是 runtime 私下约定，
而是协议里正式存在的返回种类：

```text
async is part of the hook wire contract,
not merely an execution optimization
```

---

### 第八部分：`isSyncHookJSONOutput(...)` / `isAsyncHookJSONOutput(...)` 看似简单，但它们是 hooks runtime 分流的核心判别器；更重要的是，这份文件还用 compile-time assertion 强制 SDK 类型与 Zod schema 一致

位置：

- sync guard `src/types/hooks.ts:182`
- async guard `src/types/hooks.ts:189`
- type equality assertion `src/types/hooks.ts:195`

这里最关键的不只是 type guard，
而是下面这段：

```ts
type _assertSDKTypesMatch = Assert<
  IsEqual<SchemaHookJSONOutput, HookJSONOutput>
>
```

这说明作者明确担心一种风险：

```text
runtime Zod schema and SDK-declared TypeScript type drifting apart
```

所以他们加了编译期断言，强迫：

```text
schema-inferred type === SDK exported type
```

这是一种非常成熟的协议一致性防线。

---

### 第九部分：`HookCallback` / `HookCallbackContext` / `HookCallbackMatcher` 说明 callback hook 不是 shell hook 的小变种，而是 hooks 体系里的另一类一等执行载体：直接运行内存中的 TS 回调，并可访问 AppState / attribution state

位置：

- context `src/types/hooks.ts:203`
- callback `src/types/hooks.ts:211`
- matcher `src/types/hooks.ts:228`

`HookCallback` 的关键点：

- `type: 'callback'`
- `callback(input, toolUseID, abort, hookIndex?, context?) => Promise<HookJSONOutput>`
- 可选 `timeout`
- 可选 `internal`

这里的 `context` 提供：

- `getAppState()`
- `updateAttributionState(...)`

这说明 callback hooks 在架构上是：

```text
in-process hooks with privileged state access
```

和 command / http / prompt / agent hooks 的边界非常不同。

尤其 `internal?: boolean` 很重要，
因为上一站已经看到：

- internal callback hooks 会被排除在 `tengu_run_hook` metrics 外
- 还能走 fast-path

所以这里定义的不只是一个函数签名，
而是 hooks 平台内部保留的“原生 hook 通道”。

---

### 第十部分：`HookProgress`、`HookBlockingError`、`PermissionRequestResult`、`HookResult`、`AggregatedHookResult` 共同定义了 hooks runtime 的结果语义层次：单 hook 结果、批量聚合结果、以及 UI / permission / lifecycle 需要消费的不同视图

位置：

- `HookProgress` `src/types/hooks.ts:234`
- `HookBlockingError` `src/types/hooks.ts:243`
- `PermissionRequestResult` `src/types/hooks.ts:248`
- `HookResult` `src/types/hooks.ts:260`
- `AggregatedHookResult` `src/types/hooks.ts:277`

这里的层次很清楚：

#### `HookResult`
更贴近单个 hook 执行结果：
- `message`
- `blockingError`
- `outcome`
- `permissionBehavior`
- `updatedInput`
- `updatedMCPToolOutput`
- `permissionRequestResult`
- `retry`

#### `AggregatedHookResult`
更贴近批量执行后的外部消费视图：
- `blockingErrors?`
- `permissionBehavior?`
- `additionalContexts?`
- `updatedInput?`
- `updatedMCPToolOutput?`
- `permissionRequestResult?`

这说明 hooks runtime 的设计并不是：

```text
run hooks -> get one bool
```

而是：

```text
single-hook execution semantics
-> batch aggregation semantics
-> downstream consumer semantics
```

所以 `types/hooks.ts` 在这里定义的是 hooks 系统的数据流分层。

---

### 第十一部分：`HookResult` 与 `AggregatedHookResult` 的字段设计，也说明 hooks 影响的不只是“阻断还是放行”，而是能同时触碰消息面、权限面、上下文面和输入改写面

例如：

#### 消息面
- `message`
- `systemMessage`

#### 控制流面
- `preventContinuation`
- `stopReason`
- `retry`

#### 权限面
- `permissionBehavior`
- `permissionRequestResult`
- `hookPermissionDecisionReason`

#### 输入/输出改写面
- `updatedInput`
- `updatedMCPToolOutput`

#### 上下文注入面
- `additionalContext`
- `initialUserMessage`

这说明 hooks 被建模成：

```text
multi-surface automation effects
```

而不是单一的 approve/block 开关。

也正因为有这层类型约束，上一站 `hooks.ts` 才能做稳定的结果聚合与事件分发。

---

### 第十二部分：整份文件最核心的架构价值，是把 hooks 从“脚本约定”提升成一套正式的、可校验的、和 SDK/runtime 同步演进的协议系统

如果把整份文件压缩，会发现它其实在做四层事：

#### 1. event vocabulary boundary
- `isHookEvent`
- `HOOK_EVENTS`

#### 2. hook wire protocol
- `promptRequestSchema`
- `syncHookResponseSchema`
- `hookJSONOutputSchema`
- sync/async guards

#### 3. permission-specific contract
- `PermissionRequestResult`
- `permissionBehaviorSchema`
- `permissionUpdateSchema`

#### 4. runtime result types
- `HookCallback`
- `HookProgress`
- `HookBlockingError`
- `HookResult`
- `AggregatedHookResult`

所以最准确的压缩表达是：

```text
types/hooks.ts = hooks 子系统的协议契约层：负责定义 hook event 边界、JSON 输出 schema、prompt 子协议、权限请求决策载荷，以及 hooks runtime 在单 hook 与聚合层面的正式结果类型
```

---

### 元

问题：**这一站真正想解决的架构问题是什么？**

回答：这一站关心的是，hooks runtime 为什么必须建立正式协议类型，而不是接受“任意 JSON”。只有把 prompt 子协议、sync/async 输出、权限结果和聚合结果都类型化，执行层才有稳定边界。

### 反

问题：**如果把这一站的设计反过来，会发生什么？**

回答：如果缺少这一层，`hooks.ts` 虽然还能跑，但永远只能在弱约束数据上勉强工作。协议一旦扩展，最先受伤的就是验证、分支判断和错误恢复。

### 空

问题：**跳出当前文件名，这一站背后更大的问题是什么？**

回答：更大的问题，是动态扩展系统怎样给外部结果建立可信 contract。类型层的意义，不是多写类型，而是给 runtime 提供可依赖的语义护栏。

### 读完这一站后，你应该抓住的 10 个事实

1. `src/types/hooks.ts` 不是执行器，而是 hooks 子系统的协议契约层，负责把 hooks 的 runtime 语义正式类型化和 schema 化。
2. `isHookEvent(...)` 说明 hook event 名称来自一套封闭的 `HOOK_EVENTS` 枚举，而不是任意字符串。
3. `promptRequestSchema` / `PromptRequest` / `PromptResponse` 定义了 hook 在执行期间向宿主请求用户选择的子协议，说明 hooks 支持受控交互而非纯单向执行。
4. `syncHookResponseSchema` 把同步 hook JSON 输出建模成“通用控制字段 + `hookSpecificOutput` 事件专属 payload”的两层结构。
5. `hookSpecificOutput` 是一个严格的 tagged union，不同 hook event 允许的字段不同，例如 `PreToolUse` 可返回 `updatedInput`，`PostToolUse` 可返回 `updatedMCPToolOutput`，`SessionStart` / `CwdChanged` 可返回 `watchPaths`。
6. `PermissionRequest` 的 hook-specific payload 被专门建模成 `allow` / `deny` 两类结构化决策，并可附带 `updatedInput`、`updatedPermissions`、`interrupt` 等正式字段。
7. `hookJSONOutputSchema` 把 sync 与 async hook 响应统一成一个顶层 union，说明 async hook 是协议里的一等返回形态，而不是运行时私有特例。
8. `isSyncHookJSONOutput(...)` / `isAsyncHookJSONOutput(...)` 是 hooks runtime 分流的关键 type guard，而 compile-time `IsEqual` 断言又确保 SDK 类型与 Zod schema 不会悄悄漂移。
9. `HookCallback` / `HookCallbackContext` 说明 callback hooks 是 hooks 系统里的另一类一等载体：可直接运行内存中的回调并访问 AppState / attribution state。
10. `HookResult` 与 `AggregatedHookResult` 定义了 hooks runtime 的单 hook 与聚合结果语义，表明 hooks 能同时影响消息、控制流、权限、输入改写与上下文注入多个表面。

---

### 现在把第 67-68 站串起来

```text
hooks.ts
  -> hook runtime：收集、匹配、执行、聚合 hooks
src/types/hooks.ts
  -> hook protocol：定义 hook 输出 schema、prompt 子协议与聚合结果类型
```

所以现在 hooks 子系统可以压缩成：

```text
hook event occurs
  -> hooks.ts builds HookInput and finds matching hooks
  -> command/http/prompt/agent/callback/function hooks execute
  -> outputs are validated against types/hooks.ts schemas
  -> results become AggregatedHookResult / PermissionRequestResult / context updates
  -> permission flow / tool loop / session lifecycle consume them
```

也就是说：

```text
hooks.ts 回答“hooks 在运行时怎样被挑出来并真正执行”
types/hooks.ts 回答“这些 hooks 在协议层被允许返回什么，以及 runtime 怎样知道这份返回值是合法的”
```

---

### 下一站建议

下一站最顺的是：

```text
src/schemas/hooks.ts
```

因为现在你已经看懂了：

- `types/hooks.ts` 主要定义的是 runtime 协议与输出 contract
- 但 hooks 还有另一层非常关键的“配置面 schema”：
  - settings 文件里的 hooks matcher / hook command / prompt / agent / http 配置到底怎样被定义
  - 哪些字段是用户配置层提供的，哪些是 runtime 追加的
- 这层如果不补，整个 hooks 子系统还缺“配置入口”那一半

下一步最自然就是把 hooks 配置 schema 层补齐：

**`src/schemas/hooks.ts` 到底怎样定义 hooks 在 settings / plugins / skills 中的配置结构、matcher 形态与不同 hook 类型的配置 schema，并作为 `hooks.ts` 运行时之前的配置验证入口。**
