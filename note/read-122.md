## 第 122 站：`src/tools/EnterWorktreeTool/`（4 个文件，~24K）

### 这是什么

EnterWorktreeTool 为会话创建一个隔离的 git worktree——模型可以用它来做代码修改，不影响主分支。与 `git worktree add` 不同，它与 Claude Code 的状态管理集成，自动处理 CWD、缓存、和 hooks 配置。

---

### 输入 Schema

```ts
{
  name?: string   // 可选：worktree 名，格式为 k-path-segment（最大 64 字符）
                   // 不提供则自动生成
}
```

### 输出 Schema

```ts
{
  worktreePath: string
  worktreeBranch?: string
  message: string
}
```

---

### 创建流程

```ts
// 1. 验证不在已有 worktree session 中
if (getCurrentWorktreeSession()) → throw 'Already in a worktree session'

// 2. 找到 git 根目录（从子目录进入 worktree 时也需要工作）
const mainRepoRoot = findCanonicalGitRoot(getCwd())
process.chdir(mainRepoRoot)

// 3. 创建 worktree
const slug = input.name ?? getPlanSlug()
const worktreeSession = await createWorktreeForSession(getSessionId(), slug)

// 4. 切换会话状态
process.chdir(worktreeSession.worktreePath)
setCwd(worktreeSession.worktreePath)
setOriginalCwd(getCwd())
saveWorktreeState(worktreeSession)

// 5. 清除依赖 CWD 的缓存
clearSystemPromptSections()
clearMemoryFileCaches()
getPlansDirectory.cache.clear?.()
```

---

### Worktree slug 验证

```ts
name.z.string().superRefine((s, ctx) => {
  validateWorktreeSlug(s)  // 验证每个 segment 的格式
})
```

只允许字母、数字、点、下划线、短横线——每个 segment 最多 64 字符。这防止路径注入。

---

### Worktree Session 存储

```ts
saveWorktreeState(worktreeSession)
```

Worktree 相关信息存储到 session storage——这样 ExitWorktreeTool、hooks、和其他工具可以查询当前 worktree 状态。`getCurrentWorktreeSession()` 只能返回当前 session 创建的 worktree。

---

### 缓存清除

Enter worktree 后必须清除所有依赖 CWD 的缓存：
- `clearSystemPromptSections()`——系统提示中的 `env_info_simple` 用 worktree 上下文重新计算
- `clearMemoryFileCaches()`——CLAUDE.md 等 memoized 文件缓存
- `getPlansDirectory.cache.clear?.()`——plan 目录缓存

---

### 读完后应抓住的 3 个事实

1. **Worktree 与 session 绑定**——EnterWorktreeTool 创建的 worktree 被 session tracking 系统管理。`getCurrentWorktreeSession()` 只返回当前 session 创建的 worktree。这防止 ExitWorktreeTool 操作由 `git worktree add` 创建的 worktree。

2. **状态恢复是对称的**——EnterWorktreeTool 设置 CWD、originalCwd、清除缓存；ExitWorktreeTool 反向操作。这是精心设计的对称状态管理。

3. **slug 格式安全**——worktree 名的验证防止路径注入。只允许安全字符，segment 长度有限制。

---

## 第 123 站：`src/tools/ExitWorktreeTool/`（4 个文件，~24K）

### 这是什么

ExitWorktreeTool 退出 worktree 会话并恢复原始目录——支持保留（keep）或删除（remove）worktree。这是 EnterWorktreeTool 的对应工具。

---

### 输入 Schema

```ts
{
  action: 'keep' | 'remove'            // 保留还是删除 worktree
  discard_changes?: boolean            // 删除时有未提交变更必须显式确认
}
```

### 输出 Schema

```ts
{
  action: 'keep' | 'remove'
  originalCwd: string
  worktreePath: string
  worktreeBranch?: string
  tmuxSessionName?: string
  discardedFiles?: number
  discardedCommits?: number
  message: string
}
```

---

### 安全守卫：`countWorktreeChanges()`

```ts
async function countWorktreeChanges(worktreePath, originalHeadCommit) {
  // git status --porcelain → changedFiles
  // git rev-list --count originalHeadCommit..HEAD → commits
  // 任一失败返回 null（fail-closed）
}
```

`null` 返回值是关键的安全设计——而不是 0/0。当 git 无法确定状态时（lock file、索引损坏、引用错误），返回 `null` 让调用者采取保守行为。

---

### 验证链

```ts
// 1. 必须在 worktree session 中
if (!getCurrentWorktreeSession()) → error

// 2. remove + 未确认变更
if (action === 'remove' && !discard_changes) {
  const summary = await countWorktreeChanges(...)
  if (summary === null) → error  // 无法验证，拒绝操作
  if (changedFiles > 0 || commits > 0) → error  // 列出变更，要求确认
}
```

如果 worktree 有未提交文件或未在原始分支上的提交，工具拒绝执行并列出具体变更。这防止意外丢失工作。

---

### `restoreSessionToOriginalCwd()`

```ts
function restoreSessionToOriginalCwd(originalCwd, projectRootIsWorktree) {
  setCwd(originalCwd)
  setOriginalCwd(originalCwd)
  if (projectRootIsWorktree) {
    setProjectRoot(originalCwd)
    updateHooksConfigSnapshot()  // 恢复 hooks 配置
  }
  saveWorktreeState(null)
  clearSystemPromptSections()
  clearMemoryFileCaches()
  getPlansDirectory.cache.clear?.()
}
```

这是 EnterWorktreeTool 的逆向操作：
- CWD 恢复为原始目录
- 如果是 `--worktree` 启动（projectRootIsWorktree），projectRoot 也恢复
- hooks 配置快照重新从原始目录读取

---

### Keep vs Remove

#### Keep
```ts
await keepWorktree()  // 保留 git worktree 和分支
restoreSessionToOriginalCwd(...)
// 如果有 tmux session，提示用户可以重新附加
```

Worktree 保留在磁盘上——用户可以稍后用 `tmux attach` 或 git 命令重新访问。

#### Remove
```ts
if (tmuxSessionName) await killTmuxSession(tmuxSessionName)
await cleanupWorktree()  // git worktree remove + branch delete
restoreSessionToOriginalCwd(...)
```

删除 worktree 和关联分支——如果有 tmux session 也先杀死。

---

### 失败关闭设计

```
Returns null when state cannot be reliably determined — callers that use
this as a safety gate must treat null as "unknown, assume unsafe" (fail-closed).
A silent 0/0 would let cleanupWorktree destroy real work.
```

这是关键的安全哲学：当无法确定 worktree 状态时（git 命令失败），拒绝删除而不是假设安全。静默的 0/0 会让 `cleanupWorktree` 销毁真实的工作。

---

### 读完后应抓住的 4 个事实

1. **对称状态管理**——EnterWorktreeTool 和 ExitWorktreeTool 是精确的反向操作。Enter 设置 CWD/originalCwd/清除缓存，Exit 恢复所有这些。`restoreSessionToOriginalCwd()` 是 EnterWorktreeTool.call() 的严格反向。

2. **Fail-closed 安全设计**——`countWorktreeChanges()` 在 git 失败时返回 `null` 而不是 0/0。这防止在索引损坏或引用错误时误删 worktree。调用者将 `null` 视为「未知，假设不安全」。

3. **变更保护**——remove 操作前有双重保护：validateInput 时检查，call() 执行时再检查。TOCTOU 窗口内状态可能改变，所以执行时重新计数。

4. **Hook 配置恢复**——当通过 `--worktree` 启动时，Exit 时调用 `updateHooksConfigSnapshot()` 从原始目录重新加载 hooks 配置。这是 `--worktree` 模式特有的，mid-session worktree 进入不会改变 hooks 快照。
