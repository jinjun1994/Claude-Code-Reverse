## 第 41 站：`src/utils/teammate.ts`

### 这是什么文件

`src/utils/teammate.ts` 是 swarm 体系里的 **统一 teammate 身份门面 + 同进程 teammate 状态辅助器**。

上一站 `teammateContext.ts` 已经看懂：

```text
in-process teammate 的身份隔离靠 AsyncLocalStorage
```

但系统里不可能让所有上层代码都自己去分别判断：

- 我是不是 in-process teammate
- 还是 tmux/process-based teammate
- 还是 team lead
- 有没有 team name / agent name / plan mode requirement

所以这一站解决的是：

```text
如何把 ALS context、dynamic runtime context、以及 teamContext / env 等不同来源，
统一收口成一套简单可复用的 teammate helper API
```

所以最准确的一句话是：

```text
teammate.ts = swarm 身份来源的统一解析门面
```

---

### 先看它在身份体系里的位置

位置：`src/utils/teammate.ts:1`

文件头注释已经写明优先级：

1. `AsyncLocalStorage`（in-process teammate）
2. `dynamicTeamContext`（tmux/process-based teammate via CLI args）

这说明这份文件并不是再定义一种新身份来源，
而是在做：

```text
把多个 teammate 身份来源整合成统一读取顺序
```

而且它还直接 re-export 了：

- `createTeammateContext`
- `getTeammateContext`
- `isInProcessTeammate`
- `runWithTeammateContext`

这进一步说明它的定位就是：

```text
上层尽量只依赖 teammate.ts，
不用关心底层到底是 ALS 还是 dynamic runtime context
```

所以它本质上是一个 facade。

---

### 第一部分：`dynamicTeamContext` 是 process-based teammate 的运行时身份槽位

位置：`src/utils/teammate.ts:40`

这里维护了一个模块级变量：

- `dynamicTeamContext`

字段包括：

- `agentId`
- `agentName`
- `teamName`
- `color`
- `planModeRequired`
- `parentSessionId`

并提供：

- `setDynamicTeamContext(...)`
- `clearDynamicTeamContext()`
- `getDynamicTeamContext()`

这说明对于 process-based teammate，系统并不总是只靠静态 env vars，
而是允许在运行时动态加入/退出团队，并把这些身份信息装进一份内存态 context。

所以这里对应的是：

```text
tmux / process teammate 的 runtime identity overlay
```

也就是说，这份文件整合的其实是：

```text
ALS context（同进程）
+
dynamicTeamContext（跨进程/运行时加入）
```

两种运行时身份来源。

---

### 第二部分：`getParentSessionId()` 暴露了 teammate 归属关系的统一读取口

位置：`src/utils/teammate.ts:29`

这里的优先顺序是：

- 先 `getTeammateContext()`
- 再 `dynamicTeamContext?.parentSessionId`

这说明对上层来说，“这个 teammate 属于哪个父 session”不应该区分它到底是：

- in-process
- 还是 tmux teammate

正确调用方式就是统一问：

- `getParentSessionId()`

所以它做的事是：

```text
把执行归属信息统一抽象成一个跨后端一致的查询接口
```

这类 helper 的价值就在于：

```text
上层逻辑不必知道 teammate 是通过哪种 transport/backend 存在的
```

---

### 第三部分：`getAgentId()` / `getAgentName()` / `getTeamName()` 是最核心的身份读取三件套

位置：

- `getAgentId()` `src/utils/teammate.ts:83`
- `getAgentName()` `src/utils/teammate.ts:94`
- `getTeamName()` `src/utils/teammate.ts:104`

它们的共同模式都是：

- 先看 ALS `getTeammateContext()`
- 再看 `dynamicTeamContext`
- `getTeamName()` 还允许最后回落到传入的 `teamContext`

这说明该文件的最核心职责就是：

```text
把“我是哪个 teammate / 属于哪个 team”统一转成简单 getter
```

其中 `getTeamName(teamContext?)` 最有意思，因为它显式照顾了 leader 场景：

```text
leader 通常没有 dynamicTeamContext，
但 AppState.teamContext 里仍然知道当前 teamName
```

所以这个函数不是单纯 teammate-only helper，
它实际上是整个 swarm 运行时关于“当前 team 名字”的统一入口之一。

---

### 第四部分：`isTeammate()` 的逻辑揭示了系统如何区分“协作者”与“主会话”

位置：`src/utils/teammate.ts:120`

它的逻辑是：

- 只要有 in-process ALS context -> true
- 否则 tmux/process teammate 需要同时有：
  - `agentId`
  - `teamName`

这说明“是否是 teammate”在这里不是模糊概念，而是明确受身份完整性约束的：

```text
必须已经拥有一份可用的 teammate 身份
```

这可以避免某些半初始化状态误判成 teammate。

同时也说明：

```text
team lead 不算 teammate
team lead 是 team runtime 中的另一种角色
```

这为后面的 `isTeamLead()` 留出了明确分工。

---

### 第五部分：`getTeammateColor()` / `isPlanModeRequired()` 说明颜色与 plan discipline 也被纳入统一身份门面

位置：

- `getTeammateColor()` `src/utils/teammate.ts:133`
- `isPlanModeRequired()` `src/utils/teammate.ts:144`

`getTeammateColor()` 很直观：

- ALS 优先
- 否则 dynamicTeamContext

但 `isPlanModeRequired()` 非常有意思：

优先级是：

1. ALS
2. dynamicTeamContext
3. 最后回落 env var `CLAUDE_CODE_PLAN_MODE_REQUIRED`

这说明 plan mode requirement 被建模成 teammate 身份语义的一部分，
而不是单纯的 UI 设置或一次性运行参数。

尤其 env var fallback 说明：

```text
旧路径 / 非动态 teammate 启动方式下，仍然保留通过环境变量传递这项约束的兼容通道
```

所以这个函数把：

- 新的 runtime identity 模型
- 老的 process/env 模型

都统一兼容起来了。

---

### 第六部分：`isTeamLead()` 不是简单“不是 teammate 就是 leader”，而是一个更细的角色判定

位置：`src/utils/teammate.ts:158`

这里的逻辑是：

1. 必须有 `teamContext.leadAgentId`
2. 如果 `myAgentId === leadAgentId` -> true
3. 否则，如果 `!myAgentId` -> true（backwards compat）
4. 其他情况 -> false

这说明系统并没有偷懒写成：

```text
!isTeammate() => isLeader
```

而是明确区分：

- 这是不是一个 team runtime
- 当前 agentId 是否匹配 leadAgentId
- 或者是否属于早期兼容路径（没有 agentId 的原始主会话）

所以 `isTeamLead()` 真正表达的是：

```text
当前会话是否拥有团队的主控制权
```

而不是仅仅“不是 teammate”。

这在 inbox poller、permission routing、mode broadcasting 这些场景里都非常关键。

---

### 第七部分：后半段几个 helper 暴露了 in-process teammate 的“等待与收尾”协作原语

位置：

- `hasActiveInProcessTeammates(...)` `src/utils/teammate.ts:200`
- `hasWorkingInProcessTeammates(...)` `src/utils/teammate.ts:215`
- `waitForTeammatesToBecomeIdle(...)` `src/utils/teammate.ts:233`

这说明 `teammate.ts` 不只是身份门面，还顺手提供了一组与 in-process teammate 生命周期协作相关的辅助函数。

#### `hasActiveInProcessTeammates(...)`
只看：

- `task.type === 'in_process_teammate'`
- `task.status === 'running'`

对应的问题是：

```text
系统里还有没有活着的同进程 teammate
```

#### `hasWorkingInProcessTeammates(...)`
进一步要求：

- `!task.isIdle`

对应的问题是：

```text
它们是不是还在真正干活，而不是已经空闲等输入
```

这两个 helper 把“活着”和“在工作”清楚地区分开了。

---

### 第八部分：`waitForTeammatesToBecomeIdle(...)` 说明 in-process teammate 的等待机制不是轮询，而是 callback 挂接

位置：`src/utils/teammate.ts:233`

这个函数的流程很值得细看：

1. 先扫描 `appState.tasks`
2. 找出所有 `running && !isIdle` 的 in-process teammate
3. 如果没有，直接 `Promise.resolve()`
4. 否则创建一个 promise
5. 通过 `setAppState(...)` 给每个 working teammate 的 `onIdleCallbacks` 追加同一个 `onIdle`
6. 如果注册时发现它已经 idle，就立即 `onIdle()`，防 race
7. 所有 callback 都触发后 resolve

这说明同进程 teammate 的“等它忙完”机制，不是：

```text
while (!idle) 轮询检查
```

而是：

```text
直接把 continuation 挂到 task state 的 onIdleCallbacks 上
等 teammate 自己在变 idle 时回调你
```

这是一种非常典型的同进程共享状态协作手法。

也再次印证了：

```text
in-process teammate 路径的核心不是 mailbox，而是共享 AppState + callback 协作
```

---

### 第九部分：`waitForTeammatesToBecomeIdle(...)` 还专门处理了 snapshot 与注册之间的 race

位置：`src/utils/teammate.ts:270`

注释直接说明：

```text
Check current isIdle state to handle race where teammate became idle
between our initial snapshot and this callback registration
```

这说明作者很明确地意识到这里有一个经典并发窗口：

- 你先扫描出 working teammate
- 然后准备挂 callback
- 但在这两步之间，它可能已经变 idle 了

如果不补这个检查，就可能永远等不到回调。

所以这里的真正语义是：

```text
把“当前状态快照”和“未来状态回调”结合起来，保证等待不会漏事件
```

这是一个很成熟的本地并发同步设计。

---

### 第十部分：这份文件把“身份解析”和“同进程协作辅助”放在一起，说明它是 swarm runtime 的高层门面，而不是单纯 identity 文件

如果只看前半段，很容易觉得它只是：

- `getAgentId()`
- `getAgentName()`
- `isTeammate()`
- `isTeamLead()`

之类的身份 helper。

但后半段又出现：

- active teammates
- working teammates
- wait until idle

这说明它的真实定位更宽：

```text
它是“所有上层想快速问一句当前 swarm 角色/状态”的统一高层门面
```

也因此它被广泛引用在：

- inbox poller
- permission 路由
- shutdown/wait logic
- 其他 swarm runtime 分支判断

所以把它记成“身份解析 facade + in-process teammate wait helpers”最准确。

---

### 读完这一站后，你应该抓住的 9 个事实

1. `teammate.ts` 不是新的身份来源，而是把 ALS teammate context、dynamic runtime context、teamContext/env 统一收口成一组通用 helper 的门面层。
2. 该文件 re-export 了 `teammateContext.ts` 的 ALS 能力，说明上层应尽量依赖这一个 facade，而不是分别面向多个身份来源编码。
3. `dynamicTeamContext` 是 process-based teammate 在运行时加入团队后的身份槽位，与 ALS 一起构成主要 runtime identity source。
4. `getAgentId()` / `getAgentName()` / `getTeamName()` 是最核心的统一身份读取接口，其中 `getTeamName()` 还显式照顾了 leader 的 `teamContext` 回退场景。
5. `isTeammate()` 与 `isTeamLead()` 明确区分了 teammate 身份和 lead 控制权，后者不是简单的 `!isTeammate()`。
6. `isPlanModeRequired()` 统一兼容 ALS、dynamic context 与 env var 三种来源，说明 plan discipline 已被纳入身份语义层。
7. `hasActiveInProcessTeammates()` 与 `hasWorkingInProcessTeammates()` 区分了“还活着”与“还在真正工作”两种不同状态。
8. `waitForTeammatesToBecomeIdle()` 通过 `onIdleCallbacks` 实现 callback-based 等待，而不是轮询，这正是 in-process teammate 路径共享状态模型的体现。
9. 该等待逻辑还专门处理了 snapshot 与回调注册之间的 race，说明它不仅是工具函数，还封装了关键的并发正确性细节。

---

### 现在把第 40-41 站串起来

```text
teammateContext.ts
  -> 定义 in-process teammate 的 AsyncLocalStorage 运行时身份隔离层
teammate.ts
  -> 把 ALS、dynamic context、teamContext/env 等身份来源统一收口为高层 teammate helper，并补充 in-process teammate 的等待协作原语
```

所以现在 teammate 身份链可以压缩成：

```text
runtime identity source
  -> ALS (in-process)
  -> dynamicTeamContext (process-based runtime join)
  -> teamContext/env fallback

public facade
  -> getAgentName / getTeamName / isTeammate / isTeamLead / isPlanModeRequired / waitForTeammatesToBecomeIdle
```

---

### 下一站建议

下一站最顺的是：

```text
src/utils/swarm/teamHelpers.ts
```

因为现在你已经看懂了：

- teammate 身份怎样统一解析
- in-process teammate 怎样等待变 idle
- lead / teammate 角色怎样区分

下一步最自然就是回到 team 元数据本体：

**team file 本身长什么样，成员是怎样加入/删除/更新 mode 的，以及 `removeMemberByAgentId` / `setMemberMode` / `readTeamFileAsync` 这些上层大量依赖的团队元数据操作到底怎样实现。**
