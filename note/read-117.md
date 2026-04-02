## 第 117 站：`src/tools/BriefTool/`（5 个文件，~224K）

### 这是什么

BriefTool（又名 `SendUserMessage`）是模型向用户发送消息的主要通道——支持纯文本消息和文件附件。在 assistant/assistant 模式下，这是模型回复用户的唯一可见输出通道——模型输出的其他文本对 TUI 用户不可见。

---

### 输入 Schema

```ts
{
  message: string                          // 必填：发给用户的消息
  attachments?: string[]                   // 可选：文件路径数组
  status: 'normal' | 'proactive'           // 回复式 vs 主动式
}
```

### 输出 Schema

```ts
{
  message: string
  attachments?: { path, size, isImage, file_uuid? }[]
  sentAt?: string                          // ISO 时间戳
}
```

---

### 双特性门

```ts
// 是否"有资格"使用 Brief
isBriefEntitled(): boolean {
  return feature('KAIROS') || feature('KAIROS_BRIEF')
    ? getKairosActive() || isEnvTruthy(CLAUDE_CODE_BRIEF) || getFeatureValue(...)
    : false
}

// 是否"激活"Brief（必须同时有资格 + 用户已 opt-in）
isBriefEnabled(): boolean {
  return feature('KAIROS') || feature('KAIROS_BRIEF')
    ? (getKairosActive() || getUserMsgOptIn()) && isBriefEntitled()
    : false
}
```

激活路径（用户 opt-in）：
- `--brief` CLI 标志
- `defaultView: 'chat'` 设置
- `/brief` 斜杠命令
- `--tools` SDK 选项
- `CLAUDE_CODE_BRIEF` 环境变量（开发测试）

Assistant 模式（`kairosActive`）绕过 opt-in——系统提示硬编码 "you MUST use SendUserMessage"。

---

### 为什么需要两个函数

`isBriefEntitled()` 决定用户 **是否被允许** 使用 Brief（构建时标志 + GB gate）。`isBriefEnabled()` 决定工具 **是否实际激活**（资格 + 用户 opt-in）。这样设计是因为：
- 工具注册需要知道是否跳过（`Tool.isEnabled()` 调用 `isBriefEnabled()`）
- 但 UI 入口点（`--brief` 标志、`defaultView` 选择器）需要在没有工具上下文时检查，所以用 `isBriefEntitled()`

GB gate `tengu_kairos_brief` 在会话中期关闭时会在 5 分钟刷新后禁用 Brief——这是作为 kill switch。

---

### 附件上传：动态导入模式

```ts
if (feature('BRIDGE_MODE')) {
  const { uploadBriefAttachment } = await import('./upload.js')
  // 并行上传所有附件
}
```

为什么动态导入？`upload.ts` 引入 `axios`、`crypto`、`zod`、OAuth 认证工具——如果静态引入，即使在非 BRIDGE_MODE 构建中也会被打包。动态导入让 Bun 可以从非 bridge 构建中完全 tree-shake 这些依赖。

---

### 上传流程

```
1. validateAttachmentPaths() — 验证所有附件文件存在且可读
2. resolveAttachments() — 串行 stat（保证顺序） + 并行上传
3. uploadBriefAttachment() — /api/oauth/file_upload
   - 只上传图片：image/png, jpg, jpeg, gif, webp → 后端生成预览/缩略图
   - 其他格式：octet-stream → 后端存储原始文件
   - 最大 30MB
4. 返回 { path, size, isImage, file_uuid? }
```

Upload 是 best-effort：任何失败（无 token、bridge 关闭、网络错误、4xx）静默返回 `undefined`，附件仍携带 `{path, size, isImage}` 用于本地渲染。

---

### Web 查看器集成

文件上传到 `/api/oauth/file_upload` 后，返回 `file_uuid`。Web 查看器通过 `file_uuid` 解析预览——因为本地文件路径对远程 Web 查看器没有意义。

桌面端/本地 TUI 通过 `path` 渲染，`file_uuid` 为 `undefined`。

---

### Proactive vs Normal 状态

```
'proactive': 模型主动发起——用户不在场时工作完成、遇到阻塞、需要输入
'normal': 回复用户刚才说的话
```

下游路由使用这个状态——proactive 消息可能走通知路径而 normal 消息直接显示在聊天中。

---

### Prompt 指导

BriefTool 的 prompt 中强调了一个关键 UX 问题：

```
SendUserMessage is where your replies go. Text outside is visible in the
detail view, but most won't open it — assume the answer is unseen here.

The real failure mode: the model puts the answer in plain text and only
says "done!" in SendUserMessage. The user sees "done!" and misses everything.
```

这防止模型犯一个常见错误：真正的回答放在普通文本中而 SendUserMessage 只说 "done"——用户只看到 "done" 而错过了回答。

---

### 输出 Schema 灵活性

```ts
attachments: optional     // 必须 optional — resumed sessions replay 旧输出
sentAt: optional          // 恢复会话 replay 时 sentAt 不存在
```

如果这些字段是 required 的，恢复会话 replay 时旧输出没有这些字段就会崩溃。

---

### 读完后应抓住的 4 个事实

1. **Brief 是唯一可见的用户输出通道**——在 assistant 模式下，模型的普通文本对用户不可见。模型必须通过 SendUserMessage 工具发送回复。这是 UX 安全的。

2. **双特性门设计**——`isBriefEntitled()` 检查资格（构建标志 + GB），`isBriefEnabled()` 检查资格 + 用户 opt-in。这样 UI 入口点和工具运行时检查分别工作而不冲突。

3. **附件是 best-effort 上传**——附件上传失败不会阻断消息，`file_uuid` 缺失时附件仍可携带 `{path, size, isImage}` 本地渲染。这是降级策略，网络问题不应该阻断用户看到消息。

4. **动态导入是关键的性能优化**——`upload.ts` 引入 axios、crypto、OAuth 工具，用 `await import()` 包裹让 Bun 可以在非 BRIDGE_MODE 构建中 tree-shake 整个模块。这是 Claude Code 构建系统的精细控制。
