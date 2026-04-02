## 第 62 站：`src/hooks/toolPermission/handlers/coordinatorHandler.ts`

### 这是什么文件

`src/hooks/toolPermission/handlers/coordinatorHandler.ts` 是工具权限 ask-path 里的 **自动化审批前置协调器**。

上一站 `interactiveHandler.ts` 已经看懂：

- ask-mode 最终兜底时，会进入 queue-driven 的本地交互审批链
- 本地用户、bridge、channel、hooks、classifier 会在 interactive dialog 阶段并发竞态
- 但还差 ask-path 更前面那一层：
  - 为什么有些 ask 不会立刻弹框
  - hooks / classifier 为什么有时会先被完整跑完
  - coordinator 分支和 interactive 分支到底怎么分工

这一站回答的是：

```text
当权限结果是 ask，且系统配置要求“先跑自动化检查再打断用户”时，
coordinatorHandler 怎样顺序执行 hooks 与 classifier，
并决定这次请求是提前收束，还是继续落到 interactive dialog
```

所以最准确的一句话是：

```text
coordinatorHandler.ts = ask-path 的自动化审批前置收敛器
```

---

### 先看它的整体定位

位置：`src/hooks/toolPermission/handlers/coordinatorHandler.ts:1`

这个文件不是：

- permission rule evaluator
- interactive dialog handler
- swarm worker 上抛 handler
- classifier / hooks 实现本体

它的职责是：

1. 在特定 ask-mode 分支里先跑 hooks
2. hooks 没解决时再跑 classifier
3. 如果自动化检查得出结论，直接返回标准 `PermissionDecision`
4. 如果都没解决，返回 `null` 让上层继续进入 interactive dialog
5. 如果自动化检查本身异常，也不要打断整条权限链，而是降级到人工确认

所以它本质上是一个：

```text
automated pre-dialog permission coordinator
```

也就是说它位于：

```text
ask result
  -> try automated resolution first
  -> unresolved ? fall through to dialog
```

这正是它名字里 `coordinator` 的含义。

---

### 第一部分：文件头注释已经明确说明它只服务 coordinator worker 场景，而且核心语义是“在 interactive dialog 之前顺序 await 自动检查”

位置：`src/hooks/toolPermission/handlers/coordinatorHandler.ts:16`

注释里直接写了三件事：

- `For coordinator workers`
- automated checks `awaited sequentially`
- unresolved 时 `fall through to the interactive dialog`

这说明它不是一般性的“多 racer 竞态入口”。

它的语义恰恰和上一站 `interactiveHandler.ts` 相反：

#### interactiveHandler
dialog 已出现，hooks / classifier 还可以在后台继续竞态

#### coordinatorHandler
dialog 还没出现，先把 hooks / classifier 跑完再说

所以这份文件代表的是 ask-path 的另一种 UX/控制流策略：

```text
prefer pre-resolution before showing any approval UI
```

而不是“先把 dialog 弹出来，再让后台检查补刀”。

---

### 第二部分：`CoordinatorPermissionParams` 很小，说明这个 handler 自己不持有复杂状态；它只是消费 `PermissionContext` 提供的共享能力来做一层顺序编排

位置：`src/hooks/toolPermission/handlers/coordinatorHandler.ts:8`

参数只有：

- `ctx`
- `pendingClassifierCheck`
- `updatedInput`
- `suggestions`
- `permissionMode`

这里没有：

- queue callback
- bridge/channel callback
- resolve function
- timer / listener / cleanup handle

这和 `interactiveHandler.ts` 形成非常鲜明的对比。

说明 coordinator handler 的设计目标不是维护复杂的等待态，
而是：

```text
take existing context
-> run a short automated pipeline
-> either return a decision or decline ownership
```

也就是说它是一个很薄的 orchestration layer，
复杂副作用全部下沉给：

- `ctx.runHooks(...)`
- `ctx.tryClassifier?.(...)`

---

### 第三部分：`const { ctx, updatedInput, suggestions, permissionMode } = params` 但没有提前取 `pendingClassifierCheck`，说明 classifier 在这里是可选的第二阶段，而 hooks 才是固定第一阶段

位置：`src/hooks/toolPermission/handlers/coordinatorHandler.ts:29`

一进函数先解构：

- `ctx`
- `updatedInput`
- `suggestions`
- `permissionMode`

`pendingClassifierCheck` 没被提出来，而是后面在 classifier 分支里通过 `params.pendingClassifierCheck` 使用。

这虽然是个小细节，但很能说明作者脑中的优先级：

```text
hooks are the default first automated lane
classifier is a conditional second lane
```

因为：

- hooks 是通用 permission automation 能力
- classifier 只在 feature 打开且工具/输入适合时才有意义

所以 ask-path coordinator 分支的顺序不是“谁都行”，而是明确的：

1. hooks first
2. classifier second
3. dialog last

---

### 第四部分：`ctx.runHooks(...)` 被放在第一位，而且注释写的是 `fast, local`，说明作者把 hooks 看作最轻量、最应该优先尝试的自动化审批源

位置：`src/hooks/toolPermission/handlers/coordinatorHandler.ts:32`

这里第一步就是：

```ts
const hookResult = await ctx.runHooks(
  permissionMode,
  suggestions,
  updatedInput,
)
if (hookResult) return hookResult
```

注释非常直接：

```text
Try permission hooks first (fast, local)
```

这说明 hooks 的定位是：

```text
cheapest automated permission resolution path
```

也就是说在 coordinator 场景下，
系统假定如果有自动化审批能先消化掉 ask，
最值得先跑的是 hook，而不是 classifier。

原因也很清楚：

- hooks 通常是本地逻辑 / 本地脚本
- 不一定需要额外推理成本
- 能直接产出 allow / deny / interrupt

所以这一步体现的是 ask-path 里的一个明确优化原则：

```text
use the fastest deterministic automation before slower inference-based checks
```

---

### 第五部分：`if (hookResult) return hookResult` 说明 hook 在这里是完整终局源，不是“给 classifier 提示一下”的辅助步骤

位置：`src/hooks/toolPermission/handlers/coordinatorHandler.ts:38`

一旦 hook 返回结果，立刻整个 handler 结束。

这里没有：

- 合并 hook result 与 classifier result
- 再做二次确认
- 继续进入下一阶段

这再次说明在架构上：

```text
hook decision is authoritative
```

也就是说当 hook 已经给出 `PermissionDecision` 时，
coordinator handler 不再做任何附加裁决。

所以这个文件虽然很短，
但它其实很清晰地重申了 PermissionContext 那一站的一个结论：

```text
PermissionRequest hooks are first-class decision sources
```

在这里它们不是 advisory signal，
而是能直接结束 ask-path 的正式审批源。

---

### 第六部分：classifier 被明确放在第二阶段，而且注释写的是 `slow, inference -- bash only`，说明它在 coordinator 分支里是更贵、更窄的备用自动化通道

位置：`src/hooks/toolPermission/handlers/coordinatorHandler.ts:40`

第二步是：

```ts
const classifierResult = feature('BASH_CLASSIFIER')
  ? await ctx.tryClassifier?.(params.pendingClassifierCheck, updatedInput)
  : null
if (classifierResult) {
  return classifierResult
}
```

注释直接说：

```text
Try classifier (slow, inference -- bash only)
```

这说明作者对 classifier 的产品定位很明确：

- 它不是所有工具通用
- 它比 hooks 更贵
- 它依赖推理/分类能力
- 但在 bash ask-path 上仍值得作为第二层自动化兜底

因此 coordinator 分支的结构不是“所有自动化并行跑”，
而是：

```text
run low-cost automation first
-> only then pay for inference-based automation
```

这和上一站 `interactiveHandler.ts` 里 classifier 作为后台 racer 的角色不同。

在这里 classifier 不是为了和用户抢速度，
而是被当成：

```text
a final automated chance to avoid showing the dialog at all
```

---

### 第七部分：`ctx.tryClassifier?.(...)` 用的是可选链，说明 classifier 在整个权限系统里始终被设计成“可插拔能力”，而不是 coordinator 分支的硬依赖

位置：`src/hooks/toolPermission/handlers/coordinatorHandler.ts:41`

这里有两层可选性：

1. `feature('BASH_CLASSIFIER')`
2. `ctx.tryClassifier?.(...)`

这表示 coordinator handler 对 classifier 的态度很克制：

```text
use classifier if available,
otherwise continue gracefully
```

也就是说这条 ask-path 自动化收敛分支不能依赖 classifier 一定存在。

这样设计的好处是：

- feature 关掉时逻辑仍然成立
- 非 bash / 非 classifier 场景仍能复用整个 coordinator 分支
- handler 本身不需要知道 classifier 内部如何实现

所以这里体现的是：

```text
coordinator flow depends on capability presence, not concrete implementation assumptions
```

---

### 第八部分：整个主流程没有任何 queue / resolve / callback 注册，说明 coordinator handler 的返回风格是“同步化的异步函数结果”，而不是 interactive 那种“建立未来某时刻 settle 的竞态系统”

位置：`src/hooks/toolPermission/handlers/coordinatorHandler.ts:26`

这个函数虽然是 `async`，
但控制流非常单纯：

- await hooks
- await classifier
- return decision or null

没有：

- `new Promise(...)`
- `createResolveOnce(...)`
- queue push/remove/update
- onAllow/onReject/onAbort wiring
- background racers

这说明它的抽象层级是：

```text
short-lived preflight check
```

而不是：

```text
open-ended approval session
```

所以可以把 coordinator handler 和 interactive handler ��比成：

#### coordinatorHandler
前置自动化过滤器

#### interactiveHandler
真正的审批会话协调器

两者共同组成 ask-path，
但承担的是完全不同时间尺度的工作。

---

### 第九部分：`catch` 逻辑非常重要——自动化检查失败不会把 ask-path 打死，而是刻意降级到人工 dialog，说明自动化审批在这里是“优化路径”，不是可靠性单点

位置：`src/hooks/toolPermission/handlers/coordinatorHandler.ts:47`

异常处理里注释写得很清楚：

```text
If automated checks fail unexpectedly, fall through to show the dialog
so the user can decide manually.
```

这说明作者的容错策略是：

```text
automation failure should degrade to human approval,
not to permission failure
```

这点非常关键。

因为 ask-path 的目标不是“证明自动化链绝对可靠”，
而是：

- 能自动解决最好
- 自动化出错也别拦死用户
- 最终仍保留人工审批兜底

所以 coordinator handler 在整个权限系统里的可靠性哲学是：

```text
automated checks are opportunistic accelerators,
manual approval remains the safety net
```

---

### 第十部分：对 `Error` 和非 `Error` throw 分开处理，而且注释强调故意不用 `toError()`，说明作者在意日志语义的可追踪性，而不是只图统一包装

位置：`src/hooks/toolPermission/handlers/coordinatorHandler.ts:49`

这里逻辑是：

```ts
if (error instanceof Error) {
  logError(error)
} else {
  logError(new Error(`Automated permission check failed: ${String(error)}`))
}
```

注释特别说明：

- non-Error throws 需要 context prefix
- 故意不用 `toError()`
- 因为 `toError()` 会丢掉这个 prefix

这说明作者并不是简单想“把任何 throw 转成 Error”。

真正关心的是：

```text
when logs are inspected later,
it should be obvious this failure came from automated permission checks
```

所以这里体现的不是通用异常规范化，
而是：

```text
context-preserving failure attribution
```

在这种自动化 fallback 场景下非常有价值，
因为日志需要帮助开发者判断：

- 是 hooks/classifier 自己崩了
- 还是 interactive dialog / 其他 ask-path 部分出问题

---

### 第十一部分：末尾注释 `Hooks already ran, classifier already consumed` 很关键，说明返回 `null` 并不代表“什么都没发生”，而是“自动化机会已经被用尽，可以放心进入 dialog”

位置：`src/hooks/toolPermission/handlers/coordinatorHandler.ts:59`

最后返回 `null` 前，注释写：

```text
Neither resolved (or checks failed) -- fall through to dialog below.
Hooks already ran, classifier already consumed.
```

这说明这里的 `null` 语义不是：

```text
I didn't handle this
```

而是：

```text
I handled the pre-dialog automation phase,
but it produced no final decision
```

这是很重要的语义区别。

因为它意味着上层后续进入 `interactiveHandler` 时，
应该把它理解为：

- 不是第一次开始处理 ask
- 而是自动化预处理阶段已经结束
- 剩下的是纯人工/交互审批阶段

也就是说 coordinator handler 是 ask-path 的真实前半段，
不是可有可无的小插件。

---

### 第十二部分：这份文件最核心的架构价值，是把 ask-path 切成“pre-dialog automation”和“interactive fallback”两个阶段，从而让系统能根据模式切换不同的用户打断策略

如果把这份文件放回前几站的上下文里，结构就很清楚了：

#### `useCanUseTool.tsx`
决定 ask 时要不要先走 coordinator 分支

#### `coordinatorHandler.ts`
如果要先跑自动化检查，就顺序尝试 hooks / classifier

#### `interactiveHandler.ts`
如果自动化没解决，再进入真正的交互审批会话

这说明 ask-path 其实被拆成了两个明显不同的阶段：

```text
Phase 1: pre-dialog automated checks
Phase 2: interactive approval fallback
```

而 coordinator handler 正是 Phase 1 的实装点。

它的架构意义不在“代码量”，
而在于把用户体验策略显式编码进了 handler 边界：

- 某些模式优先少打扰用户
- 某些模式则可以尽快显示 dialog，再让自动化继续后台竞态

这就是它真正的价值。

---

### 第十三部分：它和 `interactiveHandler.ts` 的关系，恰好体现 ask-path 中两种自动化策略的差别：前置 await vs 显示后竞态

把第 61 站和第 62 站并排看，差异就非常鲜明：

#### `coordinatorHandler.ts`
- dialog 还没出现
- hooks 先跑
- classifier 再跑
- 都是顺序 await
- 有结果就直接结束
- 无结果才允许进入 dialog

#### `interactiveHandler.ts`
- dialog 已经出现
- hooks / classifier 在后台继续跑
- 本地用户 / bridge / channel / hook / classifier 并发竞态
- 谁先赢谁结束

所以二者不是重复逻辑，
而是在同一 ask-path 里承担不同策略：

```text
coordinatorHandler = resolve before interrupting the user
interactiveHandler = keep resolving while the user can already act
```

这正是你前面在 `useCanUseTool.tsx` 看到的
`awaitAutomatedChecksBeforeDialog` 这个开关的真正落点。

---

### 元

问题：**这一站真正想解决的架构问题是什么？**

回答：这一站要解释的是，为什么 ask 不一定立刻打断用户，而要先跑一轮自动化审批协调。它把 hooks 和 classifier 放在对话框之前，尝试先把可自动收束的问题消掉。

### 反

问题：**如果把这一站的设计反过来，会发生什么？**

回答：如果反过来一律先弹框，系统会把本可自动解决的请求全部推给用户。这样不仅体验更差，也让 hooks 和 classifier 失去作为前置过滤器的架构意义。

### 空

问题：**跳出当前文件名，这一站背后更大的问题是什么？**

回答：更大的问题，是人在权限链路中应该介入多早。`coordinatorHandler` 展示的是一种更成熟的答案：先机器收敛，剩下的再打断人。

### 读完这一站后，你应该抓住的 10 个事实

1. `coordinatorHandler.ts` 是 ask-path 里的自动化审批前置协调器，不负责 UI，不负责规则求值，而是负责在 dialog 之前先跑自动化检查。
2. 它只消费很薄的一组输入：`ctx`、`updatedInput`、`suggestions`、`permissionMode`、`pendingClassifierCheck`，说明它是一个轻量 orchestration 层。
3. 它的自动化顺序是固定的：先 hooks，再 classifier，最后才可能 fall through 到 interactive dialog。
4. hooks 在这里被当作 `fast, local` 的首选自动化审批源，一旦有结果就立刻返回，说明 hook decision 是完整终局源。
5. classifier 在这里是第二阶段、成本更高、范围更窄的备用自动化通道，定位是 `slow, inference -- bash only`。
6. `feature('BASH_CLASSIFIER')` 加 `ctx.tryClassifier?.(...)` 说明 classifier 仍是可选能力，coordinator 分支不会把它当成硬依赖。
7. 这个 handler 没有 queue、resolve-once、callbacks 或后台 racers，说明它实现的是短生命周期 preflight，而不是长生命周期审批会话。
8. 自动化检查出错时不会导致 ask-path 失败，而是记录日志后降级到人工 dialog，体现了“自动化是优化路径，人工审批是可靠兜底”。
9. 对非 `Error` throw 的特殊包装说明作者很在意日志里的失败归因上下文，尤其要保留 `Automated permission check failed` 这种明确前缀。
10. `return null` 的真实语义不是“没处理”，而是“前置自动化阶段已经跑完但没有结论，现在可以进入 interactive fallback”。

---

### 现在把第 61-62 站串起来

```text
interactiveHandler.ts
  -> ask-path 的本地交互审批与多审批源竞态协调器
coordinatorHandler.ts
  -> ask-path 的自动化前置阶段：先顺序跑 hooks / classifier，再决定是否需要进入 interactive dialog
```

所以现在 ask-path 可以进一步压缩成：

```text
permission result = ask
  -> if mode requires pre-dialog automation:
       run coordinatorHandler
         -> hooks first
         -> classifier second
         -> resolved ? done
         -> unresolved ? continue
  -> else / or still unresolved:
       run interactiveHandler
         -> queue item + local/remote/hook/classifier race
```

也就是说：

```text
interactiveHandler.ts 回答“dialog 出现后，多个审批源怎样竞争并收尾”
coordinatorHandler.ts 回答“dialog 出现前，系统怎样先用自动化检查尽量把 ask 请求提前收束”
```

---

### 下一站建议

下一站最顺的是：

```text
src/utils/permissions/permissions.ts
```

因为现在你已经看懂了：

- `useCanUseTool.tsx` 怎样调度 ask-path
- `PermissionContext.ts` 提供了哪些共享能力
- `coordinatorHandler.ts` 怎样做前置自动化检查
- `interactiveHandler.ts` 怎样做最终交互审批与竞态收尾
- 但 ask / allow / deny 这些 `PermissionDecision` 最底层到底怎样算出来，还没真正拆开：
  - `hasPermissionsToUseTool(...)` 怎样判定
  - permission mode / rule / tool 特性怎样影响结果
  - `pendingClassifierCheck` / `suggestions` / `blockedPath` 这些 ask 附加上下文从哪里来

下一步最自然就是回到更底层的判定器：

**`permissions.ts` 到底怎样把工具、输入、permission mode、规则与上下文综合成 allow / deny / ask 三种权限结果，并为后续 coordinator / swarm / interactive handlers 提供它们消费的结构化判定结果。**
