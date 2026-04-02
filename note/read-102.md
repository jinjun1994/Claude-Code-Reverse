## 第 102 站：`src/tools/PowerShellTool/`（14 个文件，~480K）

### 这是什么

PowerShellTool 是 Windows 平台的 shell 工具实现，架构完全复制了 BashTool 的模式：安全验证 → 权限检查 → 路径约束 → 只读判定 → 执行管理。差异在于 PowerShell 的语法差异（cmdlet 名称大小写不敏感、参数以 `-` 开头但用 `:` 分隔等）和特定的安全威胁面。

---

### 文件映射：BashTool ↔ PowerShellTool

| BashTool 文件 | PowerShellTool 文件 | 用途 |
|---|---|---|
| BashTool.tsx | PowerShellTool.tsx | 核心执行链 |
| bashPermissions.ts | powershellPermissions.ts | 权限判定 |
| bashSecurity.ts | powershellSecurity.ts | 安全验证 |
| pathValidation.ts | pathValidation.ts | 路径约束 |
| readOnlyValidation.ts | readOnlyValidation.ts | 只读判定 |
| modeValidation.ts | modeValidation.ts | 模式验证 |
| commandSemantics.ts | commandSemantics.ts | 语义分析 |
| shouldUseSandbox.ts | 共享 BashTool/shouldUseSandbox | 沙箱选择 |
| gitSafety.ts | gitSafety.ts | Git 安全路径 |
| commonParameters.ts | commonParameters.ts | PowerShell 通用参数 |

---

### PowerShellTool.tsx 核心

与 BashTool.tsx 相同的核心模式：
- 输入 schema：`{ command, timeout, description, run_in_background, dangerouslyDisableSandbox }`
- `runShellCommand()` 三阶段执行（初始等待 → 进度轮询 → 收尾）
- `EndTruncatingAccumulator` 输出截断
- 大输出持久化到 `tool-results/`
- 图片输出检测和尺寸限制
- 后台化机制：显式/超时/助手模式/用户操作

#### PowerShell 特有处理
- **EOL 强制使用 `\n`** 而非 `os.EOL`——`\r\n` 在 Windows 上会破坏 Ink 终端渲染
- **命令语义分类**对应 PowerShell cmdlet：
  - 搜索：`Select-String`、`Get-ChildItem`、`Findstr`、`Where.exe`
  - 读取：`Get-Content`、`Get-Item`、`Test-Path`、`Resolve-Path`
  - 列表：`Get-ChildItem`（非递归）
  - 静默：`Move-Item`、`Copy-Item`、`Remove-Item`、`New-Item`
  - 中性：`Write-Host`、`Write-Output`、`echo`

---

### powershellPermissions.ts 核心

#### `powershellToolHasPermission()` 流程
```
1. 解析：parsePowerShellCommand(command) → AST
2. 精确匹配 deny/allow 检查
3. 前缀/通配符 deny/ask 规则匹配
4. Parse 失败 → ask（解析错误信息）
5. deny 优先于 ask
6. 子命令迭代检查：每个语句单独验证
7. 只读检查：isReadOnlyCommand()
8. 路径约束：checkPathConstraints()
9. Git 安全检查
10. 合并结果
```

#### 关键安全修复
**前缀 ask defer 机制**：之前 `ask` 规则在子命令 deny 检查前直接返回，导致 `Get-Process; Invoke-Expression evil` 有 ask 规则时 ask 直接返回，deny 从不触发。现在 `ask` 结果被延迟到 `decisions[]` 数组中，在所有子命令检查完成后才合并。

---

### PowerShell 安全模型特有挑战

#### cmdlet 大小写不敏感
`Select-String`、`select-string`、`SELECT-STRING` 都是同一个 cmdlet。规则匹配必须规范化（全部小写或忽略大小写比较）。

#### 外部命令解析器
PowerShell 的命令解析与 bash 完全不同：使用 `parsePowerShellCommand()` AST（`../../utils/powershell/parser.js`）而非 shell-quote/tree-sitter。解析器处理：
- Module-qualified cmdlets（`Microsoft.PowerShell.Utility\Write-Host`）
- 参数绑定（`-Name:value` / `-Name value`）
- 管道和操作符（`|`、`&&`、`||`、`;`）
- 重定向（`>`、`>>`、`2>`、`*>&1`）

#### 模块前缀剥离
`stripModulePrefix()` 函数处理 cmdlet 的模块限定名，防止 `Module.SafeCmdlet` 与 `SafeCmdlet` 的匹配差异被利用。

---

### readOnlyValidation.ts（PowerShell 版本）

#### `isReadOnlyCommand()`
- PowerShell 特有的只读 cmdlet 列表
- 大小写不敏感匹配
- 处理 PowerShell 特有的参数格式

#### `isAllowlistedCommand()`
- 命令白名单验证——cmdlet 必须在允许列表中
- 处理模块限定名

---

### gitSafety.ts

PowerShell 版本的 Git 路径安全：
- `isDotGitPathPS()` — 路径是否为 `.git` 相关
- PowerShell 特有的路径规范化（大小写不敏感 Windows 路径）

---

### 与 BashTool 的关键差异

| 方面 | BashTool | PowerShellTool |
|---|---|---|
| 命令解析 | tree-sitter bash / shell-quote | PowerShell AST parser |
| 安全验证器 | 23 个独立正则检查 | 更少的检查，依赖 AST 解析 |
| 大小写 | 精确匹配 | 忽略大小写 |
| 环境变量 | `VAR=val` 前缀 | `$env:VAR = val` 赋值语句 |
| 包装器 | `timeout`/`nice`/`nohup` | 较少包装器需要处理 |
| Zsh 特定 | 大量检查 | 无（PowerShell 没有同等问题） |
| UNC 路径 | 不适用 | 需要检查 WebDAV 攻击 |
| 共享组件 | `isCommandSafeViaFlagParsing` 等 | 共享 `COMMAND_ALLOWLIST` |

---

### 元

问题：**这一站真正想解决的架构问题是什么？**

回答：这一站的重点是为 Windows 世界复制一套与 BashTool 对齐、但适配 PowerShell 语法的安全执行框架。大小写不敏感、cmdlet 语义、AST 解析和 ask 延迟合并，说明它不是简单换个 shell 壳子而已。

### 反

问题：**如果把这一站的设计反过来，会发生什么？**

回答：如果照搬 Bash 的字符串匹配思路，PowerShell 的模块前缀、参数绑定和大小写宽松都会把权限判断搞得失真。那样看似复用了架构，其实把平台差异变成了安全漏洞入口。

### 空

问题：**跳出当前文件名，这一站背后更大的问题是什么？**

回答：更大的问题是，多平台工具系统能不能做到“表面行为一致、底层判定各自原生”。这一站的价值，就在于它把跨平台一致性建立在各自语法现实之上，而不是强行统一。

### 读完这一站后，你应该抓住的 4 个事实

1. PowerShellTool 的权限模型使用统一的 AST 解析（`parsePowerShellCommand`）替代 BashTool 的 tree-sitter/shell-quote 双层模型——这意味着 PowerShell 命令的安全验证更单一但更依赖于解析器的正确性。BashTool 有两个独立解析路径作为防御纵深，而 PowerShellTool 依赖单个 parser。

2. PowerShell 的「延迟 ask」安全修复是关键的排序 bug：前缀/通配符 ask 规则曾经在子命令 deny 检查前返回，导致带有 ask 规则的复合命令永远不触及 deny。BashTool 也有同样问题但被稍后修复。

3. PowerShellTool 与 BashTool 共享 `shouldUseSandbox.ts` 和 `readOnlyCommandValidation` 中的命令列表——这是设计上的一致性，两种 shell 的安全模型应该对相同的文件操作产生相同的结果。

4. PowerShellTool 的 cmdlet 匹配必须是大小写不敏感的——`SELECT-STRING`、`Select-String`、`select-string` 都应该匹配同一个权限规则。这增加了额外的规范化层，不同于 BashTool 的精确字符串匹配。
