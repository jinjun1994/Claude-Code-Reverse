## 第 99 站：`src/tools/BashTool/bashPermissions.ts`（2621 行）

### 这是什么文件

`bashPermissions.ts` 是 BashTool 的权限判定中枢——将命令解析、安全验证、规则匹配和 classifier 集成在一起，决定一个 shell 命令是 allow/ask/deny。

---

### 核心出口：`bashToolHasPermission()`

```ts
async function bashToolHasPermission(
  input: { command },
  context: ToolUseContext,
  getCommandSubcommandPrefixFn
) → PermissionResult
```

---

### 权限决策流程（按顺序）

#### 0. 解析和安全检查
```
TREE_SITTER_BASH_SHADOW (shadow 模式)
  → parseCommandRaw() 获取 AST
  → parseForSecurityFromAst() 分析
  → 记录 shadow telemetry 但强制走 legacy 路径

AST 结果:
  'too-complex' → 检查 deny 规则 → ask
  'simple' → checkSemantics() → deny/ask/passthrough
  'parse-unavailable' → 回退到 legacy shell-quote 路径
```

#### 1. 沙箱 auto-allow
如果使用沙箱且 `shouldUseSandbox(input)` 为 true，则检查沙箱自动允许规则。

#### 2. 精确匹配
```ts
bashToolCheckExactMatchPermission() // allow/deny/passthrough
```
精确命令的 allow/deny 是最优先的规则类型。

#### 3. Bash prompt deny/ask 规则（并行 LLM classifier）
```ts
Promise.all([
  classifyBashCommand(input.command, getCwd(), denyDescriptions, 'deny'),
  classifyBashCommand(input.command, getCwd(), askDescriptions, 'ask'),
])
```
- Deny 优先于 ask
- 两个都需要高置信度才能生效
- 使用 Haiku 模型进行分类

#### 4. 操作符权限检查
```ts
checkCommandOperatorPermissions() // 处理管道、&&、|| 等复合命令
```
- 对管道命令中的每个段单独检查
- 'allow' 的段仍然需要验证原始输出的路径约束和安全模式

#### 5. 旧版解析器门控（parse-unavailable 时）
```ts
bashCommandIsSafeAsync() // 23 个验证器链
// 如果是 misparsing ask → 直接 ask（除非有精确 allow）
// 剥离安全 heredoc 后再检查剩余部分
```

#### 6. 子命令拆分
```ts
astSubcommands ?? shadowLegacySubs ?? splitCommand(input.command)
filterCdCwdSubcommands() // 移除 `cd ${cwd}` 前缀
```
超过 50 个子命令 → ask（防止 ReDoS 导致的事件循环饥饿）。

#### 7. 多 cd 命令检查
多个 `cd` 命令 → ask（单命令中多次目录切换不清楚意图）。

#### 8. cd + git 混合检查
复合命令中同时包含 `cd` 和 `git` → ask（防止裸 git 仓库的 core.fsmonitor 攻击）。

#### 9. 每个子命令的权限检查
```ts
subcommands.map(cmd => bashToolCheckPermission({ command: cmd }, ...))
```

#### 10. 输出重定向验证
```ts
checkPathConstraints(input, getCwd(), ..., astRedirects, astCommands)
```
在原始命令上验证，因为 splitCommand 剥除了重定向。

#### 11. 合并结果
- 任何子命令 deny → deny
- 单 ask（只有一个非-allow 子命令）→ ask
- 全部 allow 且无命令注入 → allow
- 否则 → checkCommandAndSuggestRules() 生成 UI 提示

---

### `stripSafeWrappers()` 包装器剥除

两阶段迭代剥除：
1. **阶段 1**：剥除注释行和安全环境变量（`TZ=UTC`、`NODE_ENV=prod` 等白名单）
2. **阶段 2**：剥除安全包装器

安全包装器白名单：
- `timeout`——带完整的 GNU flag 白名单（`--foreground`/`--kill-after=N`/`-k N`/`--signal=TERM` 等），flag 值使用 `[A-Za-z0-9_.+-]` 白名单
- `time`——直接放行
- `nice`——三种形式：`nice cmd` / `nice -n N cmd` / `nice -N cmd`
- `stdbuf`——短 flag 融合形式：`-o0`/`-eL` 等
- `nohup`——直接放行

危险前缀不被建议为规则：`sh`/`bash`/`zsh`/`env`/`xargs`/`sudo`/`nice`/`timeout`/`time`——这些包装器允许任意代码执行。

---

### matchWildcardPattern()

通配符匹配，支持：
- `*` 匹配任意字符
- `?` 匹配单字符
- `**` 匹配路径分隔符
- 大小写不敏感

---

### Speculative Classifier 机制

```ts
peekSpeculativeClassifierCheck()  // 预检是否可以分类
startSpeculativeClassifierCheck()  // 启动后台检查
consumeSpeculativeClassifierCheck() // 获取结果
awaitClassifierAutoApproval()      // 等待后台审批
executeAsyncClassifierCheck()      // 执行异步分类
```

这些机制允许分类器在后台异步运行，可能在用户响应提示前自动批准命令。

---

### 关键安全修复记录

1. **CC-643**：subcommand 数量上限 50，防止 splitCommand 在复杂命令下产生指数级子命令→每个子命令调用 tree-sitter + ~20 validators + logEvent→微任务链饿死事件循环。
2. **GH#11380**：复合命令最多建议 5 条规则——防止单提示保存过多噪声规则。
3. **HackerOne #3543050**：`stripSafeWrappers()` 阶段 2 不剥环境变量——因为在 `timeout VAR=val cmd` 中 VAR=val 是 cmd 的参数而非真正的赋值。
4. **timeout flag 值注入**：原来使用 `[^ \t]+` 匹配值，被 `timeout -k$(id) 10 ls` 绕过——现在必须匹配 `[A-Za-z0-9_.+-]+`。
5. **cd + git 核心攻击**：`cd evil && git status` 从有 core.fsmonitor 的裸仓库执行 → RCE。cd 路径检查单独检查不到 git 段，git 段也看不到 cd 段——必须在复合命令级别检查。
6. **`>` 管道绕过**：`echo 'x' | xargs printf '%s' >> /tmp/file`——管道处理剥除重定向后再检查每段，两段都被允许但 `>>` 重定向没被验证。现在"allow"的管道段仍然检查原始输出路径约束。

---

### `getSimpleCommandPrefix()`

提取命令前缀用于规则建议：
```
'git commit -m "fix"' → 'git commit'
'NODE_ENV=prod npm run build' → 'npm run'
'MY_VAR=val npm run build' → null（非安全变量）
'ls -la' → null（flag，非子命令）
```
第二词必须匹配 `/^[a-z][a-z0-9]*(-[a-z0-9]+)*$/`——排除 flag、文件名、路径、URL、数字。

---

### 元

问题：**这一站真正想解决的架构问题是什么？**

回答：这一站真正要说明的是，bash 命令权限为什么不能只看命令名，而要把解析、安全验证、规则匹配与 classifier 放在一起。它是 BashTool 从“能否执行”到“应否放行”的真正决策中枢。

### 反

问题：**如果把这一站的设计反过来，会发生什么？**

回答：如果安全验证和权限判断彼此脱节，就会出现“语义危险但规则允许”或“规则不明确但安全层已看出问题”的断裂。那样 ask/allow/deny 将不再对应真实风险。

### 空

问题：**跳出当前文件名，这一站背后更大的问题是什么？**

回答：更大的问题，是通用命令执行器怎样把语法理解、风险理解和产品权限模式合成一个决定。`bashPermissions.ts` 就是这种合成点。

### 读完这一站后，你应该抓住的 6 个事实

1. bashToolHasPermission 的执行顺序很关键：AST 解析 → 沙箱检查 → 精确匹配 → LLM classifier deny/ask → 操作符检查 → 旧版安全门控 → 子命令拆分 → 多 cd 检查 → cd+git 检查 → 每子命令权限 → 输出路径约束 → 合并结果。任何步骤的 deny 直接返回；多个 ask 时才合并到 UI 中让用户决定。
2. Tree-sitter shadow 模式（`TREE_SITTER_BASH_SHADOW`）是一个观察性 A/B 测试——parse 结果用于记录 telemetry 但强制走 legacy 路径，避免新解析器的 bug 直接影响用户。这确保新代码在发布前可以在真实流量下被验证。
3. `stripSafeWrappers()` 是一个递归迭代过程——只要结果改变就继续剥，因为包装器可以嵌套（`timeout nohup cmd`）。两阶段的划分（环境变量 vs 包装器）是关键：在包装器之后出现的环境变量赋值被 bash 当作命令而非赋值来执行。
4. cd + git 复合命令检查是防止裸仓库 fsmonitor RCE 的关键安全修复。cd 和 git 分开看起来都安全，但 `cd evil-repo && git status` 执行 evil-repo/.git/config 中的 core.fsmonitor——这是一个经过验证的攻击路径。
5. Speculative classifier 机制支持命令的自动批准——分类器在用户提示时已经开始在后台运行，如果置信度足够高可以在用户回应前批准确定命令。这在 auto 模式下最关键。
6. CC-643 是一个关键的性能 bug——splitCommand 在复杂命令上可能产生指数级数量的子命令，每个子命令触发完整的安全检查链和 logEvent 调用（每个 logEvent 触发 /proc/self/stat 读），最终饿死事件循环并冻结 REPL。50 个子命令上限是一个保守的安全帽。
