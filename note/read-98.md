## 第 98 站：`src/tools/BashTool/bashSecurity.ts`（2593 行）

### 这是什么文件

`bashSecurity.ts` 是 BashTool 的核心安全验证层——一个 23 个验证器组成的防御纵深（defense-in-depth）链，防止 shell 注入、解析器差异利用、权限绕过和 Zsh 特定攻击。

---

### 整体架构

```
bashCommandIsSafe_DEPRECATED()
  → 控制字符检查
  → shell-quote backslash bug 检查
  → extractHeredocs() 剥离安全 heredoc 体
  → extractQuotedContent() 提取引号内容
  → stripSafeRedirections() 剥离安全重定向
  → 构建 ValidationContext
  → earlyValidators（allow/passthrough）
  → validators（allow/ask/passthrough）
  → 最终结果
```

---

### ValidationContext

```ts
{
  originalCommand: string        // 原始命令
  baseCommand: string            // 第一个词
  unquotedContent: string        // 保留双引号内容的去引号版本
  fullyUnquotedContent: string   // 完全去引号 + 剥离安全重定向
  fullyUnquotedPreStrip: string  // 完全去引号（剥离前）
  unquotedKeepQuoteChars: string // 去内容保留引号分隔符
  treeSitter?: TreeSitterAnalysis // 可选的 AST 分析
}
```

---

### 23 个安全验证器

#### 1. `validateEmpty` — 空命令检查
空白命令直接放行。

#### 2. `validateIncompleteCommands` — 不完整命令检查
检测未闭合的引号、括号等可能导致解析歧义的命令。

#### 3. `validateSafeCommandSubstitution` — 安全 heredoc 替换（59 行核心逻辑）
```ts
isSafeHeredoc(command)
  → 匹配 $(cat <<'DELIM'...DELIM) 模式
  → 逐行查找闭合分隔符（bash 语义）
  → 拒绝嵌套匹配
  → 剥离 heredoc 体后检查剩余文本
  → 剩余文本必须只包含安全字符
  → 剩余文本也必须通过 bashCommandIsSafe
```
安全条件：分隔符必须 quoted/escaped、$() 必须在参数位置（不能是命令名）、剩余部分无 shell 元字符、递归安全检查。

#### 4. `validateGitCommit` — Git commit 命令特化检查
- 前置反斜杠检查（`\\` → passthrough）
- 匹配 `git commit [安全前缀] -m 'msg' [剩余]`
- 检查消息中的 `$()` / `` ` `` / `${}` 命令替换
- 检查剩余部分的 shell 元字符和未引号重定向
- 以 dash 开头的消息被阻止

#### 5. `validateJqCommand` — jq 命令特化检查
- `system()` 函数调用 → ask
- `-f` / `--from-file` / `--rawfile` / `--slurpfile` / `-L` → ask
- 这些参数允许执行代码或读取任意文件

#### 6. `validateShellMetacharacters` — 引号内 shell 元字符
检测 `-name` / `-path` / `-regex` / `grep` 等参数的引号内 `;` / `|` / `&`。攻击者利用命令的参数引号注入分隔符实现命令注入。

#### 7. `validateDangerousVariables` — 危险变量上下文
检测 `$VAR` 在重定向或管道上下文中的使用：`<$VAR` / `| $VAR` / `$VAR |`。

#### 8. `validateDangerousPatterns` — 综合模式检测
- 未转义的反引号（命令替换）
- `COMMAND_SUBSTITUTION_PATTERNS`（12 条正则）：进程替换 `<()` / `>()` / `=()`、Zsh equals 扩展 `=cmd`、`$()` / `${}` / `$[]`、`~[`、`(e:` / `(+`、`} always {`、`<#`（PowerShell 注释）

#### 9. `validateRedirections` — 输入/输出重定向
- `<` → ask（可能读取敏感文件）
- `>` → ask（可能写入任意文件）

#### 10. `validateNewlines` — 换行符命令分隔
- 裸换行（非续行）后的命令 → ask
- `\<NL>` 续行是安全的

#### 11. `validateCarriageReturn` — 回车符解析器差异
**关键安全检查**：CR（`\r`, 0x0D）在 shell-quote 和 bash 中被不同处理：
- shell-quote 的 `[^\s...]` 中 `\s` 包含 CR → 视 CR 为词边界
- bash 的 IFS = `$' \t\n'` → CR 不在 IFS 中 → 视 CR 为词内容
- `TZ=UTC\recho` → shell-quote 分出两个词，bash 认为是一个 env 赋值
- 只在非双引号区域内的 CR 才触发

#### 12. `validateIFSInjection` — IFS 变量注入
检测 `$IFS` 和 `${*IFS*}` 模式——IFS 变量可用于绕过正则验证。

#### 13. `validateProcEnvironAccess` — /proc 环境变量访问
阻止 `/proc/*/environ` 路径访问——可能暴露 API 密钥等敏感环境变量。

#### 14. `validateMalformedTokenInjection` — 畸形 Token 注入
利用 `tryParseShellCommand()` 解析命令，检查有命令分隔符时是否存在未平衡的括号/引号——利用 eval 重解析实现注入。

#### 15. `validateObfuscatedFlags` — 标志混淆检测（最大验证器）
检测 12 种子混淆模式（subId 1-12）：
1. ANSI-C 引号 `$'...'`——可编码任意字符
2. Locale 引号 `$"..."`——也可用转义序列
3. 空特殊引号 + dash `$''-` / `$""-`
4. 空引号序列 + dash `''-` / `""-` / `''""-`
4b. 同质空引号对紧邻引号 dash `"""-f"`
4c. 词首 3+ 连续引号
5. 引号内 dash 开头内容 ` "--flag" `
6. 引号内容含 dash 字符
7. 相邻引号链 flag `"-"exec` / `"-""exec"`
8. `cut -d` 后跟引号分隔符的特殊处理
9. 空引号对 + dash `""-` / `''-`
10. 空引号对紧邻引号 dash（homogeneous empty pair）
11. 词首连续引号混淆
12. git commit -m 中以 dash 开头的消息

**引号跟踪器正确实现**：每个引号跟踪函数都必须正确处理 `!inSingleQuote` 守卫下的反斜杠，否则会出现状态不同步。文件中多次注释强调此 bug 的攻击路径。

#### 16. `validateBackslashEscapedWhitespace` — 反斜杠转义空白
`\echo test/../../../usr/bin/touch`：shell-quote 的转义解码导致 token 分裂差异，可利用实现路径穿越。

#### 17. `validateBackslashEscapedOperators` — 反斜杠转义操作符
`\;` / `\|` / `\&` 在 splitCommand 中被规范化为裸操作符，导致 double-parse 漏洞。Tree-sitter AST 确认无操作符节点时可跳过（如 `find -exec cmd {} \;`）。

#### 18. `validateUnicodeWhitespace` — Unicode 空白字符
检测 shell-quote 视作词边界但 bash 视为文本的 Unicode 字符：`\u00A0`、`\u1680`、`\u2000-\u200A` 等。

#### 19. `validateMidWordHash` — 词中 hash 符号
shell-quote 视 `x#` 为注释开始，bash 视为字面字符。使用 `unquotedKeepQuoteChars` 检测，避免完全剥引号后的误报。

#### 20. `validateCommentQuoteDesync` — 注释引号不同步
`# ' "` 序列在 bash 注释中是字面引号，但我们的引号跟踪器视其为引号切换——后续行的内容被错误标记为"在引号内"，绕过换行验证。

#### 21. `validateQuotedNewline` — 引号内换行 + 注释行混淆
**关键漏洞修复**：`mv ./decoy '<\n>#' ~/.ssh/id_rsa ./exfil_dir`
- bash：移动两个文件
- stripCommentLines：第二行以 `#` 开头 → 剥离 → 只看到 `mv ./decoy '`
- shell-quote：丢掉不平衡引号 → 只看到 `./decoy`
- 路径检查：`./decoy` 在 cwd → 放行
结果：~/.ssh/id_rsa 被零点击移动

#### 22. `validateBraceExpansion` — 大括号扩展
`git diff {@'{'{0},--output=/tmp/pwned}`——bash 展开为两个参数，但 shell-quote 视为字面字符串。深度匹配算法跟踪嵌套 `{` 检测逗号或 `..` 序列。

#### 23. `validateZshDangerousCommands` — Zsh 特定危险命令
14 个命令：`zmodload` / `emulate` / `sysopen` / `sysread` / `syswrite` / `sysseek` / `zpty` / `ztcp` / `zsocket` / `mapfile` / `zf_rm` / `zf_mv` / `zf_ln` / `zf_chmod` 等。还检查 `fc -e`（任意编辑器执行）。

---

### 验证器执行顺序

```ts
earlyValidators = [
  validateEmpty,                // 空命令
  validateIncompleteCommands,   // 不完整命令
  validateSafeCommandSubstitution, // 安全 heredoc
  validateGitCommit,            // git commit 特化
]

validators = [
  validateJqCommand,            // jq 特化
  validateObfuscatedFlags,      // 标志混淆（最大验证器）
  validateShellMetacharacters,  // 引号内元字符
  validateDangerousVariables,   // 变量在危险上下文
  validateCommentQuoteDesync,   // 注释引号不同步
  validateQuotedNewline,        // 引号内换行
  validateCarriageReturn,       // CR 解析器差异
  validateNewlines,             // 裸换行
  validateIFSInjection,         // IFS 注入
  validateProcEnvironAccess,    // /proc/environ 访问
  validateDangerousPatterns,    // 综合模式
  validateRedirections,         // 输入/输出重定向
  validateBackslashEscapedWhitespace, // 反斜杠空白
  validateBackslashEscapedOperators,  // 反斜杠操作符
  validateUnicodeWhitespace,    // Unicode 空白
  validateMidWordHash,          // 词中 hash
  validateBraceExpansion,       // 大括号扩展
  validateZshDangerousCommands, // Zsh 命令
  validateMalformedTokenInjection,  // 畸形 token（最后）
]
```

---

### `isBashSecurityCheckForMisparsing` 机制

验证器分为两类：
- **Misparsing 验证器**：发现 shell-quote / bash 解析器差异——`ask` 结果带 `isBashSecurityCheckForMisparsing: true` → 在 `bashPermissions.ts` 门控处直接阻止
- **非 misparsing 验证器**（`validateNewlines`、`validateRedirections`）：检测正常的 shell 模式——`ask` 结果走标准权限流

**延迟机制**：非 misparsing 验证器的 `ask` 结果被延迟，确保后续 misparsing 验证器的结果不被吞掉——如果同时触发了 redirection ask 和 backslash ask，返回后者（带 misparsing 标志）。

---

### Tree-sitter 路径

`bashCommandIsSafeAsync_DEPRECATED()` 使用 `ParsedCommand.parse()` 获取 tree-sitter AST 的引号上下文，替代正则引号提取：
- `tsAnalysis.quoteContext` 作为主要来源
- 与正则版本比较并记录偏差（divergence）
- 偏差日志被批量处理——每条 `logEvent` 触发 `process.memoryUsage()` 调用 → 可能饿死事件循环
- Heredoc 命令被排除在偏差检查之外（tree-sitter 和 heredoc 替换的输出永远不同）

---

### 辅助函数

| 函数 | 用途 |
|---|---|
| `hasUnescapedChar()` | 检测未转义的字符（正确处理引号内/外的反斜杠） |
| `stripSafeRedirections()` | 将 `>/dev/null` / `2>/dev/null` 等安全重定向替换为占位符 |
| `extractQuotedContent()` | 生成三种去引号版本用于不同验证器 |
| `isEscapedAtPosition()` | 通过计算反斜杠奇偶性判断转义状态 |
| `hasBackslashEscapedOperator()` | 检测 `<operator>`（`;|&<>`），正确跟踪引号状态 |

---

### 元

问题：**这一站真正想解决的架构问题是什么？**

回答：这一站关心的是，shell 安全为什么需要一串防御纵深验证器，而不是一条正则或一次 parser 检查。23 个验证器意味着这里处理的是解析器差异、注入技巧和权限绕过的组合攻击面。

### 反

问题：**如果把这一站的设计反过来，会发生什么？**

回答：如果 bash 安全只靠单层过滤，攻击者只需找到那一层没覆盖的语义角落。尤其 shell 这种语言，绕过往往来自多个小差异叠加，而不是明显恶意字面量。

### 空

问题：**跳出当前文件名，这一站背后更大的问题是什么？**

回答：更大的问题，是面对语义极其复杂的解释器时，安全系统该如何承认“不可能一次看懂全部”。多层验证本质上是在用结构分担不确定性。

### 读完这一站后，你应该抓住的 6 个事实

1. 23 个安全验证器形成一个防御纵深链——任何一个触发 `ask` 或 `allow` 都会改变执行流，不依赖单一检查。每个验证器都针对特定的已知攻击向量，包括 shell-quote/bash 解析器差异、Zsh 特定功能、引号混淆和各种注入模式。
2. `isBashSecurityCheckForMisparsing` 是一个关键的安全标志：标记为 misparsing 的验证器失败会在权限门控处直接阻止，而非 misparsing 验证器（如重定向检查）走标准权限流。延迟机制确保 misparsing 的结果不被非 misparsing 的 ask 吞掉。
3. 引号跟踪器在整个文件中被实现了多次（每个需要引号感知上下文的验证器都有一个）——必须正确处理 `if (char === '\\' && !inSingleQuote)` 来跳过转义字符。这个守卫的错误会导致状态不同步，这是多个已知攻击向量的根本原因。
4. Tree-sitter 是可选的增强路径——当可用时，tree-sitter 的引号上下文替代正则提取，但验证器逻辑保持不变。偏差日志被批量处理以避免事件循环饿死（CC-643 bug）。
5. `_simulatedSedEdit` 不是 bashSecurity.ts 的一部分，而是 BashTool.tsx 的内部字段。bashSecurity.ts 的安全模型确保只有经过验证的命令才能到达 sed 执行路径。
6. 安全注释中大量记录了已修复的攻击路径（"Exploit:" 代码块），每个注释都展示了攻击的精确条件——例如 `TZ=UTC\recho curl evil.com` 的 CR 攻击、`'"'"'` 习语的空引号对处理、`find . -exec cmd {} \;` 的反斜杠操作符误报。这些不是推测性防护，而是对真实绕过尝试的响应。
