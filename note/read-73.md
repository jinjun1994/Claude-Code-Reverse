## 第 73 站：`src/utils/hooks/AsyncHookRegistry.ts`

### 这是什么文件

`src/utils/hooks/AsyncHookRegistry.ts` 是 hooks 子系统里的 **异步 command hook 生命周期管理中心：负责登记后台运行的 hook 进程、周期性轮询 stdout 解析结果、在命令完成后提取 JSON 响应、清理已完成/被杀死的 hook、以及把最终结果通知给模型和 UI。**

上一站 `src/utils/hooks/sessionHooks.ts` 已经看懂 session-scoped hooks 的内存态管理。而在这之前的 `hooks.ts` 里已经看到 async hook 分支在 runtime 中非常重要；`schemas/hooks.ts` 中 `async` / `asyncRewake` 已作为可选字段进入持久化配置；`types/hooks.ts` 中 `AsyncHookJSONOutput` 也是正式协议的一部分。

这一站回答的是：

```text
当一条 hook 声明为 async 时，它在后台是怎样被
登记、跟踪、轮询、解析、清理和通知的
```

所以最准确的一句话是：

```text
AsyncHookRegistry.ts = async command hook 的后台进程层：负责把后台运行的 hook 进程纳入全局 registry，周期性扫描 stdout 中的 JSON 响应，在进程完成后解析输出并触发通知，支持 session/env 失效与优雅清理
```

---

### 先看它的整体定位

位置：`src/utils/hooks/AsyncHookRegistry.ts:1`

这个文件不是：

- hook 配置 schema
- hook 展示元数据
- hook settings loader
- session hook 注册器

它的职责是：

1. 把 `async: true` 的 hook 进程纳入全局 registry
2. 启动 progress interval 轮播提示
3. 周期性 `checkForAsyncHookResponses()` 扫描 stdout
4. 从 stdout 中提取第一条非-async JSON 作为最终响应
5. 清理完成的 hook（停止 progress、关闭 shell）
6. SessionStart 完成后触发 env cache 失效
7. 批量终了所有 pending hooks
8. 提供测试清桩工具函数

所以它本质上是一个：

```text
background hook process manager
```

也就是说：

- `hooks.ts` 负责**启动** async hooks
- `AsyncHookRegistry.ts` 负责**跟踪**它们的生命周期

---

### 第一部分：`PendingAsyncHook` 类型揭示了后台 hook 的完整跟踪元数据

位置：`src/utils/hooks/AsyncHookRegistry.ts:12`

包括：

- `processId`：底层 shell 进程的唯一标识
- `hookId`：hook 逻辑标识
- `hookName`：展示/调试用名
- `hookEvent`：触发事件
- `toolName`
- `pluginId`
- `startTime`
- `timeout`
- `command`
- `responseAttachmentSent`：响应是否已投递到消息流
- `shellCommand`：实际进程句柄
- `stopProgressInterval`：停止进度轮播

这说明 async hook 的跟踪语义不是"等它结束"那么简单，
而是需要同时关注：

1. 进程是否在跑
2. stdout 是否有 JSON 结果
3. 响应是否已交付给 UI/模型
4. 进度提示是否在轮询
5. 超时/清理机制

这是一个典型的"后台长任务跟踪模型"。

---

### 第二部分：`pendingHooks` 是个全局 `Map<processId, PendingAsyncHook>`，说明它是跨 session 的全进程单例注册表，而不是 per-session scoped

位置：`src/utils/hooks/AsyncHookRegistry.ts:28`

这和 session hooks 那边的 `Map<sessionId, SessionStore>` 完全不同。

async hook 进程层使用：

```ts
const pendingHooks = new Map<string, PendingAsyncHook>()
```

这意味着：

- 整个 CLI 进程共享一个 async hook 列表
- 所有 async hook 不论哪个 session 触发都在这里被跟踪
- 轮询 `checkForAsyncHookResponses()` 一次性扫描全部

这和 CLI 架构是一致的，因为：

- CLI 本身是单 session 进程
- 但 registry 设计没有硬绑 session 概念

---

### 第三部分：`registerPendingAsyncHook(...)` 是入口，它除了登记 registry，还立即启动了一个 `startHookProgressInterval(...)`，说明 async hooks 在后台运行时会持续推送 UI 进度提示

位置：`src/utils/hooks/AsyncHookRegistry.ts:30`

它做的事是：

1. 取 `asyncResponse.asyncTimeout` 或默认 15s
2. `startHookProgressInterval(...)` 启动一个周期性回调
3. 回调会从 `shellCommand.taskOutput` 中读取实时 stdout/stderr
4. 把进程注册进 `pendingHooks`

这说明 async hooks 不仅是在后台静默跑，
还会持续给 UI/日志层反馈：

```text
"hook X is still running, here's its latest output"
```

这是 async hook 用户体验的关键部分：因为模型不会立即等结果，
但用户需要知道它还在跑。

---

### 第四部分：`getPendingAsyncHooks()` 只返回 `responseAttachmentSent === false` 的 hook，说明 registry 虽然保存所有注册过的 hook，但外部视角只关心"还没交付结果"的那些

位置：`src/utils/hooks/AsyncHookRegistry.ts:85`

这说明 `responseAttachmentSent` 在这里扮演了一个：

```text
delivery sentinel
```

- `false`：还在 pending
- `true`：结果已处理，不再对外暴露

这让轮询方只需要关注活跃 hook 的响应。

---

### 第五部分：`finalizeHook(...)` 是清理中枢，它说明 hook 的生命周期结束时必须按固定顺序关闭：停 progress、读最终 stdout/stderr、清理 shell 句柄、然后 emit 结果事件

位置：`src/utils/hooks/AsyncHookRegistry.ts:91`

流程是：

1. `stopProgressInterval()`
2. 读 stdout / stderr
3. `shellCommand.cleanup()`
4. `emitHookResponse(...)`

这说明 async hook 结束时不是一个简单清理，
而是有一个正式的"结果发布"步骤。

`emitHookResponse` 把最终 output 发给：

- UI 展示层
- 日志
- 可能还有模型 transcript

所以 `finalizeHook` 是：

```text
cleanup + emit result event
```

---

### 第六部分：`checkForAsyncHookResponses()` 是这个文件真正的核心，它是一个周期性轮询函数，会扫描所有 pending hooks 的 stdout，提取已完成进程的 JSON 响应行

位置：`src/utils/hooks/AsyncHookRegistry.ts:113`

整个函数的工作流是：

**第一步：snapshot**

```ts
const hooks = Array.from(pendingHooks.values())
```

先拍快照再处理，
因为处理过程中会 `pendingHooks.delete(...)`。

**第二步：并发扫所有 hooks**

```ts
Promise.allSettled(hooks.map(async hook => { ... }))
```

每个 hook 依次检查：

1. 有 shell command 吗？
   - 没有 -> remove
2. shell status 是 `killed` 吗？
   - 是 -> remove
3. shell status 是 `completed` 吗？
   - 不是 -> skip
4. `responseAttachmentSent` 或 stdout 没内容？
   - 是 -> remove
5. 逐行扫描 stdout
6. 找到 JSON 行且不带 `"async": true` -> 作为响应
7. `responseAttachmentSent = true`
8. `finalizeHook(...)`
9. 返回 response payload

**第三步：聚合结果**

遍历 `allSettled` 结果：

- `remove` -> 删除 registry 条目
- `response` -> 推入 responses，删除 registry 条目
- 记录 SessionStart 完成标志

**第四步：后处理**

如果 SessionStart 有 async hook 完成了：

```ts
invalidateSessionEnvCache()
```

这说明 async hook 的 env-file 输出会影响后续 BashTool 环境，
所以完成后必须刷新 env cache。

最有趣的是：这个函数返回的是 sync 格式的钩子响应列表。
也就是说：

```text
async hook 在 "启动时不阻塞"
轮询函数 "完成后返回和 sync hook 一样的响应形态"
```

---

### 第七部分：JSON 行逐行扫描逻辑里 "line.trim().startsWith('{')" 再 jsonParse 的简单启发式方法暴露了 async hook 输出的实用主义设计

位置：`src/utils/hooks/AsyncHookRegistry.ts:183`

```ts
for (const line of lines) {
  if (line.trim().startsWith('{')) {
    try {
      const parsed = jsonParse(line.trim())
      if (!('async' in parsed)) {
        response = parsed
        break
      }
    } catch {
      // Failed to parse
    }
  }
}
```

这说明 async hook 的 stdout 可能包含：

- 非 JSON 的普通输出行
- `{"async": true, "asyncTimeout": ...}` 初始声明行
- 最终结果 JSON（不带 `async` 键）

所以这里的策略是：

```text
find the last non-async JSON line in stdout
```

如果脚本只是 echo 了一些非 JSON 内容，
那 JSON 解析失败就忽略，
最终返回空 response `{}`。

这非常实用主义——
不要求 hook 脚本严格遵循某种协议流，
只是"能抓到最后一条有效 JSON 就行"。

---

### 第八部分：`allSettled` 后的隔离策略很关键，说明即使某个 hook 的回调抛异常了，registry 也不会泄漏 side effect——已完成的 finalize 和 attachmentSent 不会被其他 hook 的错误阻断

位置：`src/utils/hooks/AsyncHookRegistry.ts:239`

注释说得很直白：

```text
allSettled — isolate failures so one throwing callback doesn't
orphan already-applied side effects from others
```

这说明：

- 某个 hook 可能在解析过程中抛出意外
- 但其他 hook 已经设置的 `responseAttachmentSent = true` 和 `finalizeHook` 必须保留
- 不用 `all` 而用 `allSettled`，确保一个失败不影响其他

这就是典型的后台进程轮询鲁棒性设计。

---

### 第九部分：`removeDeliveredAsyncHooks(...)` 只删除已交付的 hooks，说明外部清理 API 也使用 delivery sentinel 做保护

位置：`src/utils/hooks/AsyncHookRegistry.ts:270`

逻辑是：

```ts
if (hook && hook.responseAttachmentSent) {
  hook.stopProgressInterval()
  pendingHooks.delete(processId)
}
```

这说明外部传入 processId 列表时，
registry 不会暴力删除仍在跑的 hooks，
只有在"响应已交付"的前提下才做清理。

---

### 第十部分：`finalizePendingAsyncHooks()` 是终了函数，它说明在会话结束或被强制清理时，已完成和仍在运行的 hooks 分别走不同的终止路径

位置：`src/utils/hooks/AsyncHookRegistry.ts:281`

逻辑很清晰：

1. snapshot 所有 hooks
2. 对于 `completed` 的：读 exit code 并用其值 finalize
3. 对于非 `killed` 但仍在跑的：先 `kill()`，再 finalize 为 `cancelled`
4. 最后 `pendingHooks.clear()`

这说明 session 结束、用户退出等场景下：

- 完成的 async hooks 仍然要交付结果
- 仍在跑的 async hooks 必须被强制杀死并标记 cancelled

这是一种"优雅降级 + 清理"的终了策略。

---

### 第十一部分：`clearAllAsyncHooks()` 暴露为测试专用函数，说明这个 registry 设计时考虑到了测试场景下的全局状态隔离

位置：`src/utils/hooks/AsyncHookRegistry.ts:304`

注释说是：

```text
Test utility function to clear all hooks
```

它先 stop 所有 progress interval，再 clear map。

这说明全局 registry 的可测试性依赖于此函数——
否则测试间状态会泄漏。

---

### 第十二部分：`shellCommand` 的 role 很关键

`AsyncHookRegistry.ts` 里大量操作都围绕 `shellCommand`：

- `.status`
- `.taskOutput.getStdout()`
- `.result`
- `.cleanup()`
- `.kill()`

这说明整个 async hook 体系的实际进程句柄是 `ShellCommand`，
不是 Node child_process / spawn 直接操作的。

Registry 只是 `ShellCommand` 的管理层包装。

所以：

```text
ShellCommand = 底层进程抽象
AsyncHookRegistry = 后台 hook 进程的注册 + 轮询 + 解析层
```

---

### 第十三部分：SessionStart async hook 完成后的 env cache 失效很微妙

位置：`src/utils/hooks/AsyncHookRegistry.ts:257`

```ts
if (sessionStartCompleted) {
  invalidateSessionEnvCache()
}
```

这说明 SessionStart hooks 的 env-file 输出会改变后续 BashTool 的可用环境变量。

如果不在 async hook 完成后刷新 cache，
后续工具调用会用 stale env vars。

这表明 async hook 不只是 observability 或通知通道，
还可能实质影响后续会话行为。

---

### 第十四部分：async hook 的 `hookEvent` 类型包括 `'StatusLine' | 'FileSuggestion'` 而不仅是 `HookEvent`

位置：`src/utils/hooks/AsyncHookRegistry.ts:16`

```ts
hookEvent: HookEvent | 'StatusLine' | 'FileSuggestion'
```

这说明 AsyncHookRegistry 的用途可能不限于普通 hook events，
还可能被其他 UI/CLI 子系统借用：

- `StatusLine`：状态行更新
- `FileSuggestion`：文件建议

这说明这个文件的定位更像：

```text
generic background CLI process reporter + notifier
```

而不只限于 hooks 子系统内部使用。

---

### 第十五部分：整份文件最核心的架构价值，是把 async hook 的进程生命周期管理从同步执行模型中拆分出来，变成一个可并发轮询、可独立清理、可进度提示的后台进程层

如果把整份文件压缩，会发现它其实在做四层事：

#### 1. 登记层
- `registerPendingAsyncHook(...)`
- `pendingHooks` registry

#### 2. 轮询/解析层
- `checkForAsyncHookResponses()`
- JSON 行扫描
- `finalizeHook(...)`

#### 3. 清理/终了层
- `removeDeliveredAsyncHooks(...)`
- `finalizePendingAsyncHooks()`
- `clearAllAsyncHooks()`

#### 4. 交付层
- `emitHookResponse(...)`
- env cache 失效

所以最准确的压缩表达是：

```text
AsyncHookRegistry.ts = async hook 的后台进程跟踪层：负责登记后台运行的 shell hook，周期性扫描 stdout 中的 JSON 响应，在进程完成/失败/被杀死后清理资源并交付结果事件，并在 SessionStart async hooks 完成后刷新 env cache
```

---

### 读完这一站后，你应该抓住的 10 个事实

1. `src/utils/hooks/AsyncHookRegistry.ts` 不是 hook 配置或 schema 层，而是 async command hooks 的全局进程跟踪管理层。
2. `PendingAsyncHook` 类型完整描述后台 hook 的运行元数据：`processId`、`hookId`、`shellCommand`、`progress interval` 句柄、`responseAttachmentSent` sentinel 等。
3. `pendingHooks` 是全局 `Map<processId, PendingAsyncHook>` 单例，所有 async hook 不论哪个 session 触发都在这里被跟踪。
4. `registerPendingAsyncHook(...)` 登记一个 async hook 并立即启动 `startHookProgressInterval(...)`，使进程在后台持续推送进度提示。
5. `checkForAsyncHookResponses()` 是核心轮询函数：并发扫描所有 pending hooks，对 `completed` 进程逐行扫描 stdout，提取第一条非-async JSON 作为最终结果，然后 `finalizeHook` 清理并 `emitHookResponse` 发布事件。
6. stdout 扫描采用逐行查找 JSON 的实用主义策略，不要求严格协议流——只要能抓到最终结果 JSON 即可，解析失败则忽略。
7. `allSettled` 确保单个 hook 回调的异常不会阻塞其他 hook 的 finalize 和 attachmentSent 标记。
8. `finalizePendingAsyncHooks()` 在会话结束时区分已完成和仍在运行的 hooks：已完成交付结果，仍在跑的 kill + 标记 cancelled。
9. SessionStart async hook 完成后触发 `invalidateSessionEnvCache()`，说明 async hooks 可能实质影响后续 BashTool 的环境变量。
10. `hookEvent` 接受 `'StatusLine' | 'FileSuggestion'` 等非标准值，说明 AsyncHookRegistry 也可能被其他 UI/CLI 子系统借用作为通用后台进程通知层。

---

### 现在把第 72-73 站串起来

```text
src/utils/hooks/sessionHooks.ts
  -> session-scoped hook 内存态注册中心与投影层
src/utils/hooks/AsyncHookRegistry.ts
  -> async command hook 的全局进程跟踪、轮询和清理层
```

所以现在 hooks 子系统可以进一步压缩成：

```text
persisted hooks (settings)
  -> hooksSettings.ts normalizes
session hooks (in-memory)
  -> sessionHooks.ts registers and projects
runtime execution
  -> hooks.ts runs sync and launches async hooks
async tracking
  -> AsyncHookRegistry.ts polls and delivers async results
protocol
  -> types/hooks.ts validates output shapes
schema
  -> schemas/hooks.ts validates persisted definitions
```

也就是说：

```text
sessionHooks.ts 回答"session 内临时 hooks 怎样被注册、存储和分类"
AsyncHookRegistry.ts 回答"async 后台 hook 进程怎样被登记、轮询、解析和清理"
```

---

### 下一站建议

下一站最顺的是以下之一：

```text
src/utils/hooks/hookHelpers.ts
src/utils/hooks/hookEvents.ts
src/utils/hooks/execHttpHook.ts
src/utils/hooks/execPromptHook.ts
src/utils/hooks/execAgentHook.ts
```

因为现在你已经看懂了：

- 主执行器 `hooks.ts`
- 协议层 `types/hooks.ts`
- schema 层 `schemas/hooks.ts`
- 配置管理层 `hooksConfigManager.ts`
- 标准化数据层 `hooksSettings.ts`
- session hook 层 `sessionHooks.ts`
- async hook 跟踪层 `AsyncHookRegistry.ts`
- 但还差具体执行辅助函数：
  - `hookHelpers.ts`（执行中的工具函数）
  - `hookEvents.ts`（事件发射层）
  - 各种具体 hook 执行器（http/prompt/agent）

而 `AsyncHookRegistry.ts` 前面已经直接依赖：

- `emitHookResponse(...)` 来自 `hookEvents.ts`
- `startHookProgressInterval(...)` 来自 `hookEvents.ts`

所以下一步最自然就是把 hook 事件发射/通知层补齐：

**`src/utils/hooks/hookEvents.ts` 到底怎样发送 hook 事件、hook progress interval 怎样工作、以及 `emitHookResponse(...)` 怎样把最终结果通知给用户和系统。**
