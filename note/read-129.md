## 第 147 站：MCP 命令子命令（4 个文件）

### 这是什么

MCP 管理面——TUI 交互命令（`/mcp`）和终端 CLI 命令（`claude mcp add/xaa`）两套入口。

---

### 两套入口

**Slash 命令路径**：
- `index.ts` → Command 描述符（`type: 'local-jsx'`）
- `mcp.tsx` → React 组件分发器（`MCPSettings`, `MCPReconnect`, `MCPToggle`）
- 状态通过 `useAppState` 和 `useMcpToggleEnabled` 管理

**终端 CLI 路径**：
- `addCommand.ts` → `claude mcp add <name> <command> [args...]`
- `xaaIdpCommand.ts` → `claude mcp xaa setup/login/show/clear`

---

### MCPToggle 组件

```ts
// 读取 MCP 客户端列表 → 过滤 ide 特殊客户端 → 批量切换
const toToggle = target === 'all'
  ? clients.filter(c => isEnabling ? c.type === 'disabled' : c.type !== 'disabled')
  : clients.filter(c => c.name === target)
for (const s of toToggle) toggleMcpServer(s.name)
```

`toggleMcpServer` 实现来自 `services/mcp/MCPConnectionManager.js`——不是 React 组件内部逻辑。

---

### `claude mcp add`

三个传输分支：`stdio`（默认）、`sse`、`http`。

**Guard 模式**：
- URL 检测警告（用户可能忘加 `--transport`）
- OAuth + stdio 产生警告（stdIO 不支持 OAuth）
- XAA 添加时验证前置条件（非认证时）

---

### XAA IdP 管理

**四步命令**：
- `setup`——配置 OIDC IdP 连接
- `login`——通过 OIDC 浏览器流缓存 id_token
- `show`——显示当前 IdP 配置
- `clear`——移除所有 IdP 配置和密钥

**Stale keychain 清理模式**——旧 vs 新 diff：
```ts
if (issuerKey(oldIssuer) !== issuerKey(options.issuer)) {
  clearIdpIdToken(oldIssuer)
  clearIdpClientSecret(oldIssuer)
}
```

`issuerKey()` 规范化尾部斜杠和 host 大小写差异——防止相同 IdP 被误判为不同。

**存储分层**：
- 非密钥配置（issuer, clientId, callbackPort）→ settings.xaaIdp
- 密钥（client secret, id_token）→ OS keychain，按规范化 issuer 键控

**安全写入**：所有验证先完成 → 写入设置 → 清除 keychain。不部分写入。

---

### 元

问题：**这一站真正想解决的架构问题是什么？**

回答：这一站讲的是 `/mcp` 与 `claude mcp ...` 两套管理入口如何并行存在，一个偏 TUI 交互，一个偏终端配置。重点不在执行 MCP 工具，而在管理面如何为连接、开关和 XAA 身份提供操作界面。

### 反

问题：**如果把这一站的设计反过来，会发生什么？**

回答：如果把所有 MCP 管理都塞进单一路径，要么命令行用户太笨重，要么 TUI 用户缺少可视入口。管理面一旦单薄，MCP 扩展能力再强也会因为难管理而难落地。

### 空

问题：**跳出当前文件名，这一站背后更大的问题是什么？**

回答：更大的问题是，协议生态不仅需要运行时接入，也需要运维面。147 站提醒我们，扩展系统成熟与否，往往取决于管理命令是否同样成体系。

### 读完后应抓住的 2 个事实

1. **两套平行入口**——`/mcp`（TUI JSX）vs `claude mcp add`（终端 CLI 直接写入 config）。数据流：terminal add → config 文件 → useAppState → slash command 读取。

2. **XAA 验证写入顺序**——所有验证 → 写设置 → 清 keychain。先验证全部，不部分状态。Stale keychain 通过旧新 diff 清理——Issuer 或 clientId 变更时自动清理旧密钥。

---

## 第 148 站：权限 Hook Handler 链（4 个文件，~1.5K）

### 这是什么

权限决策的责任链——三个 handler 按顺序尝试，返回 null 的传给下一个。协调器、swarm 工人、交互模式各有各的处理方式。

---

### 责任链

```
[Incoming tool execution request]
        |
        v
handleCoordinatorPermission  (协调器 worker)
  ├── Await hooks（依次）
  ├── 尝试 classifier（仅 bash）
  └── 返回 null → fall through
        |
handleSwarmWorkerPermission  (swarm worker)
  ├── 本地 classifier（awaited）
  ├── 注册回调 BEFORE 发送请求（防止 leader 先回复）
  └── 返回 null → fall through
        |
handleInteractivePermission  (main agent)
  ├── 立即 push 到 ToolUseConfirm 队列
  ├── 四路 racer：bridge / channel relay / hooks / classifier
  └── claim() 保证只有一方获胜
```

---

### coordinatorHandler.ts

**Sequential check**——Coordinator 是父进程，不需要转发。先 await hooks，再尝试 classifier。任一返回决定就短路。

**Safe degradation**——自动化检查中捕获错误 → log → fall through 到交互对话框。

---

### swarmWorkerHandler.ts

**Race prevention 模式**——先注册回调，再发送请求：
```ts
// 注册在前，避免 leader 在回调就绪前响应
registerPermissionCallback({ onAllow, onReject })
sendPermissionRequestViaMailbox(request)
```

**不可显示 UI**——Worker 没有本地 UI，权限请求通过 mailbox 转发给 coordinator。Coordinator 的 ToolUseConfirmQueue 显示标准 UI（BashPermissionRequest、FileEditToolDiff 等）。

---

### interactiveHandler.ts —— 最复杂的 handler

**多路 Racer 系统**：

| Racer | 来源 | 何时启用 |
|---|---|---|
| Bridge (CCR/claude.ai) | 有 `bridgeCallbacks` | CCR web UI 连接时 |
| Channel relay | Telegram/iMessage 等 | `KAIROS_CHANNELS` 开启 |
| Permission hooks | 异步，后台 | `!awaitAutomatedChecksBeforeDialog` |
| Bash classifier | 异步，后台 | 仅 bash 且有 `pendingClassifierCheck` |

**`claim()` 原子守卫**：`createResolveOnce` 返回的 `claim()` 只在第一次调用时返回 true。所有 racer 路径都先检查 `claim()`。

**用户交互打断 classifier**：
```ts
onUserInteraction() {
  const GRACE_PERIOD_MS = 200
  if (Date.now() - permissionPromptStartTimeMs < GRACE_PERIOD_MS) return
  userInteracted = true
  clearClassifierChecking(ctx.toolUseID)
}
```

用户开始交互后 200ms 宽限期外，classifier 被 kill。但如果用户在 200ms 内就交互，classifier 仍然运行完成（避免闪退）。

**Checkmark 显示模式**：classifier 自动批准时显示 3s（终端聚焦）或 1s 的 checkmark，用户可 Esc 提前关闭。

**Channel relay 的 unsubscriber 修复**：
```ts
channelUnsubscribe = () => {
  mapUnsub()  // 清理订阅映射
  channelSignal.removeEventListener('abort', channelUnsubscribe!)  // 清理事件监听
}
```
包裹式清理修复了之前的闭包泄露。

---

### autoModeState.ts

**三角标志**：
```ts
autoModeActive          // 是否在 auto 模式中
autoModeFlagCli         // 命令行是否传了 --auto
autoModeCircuitBroken   // GrowthBook 是否硬禁用了 auto（进入后）
```

**Circuit breaker 逻辑**：GrowthBook 返回 `tengu_auto_mode_config.enabled === 'disabled'` 时设置 `autoModeCircuitBroken = true`。一旦触发，`isAutoModeGateEnabled()` 阻止 SDK 和显式重新进入。

---

### denialTracking.ts

**双重限制**：
```ts
maxConsecutive: 3     // 连续 3 次拒绝 → fallback
maxTotal: 20          // 累计 20 次拒绝 → fallback
```

**成功不完全重置**：`recordSuccess` 只重置 `consecutiveDenials = 0`。`totalDenials` 不递减——早期 burst 的拒绝仍可能后期累计超限。

---

### 元

问题：**这一站真正想解决的架构问题是什么？**

回答：这一站进一步细化权限请求在协调器、swarm worker 与交互主会话中的处理链，说明“权限”不是一个同步 if 判断，而是一条带竞争者的责任链。尤其 interactive handler 的多路 racer，把 bridge、channel、hooks 和 classifier 都纳入同一仲裁点。

### 反

问题：**如果把这一站的设计反过来，会发生什么？**

回答：如果这些来源没有统一 claim 机制，最先出现的就是多方同时决策、重复响应或晚到结果覆盖早到结果。权限系统会从稳重闸门变成竞速现场。

### 空

问题：**跳出当前文件名，这一站背后更大的问题是什么？**

回答：更大的问题是，在多入口、多代理、多自动化检查并存时，最终由谁来做权限决定。148 站给出的答案不是某个单点，而是一条可短路、可落空、但只能赢一次的链条。

### 读完后应抓住的 4 个事实

1. **三 handler 责任链**——coordinator（seq check）→ swarm（mailbox forward）→ interactive（multi-racer）。null 返回表示 fall through，composable。

2. **Interactive 的四路 racer**——bridge / channel relay / hooks / classifier 并行竞争。`claim()` 原子守卫保证只有一个决定生效。用户交互 200ms 宽限期外 kill classifier。

3. **Swarm worker 先注册回调再发请求**——防止 leader 在回调就绪前已经响应。这是 mailbox 异步通信的 race condition 修复。

4. **Denial 双重限制 + 不完全重置**——连续 3 和累计 20 两个阈值。Success 只重置 consecutive，total 保持不变——早期拒绝 burst 仍可能后期触发 fallback。
