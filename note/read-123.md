## 第 124 站：`src/tools/GlobTool/`（3 个文件，~20K）

### 这是什么

GlobTool 是文件模式匹配工具——类似于 Bash 中的 `find` 或 `ls` 模式搜索，但它使用 fast-glob 库实现。与 GrepTool 搜索内容不同，GlobTool 只搜索文件名。

---

### 输入 Schema

```ts
{
  pattern: string     // glob 模式："**/*.js", "src/**/*.ts"
  path?: string       // 搜索目录，省略时用 CWD
}
```

### 输出 Schema

```ts
{
  durationMs: number
  numFiles: number
  filenames: string[]       // 匹配文件（相对路径）
  truncated: boolean        // 超过 100 被截断
}
```

---

### 关键特征

#### 权限匹配器

```ts
async preparePermissionMatcher({ pattern }) {
  return rulePattern => matchWildcardPattern(rulePattern, pattern)
}
```

权限规则中的 glob 模式与实际 glob 模式匹配——这允许用户设置类似 `allow read **/*.ts` 的规则。

#### 实现抽象

```ts
const fs = getFsImplementation()
// ...
const { files, truncated } = await glob(pattern, path, { limit }, signal, permContext)
```

`getFsImplementation()` 返回 `realFs` 或 `mockFs`——允许测试中注入虚拟文件系统。`glob()` 是内部封装的 fast-glob，不是直接调用外部库。

#### 结果截断

```ts
const limit = globLimits?.maxResults ?? 100
```

默认最大 100 个结果——防止大目录（如 node_modules）返回数千条结果浪费 token。

---

### 与 GrepTool 的对比

| 维度 | GlobTool | GrepTool |
|---|---|---|
| 搜索目标 | 文件名 | 文件内容 |
| 底层引擎 | fast-glob | ripgrep |
| 输出 | 文件路径数组 | 匹配行 + 行号 |
| 截断 | 100 个结果 | 250 行 |
| 分页 | 无 | offset + head_limit |

GlobTool 更轻量——只做文件名匹配，不做内容扫描。

---

### 元

问题：**这一站真正想解决的架构问题是什么？**

回答：这一站把文件名搜索单独抽出来，与 Grep 的内容搜索形成分工。它通过 fast-glob、权限匹配器和结果上限，强调的是“轻量定位文件”的能力边界，而不是万能搜索。

### 反

问题：**如果把这一站的设计反过来，会发生什么？**

回答：如果把文件查找和内容查找混成一件事，简单的定位也会被更重的全文扫描拖慢，还更容易浪费上下文。系统最终会缺少一个足够便宜的第一步。

### 空

问题：**跳出当前文件名，这一站背后更大的问题是什么？**

回答：更大的问题是，探索代码库时是否应该先解决“去哪里看”，再解决“看到了什么”。GlobTool 给这个探索顺序提供了清晰、廉价的起点。

### 读完后应抓住的 2 个事实

1. **GlobTool 是 GrepTool 的补充**——GlobTool 搜索文件名，GrepTool 搜索内容。两者经常一起使用：先用 Glob 找文件，再用 Grep 搜内容。

2. **结果截断保护**——默认 100 个结果限制防止大目录返回海量结果。

---

## 第 125 站：`src/tools/SleepTool/`（2 个文件）

### 这是什么

SleepTool 是模型等待的工具——不占用 shell 进程，比 `Bash("sleep N")` 更高效。这是 Claude Code 内部的定时器而非外部命令。

---

### 输入 Schema

```ts
{
  duration: number    // 秒
}
```

### 输出 Schema

```ts
{}  // 简单确认
```

---

### 特征

#### 并发安全

```ts
isConcurrencySafe() { return true }
```

模型可以同时调用 Sleep + 其他工具——Sleep 不会阻塞其他操作。

#### Tick 提示

```
You may receive <tick> prompts — these are periodic check-ins.
Look for useful work to do before sleeping.
```

Sleep 期间 Claude 可以接收 `tick` 提示——这是定期检查是否有工作可做。这防止模型长时间无响应。

#### 5 分钟缓存过期

```
Each wake-up costs an API call, but the prompt cache expires after 5 minutes
of inactivity — balance accordingly.
```

Sleep > 5 分钟会导致 prompt 缓存过期——下次唤醒需要全量 token。引导模型在长时间等待时用更长的间隔。

#### 用户可中断

```
The user can interrupt the sleep at any time.
```

用户不必等待 Sleep 完成——可以提前发送消息中断。

---

### 元

问题：**这一站真正想解决的架构问题是什么？**

回答：这一站把等待从 shell 命令里拆出来，变成 Claude Code 内部的并发安全定时器。它看起来很小，但 tick 提示、可中断和 5 分钟缓存边界都说明“等待”本身也需要被系统理解。

### 反

问题：**如果把这一站的设计反过来，会发生什么？**

回答：如果继续让模型用 `Bash("sleep")`，就会无谓占住外部进程，还把等待和 shell 资源绑死。更糟的是，模型会更难意识到长等待对 prompt cache 的真实成本。

### 空

问题：**跳出当前文件名，这一站背后更大的问题是什么？**

回答：更大的问题是，AI 在非工作时段应该如何“什么都不做”，且不把系统资源白白耗掉。SleepTool 其实是在把空转行为制度化。

### 读完后应抓住的 2 个事实

1. **Sleep 是内部定时器**——不启动外部进程，比 Bash("sleep") 更高效。它不会占用 shell 槽。

2. **5 分钟缓存边界**——Sleep >5 分钟会导致 prompt 缓存过期。这引导模型选择更短间隔或考虑缓存成本。

---

## 第 126 站：`src/tools/PowerShellTool/`（12 个文件，~68K）

### 这是什么

PowerShellTool 是 Windows PowerShell 的集成执行工具——与 BashTool 不同，它不仅仅是子进程调用，而是有完整的命令语义分析、权限分类、和 Git 安全保障。

---

### 架构概览

文件组成：

| 文件 | 行数 | 职责 |
|---|---|---|
| `PowerShellTool.tsx` | 1000 | 主工具定义 + 执行 |
| `pathValidation.ts` | 2049 | 路径验证和安全 |
| `powershellSecurity.ts` | 1090 | PowerShell 安全机制 |
| `readOnlyValidation.ts` | 1823 | 只读命令分析 |
| `powershellPermissions.ts` | 1648 | 权限分类 |
| `modeValidation.ts` | 404 | 不同模式的命令验证 |
| `gitSafety.ts` | 176 | Git 安全操作指南 |
| `destructiveCommandWarning.ts` | 109 | 破坏性命令警告 |
| `commandSemantics.ts` | 142 | 命令语义分类 |
| `clmTypes.ts` | 211 | 类型定义 |
| `commonParameters.ts` | 30 | 常见参数常量 |
| `prompt.ts` | 145 | 提示词 |

### 核心特征

与 BashTool 的简单子进程执行不同，PowerShellTool 有：

1. **命令语义分析**——`commandSemantics.ts` 分类哪些命令是只读、破坏性、安全的
2. **路径安全验证**——`pathValidation.ts`（2049 行）处理路径注入、NTLM 泄露、UNC 路径
3. **权限分类**——与 BashTool 的 `checkPermissions` 类似但更细粒度
4. **Git 安全**——`gitSafety.ts` 有 Git 操作指南（`push --force`、`reset --hard` 等）
5. **模式验证**——`modeValidation.ts` 根据当前权限模式限制命令
6. **破坏性警告**——`destructiveCommandWarning.ts` 在执行前向用户显示具体警告

### 与 BashTool 的区别

- BashTool 是通用 shell 执行器——任何命令都能运行
- PowerShellTool 在 Bash 之上有一层语义分类层——它理解 PowerShell 的具体命令（`Get-ChildItem`、`Set-Content` 等）并做更细粒度的权限和分类

这是专门为 Windows 用户做的工具。

---

### 元

问题：**这一站真正想解决的架构问题是什么？**

回答：这一站再次把焦点放在 Windows shell 的完整语义层上，尤其突出 `pathValidation.ts`、`readOnlyValidation.ts` 和 `powershellPermissions.ts` 的规模。它强调的是 PowerShell 不是 Bash 的附庸，而是需要自己整套规则学的执行环境。

### 反

问题：**如果把这一站的设计反过来，会发生什么？**

回答：如果把 PowerShell 只当成“Windows 版 Bash”，cmdlet 级语义、安全模型和 Git 路径处理都会被压扁。那样表面统一了接口，实际上牺牲了平台原生的判断精度。

### 空

问题：**跳出当前文件名，这一站背后更大的问题是什么？**

回答：更大的问题仍然是，多平台命令执行怎样既共用抽象，又不牺牲本地语义。126 站比 102 站更像一次总括，强调这套复杂度不是偶发，而是平台现实本身。

### 读完后应抓住的 2 个事实

1. **PowerShellTool 有复杂的语义分析层**——不是简单的子进程包装器。`commandSemantics.ts`、`readOnlyValidation.ts`（1823 行）、`pathValidation.ts`（2049 行）一起形成安全屏障。

2. **Git 安全是内置的**——`gitSafety.ts` 有 Git 操作指南，与 BashTool 的通用 shell 安全不同。

---

## 下一步

剩余未处理工具目录：
- TeamCreateTool / TeamDeleteTool
- RemoteTriggerTool
- McpAuthTool / ListMcpResourcesTool / ReadMcpResourceTool
- REPLTool
- SyntheticOutputTool
- ConfigTool (文档待写)
- shared/
