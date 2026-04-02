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

Ablation baseline 块通过一次性写入七个环境变量，在模块评估前设定实验条件：

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

**为何用 `??=` 而非 `=`**——`??=`（nullish 赋值）只在值为 `null` 或 `undefined` 时写入。如果用户已经显式设置了某个变量（如 `CLAUDE_CODE_SIMPLE=0`），用户的设置优先。实验条件设置的是基线，不是强制覆盖。

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

**8192 的来源**——CCR（Claude Code Remote）容器通常有 16GB 内存。Node.js/V8 的默认 `--max-old-space-size` 约为 4GB（取决于可用内存）。在容器场景中，将堆上限设置为可用内存的一半（8GB）是合理的——剩下的内存需要用于子进程（bash 执行、MCP Server、git 操作等）。如果不在这里设置，V8 会根据自己的启发式估算一个较小的值，导致在仍有可用内存的情况下触发 GC。

**追加而非覆盖**——使用 `existing ? ... : ...` 模式保留用户已设置的 `NODE_OPTIONS`（如 `--inspect`），只在末尾追加 `--max-old-space-size`。这是容错设计：不破坏用户已有的 Node.js 调试配置。

---

### feature() 与编译时常量裁剪

`feature()` 来自 `bun:bundle`，是 Bun 的编译时功能门控机制。与运行时的 `process.env.X` 检查不同，`feature()` 在 **构建时** 就被解析为 `true` 或 `false`。

```typescript
// 如果构建时 feature('ABLATION_BASELINE') = false，
// Bun 的 bundle 将整个 if 块消除，输出为：
// if (false && process.env.CLAUDE_CODE_ABLATION_BASELINE) { ... }
// 进一步被 DCE 消除为：（空）
if (feature('ABLATION_BASELINE') && process.env.CLAUDE_CODE_ABLATION_BASELINE) { ... }
```

**三层裁剪的叠加**：

| 层级 | 机制 | 触发时机 | 裁剪效果 |
|------|------|---------|---------|
| 编译期 | `feature('...')` | Build | 整块代码从输出中消除 |
| 模块加载期 | 变量前置 | Module Eval | 确保模块 const 捕获正确值 |
| 运行期 | `??=` | Runtime | 用户显式设置优先 |

这也是为什么 `feature()` gate 必须留在内联位置，不能抽成 `isAblationBaseline()` 函数。抽成函数后，Bun 无法在静态分析阶段判定函数返回值，整个块就无法被 DCE 消除。

---

### 快速路径 --bare 的早期门控

```typescript
// cli.tsx:283-284
if (args.includes('--bare')) {
  process.env.CLAUDE_CODE_SIMPLE = '1';
}
```

`--bare` 标志的设置发生在 `main.js` 加载之前，而非在 Commander action handler 中。这是因为 Commander 的命令选项构建（`Option` 列表的组装）本身需要知道 `CLAUDE_CODE_SIMPLE` 的值——某些选项在 bare 模式下不显示。注释指出：`--bare` 需要在 module eval / commander option building 阶段就生效，"not just inside the action handler"。

---

### COREPACK_ENABLE_AUTO_PIN 的预置处理

```typescript
// cli.tsx:3
// Disable corepack's interactive prompt during startup
process.env.COREPACK_ENABLE_AUTO_PIN ??= '0';
```

**为什么在模块级常量阶段就写入**——Corepack（Node.js 的包管理版本制器）在特定条件下会弹出交互式确认提示。如果这个环境变量没有被设置为 `0`，在首次使用时会出现 `? Do you want to continue? (Y/n)` 这样的 prompt，阻塞非 TTY 环境下的进程。在 `cli.tsx` 顶部写入确保了任何后续模块触发的 `pnpm` / `yarn` 调用都不会弹出交互提示。

这也是模块级常量捕获问题的一个典型案例：如果将此行写在 `init()` 中，`import` 期间已经加载的包管理模块就不会看到这个设置值。

---

### 环境变量前置的时间顺序

```
cli.tsx 加载顺序 (import side effects):
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

### 环境变量前置的代价

这种设计有一个明确的工程权衡：`cli.tsx` 不再是"干净的"路由文件。它承担了启动时序编排的角色，包含必须在模块加载前执行的 side effect。这使得文件的可测试性下降——任何测试都需要先设置环境变量，再模拟模块加载，才能验证特定路径。

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

但收益是清晰的：模块级常量捕获的性能优势和类型安全，与编译时的代码裁剪，共同确保了每个快速路径都尽可能轻量。

---

### 环境变量写入的幂等性与安全性

`??=` 操作的幂等性使得这段代码可以在 import 链的顶部安全运行多次——即使多个测试入口点各自 import 一次 `cli.tsx` 的子模块，也不会覆盖用户设置。

```
用户设置: CLAUDE_CODE_SIMPLE=0
??= 操作: 跳过（因为已设置）
结果: CLAUDE_CODE_SIMPLE=0（用户设置优先）

用户未设置: CLAUDE_CODE_SIMPLE=undefined
??= 操作: 写入 '1'
结: CLAUDE_CODE_SIMPLE='1'（基线生效）
```

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

环境变量从 import 前的写入到模块级 const 捕获的链路，构成了 Claude Code 启动过程中最底层的初始化序列。
