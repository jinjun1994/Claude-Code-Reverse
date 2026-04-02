## 第 39 站：`src/utils/swarm/spawnInProcess.ts`

### 这是什么文件

`src/utils/swarm/spawnInProcess.ts` 是 in-process teammate 体系里的 **创建器 + 杀死器**。

上一站 `InProcessTeammateTask.tsx` 已经看懂：

```text
in-process teammate 作为一种 Task 存在于 AppState 里
可以 request shutdown / kill / 注入消息 / 查询运行实例
```

而这一站要解决的是：

```text
这个任务实体最初是怎样被 spawn 出来的
怎样生成身份
怎样创建 AsyncLocalStorage context
怎样注册到 AppState
怎样清理与 kill
```

所以最准确的一句话是：

```text
spawnInProcess.ts = in-process teammate 的启动注册器与终止回收器
```

---

### 先看它的整体定位

位置：`src/utils/swarm/spawnInProcess.ts:1`

文件头注释已经把边界划得很清楚：

它负责：

1. 创建 `TeammateContext`
2. 创建 `AbortController`
3. 在 AppState 里注册 `InProcessTeammateTaskState`
4. 返回 spawn 结果给 backend

它**不**负责：

- 实际 agent execution loop

注释甚至直接点明：

```text
The actual agent execution loop is handled by InProcessTeammateTask component
```

所以第一结论是：

```text
spawnInProcess.ts 管“生”和“死”，不管完整执行过程本身
```

---

### 第一部分：`SpawnContext` 很小，说明 spawn 阶段只依赖最少的宿主能力

位置：`src/utils/swarm/spawnInProcess.ts:47`

`SpawnContext` 只有：

- `setAppState`
- 可选 `toolUseId`

这说明 spawn 这一步的依赖被刻意压得很薄：

```text
只要能注册状态、可选关联到当前 toolUse，
就足以把一个 in-process teammate 生出来
```

它不要求完整 `ToolUseContext`、不要求 query loop 全量依赖。

所以这里体现的是一种很干净的创建边界：

```text
spawn 是状态注册动作，不是完整运行时绑定动作
```

---

### 第二部分：`InProcessSpawnConfig` 展示了创建 teammate 时真正被固定下来的身份与策略信息

位置：`src/utils/swarm/spawnInProcess.ts:56`

spawn 配置里有：

- `name`
- `teamName`
- `prompt`
- `color`
- `planModeRequired`
- `model`

这说明创建一名 in-process teammate 时，最重要的不是“开个线程”这种底层操作，而是先确定：

```text
它是谁
它属于哪个 team
它的初始任务是什么
它要不要先进入 plan mode
它用什么 model
```

也就是说，在 Claude Code 的 swarm 体系里，teammate 首先是一个有身份和任务语义的执行体，然后才是一个并发运行单元。

---

### 第三部分：`formatAgentId(name, teamName)` 说明 agentId 是由语义身份推导出的稳定标识

位置：`src/utils/swarm/spawnInProcess.ts:111`

这里会先生成：

- `agentId = formatAgentId(name, teamName)`
- `taskId = generateTaskId('in_process_teammate')`

这说明系统里明确区分了两类 ID：

#### `agentId`
```text
描述“这个 teammate 是谁”
通常稳定映射到 name@team
```

#### `taskId`
```text
描述“这次运行实例/任务对象是谁”
用于 AppState / Task 系统管理
```

所以这里非常清楚地把：

```text
身份标识
vs
执行实例标识
```

分开了。

这也解释了上一站为什么会有“同一个 agentId 可能对应多个历史 task，需要优先找 running 的那个”。

---

### 第四部分：它特意为 teammate 创建独立 `AbortController`，而不是复用 leader 当前 query 的 abort

位置：`src/utils/swarm/spawnInProcess.ts:119`

这里的注释非常关键：

```text
Teammates should not be aborted when the leader's query is interrupted
```

所以它显式创建：

- `const abortController = createAbortController()`

而不是继承 leader 当前请求的 abortController。

这说明 in-process teammate 在同一进程中运行，但生命周期并不从属于 leader 当前这一次 turn。

也就是说：

```text
同进程 != 同取消域
```

这点非常重要，因为它说明 teammate 是“长期并发协作实体”，而不是 leader turn 内部的临时子步骤。

---

### 第五部分：`parentSessionId` 把 teammate 执行挂回到 leader 会话上下文

位置：`src/utils/swarm/spawnInProcess.ts:124`

这里通过：

- `getSessionId()`

拿到当前 session，并把它写入：

- `identity.parentSessionId`
- `createTeammateContext(... parentSessionId ... )`

这说明 teammate 虽然有自己的 agent 身份，但仍然属于 leader 当前 session 的协作树。

所以 `parentSessionId` 的作用更像：

```text
把分裂出来的 teammate 重新挂回原始会话族谱
```

这会影响：

- transcript 关联
- telemetry/trace 可视化
- 团队层级关系理解

它是 swarm 执行树的一条关键连线。

---

### 第六部分：`identity` 与 `teammateContext` 是两份相似但用途不同的对象

位置：`src/utils/swarm/spawnInProcess.ts:127`

这里先创建：

- `identity: TeammateIdentity`

再创建：

- `teammateContext = createTeammateContext(...)`

这和之前 `types.ts` 的注释正好对上：

#### `identity`
- plain data
- 存进 AppState
- 用于持久/渲染/查询

#### `teammateContext`
- runtime context
- 给 `AsyncLocalStorage` 用
- 带 `abortController`
- 为实际 agent execution 提供隔离身份

所以这里非常明确地表达了两层建模：

```text
AppState 里的 teammate 身份镜像
vs
运行时 AsyncLocalStorage 里的 teammate 执行上下文
```

这也是 in-process 模型成立的关键：

```text
共享同一进程，但每个 teammate 仍然要有独立的运行时身份上下文
```

---

### 第七部分：Perfetto 注册说明 in-process teammate 会被纳入父子 agent tracing 体系

位置：`src/utils/swarm/spawnInProcess.ts:149`

如果开启 tracing：

- `registerPerfettoAgent(agentId, name, parentSessionId)`

kill 时再：

- `unregisterPerfettoAgent(agentId)`

这说明 teammate 的 spawn/kill 不只是本地状态操作，还会同步到 tracing 系统里，形成可视化层级关系。

所以从可观测性角度讲：

```text
teammate 不是隐藏在 leader 体内的匿名并发任务，
而是被正式视作执行树中的一个子 agent 节点
```

这与 `parentSessionId` 一起构成了完整的 tracing 归属关系。

---

### 第八部分：`taskState` 初始化揭示了 in-process teammate 的“出生状态”

位置：`src/utils/swarm/spawnInProcess.ts:154`

这里初始化时设置了很多关键字段：

- `status: 'running'`
- `identity`
- `prompt`
- `model`
- `abortController`
- `awaitingPlanApproval: false`
- `spinnerVerb` / `pastTenseVerb`
- `permissionMode: planModeRequired ? 'plan' : 'default'`
- `isIdle: false`
- `shutdownRequested: false`
- `lastReportedToolCount: 0`
- `lastReportedTokenCount: 0`
- `pendingUserMessages: []`
- `messages: []`

这里最重要的几点是：

#### 1. 如果 `planModeRequired`，一出生 permission mode 就是 `plan`
这说明 in-process teammate 的 plan discipline 是创建时就固定进去的，不是后面临时决定。

#### 2. `messages` 先初始化为空数组
注释明确说是为了让 `getDisplayedMessages` 立即可用。

#### 3. `isIdle` 初始为 `false`
说明新 spawn 的 teammate 默认是“马上开始干活”，不是等待输入。

所以可以把这个初始化理解成：

```text
一名新 teammate 出生时，已经是一个可立即运行的、本地注册好的、带权限模式和 UI 表现元数据的任务实体
```

---

### 第九部分：cleanup registry 接入说明 teammate 还受会话级优雅关停机制管理

位置：`src/utils/swarm/spawnInProcess.ts:182`

这里调用：

- `registerCleanup(async () => { abortController.abort() })`

并把返回的 `unregisterCleanup` 存进 taskState。

这说明除了显式 kill 外，in-process teammate 还会参与整套全局 cleanup 机制。

也就是说：

```text
当 Claude 整体需要做 graceful shutdown 时，
这些 in-process teammate 也会收到统一清理信号
```

因此它不是 AppState 里的孤儿对象，而是被纳入了会话生命周期管理总线。

---

### 第十部分：`registerTask(taskState, setAppState)` 才是 spawn 真正完成的那一刻

位置：`src/utils/swarm/spawnInProcess.ts:190`

前面做了很多准备，但真正让 teammate “存在” 的一刻其实是：

- `registerTask(taskState, setAppState)`

从这之后：

- UI 能看到它
- task framework 能管理它
- 导航能选中它
- runner 能以 taskId 为索引继续工作

所以从系统角度，spawn 的本质不是创建 JS 对象，而是：

```text
把这个 teammate 实体正式挂进共享任务状态树
```

只有到这一步，它才从“准备中的上下文”变成“系统中的活实体”。

---

### 第十一部分：spawn 返回值不是执行结果，而是一份“已注册运行体描述”

位置：`src/utils/swarm/spawnInProcess.ts:74`

`InProcessSpawnOutput` 包括：

- `success`
- `agentId`
- `taskId`
- `abortController`
- `teammateContext`
- `error`

这说明 spawn 的输出语义不是：

```text
teammate 做完了什么
```

而是：

```text
teammate 是否成功被创建并注册好了，
以及后续 backend/runner 还需要哪些句柄继续驱动它
```

所以 spawn 是一次“控制面创建操作”，不是业务执行操作。

---

### 第十二部分：`killInProcessTeammate(...)` 展现了终止流程的完整回收链

位置：`src/utils/swarm/spawnInProcess.ts:218`

kill 流程并不只是 `abort()` 一句，而是很完整：

1. 在 `setAppState` 中找到 task
2. 只处理 `type === 'in_process_teammate'` 且 `status === 'running'`
3. 取出 `teamName / agentId / toolUseId / description`
4. `abortController.abort()`
5. 调 `unregisterCleanup?.()`
6. 调 `onIdleCallbacks`，解除等待者阻塞
7. 从 `teamContext.teammates` 中移除
8. 把 task 状态改成 `killed`
9. 清空大量运行时字段：
   - `onIdleCallbacks`
   - `pendingUserMessages`
   - `inProgressToolUseIDs`
   - `abortController`
   - `unregisterCleanup`
   - `currentWorkAbortController`
10. 仅保留最后一条 `messages` 用于停止后的 UI 展示
11. 在 state updater 外：
   - `removeMemberByAgentId(...)`
   - `evictTaskOutput(taskId)`
   - `emitTaskTerminatedSdk(...)`
   - 延迟 `evictTerminalTask(...)`
12. 最后 `unregisterPerfettoAgent(agentId)`

这说明 kill 在这里被建模成一次完整的回收事务，而不是简单停止执行。

---

### 第十三部分：为什么 kill 时要保留最后一条消息

位置：`src/utils/swarm/spawnInProcess.ts:287`

kill 后它会把：

- `messages`

裁成只保留最后一条。

这说明设计目标不是彻底清空 UI，而是：

```text
让已停止 teammate 在短暂保留期内仍有一个最小可见终态，
而不是瞬间完全消失得无迹可寻
```

结合后面的：

- `STOPPED_DISPLAY_MS`
- `evictTerminalTask(...)`

可以看出这是一个“短暂留痕、随后逐出”的 UX 设计。

---

### 第十四部分：`kill` 与 `spawn` 正好组成同进程 teammate 的注册/反注册对称结构

如果把两半并排看：

#### spawn
```text
生成 agentId/taskId
创建 identity
创建 AsyncLocalStorage teammateContext
创建 abort controller
注册 Perfetto
注册 cleanup
registerTask 到 AppState
```

#### kill
```text
abort controller
取消 cleanup
唤醒等待者
从 teamContext / team file 删除
更新 task 为 killed
清运行时字段
发 SDK terminated
延时驱逐 terminal task
注销 Perfetto
```

这说明该文件最本质的作用就是：

```text
维护 in-process teammate 在系统中的“注册态”生命周期
```

所以它不是单点工具函数，而是生命周期边界模块。

---

### 读完这一站后，你应该抓住的 9 个事实

1. `spawnInProcess.ts` 负责 in-process teammate 的创建注册与终止回收，而不负责完整执行 loop 本身。
2. `agentId` 与 `taskId` 分别代表稳定身份与运行实例，两者被明确区分。
3. in-process teammate 虽然与 leader 共进程，但拥有独立的 `AbortController`，说明它不从属于 leader 当前 query 的取消域。
4. `identity` 是写进 AppState 的 plain-data 身份镜像，而 `teammateContext` 是给 `AsyncLocalStorage` 使用的运行时隔离上下文。
5. `parentSessionId` 与 Perfetto 注册共同把 teammate 纳入 leader 会话的执行树与 tracing 层级。
6. spawn 时就会固定 `permissionMode`（需要 plan mode 的 teammate 直接从 `plan` 开始），并初始化 UI/进度/消息等运行状态。
7. `registerTask(...)` 是 teammate 真正“出生”为系统内活实体的那一刻。
8. `killInProcessTeammate(...)` 实现的是完整回收事务：abort、解除 cleanup、唤醒等待者、移出 teamContext/team file、清运行时字段、发 SDK 终止事件、延迟驱逐 UI、注销 tracing。
9. kill 后只短暂保留最小终态痕迹，再通过 `STOPPED_DISPLAY_MS` 驱逐，体现了“可见结束态 + 延迟清理”的 UX 策略。

---

### 现在把第 38-39 站串起来

```text
InProcessTeammateTask.tsx
  -> 把 in-process teammate 作为 Task 类型暴露给任务系统，并提供基础状态操作入口
spawnInProcess.ts
  -> 负责真正创建该任务实体、注入身份/上下文/abort/cleanup/tracing，并在 kill 时完整回收
```

所以 in-process teammate 现在可以先压缩成：

```text
task shell = InProcessTeammateTask.tsx
spawn/kill lifecycle = spawnInProcess.ts
runtime identity isolation = teammateContext + AsyncLocalStorage
```

---

### 下一站建议

下一站最顺的是：

```text
src/utils/teammateContext.ts
```

因为现在你已经看懂了：

- in-process teammate 怎样被 spawn
- identity 与 runtime context 为什么要分开
- AsyncLocalStorage context 在创建阶段怎样被准备好

下一步最自然就是继续看这条链的隔离核心：

**`createTeammateContext(...)` / 相关上下文访问函数到底怎样定义 teammate 身份边界，并让同进程多个 agent 在共享 Node.js 进程里仍保持各自独立的运行时身份。**
