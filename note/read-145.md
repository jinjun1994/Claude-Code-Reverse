## 第 200 站：Bootstrap State + Structured IO + Upstream Proxy + Vim Mode

### bootstrap/state.ts —— 全局状态管理核心

**1,758 行的全局状态管理核心**——整个代码库中最重要的单文件之一，定义 `State` 类型和所有 state accessors。

**核心设计原则**——文件顶部注释 `// DO NOT ADD MORE STATE HERE - BE JUDICIOUS WITH GLOBAL STATE`。循环依赖防护通过 dependency injection（如 `scratchpadDir` 作为参数传入而不是 `import`）。

**State 类型核心字段**：
- `projectRoot` / `originalCwd` / `cwd`——稳定的项目根标识（不受 EnterWorktreeTool 影响）用于历史/skills/sessions 身份识别
- `sessionId` / `sessionMode`——会话身份标识和模式
- `totalCostUSD` / `totalAPIDuration` / `totalToolDuration`——成本和工作追踪
- `turn*` 系列——轮次级别计数器（tool count/hook count/classifier count/duration）
- `modelUsage`——按模型名追踪 token/成本
- `channels`——插件/服务器注册表（plugin/server allowlist with dev-channel gating）
- Telemetry 组件——`meter`/`sessionCounter`/`costCounter`/`tokenCounter` 等 OpenTelemetry 指标
- `strictToolResultPairing`——HFI 严格模式：工具结果不匹配时抛出而非用合成占位修复

**访问模式**——通过 `createSignal()` 的发布/订阅。9 个信号（signal）用于 UI 订阅：
- `onSessionStateChanged`、`onProjectRootUpdate`、`onSessionIdUpdate`
- `onLastInteractionTime`、`onPermissionModeChanged`
- `onStatsStoreAvailable`、`onUserMsgOptIn`、`onChannelsChanged`
- `onClientTypeUpdate`

**隔离设计**——forked agent 隔离所有 mutable 回调，有三个 opt-in 标志可共享（shareSetAppState/shareAbortController/shareSetResponseLength）。这是安全设计。

**Channel 注册系统**——`tryAddPluginChannel()` / `tryRemovePluginChannel()` / `tryAddServerChannel()` / `tryRemoveServerChannel()`——dev-channel 门控，`--dangerously-load-development-channels` 标记的开发条目在 `ChannelEntry.dev` 字段单独区分，不影响 allowlist-bypass。

---

### Structured IO（cli/structuredIO.ts）——SDK 协议桥

**859 行的结构化 I/O 桥**——stdin/stdout 和 SDK 控制协议之间的关键边界。

**NDJSON 消息解析**——`read()` 异步生成器增量读取输入流，处理 `\n` 分隔的 JSON 行。支持 `prependedLines` 注入（在已迭代的消息之间仍可追加用户消息）。

**工具使用 ID 去重**——`resolvedToolUseIds` 是有界 FIFO（MAX=1000），防止 WebSocket 重连等重复 `control_response` 导致重复 assistant 消息注入从而触发 API 400 "tool_use ids must be unique"。最旧项通过 `Set.values().next()` 逐出。

**三类请求路由**：
1. `createCanUseTool()`——SDK 权限请求。先检查本地 `hasPermissionsToUseTool()`，如果 undecided 则 hook 和 SDK prompt 并行竞争（`Promise.race`），先到先赢
2. `createHookCallback()`——Hook 回调通过 `sendRequest` 到 SDK consumer
3. `createSandboxAskCallback()`——沙盒网络权限通过 synthetic `SandboxNetworkAccess` 工具名复用 `can_use_tool` 协议

**`createCanUseTool()` 的 Hook 竞争**——hook 决定允许 → abort SDK pending request → 返回 hook 决定。hook 无决定 → 等 SDK 响应。SDK 先响应 → hook 在后台运行但结果忽略。失败时默认 deny。

**`SandboxNetworkAccess` 技巧**——不引入新协议 subtype，复用 `can_use_tool` 用合成工具名。SDK hosts（VS Code, CCR）看到常规权限提示要求网络访问。

**`handleElicitation()`**——MCP 服务器的用户授权请求（elicitation），发送 `subtype: 'elicitation'` 等待 consumer 响应。失败时返回 cancel。

**`sendMcpMessage()`**——MCP 消息通过 SDK 协议传输。

**输入关闭安全处理**——input stream 关闭时 reject 所有 pending requests 带 "Tool permission stream closed before response received" 错误。

---

### Upstream Proxy（upstreamproxy/）——CCR 容器认证隧道

**CCR 容器侧连接代理**。在 CCR session 容器内运行时，将本地 HTTP CONNECT 代理为 WebSocket 隧道到 CCR server。

**5 步初始化链**：
1. 读取 `/run/ccr/session_token`——heap-only token
2. 调用 `prctl(PR_SET_DUMPABLE, 0)`——通过 Bun FFI 到 `libc.so.6`，防止同 UID ptrace 从堆中 scrape token（Linux 专用）
3. 下载 upstreamproxy CA cert 并合并到系统 bundle——所有子进程（curl/gh/python）都信任 MITM proxy
4. 启动本地 CONNECT→WebSocket relay——`relay.ts`，TCP server 监听 localhost，接收 HTTP CONNECT 请求
5. **relayer 确认后才 unlink token 文件**——如果失败，supervisor 重启可重试

**所有步骤失败打开**——错误时仅 warning 然后禁用代理。绝不影响正常会话运行。

**Proto 帧封装**——手动编码 `UpstreamProxyChunk` protobuf（field_number=1, wire_type=2 → tag 0x0a + varint length + bytes）。避免 protobufjs 依赖。`MAX_CHUNK_BYTES = 512KB`（Envoy 限制）。`PING_INTERVAL_MS = 30s`（sidecar idle timeout 50s）。

**NO_PROXY 列表**——覆盖 loopback、RFC1918、IMDS、包注册表、GitHub。anthropic.com 三种形式（`*.`/`.`/apex）因为不同运行时解析不同：Bun/curl/Go 用 glob，Python urllib/httpx 用 suffix match。

**prctl 安全**——`PR_SET_DUMPABLE=0` 阻止 `gdb -p $PPID` 从堆中读取 token。通过 Bun FFI 调用 `libc.so.6` 的 `prctl()`，非 Linux 时静默跳过。

---

### Vim Mode State Machine（vim/）——完整的编辑器模式

**1,513 行的 vim 模式状态机**。类型即文档。

**两层状态**：
- **VimState**: `INSERT`（记录 insertedText）vs `NORMAL`（CommandState 机器）
- **CommandState**: idle → operator/count/find/g/replace/indent → execute

**`transitions.ts`**——主分发函数 `transition(state, input, ctx)` 基于当前 state type 路由到 `from*()` 函数。每个 state 类型一个转换函数，知道它等待什么输入。`handleNormalInput()` 共享处理 idle 和 count 状态的常规输入。

**支持的操作**：
- Operators: `d`/`c`/`y`（delete/change/yank）
- Motions: `h`/`j`/`k`/`l`、`w`/`b`/`e`/`W`/`B`/`E`、`0`/`^`/`$`
- Vim 特例: `f`/`F`/`t`/`T` 字查找、`g` 前缀（`gg`/`gj`/`gk`）
- 文本对象: `i`/`a` + `w`/`W`/`"`/`'/``/`(`/`b`/`[`/`]`/`{`/`B`/`<`/`>`/`B`
- 行操作: `dd`/`cc`/`yy`、`D`/`C`/`Y`、`o`/`O`、`J`
- 替换: `r<char>`（`r<BS>` 取消替换）
- 缩进: `>>`/`<<`
- 其他: `~` 切换大小写、`x` 删除、`.` 重复、`;`/`,` 重复查找、`u` 撤销、`p`/`P` 粘贴

**持久化状态**——`PersistentState`：`lastChange`（点重复）、`lastFind`（`;`/`,` 重复）、`register`（粘贴缓冲）。

**MAX_VIM_COUNT = 10000**——计数前缀上限。

---

### 元

问题：**这一站真正想解决的架构问题是什么？**

回答：这一站像一张压缩总图，把全局状态、SDK 协议桥、CCR 容器代理和 Vim 状态机并排展示。它真正强调的是 Claude Code 不只是业务逻辑集合，而是同时拥有状态核心、进程外协议边界、容器网络底座和本地编辑引擎的复合系统。

### 反

问题：**如果把这一站的设计反过来，会发生什么？**

回答：如果这些底座层各自无中心，最先崩掉的不会是某个功能，而是整体协同：状态访问会产生循环依赖，structured IO 会乱序，代理链路会不安全，Vim 输入也会退化为键位补丁。那时系统还能跑，但已经没有稳固地基。

### 空

问题：**跳出当前文件名，这一站背后更大的问题是什么？**

回答：更大的问题是，一个成熟 CLI 智能体产品究竟由哪些“非业务骨架”共同撑起。这里讨论的是 Claude Code 的底座复合性。

### 读完后应抓住的 3 个事实

1. **bootstrap/state.ts 的循环依赖防护**——文件顶部标注 "DO NOT ADD MORE STATE"，关键依赖通过参数注入而非 import。`scratchpadDir` 和 `telemetry` 组件等依赖倒置。9 个 signal-based 观察者模式提供 React `useSyncExternalStore` 兼容性。这是代码库中最重要的文件——几乎所有其他模块都依赖它。

2. **Upstream Proxy 的安全链路**——5 步初始化是安全-first 设计：token 从文件系统读入 heap → prctl 禁 same-UID `ptrace` → 下载 CA cert（MITM TLS 注入所需）→ WebSocket relay start → token 文件删除。所有步骤 fail-open——broken proxy 绝不破会话。这使 CCR 容器内的所有 HTTP 流量通过 org-configured 代理，注入 Datadog API key 等凭证到子请求。

3. **Vim 状态机：类型即文档**——`vim/types.ts` 中的状态图注释是整个系统的文档。TypeScript 的联合类型确保 switch-exhaustive——漏掉任何输入都会产生类型错误。这是正确性-first 设计——编译器保证了完整性。`dd`/`cc`/`yy`（同键重复）作为 line-operation 处理，与 `d<motion>`（operator-motion 组合）区分。
