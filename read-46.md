## 第 46 站：`src/utils/swarm/backends/InProcessBackend.ts`

### 这是什么文件

`src/utils/swarm/backends/InProcessBackend.ts` 是 swarm backend 体系里的 **in-process teammate 的 `TeammateExecutor` 实现**。

上一站 `PaneBackendExecutor.ts` 已经看懂：

```text
pane backend 路径要经过：
pane host
-> CLI bootstrap
-> mailbox protocol
最后才被包装成统一的 TeammateExecutor
```

那另一半对照实现自然就是：

- 不建 pane
- 不发 shell command
- 不启动独立 Claude CLI 进程
- 直接在当前 Node.js 进程里 spawn teammate task

所以这一站回答的是：

```text
in-process teammate 怎样直接实现同一套 TeammateExecutor 接口，
并把 spawn/terminate/kill/isActive 接到 AppState + task + ALS runner 这一套本地模型上
```

所以最准确的一句话是：

```text
InProcessBackend.ts = in-process teammate 生命周期接口的统一后端实现
```

---

### 先看它的整体定位

位置：`src/utils/swarm/backends/InProcessBackend.ts:1`

文件头注释把差异讲得很清楚：

#### in-process teammate
- 和 leader 共用同一 Node.js 进程
- 用 `AsyncLocalStorage` 做身份隔离
- 共享 API client / MCP connections 等资源
- 终止方式靠 `AbortController`
- 通信仍然复用 mailbox

这说明它虽然和 pane backend 走的是完全不同的执行承载方式，
但在更高层 teammate 语义上，仍然要对齐成同一套：

- `spawn()`
- `sendMessage()`
- `terminate()`
- `kill()`
- `isActive()`

所以它的定位就是：

```text
用 in-process runtime primitives 实现 TeammateExecutor 这份跨后端合同
```

---

### 第一部分：`InProcessBackend implements TeammateExecutor` 说明 in-process 不是特殊旁路，而是正式 backend 实现之一

位置：`src/utils/swarm/backends/InProcessBackend.ts:38`

类定义直接是：

- `export class InProcessBackend implements TeammateExecutor`
- `readonly type = 'in-process'`

这意味着 in-process 模式在架构上不是某种“临时特判”。

它和：

- tmux
- iTerm2

一样，都是 backend 体系里的正式一员。

也就是说：

```text
registry 不是在 “normal backend” 和 “特殊 in-process hack” 之间切换，
而是在多个正式 backend implementation 之间切换
```

这很重要，因为它解释了为什么整个 swarm backend 抽象设计得这么完整。

---

### 第二部分：它和 `PaneBackendExecutor` 一样也要求先 `setContext()`，说明不论哪种 backend，spawn 都依赖 leader 当前 ToolUseContext

位置：`src/utils/swarm/backends/InProcessBackend.ts:41`

这里同样有：

- `private context: ToolUseContext | null = null`
- `setContext(context)`

而且注释也写了：

```text
TeammateTool 在 spawn 前必须先设置 context
```

这说明无论是 pane backend 还是 in-process backend，
spawn teammate 都不是一个脱离 leader 当前 query/runtime 上下文的纯创建动作。

两者都需要 leader 当前环境中的能力，例如：

- `setAppState`
- `getAppState`
- 当前工具调用上下文
- 当前权限策略
- 其他运行时设施

所以 backend 抽象层有一个很强的共性：

```text
spawn teammate = leader runtime 的派生操作
```

不是“在真空里启动一个新 agent”。

---

### 第三部分：`isAvailable()` 永远返回 true，说明 in-process backend 被建模成无外部依赖的保底执行路径

位置：`src/utils/swarm/backends/InProcessBackend.ts:55`

这里非常简单：

- `async isAvailable() { return true }`

这说明和 pane backend 不同，
in-process 不需要检测：

- tmux 有没有装
- 是否在 iTerm2
- it2 CLI 是否可用
- pane 宿主是否存在

它只依赖当前 Claude Code 进程本身。

所以在 backend 体系里，in-process 的定位非常明确：

```text
zero-external-dependency teammate execution mode
```

这也正是它能成为 auto mode fallback 的原因。

---

### 第四部分：`spawn()` 的前半段说明 in-process teammate 的创建先走 `spawnInProcessTeammate(...)` 做注册，再走 `startInProcessTeammate(...)` 做真正执行

位置：`src/utils/swarm/backends/InProcessBackend.ts:72`

`spawn()` 的主流程是：

1. guard context
2. 调 `spawnInProcessTeammate(...)`
3. 如果成功，再调 `startInProcessTeammate(...)`
4. 返回统一的 `TeammateSpawnResult`

这和之前 `spawnInProcess.ts` / `teammateContext.ts` 的理解完全对上。

也就是说 in-process backend 把 teammate 出生明确拆成两步：

#### 第一步：注册态创建
`spawnInProcessTeammate(...)` 负责：
- 生成 identity
- 生成 taskId
- 创建 `AbortController`
- 创建 `TeammateContext`
- 注册 AppState task

#### 第二步：执行态启动
`startInProcessTeammate(...)` 负责：
- 真正启动 agent loop
- 把 ALS context 激活起来
- 让 teammate 开始运行

所以这一点非常关键：

```text
in-process backend 的 spawn 不是单函数一步到位，
而是“先创建实体，再启动执行”的两阶段启动链
```

---

### 第五部分：`spawnInProcessTeammate(...)` 这里只传了最小配置，说明底层注册阶段与上层执行策略仍然是分层的

位置：`src/utils/swarm/backends/InProcessBackend.ts:87`

传给 `spawnInProcessTeammate(...)` 的只有：

- `name`
- `teamName`
- `prompt`
- `color`
- `planModeRequired`

而像这些更丰富的字段：

- `model`
- `systemPrompt`
- `systemPromptMode`
- `permissions`
- `allowPermissionPrompts`

并没有传给注册阶段。

它们是在后面的 `startInProcessTeammate(...)` 才被使用。

这说明作者刻意保持：

```text
spawn registration layer
vs
execution behavior layer
```

的边界。

也就是说：

- `spawnInProcessTeammate` 管创建/注册壳体
- `startInProcessTeammate` 管真正的 agent 行为配置

这和之前看到的 `spawnInProcess.ts 管生死，不管完整 loop` 的理解完全一致。

---

### 第六部分：成功后立刻 `startInProcessTeammate(...)`，说明 `InProcessBackend` 才是把“注册态 teammate”变成“真正运行中的 teammate”的 backend 组装层

位置：`src/utils/swarm/backends/InProcessBackend.ts:98`

这里的条件很明确：

- `result.success`
- `result.taskId`
- `result.teammateContext`
- `result.abortController`

都存在，才启动执行。

这说明 `InProcessBackend` 是真正把：

- spawn result
- runtime context
- task id
- tool use context
- behavioral config

组装到一起的那一层。

换句话说：

```text
spawnInProcess.ts 负责生产零件
InProcessBackend.ts 负责把零件接成可运行 teammate
```

所以它不是简单转发器，
而是 in-process teammate backend 的真正 orchestration layer。

---

### 第七部分：传给 `startInProcessTeammate(...)` 的 `identity` 重新显式展开，说明执行层仍然偏好 plain identity object，而不是直接到处传整包 spawn result

位置：`src/utils/swarm/backends/InProcessBackend.ts:107`

这里传了一个新的 `identity`：

- `agentId`
- `agentName`
- `teamName`
- `color`
- `planModeRequired`
- `parentSessionId`

而不是直接把 `result.teammateContext` 或 `config` 整包甩进去。

这说明作者在 runner 启动边界仍然保持数据形状清晰：

```text
plain identity
runtime teammateContext
execution config
```

三者分开传。

这和前面整个 swarm 模型的建模风格完全一致：

- identity 不是 context
- context 不是 task state
- spawn config 也不是 identity

所以这段再次印证了系统在这方面的分层很稳定。

---

### 第八部分：`toolUseContext: { ...this.context, messages: [] }` 是一个非常关键的内存边界处理

位置：`src/utils/swarm/backends/InProcessBackend.ts:119`

这里的注释非常重要：

```text
teammate 从不读取 toolUseContext.messages；
runAgent 会自己用 createSubagentContext 覆盖它。
如果把 parent conversation 传进去，会把它整段 pin 在 teammate 生命周期里。
```

所以这里刻意做了：

- `toolUseContext: { ...this.context, messages: [] }`

这说明作者非常明确地意识到一个典型内存问题：

```text
同进程 teammate 如果长期持有 parent query 的完整 messages 数组，
会把父会话上下文整段额外钉在内存里
```

因此这里做的是：

```text
保留 toolUseContext 的能力壳层，
但主动切断不需要的历史消息引用
```

这是一个非常有价值的实现细节，
因为它说明 in-process backend 不只是追求功能正确，
还非常在意共享进程下的内存生存期。

---

### 第九部分：`allowedTools` / `allowPermissionPrompts` / `systemPrompt` 都在启动执行时注入，说明 in-process teammate 的行为策略是在 runner 启动时最终定型的

位置：`src/utils/swarm/backends/InProcessBackend.ts:124`

传给 `startInProcessTeammate(...)` 的行为配置包括：

- `model`
- `systemPrompt`
- `systemPromptMode`
- `allowedTools`
- `allowPermissionPrompts`

这说明 in-process backend 最终把 teammate 当成一个：

```text
已注册的本地任务实体
+
一组即将启动的执行策略参数
```

然后在 runner 启动边界把两者合并。

也就是说：

```text
真正的 teammate 执行人格/权限配置，
是在 startInProcessTeammate 这一跳生效的
```

这和 pane backend 通过 CLI flags/env 注入行为的方式形成鲜明对照。

---

### 第十部分：`sendMessage()` 说明即使是 in-process teammate，也在 backend 抽象层继续复用 mailbox 语义，而不直接调用 task helper

位置：`src/utils/swarm/backends/InProcessBackend.ts:145`

这一段很有意思。

尽管我们前面已经知道：

- in-process teammate 在更底层可以通过 AppState / pendingUserMessages / task helper 协作

但这里的 `sendMessage()` 仍然选择：

1. `parseAgentId(agentId)`
2. 拆出 `agentName` / `teamName`
3. `writeToMailbox(...)`

而注释直接说：

```text
All teammates use file-based mailboxes for simplicity.
```

这说明在 `TeammateExecutor` 这一抽象层，
作者刻意统一了：

```text
cross-backend message delivery semantics = mailbox
```

哪怕 in-process 底层本来有更直接的共享状态路径，
这个接口层也选择保持一致。

所以要特别记住：

```text
in-process teammate 的内部执行模型是 shared AppState
但跨-backend executor 接口的 sendMessage 语义仍统一成 mailbox transport
```

---

### 第十一部分：`terminate()` 说明 in-process teammate 的优雅关闭同样走 shutdown request 协议，只是额外同步 task 上的 `shutdownRequested`

位置：`src/utils/swarm/backends/InProcessBackend.ts:182`

terminate 的流程是：

1. guard context
2. 通过 `findTeammateTaskByAgentId(...)` 找当前 task
3. 如果已经 `shutdownRequested`，直接返回 true
4. 生成 `requestId`
5. `createShutdownRequestMessage(...)`
6. `writeToMailbox(...)`
7. `requestTeammateShutdown(task.id, setAppState)`

这说明 in-process teammate 的 graceful shutdown 并不是一条“完全不同的内部特判路径”。

它仍然遵守：

```text
shutdown_request mailbox protocol
```

只是由于它还拥有本地 task state，
所以会顺手把：

- `shutdownRequested`

同步到本地 AppState。

所以准确说它做了两件事：

#### 1. 协议层通知
让 teammate 自己按 shutdown 流程处理

#### 2. 本地状态层标记
让 UI / runtime 知道“已经在请求关闭中”

这正是 in-process backend 相比 pane backend 的额外能力。

---

### 第十二部分：`kill()` 直接委托 `killInProcessTeammate(...)`，说明 in-process force kill 的核心语义是 task/abort 生命周期终止，而不是宿主资源终止

位置：`src/utils/swarm/backends/InProcessBackend.ts:255`

这里的流程是：

- 找 task
- 调 `killInProcessTeammate(task.id, setAppState)`

而不是像 pane backend 那样 `killPane(...)`。

这说明 in-process teammate 的 force kill 本质是：

```text
终止本地任务实体及其 async execution chain
```

而不是：

```text
终止外部宿主 pane/process 资源
```

所以对照 pane backend：

#### pane kill
- host resource kill

#### in-process kill
- AppState/task/AbortController 驱动的本地执行终止

这非常准确地反映了两种后端的根本差异。

---

### 第十三部分：`isActive()` 的实现比 pane backend 更强，因为它直接依赖 task state + AbortController

位置：`src/utils/swarm/backends/InProcessBackend.ts:292`

这里判断活跃状态的方式是：

- 找 task
- `task.status === 'running'`
- `task.abortController?.signal.aborted ?? true`
- `active = isRunning && !isAborted`

这说明 in-process backend 的 `isActive()` 不是像 pane backend 那样的 best-effort tracking map，
而是直接依赖：

```text
本地真实 task state
+
真实 abort signal 状态
```

所以它的活跃性判断要更接近实际执行真相。

也就是说：

```text
in-process backend 在本进程内拥有更强的可观测性，
因此 TeammateExecutor.isActive() 的实现质量也更高
```

这正是共享进程模型的一个优势。

---

### 第十四部分：这份文件揭示了 in-process backend 的真实运行模式是“本地 task/runtime 控制 + mailbox 协议兼容”的混合体

如果把整份文件压缩，会发现它不是简单“完全不用 mailbox”。

它实际是两套东西叠在一起：

#### 1. 本地执行/状态模型
- `spawnInProcessTeammate(...)`
- `startInProcessTeammate(...)`
- `findTeammateTaskByAgentId(...)`
- `requestTeammateShutdown(...)`
- `killInProcessTeammate(...)`
- `task.status` / `AbortController`

#### 2. 协议兼容模型
- `writeToMailbox(...)`
- `createShutdownRequestMessage(...)`
- `sendMessage()` 走 mailbox
- `terminate()` 也发 mailbox request

所以 in-process backend 最准确的理解是：

```text
执行态靠 shared AppState + ALS + task model
通信/控制协议层仍尽量复用 mailbox family
```

这正是前几站一直在显露的模式，到这里彻底坐实了。

---

### 第十五部分：和 `PaneBackendExecutor` 对照后，可以看清楚同一 `TeammateExecutor` 接口下的两条实现路线

这一点很重要，值得并排记一次。

#### `PaneBackendExecutor`
```text
spawn = create pane + inject CLI command + mailbox prompt
terminate = mailbox shutdown_request
kill = killPane
isActive = tracking map best-effort
```

#### `InProcessBackend`
```text
spawn = register in-process task + start in-process runner
terminate = mailbox shutdown_request + local shutdownRequested flag
kill = killInProcessTeammate
isActive = task.status + AbortController signal
```

这说明统一接口之下，
两条路线的差别主要在：

- 执行承载体不同：pane/process vs task/ALS
- force kill 机制不同：host kill vs local abort
- active 可观测性强度不同：best-effort vs direct state

而相同点在于：

- 都是 `TeammateExecutor`
- 都能 `spawn/sendMessage/terminate/kill/isActive`
- 都尽量复用 mailbox 协议做消息/控制面通信

这就是 swarm backend 抽象真正成功的地方。

---

### 读完这一站后，你应该抓住的 10 个事实

1. `InProcessBackend.ts` 是 in-process teammate 的正式 `TeammateExecutor` 实现，不是特殊旁路代码。
2. 它和 `PaneBackendExecutor` 一样要求先 `setContext()`，说明 spawn teammate 本质上都是从 leader 当前 ToolUseContext 派生出来的控制面操作。
3. `isAvailable()` 永远返回 true，说明 in-process backend 被建模成无外部依赖的保底执行模式。
4. in-process spawn 是两阶段的：先 `spawnInProcessTeammate(...)` 创建/注册 task 与 runtime context，再 `startInProcessTeammate(...)` 真正启动 agent execution loop。
5. 行为配置如 `model`、`systemPrompt`、`allowedTools`、`allowPermissionPrompts` 不是注册阶段处理，而是在 runner 启动阶段最终生效。
6. `toolUseContext: { ...this.context, messages: [] }` 是一个重要的内存边界优化，避免父会话完整消息历史被同进程 teammate 生命周期长期 pin 住。
7. `sendMessage()` 虽然面对的是 in-process teammate，但在 executor 抽象层仍统一复用 mailbox transport，而不是直接操作 task helper。
8. `terminate()` 同样走 `shutdown_request` mailbox 协议，但会额外同步本地 task 的 `shutdownRequested` 标志，体现协议层和本地状态层双写。
9. `kill()` 直接委托 `killInProcessTeammate(...)`，说明 in-process force kill 的本质是 task/abort 生命周期终止，而不是宿主 pane/process 终止。
10. `isActive()` 直接依赖 task 状态和 `AbortController`，因此比 pane backend 的 best-effort tracking 更接近真实运行状态。

---

### 现在把第 45-46 站串起来

```text
PaneBackendExecutor.ts
  -> 把 pane host + CLI bootstrap + mailbox protocol 适配成统一的 TeammateExecutor
InProcessBackend.ts
  -> 把 AppState task + ALS runner + mailbox-compatible control flow 适配成同一套 TeammateExecutor
```

所以现在 backend 实现层可以先压缩成：

```text
pane-based route
  -> PaneBackendExecutor

in-process route
  -> InProcessBackend

shared abstraction
  -> TeammateExecutor
```

也就是说：

```text
两种后端执行承载完全不同，
但都被压平到同一 teammate lifecycle contract 下面
```

---

### 下一站建议

下一站最顺的是：

```text
src/utils/swarm/inProcessRunner.ts
```

因为现在你已经看懂了：

- `InProcessBackend` 怎样先 spawn 再 start
- in-process teammate 怎样接入 task / AbortController / mailbox 协议
- 真正的 agent execution loop 是通过 `startInProcessTeammate(...)` 启动的

下一步最自然就是继续往真正的运行内核下钻：

**`startInProcessTeammate(...)` 到底怎样把 task、ALS teammateContext、toolUseContext、runAgent/query loop 串起来，并让同进程 teammate 真正开始工作。**
