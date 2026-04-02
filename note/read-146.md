## 第 201 站：Transport 层完整架构

### 四层 Transport 架构

**总览**——`cli/transports/` 和 `cli/structuredIO.ts` 构成完整的 client-server 通信栈。

```
StructuredIO (SDK protocol bridge)
  ├── HybridTransport (WS reads + HTTP POST writes)
  │     ├── SerialBatchEventUploader (batched, serialized writes)
  │     └── WebSocketTransport (auto-reconnect, keepalive)
  ├── SSETransport (server-sent events, receive-only)
  └── ccrClient.ts (CCR client protocol)
```

---

### WebSocketTransport——可靠的 WebSocket 通道

**状态机**——`idle → connected → reconnecting → closing → closed` 五状态。

**自动重连策略**：
- 指数退避：base 1s，max 30s
- 600s (10 分钟) 总时间预算后放弃
- **睡眠检测**——重连间隔 > 60s 视为系统睡眠/唤醒，重置预算（server 可能已 reaped session）
- **永久关闭码**——`1002`（协议错误）、`4001`（session expired/not found）、`4003`（unauthorized）立即停止重连

**Keepalive 机制**：
- `KEEP_ALIVE_FRAME = '{"type":"keep_alive"}\n'`——每 5 分钟发送一次
- `PING_INTERVAL = 10s`——WebSocket 级 ping
- `lastSentId` tracking——用于重连后的消息去重

**消息顺序保证**——`lastId` 用于检测乱序到达的消息。`CircularBuffer` 用于重连期间的消息缓冲（max 1000 条）。

---

### HybridTransport——读/写分离

**为何分离**——Bridge mode 通过 `void transport.write()` fire-and-forget。如果直接并发 POST → 并发 Firestore 写同一文档 → 冲突 → 重试风暴 → 告警。`SerialBatchEventUploader` 序列化写入。

**架构**：
- **读路径**——继承 `WebSocketTransport`（WS 接收消息）
- **写路径**——`SerialBatchEventUploader` 批量 POST

**`stream_event` 缓冲**——内容 delta 累积 100ms 再 enqueue，减少 POST 次数。非 stream 写入时先 flush buffer 保证顺序。

**POST 配置**：
- `BATCH_FLUSH_INTERVAL_MS = 100`
- `POST_TIMEOUT_MS = 15000`
- `CLOSE_GRACE_MS = 3000`——shutdown 时最佳努力排空
- `maxBatchSize = 500`——session-ingress 接受任意大小

**`convertWsUrlToPostUrl()`**——从 WebSocket URL 推导对应的 HTTP POST URL。

---

### SerialBatchEventUploader——序列化批量写入引擎

**设计目标**——保证最多 1 个 POST 在飞行，事件在队列中序列化，有指数退避和反压。

**关键行为**：
- `enqueue()`——添加到 pending buffer，触发 `drain()` 如果没有在飞行
- `flush()`——阻塞直到 pending 为空
- **反压**——队列满时 `enqueue()` 返回 Promise 阻塞调用者
- **指数退避**——`baseDelayMs → maxDelayMs`，带 `jitterMs` 抖动
- **RetryableError**——特殊错误类型携带 `retryAfterMs`（如 429 Retry-After），覆盖默认退避
- **maxConsecutiveFailures**——超过则丢弃批次并继续，不调用者不卡死

**`RetryableError` 的安全机制**——`retryAfterMs` 被 clamp 到 `[baseDelayMs, maxDelayMs]` 范围并加抖动，防止失控循环或卡住客户端。多个会话共享限速时避免同时重传。

---

### SSETransport——单向接收

**只接收通道**——用于不需要写路径的场景（如 print/headless 模式的部分功能）。

---

### StructuredIO——SDK 协议层

**NDJSON 流解析**——`read()` async generator 处理 `\n` 分隔的 JSON 行。`splitAndProcess` 在 for-await 之前和每个 block 后都调用，确保 `prependedLines` 插入在同一 block 的两个消息之间也能正确处理。

**`replayUserMessages` 标志**——当为 true 时，control_response 会 yield 回主循环（print mode 需要）；否则静默处理。

**`sendRequest()` 模式**——创建 `PendingRequest`（resolve/reject/schema/request），注册到 `pendingRequests` Map，通过 `outbound` Stream 排队写入。

**`outbound` Stream**——`sendRequest()` 和 `print.ts` 都从这里排队。drain loop 是唯一的 writer。防止 `control_request` 抢占排队的 `stream_event`。

**注入控制响应**——`injectControlResponse()` 供 bridge 使用，将 claude.ai 的权限决议注入 SDK 权限流。同时发送 `control_cancel_request` 取消 SDK consumer 的 `canUseTool` callback。

**Sandbox 网络权限**——`createSandboxAskCallback()` 复用 `can_use_tool` 协议通过 `SandboxNetworkAccess` 合成工具名。不需要新的协议 subtype。

---

### 元

问题：**这一站真正想解决的架构问题是什么？**

回答：这一站真正要回答的是：**Claude Code 一旦跨出本地进程，消息到底怎样才能可靠地发出去、收回来、保持顺序，并在中断后继续活着。** 所以这里讲的不是某个 WebSocket 类怎么写，而是整个 Transport 层为什么必须独立：WebSocketTransport 负责连接生命周期，HybridTransport 处理读写分离，SerialBatchEventUploader 解决并发写冲突，StructuredIO 则把这些通信细节接回 Claude Code 自己的协议世界。

### 反

问题：**如果把这一站的设计反过来，会发生什么？**

回答：如果反过来做，不单独建立 Transport 层，而让上层业务自己关心重连、批量写入、消息去重、stdout 排队，那最先失控的就是时序。Bridge mode 的并发写冲突、sleep 之后的重连预算、`control_request` 和 `stream_event` 的先后关系，都会变成局部补丁。通信问题最可怕的地方就在这里：平时像细节，出事时却会一起爆。

### 空

问题：**跳出当前文件名，这一站背后更大的问题是什么？**

回答：跳出这些 transport 实现之后，更大的问题其实是：**一个会话型 AI 系统，怎样跨进程、跨网络、跨协议地持续保持“像同一个会话”那样工作。** 具体传输方式以后可以变，但可靠传输、顺序保证、恢复能力和协议桥接，永远都会是远程 Claude Code 的基础问题。

### 读完后应抓住的 3 个事实

1. **HybridTransport 的写序列化解决并发写冲突**——Bridge mode 的 `void transport.write()` fire-and-forget 导致并发 Firestore 文档写冲突。`SerialBatchEventUploader` 保证最多 1 POST 在飞行，事件排队等待。这是生产级修复——来自真实的 oncall 重试风暴事件。

2. **睡眠检测保护**——`SLEEP_DETECTION_THRESHOLD_MS = 60s`——如果重连间隔超 60s 视为系统睡眠/唤醒，重置重连预算而不是继续指数退避。server 可能在睡眠期间 reaped session，会用永久关闭码（1002/4001）拒绝。这是可靠性设计。

3. **StructuredIO 的 `outbound` Stream**——所有输出都经过 `outbound` Stream 排队。drain loop 是唯一的 stdout writer。这防止了 `control_request` 从 `sendRequest()` 覆盖 `stream_event` 从 `print.ts`。这是消息顺序保证设计。
