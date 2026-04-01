## 第 72 站：`src/utils/hooks/sessionHooks.ts`

### 这是什么文件

`src/utils/hooks/sessionHooks.ts` 是 hooks 子系统里的 **session-scoped hook 存储层、内存态 function hook 注册中心，以及 session hooks 到 runtime/管理层可消费视图的桥接层**。

上一站 `src/utils/hooks/hooksSettings.ts` 已经看懂：

- 那一层会把 settings hooks 和 session hooks 摊平成统一 `IndividualHookConfig[]`
- 也说明 session hooks 是 hooks 生态里的正式来源之一
- 但还差另一半：
  - session hooks 到底存在哪里
  - 为什么它们是按 `sessionId` 分隔的
  - function hooks 是怎样单独建模的
  - 为什么 `getSessionHooks(...)` 要排除 function hooks
  - `onHookSuccess` callback 又怎样跟某条 session hook 绑定

这一站回答的是：

```text
session-scoped hooks 在内存里怎样保存、增删和清空；
function hooks 怎样作为 runtime-only 验证 hook 被纳入同一套 session 存储；
以及 session hooks 怎样被拆分投影成 HookCommand 视图和 FunctionHook 视图供不同层消费
```

所以最准确的一句话是：

```text
sessionHooks.ts = session hooks 的内存态管理层：负责按 session 保存/删除 hook，支持 runtime-only function hooks，并把 session hook 状态投影为 runtime 与管理层可消费的不同视图
```

---

### 先看它的整体定位

位置：`src/utils/hooks/sessionHooks.ts:1`

这个文件不是：

- hook runtime 执行器
- hook schema 文件
- persisted settings loader
- plugin hook 注册器

它的职责是：

1. 定义 session hook 的内存数据结构
2. 定义 runtime-only `FunctionHook`
3. 提供 session hook / function hook 的添加入口
4. 提供移除与清空入口
5. 提供按 event/session 读取 session hooks 的 API
6. 把 session hooks 拆成：
   - 可投影成 `HookCommand[]` 的普通 hooks
   - 不能持久化的 `FunctionHook[]`
7. 提供查回某条 hook 的 `onHookSuccess` 绑定信息

所以它本质上是一个：

```text
session hook registry
+ in-memory hook store
+ session hook projection layer
```

也就是说：

- `hooksSettings.ts` 只消费 session hooks
- `sessionHooks.ts` 才是真正定义它们怎样存、怎样删、怎样分类的地方

---

### 第一部分：文件开头的 `OnHookSuccess` 很关键，说明 session hook 不只是“执行一些 hook”，还支持把某条 hook 成功后的聚合结果回传给注册方

位置：`src/utils/hooks/sessionHooks.ts:9`

它的签名是：

```ts
(
  hook: HookCommand | FunctionHook,
  result: AggregatedHookResult,
) => void
```

这说明 session hook entry 里除了 hook 本体，
还可以带一个：

```text
post-success callback channel
```

也就是说某些 session hooks 的注册方并不只想“让它执行”，
还想在 hook 成功完成后拿到：

- 哪条 hook 成功了
- 聚合后的 hook 结果是什么

这说明 session hooks 在系统里不仅是配置输入，
也可以被用作一种：

```text
runtime observer / follow-up trigger
```

---

### 第二部分：`FunctionHookCallback` 与 `FunctionHook` 特别重要，它们说明 session hooks 体系里内建了一类完全不走外部 command/http/prompt/agent 的 runtime-only hook：直接在内存里执行 TS 回调

位置：

- callback type `src/utils/hooks/sessionHooks.ts:15`
- hook type `src/utils/hooks/sessionHooks.ts:24`

`FunctionHookCallback` 的签名是：

```ts
(messages: Message[], signal?: AbortSignal) => boolean | Promise<boolean>
```

`FunctionHook` 则包括：

- `type: 'function'`
- `id?`
- `timeout?`
- `callback`
- `errorMessage`
- `statusMessage?`

这说明 function hook 的语义不是“执行脚本并看 stdout/stderr”，
而是：

```text
run in-process validation logic over message history
```

而且注释已经明确写了：

```text
Session-scoped only, cannot be persisted to settings.json.
```

这再次印证前面几站的结论：

```text
runtime-only hooks and persisted hooks are different universes
```

---

### 第三部分：`FunctionHookCallback` 返回 `boolean | Promise<boolean>` 很值得注意，说明 function hook 的设计目标不是通用副作用执行器，而是偏向“同步/异步检查器”

位置：`src/utils/hooks/sessionHooks.ts:15`

注释说得很直白：

```text
returns true if check passes, false to block
```

这说明 function hooks 的核心产品语义更像：

```text
in-memory validation gate
```

而不是像 command hook 那样能产出一整套复杂 JSON 协议。

也就是说 function hook 在设计上更轻：

- 输入是消息
- 输出是 pass/fail
- 附带一个 `errorMessage` 作为失败说明

所以这一类 hook 更像：

```text
local assertion hooks
```

---

### 第四部分：`SessionHookMatcher` 很关键，因为它说明 session hooks 的存储单元并不是直接 `matcher -> HookCommand[]`，而是 `matcher -> { hook, onHookSuccess }[]`，说明存储层保留了比运行时投影视图更多的信息

位置：`src/utils/hooks/sessionHooks.ts:33`

结构是：

```ts
{
  matcher: string,
  skillRoot?: string,
  hooks: Array<{
    hook: HookCommand | FunctionHook,
    onHookSuccess?: OnHookSuccess,
  }>
}
```

这说明 session hook 存储层比普通 persisted `HookMatcher` 多了两类运行时信息：

1. `FunctionHook`
2. `onHookSuccess`

也就是说 sessionHooks.ts 保存的是：

```text
rich runtime matcher entries
```

而不是 persisted schema 那种纯配置数据。

这也解释了为什么后面要有“转换/投影”函数——
因为原始存储结构比公开给其他层的结构更富。

---

### 第五部分：`SessionStore` 和 `SessionHooksState` 明确说明 hooks 是按 `sessionId` 隔离保存的，每个 session 有自己的一套 `HookEvent -> SessionHookMatcher[]` 状态

位置：

- `SessionStore` `src/utils/hooks/sessionHooks.ts:42`
- `SessionHooksState` `src/utils/hooks/sessionHooks.ts:62`

这里的形态是：

```text
SessionHooksState = Map<sessionId, SessionStore>
SessionStore = {
  hooks: {
    [event in HookEvent]?: SessionHookMatcher[]
  }
}
```

这说明 session hooks 的作用域不是全局进程，
而是：

```text
session-local runtime hook registry
```

也就是说：

- 不同 session 彼此隔离
- 当前 session 结束可整体清空
- 同一个 AppState 可以同时挂多份 session hook store

这和“settings hooks 是跨 session 持久化的”形成了鲜明对比。

---

### 第六部分：关于 `SessionHooksState = Map<string, SessionStore>` 的长注释非常重要，说明这里选 Map 不是风格问题，而是为了解决高并发 agent 场景下的 O(N²) 拷贝和 store listener 风暴

位置：`src/utils/hooks/sessionHooks.ts:48`

注释说得非常具体：

- 用 `Map`，`.set/.delete` 不改变容器 identity
- mutator 返回 `prev` 原对象
- 这样 store 的 `Object.is(next, prev)` 会直接短路
- 不会通知 listeners
- session hooks 又不是响应式读取，而是 query loop 里 snapshot 读取

更关键的是性能分析：

- 并发 `N` 个 schema-mode agents 时
- 若用 `Record + spread`
- 每次 add 都复制增长中的 map
- 总成本 O(N²)
- 还会触发所有 store listeners

而 Map 则是：

- `.set()` O(1)
- 零 listener fires

这说明作者在这里做的是一个非常有意识的选择：

```text
optimize session hook state for high-concurrency write-heavy runtime usage,
not reactive UI ergonomics
```

这也是整份文件里最有架构味道的一点。

---

### 第七部分：`addSessionHook(...)` 与 `addFunctionHook(...)` 分成两个入口，说明虽然二者最后进入同一个 session store，但产品语义上作者刻意区分“普通 session hook”和“function hook”两种注册方式

位置：

- `addSessionHook` `src/utils/hooks/sessionHooks.ts:68`
- `addFunctionHook` `src/utils/hooks/sessionHooks.ts:93`

二者的区别：

#### `addSessionHook`
- 传入现成 `HookCommand`
- 可带 `onHookSuccess`
- 可带 `skillRoot`

#### `addFunctionHook`
- 传入 callback + errorMessage
- 构造 `FunctionHook`
- 自动生成/接收 `id`
- 默认 `timeout = 5000`
- 返回 hook ID，方便删除

这说明 function hook 被当成：

```text
special runtime-only session hook subtype
with its own lifecycle ergonomics
```

尤其“返回 ID 供移除”这点，
也说明 function hook 的典型使用场景很像：

```text
temporarily install a validation callback,
then later tear it down precisely
```

---

### 第八部分：`addFunctionHook(...)` 的默认 ID 生成方式很朴素：`Date.now() + Math.random()`，说明 function hook ID 的目标只是 session 内临时唯一，而不是稳定可重建标识

位置：`src/utils/hooks/sessionHooks.ts:105`

这说明这里对 ID 的要求不是：

- 可持久化
- 可复现
- 跨进程稳定

而只是：

```text
good enough temporary uniqueness for in-memory removal
```

这和 function hook 的定位完全一致：

```text
ephemeral, session-only, runtime-scoped
```

---

### 第九部分：`addHookToSession(...)` 是真正的内部写入口，它说明 session hook 存储的 key 不是只有 matcher，还包括 `skillRoot`；也就是说 session hooks 支持 matcher + skill-scope 的复合分桶

位置：`src/utils/hooks/sessionHooks.ts:167`

它找 existing matcher 的方式是：

```ts
m => m.matcher === matcher && m.skillRoot === skillRoot
```

这说明一个 matcher bucket 的身份不是单纯 `matcher` 字符串，
而是：

```text
matcher + skillRoot
```

也就是说 session hooks 允许：

- 同样的 matcher
- 来自不同 skill scope
- 被分成不同 matcher entry

这很关键，
因为 skill-scoped session hooks 显然需要比一般 settings hooks 更多 provenance 维度。

---

### 第十部分：`addHookToSession(...)` 在写入时不会去重，而是简单 append，说明 session hooks 的设计不是“声明式唯一配置”，而是“运行时注册表”

位置：`src/utils/hooks/sessionHooks.ts:185`

逻辑是：

- 找到现有 matcher -> 直接 `hooks: [...existingMatcher.hooks, { hook, onHookSuccess }]`
- 没找到 matcher -> 新建 matcher entry

这里没有做 `isHookEqual` 去重。

这说明 session hooks 的设计哲学更像：

```text
registration log / runtime accumulation
```

而不是：

```text
declarative normalized settings state
```

也就是说同一条 hook 可以被多次注册；
如果注册方不想重复，得自己控制。

---

### 第十一部分：`removeFunctionHook(...)` 与 `removeSessionHook(...)` 的删除策略并不相同，说明 function hooks 和普通 hooks 在身份模型上确实是两套体系

位置：

- `removeFunctionHook` `src/utils/hooks/sessionHooks.ts:120`
- `removeSessionHook` `src/utils/hooks/sessionHooks.ts:225`

区别很清楚：

#### function hook
按 `id` 删除

#### 普通 session hook
按 `isHookEqual(...)` 删除

这说明：

- function hook 的唯一稳定删除主键是 ID
- 普通 hook 的删除主键是语义身份（command/prompt/url + if + shell 等）

也就是说作者并没有强行统一两者，
而是让每类 hook 用最合理的 identity model。

---

### 第十二部分：删除逻辑里“空 matcher 桶要删掉，空 event 也要删掉”这一整套清理很重要，说明 session hook store 会主动保持紧凑结构，而不是留下空壳 bucket

位置：

- `removeFunctionHook` `src/utils/hooks/sessionHooks.ts:135`
- `removeSessionHook` `src/utils/hooks/sessionHooks.ts:240`

删除流程大致是：

1. 先对每个 matcher 过滤 hooks
2. hooks 为空的 matcher 整桶删掉
3. event 下 matcher 全空，则 event key 也删掉
4. 更新 `sessionHooks`

这说明存储层的目标不是“保留历史结构”，
而是：

```text
maintain a compact live registry
```

这样读路径 `getSessionHooks(...)` / `getSessionFunctionHooks(...)` 也更干净。

---

### 第十三部分：`SessionDerivedHookMatcher` 与 `convertToHookMatchers(...)` 很关键，说明从 session store 导出给外界的普通 hooks 视图时，会显式过滤掉 function hooks，只保留 `HookCommand[]`

位置：

- type `src/utils/hooks/sessionHooks.ts:271`
- convert `src/utils/hooks/sessionHooks.ts:282`

这里的关键注释是：

```text
Filter out function hooks - they can't be persisted to HookMatcher format
```

这说明：

- session store 里允许普通 hook 与 function hook 混存
- 但一旦要导出成类似 `HookMatcher` 的视图
- function hook 必须被剔除

因为这类视图是给：

- `hooksSettings.ts`
- `hooks.ts`
- 以及其他期待 `HookCommand[]` 的层

使用的。

所以这一层在做的是：

```text
rich runtime session store
-> persisted-like hook matcher projection
```

---

### 第十四部分：`getSessionHooks(...)` 返回 `Map<HookEvent, SessionDerivedHookMatcher[]>` 而不是 plain object，说明作者在读取层也延续了“event 稀疏、只返回存在项”的思路

位置：`src/utils/hooks/sessionHooks.ts:302`

逻辑是：

- 没 store -> `new Map()`
- 指定了 `event` -> 只返回该 event 的 matcher map
- 否则遍历 `HOOK_EVENTS`，把存在的项放进 result

这说明 `getSessionHooks(...)` 的语义不是：

```text
return a full dense event record
```

而是：

```text
return only the effective session hook entries that exist
```

这和 persisted hooks 那边 `partialRecord` 的思路其实很一致：

```text
sparse event-keyed representation
```

---

### 第十五部分：`getSessionFunctionHooks(...)` 把 function hooks 单独导出成另一份 map，说明 function hooks 在消费面上是明确走旁路的，不与普通 `HookCommand` 视图混用

位置：`src/utils/hooks/sessionHooks.ts:345`

它会：

1. 从 `SessionHookMatcher[]` 中提取 `h.hook.type === 'function'`
2. 组成：

```ts
{
  matcher: sm.matcher,
  hooks: FunctionHook[]
}
```

3. 只保留有 function hooks 的 matcher
4. 返回 `Map<HookEvent, FunctionHookMatcher[]>`

这说明 function hooks 虽然存在于同一个 session store，
但在读取/消费层是被刻意拆开的：

```text
command/prompt/agent/http session hooks
vs
function validation hooks
```

这正是因为它们：

- 不能持久化
- 不能投影成 `HookMatcher`
- 执行模型也不同

---

### 第十六部分：`getSessionHookCallback(...)` 很关键，因为它说明有些调用方需要回到 session store 原始 entry，而不是只拿 projection 后的 hook 列表；原因正是要拿到 `onHookSuccess`

位置：`src/utils/hooks/sessionHooks.ts:397`

它返回的是：

```ts
{
  hook: HookCommand | FunctionHook,
  onHookSuccess?: OnHookSuccess
} | undefined
```

查找逻辑是：

- 先找到 sessionId 的 store
- 再按 event 找 matcher entries
- matcher 命中后，用 `isHookEqual(h.hook, hook)` 找对应 hook entry

这说明 projection 层虽然方便，
但有信息会丢失：

- `onHookSuccess`
- 原始 runtime entry 结构

所以这个函数的存在说明：

```text
some runtime consumers need to rehydrate from normalized hook identity
back to the full session entry with callbacks
```

---

### 第十七部分：`matcherEntry.matcher === matcher || matcher === ''` 这个细节有点特别，说明 callback 查找时允许空 matcher 作为“通配查找模式”

位置：`src/utils/hooks/sessionHooks.ts:420`

也就是说如果传进来的 `matcher === ''`，
它会在该 event 的所有 matcher 桶里找相等 hook。

这说明 API 的设计考虑到了两种使用方式：

1. 知道具体 matcher，就精确查
2. 不知道或不关心 matcher，就跨该 event 全局找

这是一个挺实用的 runtime lookup 设计。

---

### 第十八部分：`clearSessionHooks(sessionId)` 说明 session hooks 的生命周期结束方式非常干脆：整 session 直接删掉，不做更细粒度清扫

位置：`src/utils/hooks/sessionHooks.ts:437`

逻辑非常直接：

```ts
prev.sessionHooks.delete(sessionId)
```

这说明 session hooks 的整体生命周期模型是：

```text
session starts -> accumulate temporary hooks
session ends -> drop the whole session hook store
```

而不是把它们看成需要单独持久化/迁移/恢复的状态。

这和注释里的定义完全一致：

```text
temporary, in-memory only, cleared when session ends
```

---

### 第十九部分：整份文件最核心的架构价值，是把 session hooks 设计成一套高并发友好的、session-local 的 runtime hook registry，同时把普通 HookCommand 与 runtime-only FunctionHook 放进同一个存储，再按消费场景投影成不同视图

如果把整份文件压缩，会发现它其实在做五层事：

#### 1. runtime-only hook typing
- `FunctionHookCallback`
- `FunctionHook`

#### 2. session-local storage model
- `SessionHookMatcher`
- `SessionStore`
- `SessionHooksState`

#### 3. mutation API
- `addSessionHook(...)`
- `addFunctionHook(...)`
- `removeSessionHook(...)`
- `removeFunctionHook(...)`
- `clearSessionHooks(...)`

#### 4. projection API
- `getSessionHooks(...)`
- `getSessionFunctionHooks(...)`
- `convertToHookMatchers(...)`

#### 5. callback rehydration API
- `getSessionHookCallback(...)`

所以最准确的压缩表达是：

```text
sessionHooks.ts = session hooks 的运行时注册与投影层：负责以 sessionId 为作用域维护内存态 hook store，把普通 HookCommand 与 runtime-only FunctionHook 一起管理，并按不同消费场景导出为 HookMatcher 风格视图、function-hook 视图或带 onHookSuccess 的原始 entry
```

---

### 读完这一站后，你应该抓住的 10 个事实

1. `src/utils/hooks/sessionHooks.ts` 不是 persisted hooks 配置层，而是 session-scoped hooks 的内存态注册与管理层。
2. `FunctionHook` 是一种 runtime-only session hook：直接在内存里执行 TS callback，不能持久化到 settings.json。
3. `FunctionHookCallback` 以 `messages` 为输入并返回布尔通过/阻断结果，说明 function hooks 更像本地验证器而不是通用外部 hook 执行器。
4. session hook store 的原始单元是 `SessionHookMatcher`，其中每条 hook 除了 hook 本体，还可附带 `onHookSuccess` callback。
5. `SessionHooksState` 使用 `Map<sessionId, SessionStore>` 而不是 `Record`，是为了在高并发 agent 场景下避免 O(N²) 拷贝和 store listener 风暴。
6. `addSessionHook(...)` 与 `addFunctionHook(...)` 虽然最后都写进同一个 session store，但普通 hook 和 function hook 在注册 ergonomics 上被刻意区分；function hook 会返回 ID 供精确删除。
7. `addHookToSession(...)` 不做去重，只是按 `matcher + skillRoot` 分桶后 append，说明 session hooks 更像运行时注册表而不是声明式唯一配置。
8. `removeFunctionHook(...)` 按 hook ID 删除，而 `removeSessionHook(...)` 按 `isHookEqual(...)` 的语义身份删除，体现了两类 hook 不同的 identity model。
9. `getSessionHooks(...)` 会把 session store 投影成只含 `HookCommand[]` 的 matcher map，并显式过滤掉 function hooks；`getSessionFunctionHooks(...)` 则单独导出 function hooks 视图。
10. `getSessionHookCallback(...)` 说明某些 runtime 调用方需要从标准化 hook 身份回查到原始 session entry，以拿到 `onHookSuccess` 这类 projection 后会丢失的运行时信息。

---

### 现在把第 71-72 站串起来

```text
src/utils/hooks/hooksSettings.ts
  -> 把多来源 hooks 摊平成 IndividualHookConfig，并定义展示/排序/比较辅助逻辑
src/utils/hooks/sessionHooks.ts
  -> 维护 session-scoped in-memory hook store，并把 session hooks 投影成普通 hooks 视图或 function hooks 视图
```

所以现在 hooks 子系统可以进一步压缩成：

```text
user/project/local settings hooks
  -> hooksSettings.ts normalizes persisted hooks
session-scoped in-memory hooks
  -> sessionHooks.ts stores and projects session hooks
management/config UI
  -> hooksConfigManager.ts groups normalized hooks and adds event metadata
runtime execution
  -> hooks.ts consumes snapshot/registered/session hooks and runs them
runtime protocol
  -> types/hooks.ts validates outputs
persisted schema
  -> schemas/hooks.ts validates config input
```

也就是说：

```text
hooksSettings.ts 回答“多来源 hooks 怎样被标准化和展示”
sessionHooks.ts 回答“session 内临时 hooks 怎样被注册、存储、分类和导出”
```

---

### 下一站建议

下一站最顺的是：

```text
src/utils/hooks/asyncHooks.ts
```

因为现在你已经看懂了：

- `hooks.ts` 里 async hook 是 runtime 的关键分支
- `schemas/hooks.ts` 里 `async` / `asyncRewake` 已经进入配置层
- `types/hooks.ts` 里 async JSON 也是正式协议的一部分
- `sessionHooks.ts` 又补齐了 in-memory hook 来源
- 但还差“异步 hook 本身怎样注册、追踪、完成、通知模型/任务”的专门状态层

而 `hooks.ts` 前面已经明确依赖过：

- async hook registry
- task notification
- async backgrounding

所以下一步最自然就是把 async hooks 的状态管理与通知桥接层补齐：

**`src/utils/hooks/asyncHooks.ts` 到底怎样表示异步 hook、怎样登记后台 hook、怎样在完成后触发通知或 rewake，并怎样与 hooks runtime 的 async JSON / asyncRewake 机制配合。**
