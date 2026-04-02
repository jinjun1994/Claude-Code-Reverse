## 第 40 站：`src/utils/teammateContext.ts`

### 这是什么文件

`src/utils/teammateContext.ts` 是 in-process teammate 体系里的 **运行时身份隔离容器**。

上一站 `spawnInProcess.ts` 已经看到：

```text
spawn in-process teammate 时
会同时创建：
- AppState 里的 identity plain data
- AsyncLocalStorage 用的 teammateContext runtime object
```

这一站就是后者的定义处。

它回答的问题是：

```text
同一 Node.js 进程里并发跑多个 teammate 时，
系统怎样知道“当前这一段执行属于哪个 teammate”，
并避免全局身份冲突
```

所以最准确的一句话是：

```text
teammateContext.ts = in-process teammate 的 AsyncLocalStorage 身份隔离层
```

---

### 先看它在整个 teammate 身份体系里的位置

位置：`src/utils/teammateContext.ts:1`

文件头注释直接给出三种身份来源：

- env vars：进程型 teammate（tmux 等）
- `dynamicTeamContext`：运行时加入的 process-based teammate
- `TeammateContext`：in-process teammate 的 AsyncLocalStorage

并且还明确说：

```text
teammate.ts 里的 helper 会优先检查 AsyncLocalStorage，
再检查 dynamicTeamContext，最后才看 env vars
```

这说明 `teammateContext.ts` 并不是旁路方案，而是：

```text
整个 teammate 身份解析优先级中的最高层来源
```

原因很简单：

```text
同进程并发场景里，如果不优先读 AsyncLocalStorage，
所有 in-process teammate 都会共享同一份进程级身份视图，立刻串台
```

所以这份文件虽然短，但在身份模型里地位很高。

---

### 第一部分：`TeammateContext` 和 `TeammateIdentity` 很像，但语义完全不同

位置：`src/utils/teammateContext.ts:18`

这里的 `TeammateContext` 包含：

- `agentId`
- `agentName`
- `teamName`
- `color`
- `planModeRequired`
- `parentSessionId`
- `isInProcess: true`
- `abortController`

看起来和前面 `types.ts` 里 AppState 存的 `TeammateIdentity` 很像，
但关键区别在于：

#### `TeammateIdentity`
```text
plain data
给 AppState / UI / 查询 / 持久视图使用
```

#### `TeammateContext`
```text
runtime-only
给当前调用链在执行时读取“我是谁”使用
```

而且 `TeammateContext` 还包含：

- `isInProcess: true`
- `abortController`

这两个明显是运行时概念，不适合只当静态身份字段。

所以这一站最重要的对照就是：

```text
identity = 状态镜像
context = 执行上下文
```

---

### 第二部分：`AsyncLocalStorage<TeammateContext>` 才是这个文件真正的核心

位置：`src/utils/teammateContext.ts:41`

这里创建了：

- `const teammateContextStorage = new AsyncLocalStorage<TeammateContext>()`

这一步的意义非常大，因为它解决的是 Node.js 同进程并发里最棘手的那个问题：

```text
多个 teammate 共用同一个 JS 进程，
但每条异步调用链都必须知道自己属于哪个 agent
```

如果只靠全局变量，就一定串；
如果每层函数都手传 context，又会把整套代码侵入得很重。

所以 AsyncLocalStorage 在这里扮演的是：

```text
每条异步执行链的隐式 teammate 身份槽位
```

也就是说：

```text
同进程 teammate 的隔离核心，不是线程，也不是子进程，
而是异步上下文本地存储
```

---

### 第三部分：`getTeammateContext()` 让任意下游代码都能在运行时问一句“我现在是谁”

位置：`src/utils/teammateContext.ts:43`

这个函数非常简单：

- `return teammateContextStorage.getStore()`

但它的架构意义很大。

因为这意味着任意底层 helper、tool、runtime hook，只要运行在某个 teammate 的 ALS 上下文里，就可以不显式传参地拿到：

- `agentId`
- `agentName`
- `teamName`
- `parentSessionId`
- `abortController`
- 等等

所以它相当于 in-process teammate 体系的一个全局访问口，但又不是普通全局变量，而是：

```text
按异步调用链隔离的局部全局
```

这是理解它最关键的一点。

---

### 第四部分：`runWithTeammateContext(...)` 才是真正“把一段执行变成某个 teammate”的入口

位置：`src/utils/teammateContext.ts:51`

这里的实现只有一句：

- `teammateContextStorage.run(context, fn)`

这说明 in-process teammate 的身份隔离并不是自动发生的，
而是某个上层启动器在执行 agent loop 前，必须显式包一层：

```text
runWithTeammateContext(teammateContext, () => { ...agent loop... })
```

也就是说：

```text
spawnInProcess.ts 负责“造出” teammateContext
真正的 runner 负责“激活”这份 context
```

只有进入 `run(...)` 的那一刻，这条异步链才真正拥有了某个 teammate 身份。

这也是为什么上一站说：

```text
spawn 创建的是上下文对象和注册态；
真正执行还要靠后续 runner 把它跑起来
```

---

### 第五部分：`isInProcessTeammate()` 体现了“快速判定”需求远多于“读取完整上下文”需求

位置：`src/utils/teammateContext.ts:66`

它只是：

- `teammateContextStorage.getStore() !== undefined`

注释还特别说：

```text
This is faster than getTeammateContext() !== undefined for simple checks.
```

这说明代码库里有很多地方其实并不需要知道完整身份细节，只是需要快速分支判断：

```text
当前是不是 in-process teammate？
```

比如上一站 `useInboxPoller.ts` 的 `getAgentNameToPoll(...)` 就是这种场景：

- 如果是 in-process teammate，就不要走 inbox poller

所以这个函数的存在，说明 in-process teammate 身份在运行时不是一个偶尔才用的特例，而是会频繁参与流程分叉。

---

### 第六部分：`createTeammateContext(...)` 的价值不在逻辑复杂，而在把运行时上下文 shape 固定下来

位置：`src/utils/teammateContext.ts:74`

这个工厂函数只是把传入配置展开后再补：

- `isInProcess: true`

看上去很简单，但它有两个重要作用：

#### 1. 统一 shape
不让各处自己临时拼 runtime context，避免字段漂移。

#### 2. 强制打上 `isInProcess: true`
这等于给这份上下文加了明确的语义标签：

```text
这不是任意 team context，
这是“同进程 teammate 执行态”的上下文
```

所以它虽然是薄工厂，但它在模型上把“in-process teammate runtime context”正式制度化了。

---

### 第七部分：`abortController` 在 context 里，说明取消/生命周期控制被视为执行上下文的一部分

位置：`src/utils/teammateContext.ts:37`

`abortController` 被放进 `TeammateContext`，而不是只存在 task state 里。

这说明在架构上，取消控制不只是“管理面数据”，而是：

```text
运行这条异步调用链时，当前 teammate 本身的一部分执行环境
```

这样任何运行在 teammate 上下文内的更下游逻辑，都可以在需要时访问到当前 teammate 的 lifecycle controller。

结合上一站的注释：

- 这个 controller 通常是 independent，不绑定 leader 当前 query 中断

可以看出这里表达的是：

```text
in-process teammate 的取消域和身份域是绑定在一起传播的
```

这是一个很强的运行时建模信号。

---

### 第八部分：`parentSessionId` 放进 context，说明 transcript / tracing 归属不该只靠静态状态回查

位置：`src/utils/teammateContext.ts:33`

把 `parentSessionId` 放进运行时上下文而不是只留在 AppState 里，意味着下游执行逻辑在运行过程中，随时都可能需要知道：

```text
当前 teammate 属于哪个父 session
```

这通常会影响：

- transcript 归档
- trace 事件归属
- 某些基于 session 的运行时逻辑

所以这里再次说明：

```text
TeammateContext 不是纯身份卡片，
而是运行时归属信息的即时读取入口
```

---

### 第九部分：这份文件真正体现的是“用 ALS 代替子进程隔离”的架构取舍

把它和 process-based teammate 路径对照，差别就很明显：

#### process-based teammate
```text
靠 env vars / 独立进程边界 / 独立 mailbox 自然隔离
```

#### in-process teammate
```text
没有进程边界
必须用 AsyncLocalStorage 主动模拟出“每条执行链独立身份上下文”
```

所以这份文件代表的是一个非常明确的架构选择：

```text
为了降低进程开销、提高同进程协作能力，
系统接受“共享一个 Node.js 进程”，
再用 ALS 补上执行身份隔离
```

这就是 in-process teammate 模型能够成立的基础。

---

### 第十部分：它和 `teammate.ts` 的关系，是“身份来源优先级的顶层插槽”

文件头注释已经提示：

- `teammate.ts` 里的 helper 会先查 ALS
- 再查 dynamicTeamContext
- 最后查 env vars

这说明很多上层代码未必直接 import 这个文件，
但它们仍会间接依赖这里，因为身份解析最终会优先经过它。

所以这份文件虽然短，却是：

```text
整个 teammate 身份系统里的最上游 runtime override 层
```

对于 in-process teammate 来说，这一层优先级如果不存在，整个身份模型都会塌。

---

### 元

问题：**这一站真正想解决的架构问题是什么？**

回答：这一站用 AsyncLocalStorage 为 in-process teammate 提供身份隔离，避免同一 Node.js 进程中多个 teammate 共享同一身份视图。它是同进程并发协作里最关键隔离层之一。

### 反

问题：**如果把这一站的设计反过来，会发生什么？**

回答：如果不优先依赖 ALS，而只是靠进程级 env 或全局变量，多 teammate 并发时一定会串台。身份一旦串掉，权限、日志和消息归属都会跟着失真。

### 空

问题：**跳出当前文件名，这一站背后更大的问题是什么？**

回答：更大的问题是，轻量级隔离能否替代多进程隔离。ALS 给出的答案是：在某些场景可以，但前提是身份上下文管理必须严密。


### 读完这一站后，你应该抓住的 8 个事实

1. `teammateContext.ts` 提供的是 in-process teammate 的 AsyncLocalStorage 身份隔离层，而不是普通状态定义文件。
2. 在 teammate 身份来源优先级里，ALS 上下文先于 dynamicTeamContext 和 env vars，因此它是 in-process teammate 的最高优先级身份来源。
3. `TeammateContext` 和 AppState 里的 `TeammateIdentity` 字段相似，但前者是运行时执行上下文，后者是状态镜像。
4. `AsyncLocalStorage<TeammateContext>` 解决的是“多个 teammate 共用同一 Node.js 进程时，如何让每条异步链都知道自己是谁”的核心隔离问题。
5. `getTeammateContext()` 提供了按异步调用链隔离的全局读取口，让下游逻辑不必层层手传 teammate 身份。
6. `runWithTeammateContext(...)` 才是真正激活 teammate 身份的入口：只有包在这层 ALS `run()` 里的执行链，才拥有该 teammate 的运行时身份。
7. `isInProcessTeammate()` 说明“当前是否在 in-process teammate 上下文中”是一个高频分支条件。
8. `abortController`、`parentSessionId` 等字段被放进 context，说明取消域和会话归属也被视为 teammate 运行时上下文的一部分。

---

### 现在把第 39-40 站串起来

```text
spawnInProcess.ts
  -> 创建 teammate identity、teammateContext、abortController，并把 task 注册进 AppState
teammateContext.ts
  -> 定义并承载 in-process teammate 的 AsyncLocalStorage 运行时身份隔离层
```

所以 in-process teammate 的创建链现在可以先压缩成：

```text
spawn
  -> create plain identity for AppState
  -> create runtime teammateContext for ALS
  -> later run agent loop inside runWithTeammateContext(...)
```

---

### 下一站建议

下一站最顺的是：

```text
src/utils/teammate.ts
```

因为这一站你已经看懂了：

- in-process teammate 的 runtime identity 怎样靠 ALS 隔离
- 为什么 `teammate.ts` 要优先读取 AsyncLocalStorage

下一步最自然就是去看统一身份门面：

**`teammate.ts` 是怎样把 ALS context、dynamicTeamContext、env vars 这三种 teammate 身份来源统一收口成 `getAgentName()` / `isTeammate()` / `isTeamLead()` / `isPlanModeRequired()` 等通用判断接口的。**
