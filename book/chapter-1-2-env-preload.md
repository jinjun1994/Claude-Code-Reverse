# 1.2 环境变量预处理与模块级常量捕获

`cli.tsx` 顶部三组环境变量操作不是"配置初始化"的常规步骤。它们解决的是一个与 JavaScript 模块系统交互的根本问题：

```text
如果某个模块在 import 时将 process.env 的值捕获为模块级 const，
那么 init() 中的环境变量写入永远无法被该模块观察到。
```

---

### 模块级常量捕获问题

Claude Code 的工具模块（BashTool、AgentTool、PowerShellTool）使用如下模式：

```typescript
// 在工具模块的顶层（import 期间执行）
const DISABLE_BACKGROUND_TASKS = process.env.CLAUDE_CODE_DISABLE_BACKGROUND_TASKS === '1';

// 后续代码中直接使用 const，不再读 process.env
```

这是合理的工程实践——模块级 `const` 的读取比每次检查 `process.env` 更快，且类型安全。但代价是：**环境变量必须在模块 eval 之前就位**。

---

### Ablation Baseline：七个实验变量

```typescript
// cli.tsx:16-26
if (feature('ABLATION_BASELINE') && process.env.CLAUDE_CODE_ABLATION_BASELINE) {
  for (const k of [
    'CLAUDE_CODE_SIMPLE',           // 简化 UI，无 spinner
    'CLAUDE_CODE_DISABLE_THINKING', // 禁用 thinking 标签
    'DISABLE_INTERLEAVED_THINKING', // 禁用交替思维
    'DISABLE_COMPACT',              // 禁用上下文压缩
    'DISABLE_AUTO_COMPACT',         // 禁用自动压缩
    'CLAUDE_CODE_DISABLE_AUTO_MEMORY', // 禁用自动记忆
    'CLAUDE_CODE_DISABLE_BACKGROUND_TASKS', // 禁用后台任务
  ]) {
    process.env[k] ??= '1';
  }
}
```

Ablation baseline 块在模块评估前设定实验条件。每个变量控制一个子系统：

| 变量 | 影响系统 | 用途 |
|------|---------|------|
| `CLAUDE_CODE_SIMPLE` | UI 渲染 | 简化模式，无 spinner |
| `CLAUDE_CODE_DISABLE_THINKING` | 模型采样 | 禁用 thinking 标签 |
| `DISABLE_INTERLEAVED_THINKING` | 模型采样 | 禁用交替思维 |
| `DISABLE_COMPACT` | 上下文压缩 | 禁用所有压缩 |
| `DISABLE_AUTO_COMPACT` | 自动压缩 | 禁用自动触发 |
| `CLAUDE_CODE_DISABLE_AUTO_MEMORY` | 记忆系统 | 禁用自动记忆注入 |
| `CLAUDE_CODE_DISABLE_BACKGROUND_TASKS` | BashTool | 禁用后台任务 |

这些变量用于实验对照（ablation study）——通过逐一关闭子系统，测量每个子系统对用户体验的影响。

---

### CCR 堆配置：容器环境适配

```typescript
// cli.tsx:7-14
if (process.env.CLAUDE_CODE_REMOTE === 'true') {
  const existing = process.env.NODE_OPTIONS || '';
  process.env.NODE_OPTIONS = existing
    ? `${existing} --max-old-space-size=8192`
    : '--max-old-space-size=8192';
}
```

**8192 的来源**——CCR 容器通常有 16GB 内存。Node.js/V8 默认约 4GB。设置为 8GB 是合理折中——剩余内存用于子进程（bash、MCP、git）。

**追加而非覆盖**——保留用户已设置的 `NODE_OPTIONS`（如 `--inspect`），只追加堆大小。不破坏用户调试配置。

---

### feature() 与编译时常量裁剪

`feature()` 来自 `bun:bundle`，在**构建时**就被解析为 `true` 或 `false`：

```typescript
if (feature('ABLATION_BASELINE') && process.env.CLAUDE_CODE_ABLATION_BASELINE) { ... }
// 如果 feature()=false，DCE 整个消除 if 块，零运行时开销
```

**三层裁剪的叠加**：

| 层级 | 机制 | 触发时机 | 裁剪效果 |
|------|------|---------|---------|
| 编译期 | `feature('...')` | Build | 整块代码从输出中消除 |
| 模块加载期 | 变量前置 | Module Eval | 确保模块 const 捕获正确值 |
| 运行期 | `??=` | Runtime | 用户显式设置优先 |

**为何不能抽成函数**——如果写成 `isAblationBaseline()`，Bun 无法在静态分析阶段判定返回值，整个块无法被 DCE 消除。

---

### 快速路径 --bare 的早期门控

```typescript
// cli.tsx:283-284
if (args.includes('--bare')) {
  process.env.CLAUDE_CODE_SIMPLE = '1';
}
```

`--bare` 在 `main.js` 加载之前设置。Commander 的选项构建需要知道 `CLAUDE_CODE_SIMPLE` 的值——某些选项在 bare 模式下不显示。注释指出：`--bare` 必须在 module eval 阶段就生效，"not just inside the action handler"。

---

### COREPACK_ENABLE_AUTO_PIN 的预置处理

```typescript
// cli.tsx:3
process.env.COREPACK_ENABLE_AUTO_PIN ??= '0';
```

Corepack 在特定条件下会弹出交互式确认提示（`? Do you want to continue? (Y/n)`），阻塞非 TTY 环境下的进程。在 `cli.tsx` 顶部写入确保任何后续模块触发的 `pnpm` / `yarn` 调用都不会弹出交互提示。

---

### 环境变量前置的时间顺序

```
cli.tsx 加载顺序（import side effects）:
  1. COREPACK_ENABLE_AUTO_PIN = '0'                ← 阻止交互式提示
  2. NODE_OPTIONS += --max-old-space-size=8192      ← CCR 堆配置
  3. ABLATION_BASELINE 七变量                      ← 实验条件基线
  4. --bare → CLAUDE_CODE_SIMPLE = '1'              ← 主函数中早期门控
  ...
main() 执行:
  5. ensureMdmRawRead() + startKeychainPrefetch()   ← 并行预取启动
  ...
import('../main.js'):
  6. 模块评估捕获所有已设置的环境变量                ← 常量固化点
```

步骤 1-4 必须在步骤 6 之前完成。步骤 5（并行预取）在 import 链期间启动，与模块加载并行运行。

---

### 环境变量写入的幂等性与安全性

`??=` 操作的幂等性使得代码可以在 import 链顶部安全运行多次：

```
用户设置: CLAUDE_CODE_SIMPLE=0
??= 操作: 跳过（因为已设置）
结果: CLAUDE_CODE_SIMPLE=0（用户设置优先）

用户未设置: CLAUDE_CODE_SIMPLE=undefined
??= 操作: 写入 '1'
结果: CLAUDE_CODE_SIMPLE='1'（基线生效）
```

---

### 环境变量注入的全局图

```
cli.tsx top-level side effects (import 前):
  COREPACK_ENABLE_AUTO_PIN    → 阻止 corepack 交互提示
  NODE_OPTIONS (max-old-space) → CCR 容器堆配置
  ABLATION_BASELINE (7 vars)   → 实验条件

main() 中:
  --bare → CLAUDE_CODE_SIMPLE  → bare 模式早期门控

import 模块评估:
  BashTool/AgentTool: const = process.env.X  → 捕获基线值

init() 运行时:
  认证、策略、会话 ID → 但这些不影响模块级 const
```

---

### 环境变量前置的代价

这种设计有一个明确的工程权衡：`cli.tsx` 不再是"干净的"路由文件。它承担了启动时序编排的角色，包含必须在模块加载前执行的 side effect。这使得文件的可测试性下降——任何测试都需要先设置环境变量，再模拟模块加载，才能验证特定路径。

但收益是清晰的：模块级常量捕获的性能优势和类型安全，与编译时的代码裁剪，共同确保了每个快速路径都尽可能轻量。

---

### 哪些模块捕获了哪些环境变量？

通过 `grep` 搜索源码可以确认以下模块级常量捕获：

```
BashTool.tsx:
  CLAUDE_CODE_DISABLE_BACKGROUND_TASKS   → 禁用后台任务（line 42）
  CLAUDE_CODE_ASSISTANT_MODE_ACTIVE      → 助手模式超时预算
  ASSISTANT_BLOCKING_BUDGET_MS           → 覆盖阻塞预算

AgentTool/constants.ts:
  CLAUDE_CODE_MAX_CONCURRENT_AGENTS       → 最大并发 Agent 数

FileReadTool/FileReadTool.ts:
  CLAUDE_CODE_READ_IMAGE_BASE64          → 图片 base64 编码开关

MCP/normalization.ts:
  CLAUDE_AGENT_SDK_MCP_NO_PREFIX         → MCP 工具名前缀开关

tools.ts:
  USER_TYPE                              → 内部用户标志（ant vs external）
  CLAUDE_CODE_DISABLE_EMBEDDED_SEARCH    → 嵌入搜索禁用
```

**关键观察**——这些模块分布在 6 个不同的目录中。没有任何一个 `init()` 函数能在所有模块加载之前运行。因此唯一可行的方案是在 `import` 之前、在文件顶部完成所有环境变量写入。

### BashTool 对 env 的依赖示例

BashTool 是最重度依赖环境变量的工具模块：

```typescript
// BashTool.tsx top-level
const ASSISTANT_BLOCKING_BUDGET_MS =
  parseInt(process.env.ASSISTANT_BLOCKING_BUDGET_MS || '15000', 10)

const PROGRESS_THRESHOLD_MS =
  parseInt(process.env.PROGRESS_THRESHOLD_MS || '2000', 10)

const DISABLE_BACKGROUND_TASKS =
  process.env.CLAUDE_CODE_DISABLE_BACKGROUND_TASKS === '1'
```

如果 `--bare` 设置 `CLAUDE_CODE_SIMPLE=1` 发生在这些 const 捕获**之后**，BashTool 的常量已经捕获了旧值，bare 模式的配置不生效。这就是时序问题的根本原因。

---

## 1.10 REPL 模式与环境变量交互

环境变量预处理的影响范围不止于工具模块——它贯穿整个 REPL 初始化和模式路由。理解这些 env 如何与 REPL 模式交互，是理解 `main.tsx` 分支逻辑的前提。

### isReplModeEnabled：三级信号链

```typescript
// src/tools/REPLTool/constants.ts:23-30
export function isReplModeEnabled(): boolean {
  if (isEnvDefinedFalsy(process.env.CLAUDE_CODE_REPL)) return false
  if (isEnvTruthy(process.env.CLAUDE_REPL_MODE)) return true
  return (
    process.env.USER_TYPE === 'ant' &&
    process.env.CLAUDE_CODE_ENTRYPOINT === 'cli'
  )
}
```

**三级信号，按优先级排列**：

| 优先级 | 环境变量 | 效果 | 用途 |
|--------|---------|------|------|
| 1 (最高) | `CLAUDE_CODE_REPL=0/false/no/off` | 硬关闭 | 内部调试 opt-out |
| 2 | `CLAUDE_REPL_MODE=1/true/yes/on` | 硬开启 | 强制开启（任何构建类型） |
| 3 (默认) | `USER_TYPE === 'ant' && ENTRYPOINT === 'cli'` | 默认开启 | 内部构建的 CLI 默认使用 REPL |

**默认行为的深层含义**——外部构建（`USER_TYPE !== 'ant'`）中，REPL 模式默认关闭。这意味着所有原始工具（Bash、Read、Edit 等）直接暴露给模型。而 REPL 模式下，这些工具被隐藏——只能通过 VM 上下文内的 REPL 工具间接使用。

### REPL 模式对工具池的影响

```typescript
// src/tools.ts:271-327
function getTools(): Tools {
  if (isReplModeEnabled()) {
    // 隐藏原始工具——只暴露 REPL 工具
    return filterBareTools(tools)
  }
  // 暴露全部原始工具
  return tools
}
```

被隐藏的工具列表包括：`Bash`、`FileReadTool`、`FileEditTool`、`Glob`、`Grep`、`FileWrite`、`NotebookEdit`、`AgentTool`。这些在 REPL 模式下通过 VM context 间接调用。

### Bare Mode vs Normal REPL

`--bare` 是最激进的环境变量写入操作之一：

```typescript
// cli.tsx:283-285 — 模块加载前设置
if (args.includes('--bare')) {
  process.env.CLAUDE_CODE_SIMPLE = '1';
}

// main.tsx:1012-1016 — Commander action handler 中再次设置
if ((options as { bare?: boolean; }).bare) {
  process.env.CLAUDE_CODE_SIMPLE = '1';
}
```

**为什么设置两次**——第一次在 `import('../main.js')` 之前，确保模块 eval 阶段的常量捕获生效。第二次在 action handler 中，处理通过 `--settings` 文件配置的 `bare` 选项（不通过 CLI 参数的情况）。两次设置保证了无论用户通过哪种方式启用 bare，行为一致。

`CLAUDE_CODE_SIMPLE=1` 的影响范围（约 30+ 个 gate 点）：

| 组件 | 文件 | 效果 |
|------|------|------|
| 工具过滤 | `src/tools.ts:273` | 只保留 Bash、Read、Edit（或 REPL） |
| SessionStart hooks | `src/utils/sessionStart.ts:47` | 返回空数组 |
| Setup hooks | `src/utils/sessionStart.ts:182` | 返回空数组 |
| Stop hooks | `src/query/stopHooks.ts:136` | 跳过 prompt suggestion、memory extraction |
| Skills 发现 | `src/skills/loadSkillsDir.ts:658` | 跳过自动发现 walks |
| CLAUDE.md 加载 | `src/context.ts:165-167` | 跳过自动发现（cwd walk） |
| MCP 自动发现 | `src/main.tsx:1809` | 返回空配置 `{ servers: {} }` |
| 启动预取 | `src/main.tsx:2346` | 跳过 quota check、bootstrap data |
| Keychain 凭证 | `envUtils.ts:50-52` | 所有 keychain 读取被跳过 |
| 插件同步 | `src/main.tsx:2555` | 跳过版本同步和清理 |
| 后台维护 | `src/main.tsx:2816` | 跳过 `startDeferredPrefetches()`、`startBackgroundHousekeeping()` |

Bare 模式的核心设计：**Auth 是纯粹的**。只读取 `ANTHROPIC_API_KEY` 环境变量或 `--settings` 中的 `apiKeyHelper`。OAuth 和 keychain 永远不被读取。

### Output Format 与传输模式分支

`--output-format` 决定输出的序列化方式，但只在非交互模式下生效：

```
判断逻辑（main.tsx:797-815）:
  isNonInteractive = 
    hasPrintFlag ||           // -p/--print
    hasInitOnlyFlag ||        // --init-only
    hasSdkUrl ||             // --sdk-url
    !process.stdout.isTTY     // stdout 非 TTY

isNonInteractive=true, outputFormat='stream-json'  → NDJSON 逐行输出
isNonInteractive=true, outputFormat='json'         → 单个 JSON 对象
isNonInteractive=true, outputFormat='text'         → 人类可读 ANSI
isNonInteractive=false                              → Ink TUI React 渲染
```

**SDK 一致性校验**：
- `--input-format=stream-json` 要求 `--output-format=stream-json`
- `--sdk-url` 要求两者都是 `stream-json`
- `--replay-user-messages` 要求两者都是 `stream-json`

### Fast Mode 环境变量路由

```typescript
// src/utils/fastMode.ts
export function isFastModeEnabled(): boolean {
  return !isEnvTruthy(process.env.CLAUDE_CODE_DISABLE_FAST_MODE)
}
```

Fast mode 的五层可用性检查：

| 检查层 | 来源 | 禁用���因 |
|--------|------|---------|
| GrowthBook kill switch | `tengu_penguins_off` | 全局关闭 |
| 原生二进制要求 | `tengu_marble_sandcastle` | 需要 bun build |
| SDK 限制 | SDK entrypoints | 默认不启用（Assistant daemon 除外）|
| 提供商限制 | Bedrock/Vertex/Foundry | 仅一级 API 支持 |
| 组织状态 | `/api/claude_code_penguin_mode` | free/preference/network_error/extra_usage_disabled |

**冷却状态管理**——Fast mode 有两个冷却触发器：
- `triggerFastModeCooldown()` — rate-limit 或 overloaded 拒绝后进入冷却
- `handleFastModeRejectedByAPI()` — API 明确说组织不允许时永久禁用

冷却期间 fast mode 暂时禁用，冷却结束后恢复。这是防止 API 过载时的反复重试循环。

### 思维（Thinking）环境变量

```typescript
// --thinking 标志（main.tsx:2462-2488，已弃用 --max-thinking-tokens）
case 'enabled':  / 'adaptive': config = { type: 'adaptive' }; break;
case 'disabled': config = { type: 'disabled' }; break;

// MAX_THINKING_TOKENS env 覆盖一切
const override = parseInt(process.env.MAX_THINKING_TOKENS, 10);
if (!isNaN(override)) {
  config = override > 0 ? { type: 'enabled', maxTokens: override } : { type: 'disabled' };
}
```

| 变量 | 影响 | 废弃关系 |
|------|------|---------|
| `CLAUDE_CODE_DISABLE_THINKING` | API 调用层硬禁用 thinking | - |
| `DISABLE_INTERLEAVED_THINKING` | 省略 `interleaved-thinking` beta header | - |
| `MAX_THINKING_TOKENS` | 覆盖 `--max-thinking-tokens` flag | `--thinking` 取代后已弃用 |

### 非交互式 Session 的 CLAUDE_CODE_ENTRYPOINT

```typescript
// main.tsx:517
process.env.CLAUDE_CODE_ENTRYPOINT = isNonInteractive ? 'sdk-cli' : 'cli';
```

这个设置影响下游的所有模块：

| `CLAUDE_CODE_ENTRYPOINT` | 含义 | 渲染方式 |
|--------------------------|------|---------|
| `cli` | 完整交互式 REPL | Ink TUI (React) |
| `sdk-cli` | 头less CLI | `runHeadless()` (print.ts) |
| `mcp` | MCP 服务器 | MCP transport |
| `local-agent` | 本地 Agent 派生 | Agent 路由 |
| `remote` | 远程会话访问 | WebSocket 桥接 |
| `sdk-ts`/`sdk-py` | SDK 消费者 | StructuredIO |

这个环境变量是进程的"身份标识"——每个模块通过检查它来决定自己的行为。

---

## 3.7 环境变量枚举与分级

`managedEnvConstants.ts` 中定义了 100+ 环境变量，按安全级别分为 4 级：

### 核心操作变量
| 变量 | 用途 |
|------|------|
| `CLAUDE_CODE_ENTRYPOINT` | 身份标识：cli, sdk-ts, sdk-py, mcp, claude-code-github-action, claude-desktop, local-agent |
| `CLAUDE_CODE_SIMPLE` | Bare 模式门控，跳过 hooks/LSP/plugin sync/MCP sync/attribution |
| `CLAUDE_CODE_REMOTE` | CCR 模式，触发 `NODE_OPTIONS --max-old-space-size=8192` |
| `CLAUDE_CODE_COORDINATOR_MODE` | 协调器模式（main 委派给 workers） |
| `CLAUDE_CODE_ABLATION_BASELINE` | L0 ablation 基线——强制禁用 thinking/memory/compact/background tasks |
| `CLAUDE_CODE_SUBPROCESS_ENV_SCRUB` | 敏感变量剥离门控 |
| `CLAUDE_CODE_DISABLE_POLICY_SKILLS` | 阻止 policy skills 加载 |
| `CLAUDE_CODE_DISABLE_NONESSENTIAL_TRAFFIC` | 减少后台流量 |

### 提供者管理的环境变量（主机锁定）
这些变量在 `managedEnvConstants.ts:14-56` 定义，通过 `CLAUDE_CODE_PROVIDER_MANAGED_BY_HOST` 门控——当主机管理时，settings.json 无法覆盖：

| 前缀 | 用途 | 变量数 |
|------|------|--------|
| `CLAUDE_CODE_USE_*` | 模型提供者选择（BEDROCK/VERTEX/FOUNDRY） | 3 个 |
| `ANTHROPIC_*_BASE_URL` | 端点配置 | 4 个 |
| `ANTHROPIC_AUTH_*` | 身份验证（Token/API Key/OAuth） | 5 个 |
| `AWS_BEARER_TOKEN_*` | AWS 凭证 | 2 个 |
| `ANTHROPIC_DEFAULT_*_MODEL` | 模型默认值（Sonnet/Opus/Haiku） | 3 组 + 每组的描述/功能 |
| `CLAUDE_CODE_SUBAGENT_MODEL` | 子 Agent 模型 | 1 个 |
| `VERTEX_REGION_CLAUDE_*` | 按模型的 Vertex 区域覆盖 | 每模型 1 个 |

### 安全环境变量（信任对话框前允许）
`managedEnvConstants.ts:108-191`：65 个变量可在信任对话框之前应用。关键包含 Anthropic 自定义头、模型配置、AWS 区域/配置文件变量、OTEL 协议变量。

**关键排除**（DANGEROUS 级别）= `ANTHROPIC_BASE_URL`, `HTTP_PROXY`, `HTTPS_PROXY`, `NODE_TLS_REJECT_UNAUTHORIZED`, `NODE_EXTRA_CA_CERTS` 以及 shell 执行设置（如 apiKeyHelper, awsAuthRefresh, statusLine, otelHeadersHelper）。

---

## 3.8 进程环境变量的剥离

`subprocessEnv()`（`src/utils/subprocessEnv.ts`）为子进程生成安全的环境副本。当 `CLAUDE_CODE_SUBPROCESS_ENV_SCRUB` 为真时，18 个环境变量族及其 `INPUT_` 前缀的重复项（GitHub Actions 自动创建）被剥离。

### 被剥离的家族
1. **身份验证凭据**：`ANTHROPIC_API_KEY`, `CLAUDE_CODE_OAUTH_TOKEN`, `ANTHROPIC_AUTH_TOKEN`, `ANTHROPIC_FOUNDRY_API_KEY`, `ANTHROPIC_CUSTOM_HEADERS`
2. **OTLP 导出头**：`OTEL_EXPORTER_OTLP_*_HEADERS`
3. **云提供商凭据**：`AWS_SECRET_ACCESS_KEY`, `AWS_SESSION_TOKEN`, `AWS_BEARER_TOKEN_BEDROCK`, `GOOGLE_APPLICATION_CREDENTIALS`, `AZURE_CLIENT_SECRET`
4. **GitHub Actions OIDC**：`ACTIONS_ID_TOKEN_REQUEST_TOKEN`, `ACTIONS_ID_TOKEN_REQUEST_URL`
5. **GitHub Actions 运行时**：`ACTIONS_RUNTIME_TOKEN`, `ACTIONS_RUNTIME_URL`
6. **claude-code-action 重复项**：`ALL_INPUTS`, `OVERRIDE_GITHUB_TOKEN`, `DEFAULT_WORKFLOW_TOKEN`, `SSH_SIGNING_KEY`

### 消费者
`subprocessEnv()` 调用者：
- MCP 服务器产生（`client.ts:954`）
- LSP 服务器产生（`LSPClient.ts:100`）
- Hook 执行（`hooks.ts:883`）
- Bash 工具（`Shell.ts:318`）
- Shell 快照（`ShellSnapshot.ts:463`）

---

## 3.9 OTEL 的自举引导

`instrumentation.ts:88-111`：当 `USER_TYPE === 'ant'` 时，`ANT_OTEL_*` 前缀的环境变量被复制到标准 `OTEL_*` 环境变量：
- `ANT_OTEL_METRICS_EXPORTER` → `OTEL_METRICS_EXPORTER`
- `ANT_OTEL_LOGS_EXPORTER` → `OTEL_LOGS_EXPORTER`
- `ANT_OTEL_TRACES_EXPORTER` → `OTEL_TRACES_EXPORTER`
- `ANT_OTEL_EXPORTER_OTLP_PROTOCOL` → `OTEL_EXPORTER_OTLP_PROTOCOL`
- `ANT_OTEL_EXPORTER_OTLP_ENDPOINT` → `OTEL_EXPORTER_OTLP_ENDPOINT`
- `ANT_OTEL_EXPORTER_OTLP_HEADERS` → `OTEL_EXPORTER_OTLP_HEADERS`

**协议级导出器**在协议 switch 语句中**动态导入**——避免在不使用时加载所有 6 个导出器（约 1.2MB）的模块图。

**Analytics Sink**（`sink.ts`）：将事件路由到 Datadog 和 1P。Datadog 门控在 `tengu_log_datadog_events` GrowthBook 特性门之前。Sink 在发送到 Datadog（通用访问）之前会剥离 `_PROTO_*` PII 键，而 1P 接收完整载荷。

---

## 3.10 Async 操作的 Promise 栅格化

### MDM 加载序列化
`settings.ts:67-108`：`mdmLoadPromise` 确保只发生一次 MDM 读取。`startMdmSettingsLoad()` 是幂等的（line 68: `if (mdmLoadPromise) return`）。`ensureMdmSettingsLoaded()` 等待进行中的加载。

### Keychain 栅格化
`keychainPrefetch.ts:39-41,81`：`prefetchPromise` 确保只触发一个 `Promise.all([oauthSpawn, legacySpawn])`。`ensureKeychainPrefetchCompleted()` 等待完成。

### GrowthBook 重新初始化序列化
`growthbook.ts:94,968-975`：`reinitializingPromise` 跟踪进行中的重新初始化。`checkSecurityRestrictionGate()`：`if (reinitializingPromise) { await reinitializingPromise }`——等待新值。`.finally(() => { reinitializingPromise = null })` 完成后清除。

### Policy Limits 加载
`policyLimits/index.ts:94-114`：`loadingCompletePromise` 带 30 秒超时（`LOADING_PROMISE_TIMEOUT_MS`）。防止死锁（如 `loadPolicyLimits()` 从未被调用时）。

---

## 3.11 GrowthBook：自举期的特性门求值

**初始化序列**（`growthbook.ts`）：
1. `getGrowthBookClient()` 创建一次 GB 客户端
2. `remoteEval: true`——服务器预评估特性
3. 5 秒超时 `client.init()`
4. 信任对话框前，GrowthBook 客户端在无认证头部的情况下创建，依赖磁盘缓存值
5. 信任对话框后，`initializeGrowthBook()` 检测并重新创建认证头部

**三种访问模式**：
| 方法 | 用途 | 阻塞性 |
|------|------|--------|
| `getFeatureValue_CACHED_MAY_BE_STALE()` | 快速路径，内存图 | 不阻塞 |
| `checkSecurityRestrictionGate()` | 等重新初始化完成 | 可能阻塞 |
| `checkGate_CACHED_OR_BLOCKING()` | 如果磁盘说 false 则缓慢 | 可能阻塞 |

**定期刷新**：ants 为 20 分钟，外部用户为 6 小时（`GROWTHBOOK_REFRESH_INTERVAL_MS`）。

**内部覆盖**：`CLAUDE_INTERNAL_FC_OVERRIDES`（JSON）仅在 `USER_TYPE === 'ant'` 时解析一次，优先于所有缓存和网络值。

---

---

## 3.12 Settings 合并管线

`applySettingsChange.ts` 中，设置从多个来源合并：

| 来源 | 文件路径 | 优先级 |
|------|---------|--------|
| Policy | `managed-settings.json` 或 MDM | 最高 |
| Flag | CLI 选项 | 高 |
| Local | `.claude/settings.local.json` | 中 |
| Project | `.claude/settings.json`（项目根） | 低 |
| User | `~/.claude/settings.json` | 最低 |

合并策略是**深度合并**——嵌套对象按来源优先级递归合并。这意味着高优先级的嵌套字段覆盖低优先级的对应字段，但不影响同一对象的其他字段。

---

## 3.13 Settings 架构与 Zod 验证

Settings 使用 Zod 模式进行验证——所有字段在启动和设置变更时进行验证。

**未知字段处理**——Zod 的 strip 行为会静默剥离未知字段。这意味着设置文件可以包含未来格式中的额外字段而不会导致验证失败——它们只是被忽略。

**`SettingsWithErrors` 类型**——设置加载可能部分失败。`SettingsWithErrors` 包含有效的设置对象和一个错误列表（解析失败的字段）。这使得即使部分设置无效，有效设置仍能被使用。

---

## 3.14 变更检测器

`changeDetector.ts`：

```
SettingsChangeDetector.initialize() — 初始化设置快照
SkillChangeDetector.initialize()  — 初始化技能目录快照
```

**轮询**——检测器周期性检查设置文件和技能目录的变化。当检测到变更时，触发热重载路径。

**SettingsChangeDetector** 监控以下来源的快照：
- 配置文件修改时间
- 设置文件内容哈希
- 策略设置变更

**SkillChangeDetector** 监控：
- `.claude/skills/` 目录的内容变更
- `.claude/commands/` 目录的内容变更（旧格式）

---

## 3.15 OAuth 端点配置

`config.ts`（MCP OAuth）中，端点通过以下序列解决：
1. 如果配置了 `authServerMetadataUrl`，直接获取
2. 否则，遵循 RFC 9728：探测 `/.well-known/oauth-protected-resource`
3. 然后，遵循 RFC 8414 获取授权服务器元数据
4. 回退到路径感知的 RFC 8414 直接探测（旧服务器）

---

## 3.16 环境变量替换

`expandEnvVars()` 扩展配置中的 `${VAR}` 和 `${VAR:-default}` 语：
- stdio：命令、参数、所有 env 值
- sse/http/ws：url 和所有 header 值
- sse-ide/ws-ide/sdk/claudeai-proxy：无扩展

缺少变量记录在 `missingVars` 中并报告为错误。

---

## 3.17 Config 文件原子写入

配置文件使用原子写入模式：
1. 写入 `.{pid}.{timestamp}.tmp` 文件
2. `fsync()` 刷新到磁盘
3. 原子化 `rename()`
4. 保留原始文件权限

这防止了写入中断时的配置文件损坏。`fsync()` 确保数据在 rename 之前持久化——这是 POSIX rename 的原子性保证但前提是目标已落盘。

---

## 3.18 信任对话框与安全变量

信任对话框是 Claude Code 的安全门控。在对话框被接受之前：
- 仅**安全环境变量**（65 个变量）被应用
- 敏感变量（`ANTHROPIC_BASE_URL`、`HTTP_PROXY`、`HTTPS_PROXY`、`NODE_TLS_REJECT_UNAUTHORIZED` 等）被阻止
- Keychain 预取被延迟但已在进行中
- MDM 加载在独立路径上运行

**`applySafeConfigEnvironmentVariables()`**——在信任对话框之前运行，应用安全变量。当对话框被接受后，完整的环境变量被应用。

---

---

## 3.19 Env Utils 工具集

`envUtils.ts` 中的核心环境变量函数：

| 函数 | 用途 |
|------|------|
| `isEnvTruthy(envVar)` | 检查 `'1' | 'true' | 'yes' | 'on'`（不区分大小写） |
| `isEnvDefinedFalsy(envVar)` | 检查 `'0' | 'false' | 'no' | 'off'` |
| `hasNodeOption(flag)` | 分割 `NODE_OPTIONS` 空白，精确匹配 |
| `isBareMode()` | 检查 `CLAUDE_CODE_SIMPLE=1` 或 `--bare`；门控约 30 个功能（hooks、LSP、插件同步、skill 目录遍历、所有 keychain/凭据读取） |
| `parseEnvVars(string[])` | 解析 `KEY=VALUE` 数组为对象 |
| `getAWSRegion()` | `AWS_REGION` 或 `AWS_DEFAULT_REGION`，默认 `us-east-1` |
| `getDefaultVertexRegion()` | `CLOUD_ML_REGION`，默认 `us-east5` |

**`env` 对象**（`env.ts` 行 316）——memoized 聚合：
- `hasInternetAccess`——`http://1.1.1.1` HEAD 请求，1s 超时
- `isCI`——来自 `CI` 环境变量
- `getPackageManagers()`——检查 npm/yarn/pnpm 可用性
- `getRuntimes()`——检查 bun/deno/node
- `detectDeploymentEnvironment()`——检测 codespaces/gitpod/replit/vercel/heroku/lambda/k8s/docker 等（200+ 行环境变量嗅探）

---

## 3.20 5 层 Settings 级联

`settings.ts`（1016 行）——5 个设置来源（优先级从低到高）：

```
userSettings → projectSettings → localSettings → flagSettings → policySettings
```

**路径解析**——`getSettingsFilePathForSource(source)`：
| 来源 | 路径 |
|------|------|
| user | `~/.claude/settings.json` |
| project | `.claude/settings.json` |
| local | `.claude/settings.local.json` |
| policy | `/Library/Application Support/ClaudeCode/managed-settings.json`（macOS） |
| flag | CLI `--settings` 标志的路径 |

**合并算法**——`loadSettingsFromDisk()`：
- 递归守卫：`isLoadingSettings` 标志
- 从 `pluginSettingsBase` 开始
- 按优先级迭代 `getEnabledSettingSources()`
- 使用 `lodash-es/mergeWith` + `settingsMergeCustomizer`
- 数组连接+去重

**安全敏感检查排除 projectSettings**——`hasSkipDangerousModePermissionPrompt()` / `hasAutoModeOptIn()` / `getUseAutoModeDuringPlan()`：
- 检查多个可信来源
- **故意排除**`projectSettings`——RCE 风险：恶意项目可绕过对话框

---

## 3.21 Settings Schema**与验证

`types.ts`（1149 行）——`SettingsSchema` 是 lazy Zod v4 schema，带 `.passthrough()`：

**关键字段**（100+）：
- `$schema`——URL 字面量指向 `https://json.schemastore.org/claude-code-settings.json`
- `env`——`z.record(z.string(), z.coerce.string())`（类型强制从数字到字符串）
- `permissions`——allow/deny/ask 规则数组，defaultMode 枚举
- `hooks`——来自 `schemas/hooks.ts` 的 HooksSchema
- `allowedMcpServers` / `deniedMcpServers`——企业 MCP allow/deny 列表
- `strictPluginOnlyCustomization`——锁定定制面：`['skills', 'agents', 'hooks', 'mcp']`

**验证管线**——`validation.ts`：
- `formatZodError(error, filePath)`——将 Zod 问题转为 `ValidationError[]`
- `filterInvalidPermissionRules(data, filePath)`——在 schema 验证前预验证权限规则数组，移除无效规则（返回警告而非错误，防止整个文件被拒绝）
- `validateSettingsFileContent(content)`——严格模式验证，用于文件编辑期间

**Validation Tips**（`validationTips.ts`）——13 个 tip 匹配器，键为 `path.code` 模式：
- `permissions.defaultMode.invalid_value` → 有效模式 + 文档链接
- `env.*.invalid_type` → 必须是字符串，数字/布尔用引号包裹
- `hooks.invalid_type` → 匹配器格式提示（管道分隔或正则）

---

## 3.22 MDM Subprocess 预取

`rawRead.ts`——并行子进程读取：

**macOS**——`plutil -convert json -o - -- {plist_path}`
**Windows**——并行 `reg query HKLM\SOFTWARE\Policies\ClaudeCode` 和 `reg query HKCU\...`
**Linux**——无操作

**`startMdmRawRead()`**——在 main.tsx 模块评估期间启动，结果稍后通过 `getMdmRawReadPromise()` 消费。

**MDM 常量**（`mdm/constants.ts`）：

| macOS plist 路径 | 优先级 |
|---|---|
| `/Library/Managed Preferences/{user}/com.anthropic.claudecode.plist` | 每用户管理（最高） |
| `/Library/Managed Preferences/com.anthropic.claudecode.plist` | 设备级 |
| `~/Library/Preferences/com.anthropic.claudecode.plist` | 用户偏好（仅 ant，本地测试） |

**Preference 域**——`com.anthropic.claudecode`

**WOW64 共享键**——Windows 注册表路径在 `SOFTWARE\Policies` 下——WOW64 共享键列表，32 位和 64 位进程看到相同的值，无重定向。

**子进程超时**——`MDM_SUBPROCESS_TIMEOUT_MS = 5000`

---

## 3.23 Remote Managed Settings HTTP 缓存

`remoteManagedSettings/index.ts`（639 行）——启动期间触发，fail-open 行为：

**Cache-first 策略**——立即应用缓存的磁盘设置以取消阻塞等待者，然后获取。设置加载时触发 `settingsChangeDetector.notifyChange('policySettings')`。

**ETag 缓存**——`fetchWithRetry(cachedChecksum?)`——最多 5 次重试，指数退避，使用 `If-None-Match` 进行基于 ETag 的 HTTP 缓存。

**校验和计算**——`computeChecksumFromSettings()`——匹配服务器的 `json.dumps(settings, sort_keys=True)`，用于 ETag 验证。

**背景轮询**——`startBackgroundPolling()`——1 小时间隔，unref'd 定时器，通过 `cleanupRegistry` 注册。

**同步缓存状态**（`syncCacheState.ts`）——`remote-settings.json` 存储在 `~/.claude/`——首次可用时调用 `resetSettingsCache()` 使后续合并读取包含策略层。

---

## 3.24 Settings 变更检测器

`changeDetector.ts`（489 行）——chokidar + 30 分钟 MDM 轮询：

**常量**：
| 常量 | 值 | 用途 |
|------|----|------|
| `FILE_STABILITY_THRESHOLD_MS` | 1000 | 等待文件写入稳定 |
| `FILE_STABILITY_POLL_INTERVAL_MS` | 500 | chokidar 轮询间隔 |
| `INTERNAL_WRITE_WINDOW_MS` | 5000 | 忽略自身写入 |
| `MDM_POLL_INTERVAL_MS` | 30 分钟 | 注册表/plist 轮询 |
| `DELETION_GRACE_MS` | 1700 | 删除后重建的宽限期 |

**架构**：
1. `initialize()`——启动 MDM 轮询，创建设置目录的 chokidar watcher。使用 `awaitWriteFinish` 确保写入稳定性，`atomic: true`，`usePolling: false`
2. 监控目标从所有设置来源构建（flagSettings 除外，可能是 $TMPDIR 中的临时文件）
3. `handleChange(path)`——获取来源，取消待定删除定时器，检查 `consumeInternalWrite()`，先触发 ConfigChange hook，如不阻止则 `fanOut(source)`
4. `fanOut(source)`——调用 `resetSettingsCache()` 然后 `settingsChanged.emit(source)`（集中式缓存重置避免 N 路抖动）
5. `startMdmPoll()`——对 mdm+hkcu 设置进行 JSON 快照，每 30 分钟比较设置变更时调用 `setMdmSettingsCache()` 和 `fanOut('policySettings')`

**内部写入抑制**——`internalWrites.ts`（38 行）——打破导入循环：从 changeDetector 提取，使 settings.ts 可标记写入而不导入 changeDetector。

**热重载应用**——`applySettingsChange.ts`（93 行）：
1. 读取新设置
2. 重新加载权限规则
3. 更新 hooks 配置快照
4. 同步权限规则
5. （仅 ant）剥离过于宽泛的 Bash 权限
6. 如需要创建设置禁用 bypass 上下文
7. 转换 auto 模式
8. 更新 AppState

---

## 3.25 macOS Keychain 预取

`keychainPrefetch.ts`（117 行）——并行启动优化：

- `startKeychainPrefetch()` 在 main.tsx 顶层触发（与 `startMdmRawRead()` 并行）
- 在 macOS 上并行运行两个 `security find-generic-password` 子进程：
  1. OAuth 凭据键（`Claude Code{OAUTH_SUFFIX}-credentials`）
  2. 遗留 API 键（`Claude Code{OAUTH_SUFFIX}`）

**`spawnSecurity(serviceName)`**——`security find-generic-password -a {user} -w -s {serviceName}`，10 秒超时，退出码 44（条目未找到）为安全 null，超时允许同步重试。

**`ensureKeychainPrefetchCompleted()`**——在 preAction 中等待，由于 ~65ms 的模块导入期间子进程已完成，几乎免费。

**Keychain 存储**（`macOsKeychainStorage.ts`，232 行）：
- `read()`——TTL 检查 → 同步 `security find-generic-password` → JSON 解析。**Stale-while-error**——瞬态 spawn 失败时提供陈旧值而非"Not logged in"
- `readAsync()`——通过 `readInFlight` 去重，基于 generation 的缓存失效
- `update(data)`——JSON → hex 编码（避免转义），使用 `security -i` stdin（绕过 CrowdStrike 等），载荷低于 `SECURITY_STDIN_LINE_LIMIT = 4032`（4096-64）时使用。溢出回退到 argv + hex
- `isMacOsKeychainLocked()`——`security show-keychain-info` 退出码 36，进程生命周期缓存

**FallbackStorage**（`fallbackStorage.ts`，71 行）——`createFallbackStorage(primary, secondary)`：
- 先尝试 primary，回退到 secondary
- 关键修复：如 primary 更新前就有数据但写入失败，`read()` 仍偏好 primary（陈旧条目遮蔽新鲜数据）。所以：这种情况下删除 primary（修复由陈旧刷新令牌导致的 `/login` 循环）

---

## 3.26 OAuth 配置与环境选择

`oauth/constants.ts`——3 个 OAuth 配置：

| 配置 | 用途 |
|------|------|
| PROD | 默认，`https://claude.ai` |
| STAGING | 仅 ant，外部构建中 DCE'd |
| LOCAL | localhost:8000 API，:4000 前端，:3000 控制台 |

**受限配置**：
- `CLAUDE_CODE_CUSTOM_OAUTH_URL`——限制为允许列表：`beacon.claude-ai.staging.ant.dev`、`claude.fedstart.com`、`claude-staging.fedstart.com`
- `CLAUDE_CODE_OAUTH_CLIENT_ID`——允许客户端 ID 覆盖（如 Xcode 集成）

**MCP Client 元数据**——`MCP_CLIENT_METADATA_URL = 'https://claude.ai/oauth/claude-code-client-metadata'`——SEP-991 MCP OAuth 客户端元数据文档。

## 3.27 Settings Sync 服务

`settingsSync/index.ts`（582 行）——双向设置同步：

**上传**——`uploadUserSettingsInBackground()`：
- 仅交互式 CLI、需要 OAuth
- 增量上传（仅变更条目）
- 10 秒超时，3 次重试，每文件 500KB 限制

**下载**——`downloadUserSettings()`：
- CCR 模式（headless）
- fire-and-forget 然后在插件安装前等待

**`redownloadUserSettings()`**——`/reload-plugins` 的新鲜下载（0 次重试）。

**同步键**——用户设置、用户内存、项目设置、项目内存（按 git remote hash 键值）。

**写入使用**`markInternalWrite()`——抑制虚假 chokidar 事件。

**资格要求**——`getAPIProvider() === 'firstParty'`且`isFirstPartyAnthropicBaseUrl()` 且具有 `user:inference` 范围的 OAuth 令牌。

## 3.28 Three-level Settings Cache

`settingsCache.ts`（81 行）——三级缓存：

| 级别 | 存储 | 失效 |
|------|------|------|
| Session | 单次合并的 `SettingsWithErrors` | `resetSettingsCache()` |
| Per-source | `Map<SettingSource, SettingsJson>` | `resetSettingsCache()` |
| Parse-file | `Map<string, ParsedSettings>`（按完整磁盘路径键值） | `resetSettingsCache()` |

+ `pluginSettingsBase`——插件提供的最低优先级基础层

**为何集中式 cache reset**——`changeDetector.fanOut()` 中的缓存重置是集中式的，以防止 N 路抖动——多个观察者不应各自独立地使缓存失效。

---
