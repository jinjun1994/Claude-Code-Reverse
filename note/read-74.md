## 第 74 站：`src/utils/hooks/hookEvents.ts`

### 这是什么文件

`src/utils/hooks/hookEvents.ts` 是 hooks 子系统里的 **事件发射与广播层：定义了 hook 执行的三种事件类型（started、progress、response），提供一个带 pending 队列的发布-订阅机制，支持 handler 注册前的事件缓存、事件节流（去重），以及 `session`/`Setup` 始终发射、其他事件按需启用的分级发射策略。**

上一站 `src/utils/hooks/AsyncHookRegistry.ts` 已经看懂 async hook 的进程跟踪与轮询层。而在那里面直接依赖的 `emitHookResponse` 和 `startHookProgressInterval` 就是从这个文件来的。

这一站回答的是：

```text
hook 执行期间和完成后，系统怎样把 started/progress/response
三种事件可靠地广播给外部 handler（UI、日志、SDK 等），
并且支持 handler 延迟注册的全套 pending + 限流机制
```

所以最准确的一句话是：

```text
hookEvents.ts = hook 的事件广播层：为 hook 执行提供 started/progress/response 三态事件，带 pending 缓冲、handler 注册、差量去重发射，以及 always-emit vs opt-in 的分级策略
```

---

### 先看它的整体定位

位置：`src/utils/hooks/hookEvents.ts:1`

这个文件不是：

- hook 执行器
- hook 状态存储
- hook schema
- async hook registry 本体

它的职责是：

1. 定义 `started`、`progress`、`response` 三种事件类型
2. 管理一个全局的 `pendingEvents` 队列
3. 提供 handler 注册/取消注册
4. 提供 `emit(...)` 带缓冲逻辑
5. 提供 `startHookProgressInterval(...)` 带差量去重的周期性进度发射
6. 提供 `shouldEmit(...)` 分级发射策略
7. 暴露 enable/disable 所有 hook 事件的控制入口
8. 提供全局状态清理

所以它本质上是一个：

```text
lightweight event pub-sub layer for hook lifecycle events
```

也就是说：

- `AsyncHookRegistry.ts` 调用这里的函数来 emit 结果
- `hooks.ts` 调用这里的函数来 emit 进度
- 而这里的函数把事件转发给最终 handler

---

### 第一部分：三类事件类型 `started` / `progress` / `response` 清晰描述了 hook 执行的生命周期三态

位置：

- `HookStartedEvent` `src/utils/hooks/hookEvents.ts:22`
- `HookProgressEvent` `src/utils/hooks/hookEvents.ts:29`
- `HookResponseEvent` `src/utils/hooks/hookEvents.ts:39`

说明 hook 在执行期间对外暴露的事件是：

1. `started`：hook 刚开始执行
2. `progress`：执行中，带 stdout/stderr/output 增量输出
3. `response`：执行完，带最终 exit code 和 outcome

这三种事件合起来就是一个典型的"长任务生命周期通知"：

```text
started -> progress (N times) -> response
```

---

### 第二部分：`ALWAYS_EMITTED_HOOK_EVENTS = ['SessionStart', 'Setup']` 非常关键，说明事件发射采取了分级策略：SessionStart 和 Setup 始终发射，其余事件需要 opt-in

位置：`src/utils/hooks/hookEvents.ts:18`

注释说明了原因：

```text
low-noise lifecycle events that were in the original
allowlist and are backwards-compatible
```

这说明这两个事件被认为是"足够安静且足够重要"的：

- SessionStart：session 初始化钩子
- Setup：项目初始化钩子

它们不会频繁触发，
所以默认发射也不会造成事件洪水。

这也说明事件广播层的设计哲学是：

```text
default safe and quiet,
selectively verbose when requested
```

---

### 第三部分：`MAX_PENDING_EVENTS = 100` 说明 pending 队列有上限，超出 oldest-first 丢弃，防止在无 handler 的场景下无限积累

位置：`src/utils/hooks/hookEvents.ts:20`

```ts
if (pendingEvents.length > MAX_PENDING_EVENTS) {
  pendingEvents.shift()
}
```

这是典型的环形缓冲策略。

也就是说：

- handler 还没注册时
- 事件先进 pending 队列
- 但如果没人消费超过 100 条
- 最老的事件会被默默丢掉

这说明系统不指望"所有事件最终都被消费"，
而是"尽量缓存近期事件等 handler 来"。

---

### 第四部分：`eventHandler` 是全局单例，说明同一时刻只有一个 active listener 消费 hook 事件

位置：`src/utils/hooks/hookEvents.ts:58`

```ts
let eventHandler: HookEventHandler | null = null
```

这里不是多 listener 模型，
而是：

```text
single subscriber at a time
```

结合 CLI 架构来看，这很合理因为：

- CLI 本身是单进程
- 通常只有一个 UI/log 层需要消费事件

---

### 第五部分：`registerHookEventHandler(...)` 最有趣的一点是处理 "handler 晚于事件到达" 的场景——handler 注册时会先 flush 所有 pending 事件，确保不丢刚发生不久的历史

位置：`src/utils/hooks/hookEvents.ts:61`

逻辑是：

1. 把 handler 设为 `eventHandler`
2. 如果 handler 非空且 pending 队列有事件
3. 全部 splice 出来逐个发给新 handler

这说明系统预期了一个常见的 CLI 时序问题：

```text
hooks might run before the handler is fully wired up
```

所以用 pending 作为"时间桥梁"。

---

### 第六部分：`emit(...)` 内部逻辑很关键，它实现了一个"有 handler 就直接发、没有就暂存"的统一发射策略，同时带最大缓冲保护

位置：`src/utils/hooks/hookEvents.ts:72`

```ts
if (eventHandler) {
  eventHandler(event)
} else {
  pendingEvents.push(event)
  if (pendingEvents.length > MAX_PENDING_EVENTS) {
    pendingEvents.shift()
  }
}
```

这是一个非常常见的 pub-sub 模式：

```text
immediate delivery if subscriber exists,
else buffer with backpressure cap
```

---

### 第七部分：`shouldEmit(...)` 实现了前面提到的分级发射策略

位置：`src/utils/hooks/hookEvents.ts:83`

逻辑是：

```ts
if ALWAYS_EMITTED_HOOK_EVENTS -> true
else if allHookEventsEnabled && valid HOOK_EVENT -> true
else -> false
```

这说明一个 hook 事件能不能被发射，取决于：

1. 是不是 SessionStart/Setup（始终发射）
2. 否则 `allHookEventsEnabled` 是否打开
3. 而且是否是合法的 `HOOK_EVENTS` 成员

这个设计非常像一个"事件访问控制层"。

---

### 第八部分：`setAllHookEventsEnabled(...)` 说明外部可以按需打开/关闭完整事件流，这通常是在 SDK `includeHookEvents` 选项或远程模式下被启用

位置：`src/utils/hooks/hookEvents.ts:184`

注释写得很清楚：

```text
Called when the SDK includeHookEvents option is set or when
running in CLAUDE_CODE_REMOTE mode
```

这说明这个文件的用途不仅是 CLI 内部日志：

- SDK 可能把 hook 事件传给第三方集成
- 远程模式可能需要完整的 hook 执行可见性

所以它是一个：

```text
SDK + remote mode hook event bridge
```

---

### 第九部分：`startHookProgressInterval(...)` 是这文件最有意思的函数——它实现了一个带差量去重的周期性进度发射机制，只有 output 变化时才真正 emit，从而避免无变化时的重复通知

位置：`src/utils/hooks/hookEvents.ts:124`

流程是：

1. 判断是否应该发射
2. 用 `setInterval` 定时调用 `getOutput()`
3. 如果 `output === lastEmittedOutput` -> skip
4. 否则更新 `lastEmittedOutput` 并 emit progress
5. `interval.unref()` 防止进程因此而保持活跃
6. 返回一个 stop 函数

这里的几个设计点值得说：

#### 差量去重

```ts
if (output === lastEmittedOutput) return
```

避免"hook 跑着但没新输出"的情况下每秒无意义 emit。

#### unref

```ts
interval.unref()
```

确保这个 interval 不会阻止 Node 进程自然退出。

#### getOutput 回调

由调用方提供：

```ts
getOutput: () => Promise<{ stdout, stderr, output }>
```

所以这个函数只负责"定时差量发射"，
不关心输出从哪里来。

这也是 `AsyncHookRegistry` 调用它时传入：

```ts
getOutput: async () => {
  const taskOutput = pendingHooks.get(processId)?.shellCommand?.taskOutput
  // ...
}
```

的原因。

---

### 第十部分：`emitHookResponse(...)` 除了 emit 事件，还无条件地把完整输出写到 debug 日志里

位置：`src/utils/hooks/hookEvents.ts:153`

```ts
const outputToLog = data.stdout || data.stderr || data.output
if (outputToLog) {
  logForDebugging(...)
}
```

这说明 response 事件在 debug 模式下总是可观测的，
即使 `shouldEmit` 最终返回 false。

事件是"给程序看的"
日志是"给人看的"

---

### 第十一部分：`clearHookEventState()` 说明这个模块的所有状态都是可重置的，支持测试或 session 切换时的清理

位置：`src/utils/hooks/hookEvents.ts:188`

```ts
eventHandler = null
pendingEvents.length = 0
allHookEventsEnabled = false
```

这保证了这个全局单例模块可以被完全清桩，
不会在测试间泄漏状态。

---

### 第十二部分：整份文件的架构价值在于把 hook lifecycle notification 从执行层中完全解耦出来

如果把整份文件压缩，会发现它其实在做四层事：

#### 1. 事件类型定义
- `started` / `progress` / `response`

#### 2. 发布-订阅机制
- `eventHandler` 单例
- `pendingEvents` 队列
- `registerHookEventHandler`

#### 3. 进度轮询
- `startHookProgressInterval`
- 差量去重
- `unref`

#### 4. 分级发射
- `ALWAYS_EMITTED_HOOK_EVENTS`
- `allHookEventsEnabled`
- `shouldEmit`

所以最准确的压缩表达是：

```text
hookEvents.ts = hook 生命周期事件广播层：定义 started/progress/response 三态事件，提供单订阅者 + pending 缓冲的 pub-sub 机制，带差量去重的进度轮询，以及 SessionStart/Setup 始终发射、其余 opt-in 的分级策略
```

---

### 元

问题：**这一站真正想解决的架构问题是什么？**

回答：这一站的元问题，是 hook 生命周期事件为什么需要一个正式广播层，而不是执行器直接通知 UI。started、progress、response 三态加上 pending 缓冲和 opt-in 发射策略，才让事件成为独立能力。

### 反

问题：**如果把这一站的设计反过来，会发生什么？**

回答：如果事件发射分散在各个执行器内部，handler 注册时机和事件去重都会变得混乱。最终最难收拾的，不是少发一条事件，而是整套可观察性根本不统一。

### 空

问题：**跳出当前文件名，这一站背后更大的问题是什么？**

回答：更大的问题，是 runtime 怎样为扩展点提供稳定的可观察接口。`hookEvents.ts` 的意义，是把 hook 的内部过程变成系统可以消费的公共信号。

### 读完这一站后，你应该抓住的 10 个事实

1. `src/utils/hooks/hookEvents.ts` 不是 hook 执行或跟踪层，而是 hook 生命周期事件的轻量 pub-sub 广播层。
2. Hook 执行对外暴露三类事件：`started`（开始）、`progress`（执行中，带增量输出）、`response`（结束，带 exit code 和 outcome）。
3. `ALWAYS_EMITTED_HOOK_EVENTS` 使 `SessionStart` 和 `Setup` 始终被发射，因为它们是低频生命周期事件且向后兼容；其余事件需要 `allHookEventsEnabled` 才发射。
4. `pendingEvents` 队列最多缓存 100 条事件，超出 oldest-first 丢弃，采用典型的环形缓冲策略。
5. `eventHandler` 是全局单例——同一时刻只有一个 active listener 消费 hook 事件。
6. `registerHookEventHandler(...)` 会在 handler 注册时 flush 所有 pending 事件，解决"事件先于 handler 到达"的时序问题。
7. `startHookProgressInterval(...)` 实现了带差量去重的进度轮询：只有 stdout/stderr 发生变化时才 emit progress，`interval.unref()` 防止 interval 阻碍进程退出。
8. `emitHookResponse(...)` 无条件写 debug 日志（给人看），有条件 emit 事件（给程序消费）。
9. `setAllHookEventsEnabled(...)` 在 SDK `includeHookEvents` 选项或远程模式下启用完整事件流，使事件广播层也充当 SDK/remote mode 的 hook 事件桥接。
10. `clearHookEventState()` 保证所有全局状态可被完全清桩，确保测试或 session 切换时不泄漏状态。

---

### 现在把第 73-74 站串起来

```text
src/utils/hooks/AsyncHookRegistry.ts
  -> async hook 的后台进程跟踪、轮询、结果提取与清理
src/utils/hooks/hookEvents.ts
  -> hook 生命周期事件的发射与广播，带 pending 队列和差量进度轮询
```

所以现在 hooks 子系统可以进一步压缩成：

```text
hooks.ts -> runs sync hooks and launches async hooks
AsyncHookRegistry.ts -> tracks async hook processes and extracts responses
hookEvents.ts -> broadcasts started/progress/response events to handlers
sessionHooks.ts -> manages session-scoped in-memory hooks
hooksSettings.ts -> normalizes multi-source hooks into unified records
hooksConfigManager.ts -> groups hooks by event/matcher with metadata
schemas/hooks.ts -> validates persisted hook config definitions
types/hooks.ts -> validates runtime hook protocol types
```

也就是说：

```text
AsyncHookRegistry.ts 回答"async 后台进程怎样被跟踪、轮询和解析"
hookEvents.ts 回答"hook 执行期间怎样广播 started/progress/response 事件给 UI/SDK/日志"
```

---

### 下一站建议

下一站最顺的是以下之一：

```text
src/utils/hooks/execPromptHook.ts
src/utils/hooks/execAgentHook.ts
src/utils/hooks/execHttpHook.ts
src/utils/hooks/hookHelpers.ts
```

前面 `hooks.ts` 已经展示了四大类非-command hook 的执行：

- prompt hooks (LLM)
- agent hooks (agentic verifier)
- http hooks (external API)
- function hooks (session-only callbacks)

其中 command hooks 的执行相对直白（通过 `ShellCommand`），
而这三种特殊执行器的细节（SSRF guard、env 变量注入、prompt 求值、agent 循环）都各自封装在独立文件里。

而 `AsyncHookRegistry.ts` 和 `hooks.ts` 都依赖过：

- http hook 执行
- prompt hook 执行
- agent hook 执行

所以下一步最自然就是把这三类特殊 hook 的执行器逐个补齐。最顺的第一站是：

**`src/utils/hooks/execHttpHook.ts` 到底怎样执行 HTTP hooks、注入环境变量、处理请求头、实现 SSRF 防护并与 hook 协议配合。**
