## 第 38 站：`src/tasks/InProcessTeammateTask/InProcessTeammateTask.tsx`

### 这是什么文件

`src/tasks/InProcessTeammateTask/InProcessTeammateTask.tsx` 是 in-process teammate 体系里的 **任务壳层 + AppState 操作入口**。

上一站 `inProcessTeammateHelpers.ts` 已经看到：

```text
in-process teammate 不走独立 inbox poller
它只需要少量 helper 把消息语义桥接回共享 AppState / task state
```

而这一站继续往真正的实体层走，回答的是：

```text
in-process teammate 在 AppState 里到底以什么任务形态存在
有哪些最基础的生命周期操作
怎样追加消息、注入用户输入、查找运行中的 teammate
```

所以最准确的一句话是：

```text
InProcessTeammateTask.tsx = in-process teammate 的任务接口壳层与状态操作入口
```

---

### 先看它的整体定位

位置：`src/tasks/InProcessTeammateTask/InProcessTeammateTask.tsx:1`

文件头注释已经明确：

- 它实现 `Task` 接口
- in-process teammate 跟 `LocalAgentTask` 不同
- 同进程运行，通过 `AsyncLocalStorage` 隔离
- 带 team-aware identity
- 支持 plan mode approval
- 既可能 idle，也可能 active

这说明这里不是 in-process teammate 的完整 runner，而是把它纳入通用任务系统的那一层壳。

所以这份文件的核心价值在于：

```text
把“同进程 teammate”正式包装成 Task 系统能理解、能管理、能展示的一类任务
```

---

### 第一部分：`InProcessTeammateTask` 对象本身非常薄，说明真正的执行逻辑不在这里

位置：`src/tasks/InProcessTeammateTask/InProcessTeammateTask.tsx:21`

导出的任务定义只有：

- `name: 'InProcessTeammateTask'`
- `type: 'in_process_teammate'`
- `kill(taskId, setAppState)` -> `killInProcessTeammate(...)`

这说明它不像某些 task 类型那样在这里直接塞大量执行逻辑。

它的真正职责更像：

```text
在 Task 注册层声明“有这么一种任务类型”，
并把 kill 操作转发给专门的 swarm runtime 实现
```

因此第一结论是：

```text
InProcessTeammateTask.tsx 更偏 task adapter，而不是 task runner
```

---

### 第二部分：`requestTeammateShutdown(...)` 体现了“请求关闭”与“立即杀死”是两个层级

位置：`src/tasks/InProcessTeammateTask/InProcessTeammateTask.tsx:32`

这里做的不是直接 kill，而是：

- 只在 `task.status === 'running'`
- 且 `!task.shutdownRequested`
- 把 `shutdownRequested` 设成 `true`

这说明 in-process teammate 的关闭语义被拆成了两层：

#### 1. 请求关闭
```text
通过状态位告诉 teammate：应该进入 shutdown 流程了
```

#### 2. 强制 kill
```text
真正的 kill 由 killInProcessTeammate(...) 执行
```

所以这个函数表达的是：

```text
优先走协作式 shutdown，而不是一上来粗暴中断
```

这和 swarm/mailbox 路径里的 shutdown request / approval 语义是对齐的。

---

### 第三部分：`appendTeammateMessage(...)` 说明 task.messages 只是 UI 镜像，不是完整历史真源

位置：`src/tasks/InProcessTeammateTask/InProcessTeammateTask.tsx:47`

这里通过：

- `appendCappedMessage(task.messages, message)`

把消息追加到任务状态里。

结合上一轮对 `types.ts` 的读取，可以知道：

- `task.messages` 只是 zoomed transcript view 的 UI 镜像
- 有上限 `TEAMMATE_MESSAGES_UI_CAP = 50`
- 完整对话另有更完整来源

所以这里最该记住的是：

```text
appendTeammateMessage(...) 不是在维护 teammate 的完整会话真相，
而是在维护 AppState 中给 UI 展示用的最近消息窗口
```

这也解释了为什么这里的追加逻辑会刻意走 capped append。

---

### 第四部分：`injectUserMessageToTeammate(...)` 揭示了“给 teammate 发话”在 in-process 模型里的真实落地方式

位置：`src/tasks/InProcessTeammateTask/InProcessTeammateTask.tsx:63`

这里做了两件事：

1. 把字符串塞进：
   - `pendingUserMessages`
2. 同时生成一条 `createUserMessage({ content: message })`
   - 追加到 `task.messages`

这说明在 in-process teammate 里，给 teammate 发一条消息并不是立刻触发一个独立 IPC，而是：

```text
先进入该 teammate 的待投递用户消息队列
runner 稍后再消费它
```

而同时更新 `task.messages` 的目的，是让 zoomed transcript UI 立即看到这条消息，获得即时反馈。

所以它实际上是：

```text
execution path: pendingUserMessages
UI path: task.messages
```

的双写。

这是一个很典型的“执行语义”和“用户感知”分离设计。

---

### 第五部分：为什么允许在 idle 时也能注入消息

位置：`src/tasks/InProcessTeammateTask/InProcessTeammateTask.tsx:68`

注释明确写了：

- 允许在 `running` 或 `idle` 时注入消息
- 只在 terminal state 才拒绝

这说明在 in-process teammate 语义里：

```text
idle 不是“已经不能再沟通了”
idle = 正在等待下一份工作/输入
```

因此用户在查看 teammate transcript 时，仍然可以给它补充信息，
系统会把消息排入 `pendingUserMessages`，等待 runner 取走。

所以 `idle` 在这里更像：

```text
ready for next prompt
```

而不是 finished。

---

### 第六部分：`findTeammateTaskByAgentId(...)` 说明 agentId 才是 in-process teammate 的稳定身份键

位置：`src/tasks/InProcessTeammateTask/InProcessTeammateTask.tsx:86`

这里按：

- `isInProcessTeammateTask(task)`
- `task.identity.agentId === agentId`

查找任务，且有一个很关键的策略：

- 优先返回 `status === 'running'` 的任务
- 否则才返回第一个 fallback

注释已经说明原因：

```text
旧的 killed/completed task 可能还留在 AppState，
同时新的 running task 已经存在，而且 agentId 相同
```

这说明系统明确考虑到了 teammate 重启 / 复用 identity 的情况。

所以这里表达的是：

```text
agentId 是身份主键，
但 AppState 里可能存在同一身份的历史任务残留，
因此查找时必须优先活跃实例
```

这是个很实际的运行时细节。

---

### 第七部分：`getAllInProcessTeammateTasks(...)` / `getRunningTeammatesSorted(...)` 说明这个模块还承担 UI 一致性约束

位置：

- `getAllInProcessTeammateTasks(...)` `src/tasks/InProcessTeammateTask/InProcessTeammateTask.tsx:110`
- `getRunningTeammatesSorted(...)` `src/tasks/InProcessTeammateTask/InProcessTeammateTask.tsx:117`

这里除了筛任务，还专门强调：

- running teammates 要按 `agentName` 字母序排序
- `TeammateSpinnerTree`
- `PromptInput` footer selector
- `useBackgroundTaskNavigation`

这三处都必须使用同一排序

这说明这个文件不只是“任务操作函数集合”，还承担一个重要职责：

```text
为多个 UI/导航部件提供统一的 teammate 排序约定
```

否则同一个 `selectedIPAgentIndex` 会在不同组件里映射到不同 teammate。

所以这里其实是在维护：

```text
跨组件共享的 teammate ordering contract
```

---

### 第八部分：结合 `types.ts` 才能看清这个 task state 的真正形状

位置：`src/tasks/InProcessTeammateTask/types.ts:13`

这次顺手读到的 `types.ts` 很关键，因为它补足了状态对象的完整结构。

其中 `InProcessTeammateTaskState` 包括：

- `identity.agentId / agentName / teamName / color / planModeRequired / parentSessionId`
- `prompt`
- `model`
- `selectedAgent`
- `abortController`
- `currentWorkAbortController`
- `unregisterCleanup`
- `awaitingPlanApproval`
- `permissionMode`
- `error`
- `result`
- `progress`
- `messages`
- `inProgressToolUseIDs`
- `pendingUserMessages`
- `spinnerVerb / pastTenseVerb`
- `isIdle`
- `shutdownRequested`
- `onIdleCallbacks`
- `lastReportedToolCount / lastReportedTokenCount`

这说明所谓 in-process teammate task，并不是一个只存 prompt/result 的轻对象，而是把：

```text
身份
执行控制器
权限状态
plan approval 状态
UI transcript 镜像
消息待投递队列
生命周期标志
进度统计
```

全部都装进了同一个任务状态容器里。

所以这一站最应该补上的一句话是：

```text
InProcessTeammateTask.tsx 提供的是操作入口，
而真正的“实体形状”定义在 types.ts 里
```

---

### 第九部分：`appendCappedMessage(...)` 暗示了一个非常现实的内存控制策略

位置：`src/tasks/InProcessTeammateTask/types.ts:89`

`TEAMMATE_MESSAGES_UI_CAP = 50` 的注释写得非常重：

- `task.messages` 只是 UI 镜像
- 完整对话在本地 `allMessages` 和磁盘 transcript
- 曾经出现过大规模 agent burst 导致 RSS 飙升
- 双份消息拷贝是主要内存成本之一

所以 `appendCappedMessage(...)` 的存在不是“小优化”，而是：

```text
为了防止 swarm 爆量并发时，AppState 中每个 teammate task 都复制整份对话历史，造成内存爆炸
```

这让 `appendTeammateMessage(...)` 的设计意图一下子清楚了：

```text
AppState 里的 teammate transcript 只是近期 UI 缓存，不是历史归档
```

---

### 第十部分：这份文件揭示了 in-process teammate 模型的一个核心原则——“执行实体是 Task，通信只是附属能力”

如果把这站和前几站对比：

#### mailbox / out-of-process 路径
你首先看到的是：
- mailbox
- protocol message
- poller
- callback 恢复

#### in-process 路径
你首先看到的是：
- Task state
- kill / shutdownRequested
- pendingUserMessages
- messages UI mirror
- running teammate list

这说明 in-process teammate 的建模重心明显不同：

```text
out-of-process teammate 先是“远端通信对象”
in-process teammate 先是“本地任务对象”
```

也就是说，同进程模型的第一性原语不是 mailbox，而是 AppState 中的一类任务状态机。

这正是为什么这一站落在 `tasks/` 目录下，而不是 `utils/swarm/`。

---

### 元

问题：**这一站真正想解决的架构问题是什么？**

回答：这一站把 in-process teammate 明确定义成一种特殊 task type，并把 request shutdown、append message、inject user message 等状态操作收在这里。它是任务壳层，不是运行内核。

### 反

问题：**如果把这一站的设计反过来，会发生什么？**

回答：如果 in-process teammate 不走统一 task 接口，管理和展示都会变成特判。生命周期操作也会失去统一入口。

### 空

问题：**跳出当前文件名，这一站背后更大的问题是什么？**

回答：更大的问题是，特殊协作者如何以标准对象的形式被平台接纳。task adapter 是一种很典型的制度化方式。


### 读完这一站后，你应该抓住的 8 个事实

1. `InProcessTeammateTask.tsx` 的核心作用是把 in-process teammate 纳入 Task 系统，而不是直接承载完整 runner 逻辑。
2. `kill()` 与 `requestTeammateShutdown()` 分别代表强制终止与协作式关闭两个层级。
3. `appendTeammateMessage(...)` 维护的是 AppState 中给 UI 展示用的最近 transcript 镜像，而不是完整对话真源。
4. `injectUserMessageToTeammate(...)` 同时更新执行队列 `pendingUserMessages` 和 UI 镜像 `messages`，体现执行路径与用户感知路径分离。
5. in-process teammate 在 `idle` 时仍然可以接收用户消息，说明 idle 的语义是“等待下一份输入”，不是终止。
6. `findTeammateTaskByAgentId(...)` 优先返回 running task，说明系统考虑了同一 `agentId` 的历史任务残留问题。
7. `getRunningTeammatesSorted(...)` 维护的是跨多个 UI/导航组件共享的 teammate 排序契约。
8. 真正完整的 `InProcessTeammateTaskState` 结构定义在 `types.ts`，其中包含身份、权限、plan approval、执行控制、消息缓存、生命周期与进度统计等多层状态。

---

### 现在把第 37-38 站串起来

```text
inProcessTeammateHelpers.ts
  -> 提供 in-process teammate 的少量状态桥接与消息语义辅助
InProcessTeammateTask.tsx
  -> 把 in-process teammate 作为一种正式 Task 类型纳入 AppState 与任务系统管理
```

所以现在可以把 in-process teammate 这条线先压缩成：

```text
helpers = 轻量桥接层
task = 本地执行实体壳层
types = 实体状态模型定义
```

---

### 下一站建议

下一站最顺的是：

```text
src/utils/swarm/spawnInProcess.ts
```

因为现在你已经看懂了：

- in-process teammate 在 AppState 里以什么任务形态存在
- 怎样被 kill / shutdown / 注入用户消息 / 查找运行实例

下一步最自然就是继续进入真正的创建与启动链路：

**in-process teammate 是怎样被 spawn 出来、怎样建立 AsyncLocalStorage 身份隔离、怎样把 task state 与实际 runner 绑到一起开始执行的。**
