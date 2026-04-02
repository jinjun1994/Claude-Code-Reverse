## 第 101 站：`src/tools/BashTool/` 剩余文件

### pathValidation.ts（~1303 行）

路径约束验证层——检查命令中涉及的文件路径是否在允许的目录范围内，以及输出重定向的目标是否安全。

#### `checkPathConstraints()`
权限流程中的路径约束入口：
1. 进程替换 `<()` `>()` 检测 → ask（AST 路径已被 too-complex 阻断）
2. 使用 AST 或 shell-quote 提取输出重定向
3. 重定向目标含 `$VAR` / `%VAR%` 展开语法 → ask
4. `validateOutputRedirections()` 验证每个重定向目标路径
5. AST 路径：用 `validateSinglePathCommandArgv()` 直接验证 AST argv
6. Legacy 路径：`splitCommand_DEPRECATED` + `validateSinglePathCommand()`

#### `validateSinglePathCommand()`
对单个子命令进行路径验证：
- 使用 `parseCommandArguments()` 提取命令的参数
- 根据命令类型（`PathCommand` 枚举：60+ 命令）调用对应的 `PATH_EXTRACTORS`
- 每个提取器返回该命令涉及的文件路径
- `validatePath()` 检查每个路径是否在允许的目录内

#### `PATH_EXTRACTORS` 模式
```ts
cat: extractFilePaths(args)          // 所有参数都是文件路径
grep: extractFilePaths(args)         // 排除 pattern 后的路径
find: extractFindPaths(args)         // 特殊的 find 路径提取
git: extractGitPaths(args)           // git 特定的路径逻辑
mv: extractSourceDestPaths(args)     // 源文件 + 目标目录
cp: extractSourceDestPaths(args)
rm: extractRemovalPaths(args)        // 包含危险路径检测
sed: extractSedFilePaths(args)       // -i/-f 参数提取
...
```

#### `stripWrappersFromArgv()`
AST 路径的包装器剥除——与 `stripSafeWrappers()` 相同逻辑但操作 argv 数组而非字符串。处理 `timeout` 的完整 flag 解析（`skipTimeoutFlags()` 函数），`nice` 的三种形式（`nice cmd` / `nice -n N cmd` / `nice -N cmd`），`stdbuf` 的短 flag 融合。

---

### sedValidation.ts（~322 行）

Sed 命令专用的安全验证——因为 sed 可以写入文件（`-i` 标志）和执行外部命令（`/e` 标志），需要特殊处理。

#### 三个 sed 安全模式
1. **`isLinePrintingCommand()`**：`sed -n 'Np'` 或 `sed -n '1p;2p;3p'`——只打印行，不修改文件
2. **`isFileReadCommand()`**：`sed 'q'` 或 `sed 'N,Md'`——只读操作（打印/删除行但不写回）
3. **`isEditWithNoOtherCommand()`**：`sed -i 's/old/new/'`——只包含一个 `s///` 命令的编辑

#### `sedCommandIsAllowedByAllowlist()`
- 允许的 flag：`-n`（静默）、`-E`（扩展正则）、`-r`（扩展正则）
- 不允许的 flag：`-i`（in-place 编辑，除非通过上面的模式检查）

#### `checkSedConstraints()`
sed 在权限检查中的入口函数——组合上面的三个模式进行判定。

---

### shouldUseSandbox.ts（153 行）

判断一个命令是否应该使用沙箱执行。

#### `shouldUseSandbox()`
```ts
1. 沙箱未启用 → false
2. dangerouslyDisableSandbox 且策略允许 → false
3. 命令包含排除命令 → false
4. 否则 → true
```

#### `containsExcludedCommand()`
- Ant 用户：检查 dynamic config（`tengu_sandbox_disabled_commands`）中的命令和子字符串
- 所有用户：检查 settings 中的 `sandbox.excludedCommands`
- 复合命令按 `&&` 分割后逐个检查——防止通过复合命令逃逸沙箱
- **固定点迭代**：对每个子命令迭代应用 `stripAllLeadingEnvVars` 和 `stripSafeWrappers`，直到无新候选产生——处理 `timeout 300 FOO=bar bazel run` 这种交错模式

---

### modeValidation.ts（115 行）

权限模式验证——检查命令是否符合当前的 permission mode（acceptEdits、default、locked 等）。

#### `checkPermissionMode()`
```ts
1. acceptEdits 模式：mv/cp/rm/echo 等写入命令自动允许
2. default 模式：标准权限检查
3. locked 模式：只读命令自动允许，写入命令 ask
```

---

### bashCommandHelpers.ts（265 行）

复合命令操作符检查——处理管道（`|`）、逻辑与（`&&`）、逻辑或（`||`）等连接符。

#### `checkCommandOperatorPermissions()`
- 对管道中每个段单独调用 `bashToolHasPermission()`
- 特殊处理：标准化 cd 命令、标准化 git 命令
- 管道段的"allow"结果仍然需要验证原始命令的路径约束

---

### destructiveCommandWarning.ts（102 行）

破坏性命令警告——针对 `rm -rf`、`mv` 覆盖等操作的额外提醒。

---

### commandSemantics.ts（140 行）

命令语义分析——与 AST 配合检查特定命令的危险性。

#### `checkSemantics()`
检查 AST 中的命令是否存在语义级别危险：
- zsh 内置命令（zmodload、emulate、eval 等）
- 已知不安全的命令模式
- 返回 `{ ok: boolean, reason?: string }`

---

### utils.ts（223 行）

BashTool 的辅助工具函数——包括子命令拆分、cd 前缀过滤等通用逻辑。

---

### toolName.ts（2 行）
工具名称常量导出。

---

### commentLabel.ts（13 行）
注释标签工具——为 Bash 命令的注释部分提供 UI 标签。

### BashToolResultMessage.tsx（190 行）
BashTool 结果的 React 渲染组件——显示命令输出、退出码、截断标记等。

### UI.tsx（184 行）
BashTool 的 UI 渲染组件组——命令执行状态、进度、结果的 React 组件。

### prompt.ts（369 行）
BashTool 的模型提示文本——生成给 LLM 的工具描述，包括使用示例、注意事项。

### sedEditParser.ts（322 行）
Sed 编辑命令解析器——解析 `s/old/new/g` 等 sed 替换模式，计算实际修改内容以便预览。

---

### 元

问题：**这一站真正想解决的架构问题是什么？**

回答：这一站的重心不是再讲 Bash 怎么执行，而是把路径校验、sed 安全、沙箱启用、命令语义这些“外围护栏”单独拉成一层。`checkPathConstraints()`、`PATH_EXTRACTORS` 和 `shouldUseSandbox()` 共同说明，真正难的是在命令真正落地前先把边界讲清楚。

### 反

问题：**如果把这一站的设计反过来，会发生什么？**

回答：如果把这些判断散回各个命令分支，最先失控的就是重定向、进程替换和 wrapper 剥离这类细碎但危险的角落。表面上少了一层抽象，实际上会让 Bash 安全策略变成到处漏风的补丁集合。

### 空

问题：**跳出当前文件名，这一站背后更大的问题是什么？**

回答：再往上看，这一站讨论的不是某个 shell 细节，而是“开放命令执行能力如何仍然保持可验证边界”。BashTool 的这些剩余文件，本质上是在回答系统怎样把自由度和约束同时保住。

### 读完这一站后，你应该抓住的 5 个事实

1. `checkPathConstraints()` 有两个路径：AST 路径（当 `astCommands` 存在时使用 `validateSinglePathCommandArgv()` 直接处理 argv）和 Legacy 路径（`splitCommand_DEPRECATED` + `validateSinglePathCommand()` 重新 tokenization）。AST 路径避免了 shell-quote 的单引号反斜杠 bug——这个 bug 会导致路径检查被完全跳过。

2. `shouldUseSandbox()` 中的排除命令检查使用固定点迭代——交替应用环境变量剥除和包装器剥除，直到不再产生新候选。这处理 `timeout 300 FOO=bar bazel run` 这种交错模式，单次剥除无法匹配 `bazel:*`。

3. Sed 有三个安全模式可以允许写入操作：line printing（`-n 'Np'`）、file read（`'q'` 或 `'Nd'`）、single edit（只含 `s/old/new/` 的 `-i`）——这些模式只包含单一的文本操作，不含分号连接的多个 sed 命令。

4. `parseCommandArguments()` 通过 `PATH_EXTRACTORS` 为 60+ 命令定义了专用的路径提取逻辑——每个命令类型知道自己哪些参数是文件路径、哪些是 flag、哪些是数据。这比通用的"所有参数都是文件路径"更精确。

5. 进程替换 `<(cmd)` 和 `>(cmd)` 在 pathValidation.ts 的顶部被直接拦截为 ask——因为执行的 `cmd` 可以写入文件而不出现在重定向目标中（如 `echo secret > >(tee .git/config)` 中的 `tee` 写入行为不被重定向分析捕获）。
