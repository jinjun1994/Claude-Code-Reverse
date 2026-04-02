## 第 153 站：API 层与服务子系统（5 个目录，~4K）

### 这是什么

API 服务层处理多后端认证、流式、重试逻辑和错误恢复。AutoDream 和 ExtractMemories 是后台记忆服务。ToolUseSummary 生成 Haiku 工具摘要。

---

### API 服务层

#### withRetry.ts —— 核心重试引擎

**异步生成器模式**：
```ts
export async function* withRetry<T>(
  getClient: () => Promise<Anthropic>,
  operation: (client, attempt) => Promise<T>,
  options: RetryOptions,
): AsyncGenerator<SystemAPIErrorMessage, T>
```

**关键决策矩阵**：

| 场景 | 行为 |
|---|---|
| 前台 529 | 重试（来自 `FOREGROUND_529_RETRY_SOURCES`） |
| 后台 529 | 立即退出——防止 3-10x 网关放大 |
| Fast mode 429/529 | 短重试；超时则进入冷却（降到标准模型，30min hold） |
| 连续 3 个 529 | 模型回退（Opus → Sonnet） |
| `CLAUDE_CODE_UNATTENDED_RETRY` | 无限重试，5min 最大退避，6hr 重置，30s 心跳 |

**后台不重试 529 的设计原因**：摘要、标题、非安全 classifer 等后台调用在拥塞级联时重试会造成 3-10x 的网关放大。立即退出。

**`x-should-retry` header**——服务端可控的重试信号。

**订阅类型检查**——Enterprise/PAYG 可重试，ClaudeAI 订阅者不能。

---

#### claude.ts —— 主 API 调用

**Streaming 流式**：`queryModelWithStreaming()` → `withStreamingVCR`（prompt cache 系统）→ `queryModel()`。

**流式 529 回退**：流式失败 → `executeNonStreamingRequest()` → `withRetry` + timeout（远端 120s / 本地 300s）。

**VCR**——prompt cache 系统，`withStreamingVCR` 封装。

---

#### client.ts —— 多后端客户端

**4 种后端**：First-party API、AWS Bedrock、Azure Foundry、Google Vertex。

每种认证不同：OAuth tokens / API keys / AWS credentials / Azure AD / GoogleAuth。

**Headers**：`x-claude-remote-container-id`、`x-client-request-id`（超时关联）、`x-anthropic-additional-protection`。

---

### AutoDream —— 后台记忆整合

**6 级门控**（从便宜到贵）：
```
1. Enabled gate（非 KAIROS、非 remote、auto-memory 开启、feature flag）
2. Time gate（距上次整合 >= minHours，默认 24h）
3. Scan throttle（每 10 分钟最多扫描一次）
4. Session gate（mtime > lastConsolidatedAt 的 transcript 数 >= minSessions，默认 5）
5. Lock 检查（PID 存活检测）
6. 实际 forked agent 执行
```

**Lock 文件设计**：
- 路径：memory 目录
- mtime = `lastConsolidatedAt`
- body = PID
- 死进程检测：PID 不存活 → 60min 后可以回收
- 崩溃恢复：回滚重写 mtime 到 acquire 前的值

**四阶段 prompt**：
1. Orient（ls/read 现有文件）
2. Gather（review logs, grep transcripts）
3. Consolidate（update/create memory files）
4. Prune（trim MEMORY.md 到 25KB）

---

### Extract Memories —— Transcript 记忆提取

**Forked agent，restricted tools**：
```
Read/Grep/Glob: 无限制（只读）
Bash: 仅 `isReadOnly()` 通过
Edit/Write: 仅 memory 目录路径
其他: 拒绝
```

**互斥设计**：`hasMemoryWritesSince()` 检查主代理是否写了 memory 文件——如果是，跳过提取并将游标移到该范围之后。防止重复工作。

**闭包状态**（`initExtractMemories()` 初始化）：
- `lastMemoryMessageUuid`：游标，跟踪已处理消息
- `inProgress`：防止重叠运行
- `pendingContext`：进行中运行的尾部提取缓存
- `turnsSinceLastExtraction`：节流（flag `tengu_bramble_lintel`，默认 1）

---

### ToolUseSummary —— Haiku 工具摘要

**目的**：为完成的工具批次生成短标签（约 30 字符），显示在移动客户端的进度行。

**Prompt 要求**：git commit subject 风格、过去时、动词+名词独特的。

示例：`"Searched in auth/"`, `"Fixed NPE in UserService"`, `"Read config.json"`

**截断**：工具输入/输出各截断 300 字符。

**静默失败**——记录错误但永不阻塞主查询。

---

### 读完后应抓住的 4 个事实

1. **后台调用不重试 529**——防止拥塞级联时 3-10x 网关放大。这是来自真实生产事件的设计——摘要/标题/non-security classifier 在拥塞期间重试会放大问题。

2. **AutoDream 6 级门控**——从便宜检查（enabled flag）到贵操作（forked agent）排列，尽早退出。Lock 文件用 PID 检测和 mtime 时间戳防止死锁和重复运行。

3. **ExtractMemory 与主代理互斥**——`hasMemoryWritesSince()` 防止提取器覆盖主代理刚写的 memory。提取的 forked agent 只能读写 memory 目录和只读工具。

4. **连续 3 个 529 触发模型回退**——Opus → Sonnet。这是生产级的降级设计——当 API 持续拥塞时，自动降级到更快的模型而不是无限等待。
