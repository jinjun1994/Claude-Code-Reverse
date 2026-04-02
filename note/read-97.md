## 第 97 站：`src/tools/BashTool/BashTool.tsx`（第 1 部分）

### 这是什么文件

`BashTool.tsx`（1100+ 行+辅助文件共 12,411 行）是 Claude Code 最重要的工具之一：Shell 命令执行的完整实现，包括安全验证、权限检查、沙箱管理、进度追踪、输出截断和图片处理。

---

### 输入 Schema

```ts
{
  command: string              // 必填
  timeout?: number             // 超时（ms）
  description?: string         // 命令用途说明
  run_in_background?: boolean  // 后台运行
  dangerouslyDisableSandbox?: boolean  // 禁用沙箱
  _simulatedSedEdit?: { filePath, newContent }  // 内部：sed 预计算结果
}
```

`_simulatedSedEdit` 是内部字段——用户确认 sed edit 预览后设置。从模型 schema 中排除（`.omit()`），否则模型可以配对无害命令与任意文件写入来绕过权限检查。

---

### 核心执行链

#### 1. Sed edit 模拟

```ts
if (input._simulatedSedEdit) {
  return applySedEdit(input._simulatedSedEdit, ...) // 直接写入，不走 shell
}
```
这确保用户预览的 sed 编辑结果与实际写入的内容完全一致。

#### 2. `runShellCommand()` 生成器

`runShellCommand()` 是一个 async generator（`AsyncGenerator<{progress}, ExecResult>`）：

```
spawnShellTask/start shell command
→ onProgress callback → resolve progressSignal
→ Promise.race([resultPromise, progressSignal])
→ yield progress 或 return result
```

进度信号通过 Promise 解析机制传播：`exec()` 的 `onProgress` 回调解析 `resolveProgress()`，唤醒生成器的 `await Promise.race()`。

#### 3. 三阶段执行

**阶段 1**: 初始等待（`PROGRESS_THRESHOLD_MS` = 2,000ms）
```ts
await Promise.race([resultPromise, new Promise(resolve => setTimeout(resolve, 2000))])
```
2 秒内完成的命令直接返回结果，无 UI 进度显示。

**阶段 2**: 进度轮询
```ts
TaskOutput.startPolling(shellCommand.taskOutput.taskId)
while (true) {
  await Promise.race([resultPromise, progressSignal])
  // yield progress
}
```

**阶段 3**: 收尾处理
- stdout 通过 `EndTruncatingAccumulator` 累计（超限截断）
- 合并 fd 的 stderr 已含在 stdout 中
- `interpretCommandResult()` 提供语义解释
- 大输出持久化到 `tool-results/` 目录

---

### 后台化机制

BashTool 支持多种后台化触发路径：

| 触发方式 | 条件 | 说明 |
|---|---|---|
| 显式请求 | `run_in_background: true` | 始终优先处理 |
| 超时触发 | `shellCommand.onTimeout()` | 默认超时时自动后台化 |
| 助手模式 | Kairos enabled && 15s budget | assistant 模式 15s 后自动后台化 |
| 用户操作 | Ctrl+B | 用户手动后台化 |

`sleep` 命令被排除在自动后台化之外（`DISALLOWED_AUTO_BACKGROUND_COMMANDS`），因为它是常见的轮询模式基础。

---

### 输出处理

#### 截断机制
`EndTruncatingAccumulator`—尾部截断累加器，保留输出首尾内容，中间部分被截断。

#### 大输出持久化
当输出文件超过 `getMaxOutputLength()` 时：
1. 复制到 `tool-results/` 目录
2. 超过 64MB 则先截断
3. 模型通过 `<persisted-output>` XML 标签获知文件路径
4. UI 不受影响——使用原始 stdout 数据

#### 图片输出
检测 stdout 中是否包含图片数据（base64 编码），解码后限制尺寸并通过 `buildImageToolResult()` 格式化为图片内容块。

---

### 命令语义分类

| 分类 | 命令集 | 用途 |
|---|---|---|
| 搜索 | find, grep, rg, ag, ack, locate, which, whereis |  collapsible search |
| 读取 | cat, head, tail, less, more, wc, stat, file, jq, awk, cut, sort, uniq, tr | collapsible read |
| 列表 | ls, tree, du | collapsible directory list |
| 静默 | mv, cp, rm, mkdir, touch, ln, cd, export | 成功时无输出 |
| 中性 | echo, printf, true, false, : | 纯输出/状态命令 |
| 禁止自动后台 | sleep | 轮询模式常用 |

---

### `isSearchOrReadBashCommand()`

用于判定命令是否可折叠显示：
1. 解析复合命令为带操作符的部分
2. 跳过重定向目标（`>`, `>>`, `>&` 后面的部分）
3. 跳过语义中性命令（echo, printf）
4. 如果所有部分都是 search/read/list，则判定为可折叠
5. 只有 `tool_result` 消息才计数（避免 `tool_use` 和 `tool_result` 双倍计数）

---

### 权限与 hook 匹配

#### `preparePermissionMatcher()`
```ts
const parsed = await parseForSecurity(command)
// 复合命令：任一子命令匹配 → 触发 hook
return pattern => subcommands.some(cmd => matchWildcardPattern(pattern, cmd))
```
这防止 `ls && git push` 绕过 `Bash(git *)` 安全 hook。

#### `checkPermissions()`
委托给 `bashToolHasPermission()`——在 `bashPermissions.ts` 中实现。

#### `isReadOnly()`
委托给 `checkReadOnlyConstraints()`——在 `readOnlyValidation.ts` 中实现。

---

### 安全验证

#### Input 验证
```ts
validateInput() {
  // 检测阻塞的 sleep 模式
  if (MONITOR_TOOL && !isBackgroundTasksDisabled && !run_in_background) {
    if (detectBlockedSleepPattern(command)) {
      return { result: false, message: "...", errorCode: 10 }
    }
  }
}
```

`detectBlockedSleepPattern()` 识别：
- `sleep N`（N ≥ 2）—— 独立的整数秒等待
- 但 `sleep 0.5` 允许（真正的速率限制/延迟）
- `sleep 5 && check` —— 建议用 Monitor tool

---

### `applySedEdit()` 模拟写入

直接写入文件而不是运行 sed 命令：
1. 读取原始内容（VS Code 通知用）
2. 跟踪文件历史（undo 支持）
3. 检测行尾格式
4. 写入新内容
5. 通知 VS Code
6. 更新文件缓存
7. 返回与 sed 相同的输出格式

---

### 元

问题：**这一站真正想解决的架构问题是什么？**

回答：这一站真正要回答的是，为什么 BashTool 会成为 Claude Code 最关键也最重的工具之一。因为它不只是执行 shell，而是同时承接安全验证、权限检查、沙箱、进度、截断和富结果处理。

### 反

问题：**如果把这一站的设计反过来，会发生什么？**

回答：如果把 Bash 当成一个轻薄命令封装器，系统最先失守的就是安全与可控性。越通用的工具越不能靠“执行后再说”，必须在执行前就有层层约束。

### 空

问题：**跳出当前文件名，这一站背后更大的问题是什么？**

回答：更大的问题，是平台怎样把最危险也最强大的通用工具纳入可信执行框架。BashTool 的复杂度，本身就是这个问题的答案。

### 读完这一站后，你应该抓住的 6 个事实

1. BashTool 的执行通过 async generator 实现——`runShellCommand()` 是一个可以 yield 进度更新并在完成时返回结果的 async generator。进度信号通过 Promise 解析机制从 `exec()` 的回调传播到生成器的 `Promise.race()`。
2. 所有后台化路径（显式/超时/助手模式/用户操作）最终都通过 `startBackgrounding()` 统一处理。`spawnBackgroundTask()` 创建后台 shell 任务并返回 taskId。如果前台任务已注册，优先用 `backgroundExistingForegroundTask()` 原地转换而非重新 spawn。
3. Assistant 模式有 15 秒的自动后台化预算——阻塞命令超过这个时间被自动移到后台。这是为了保持 assistant 会话的响应性。`sleep` 命令被排除在自动后台化之外。
4. 输出处理中 stderr 已被合并到 stdout（merged fd）——Shell 使用合并的文件描述符。这意味着错误消息不在 `result.stderr` 中，而在 `result.stdout` 中。`interpretCommandResult()` 通过语义规则从 stdout 中识别错误。
5. `_simulatedSedEdit` 是内部字段且从模型 schema 中排除。当用户确认 sed 编辑预览后，这个字段被设置，工具直接写入文件而不是运行 sed。这是确保预览与实际写入完全一致的关键安全机制——如果模型能看到这个字段，它可能配对无害命令与任意文件写入。
6. 大输出持久化使用 hard link（link）优先，失败才 copyFile——这避免了 64MB+ 文件的双重磁盘占用。UI 使用原始 stdout 数据（不受持久化影响），而模型通过 `<persisted-output>` 标签获得一个可读取的文件路径。
