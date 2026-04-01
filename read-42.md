## 第 42 站：`src/utils/swarm/teamHelpers.ts`

### 这是什么文件

`src/utils/swarm/teamHelpers.ts` 是 swarm 体系里的 **team 元数据文件门面 + 会话清理/回收辅助器**。

前一站 `teammate.ts` 已经看懂：

```text
上层会通过统一 helper 判断：
- 我是不是 teammate / team lead
- 当前 agentId / agentName / teamName 是什么
- in-process teammate 是否还在工作、是否要等待它 idle
```

但仅有“当前我是谁”还不够。

swarm 还必须解决另一类问题：

- team 的持久元数据到底存在哪
- 成员列表怎样增删改
- 成员 mode / active 状态怎样同步
- hidden pane / allowed path 这类团队共享状态怎样维护
- session 异常退出后 team 目录、tasks 目录、worktree、pane 怎样回收

所以这一站回答的是：

```text
team 作为一个可持久化、可恢复、可清理的 swarm 实体，
它的元数据文件长什么样，以及围绕它有哪些基础维护操作
```

所以最准确的一句话是：

```text
teamHelpers.ts = swarm 的 team metadata control plane
```

---

### 先看它的整体定位

位置：`src/utils/swarm/teamHelpers.ts:1`

这个文件做的事情其实分成两大块：

#### 1. team file 读写与成员元数据维护
比如：

- `readTeamFile()`
- `readTeamFileAsync()`
- `writeTeamFileAsync()`
- `removeTeammateFromTeamFile()`
- `removeMemberByAgentId()`
- `setMemberMode()`
- `setMultipleMemberModes()`
- `setMemberActive()`

#### 2. session 结束时的团队回收
比如：

- `registerTeamForSessionCleanup()`
- `cleanupSessionTeams()`
- `killOrphanedTeammatePanes()`
- `cleanupTeamDirectories()`
- `destroyWorktree()`

这说明它不是单纯的 `config.json` 读写工具，
而是：

```text
围绕 team 持久元数据生命周期的一层高层 helper 集合
```

也就是说：

```text
teammate.ts 偏“当前身份门面”
teamHelpers.ts 偏“团队实体门面”
```

---

### 第一部分：`inputSchema` 暗示这个模块还承担了 TeamCreate / cleanup 这类控制面输入定义

位置：`src/utils/swarm/teamHelpers.ts:19`

一开头就定义了：

- `operation: 'spawnTeam' | 'cleanup'`
- `agent_type`
- `team_name`
- `description`

这说明这份文件虽然不直接等于某个 Tool，
但它显然被 team 相关 tool 或 handler 当作公共控制面模型使用。

也就是说这里并不是随便放几个内部类型，
而是在固定：

```text
team 控制操作的公共输入 shape
```

这类设计的意义在于：

- team create / cleanup 语义保持一致
- tool 层和 helper 层共享同一套 schema
- 后续 team 相关操作能围绕同一份基础模型扩展

所以从模块分层看：

```text
它位于“team tool/handler”与“team file/runtime state”之间
```

---

### 第二部分：`TeamFile` 才是整个 team runtime 的持久化核心模型

位置：`src/utils/swarm/teamHelpers.ts:64`

这里定义的 `TeamFile` 非常关键：

- `name`
- `description`
- `createdAt`
- `leadAgentId`
- `leadSessionId`
- `hiddenPaneIds`
- `teamAllowedPaths`
- `members[]`

其中每个 member 又包含：

- `agentId`
- `name`
- `agentType`
- `model`
- `prompt`
- `color`
- `planModeRequired`
- `joinedAt`
- `tmuxPaneId`
- `cwd`
- `worktreePath`
- `sessionId`
- `subscriptions`
- `backendType`
- `isActive`
- `mode`

这说明 team file 记录的不是“静态 roster 名单”这么简单，
而是把很多运行时关键事实都落盘了：

```text
谁是 lead
谁在 team 里
每个人在哪个 backend 上运行
pane / cwd / worktree 在哪
当前 mode 是什么
当前是否 active
```

所以它真正表达的是：

```text
TeamFile = swarm 团队的持久化控制面快照
```

这和前面看到的 `teammateContext` / `dynamicTeamContext` 很不一样：

#### teammate runtime context
```text
面向“当前执行链是谁”
```

#### TeamFile
```text
面向“整个团队现在长什么样”
```

因此这份文件是：

```text
identity facade 背后的 team-wide durable state
```

---

### 第三部分：`hiddenPaneIds` / `teamAllowedPaths` 说明 team file 不只是成员表，还承担团队 UI 与权限共享状态

位置：`src/utils/swarm/teamHelpers.ts:70`

这两个字段很值得注意。

#### `hiddenPaneIds`
它表达的是：

```text
哪些 teammate pane 目前被 UI 隐藏
```

这说明 pane 可见性不是纯前端临时状态，
而是团队级共享元数据的一部分。

#### `teamAllowedPaths`
字段结构是：

- `path`
- `toolName`
- `addedBy`
- `addedAt`

这说明某些“大家都可以直接编辑的路径许可”会被记到 team file 里，
成为一个跨 teammate 共享的权限例外表。

所以 team file 里装的并不只是：

- team membership

还包括：

- pane 展示状态
- 团队共享路径权限策略

这进一步说明：

```text
TeamFile 是 team control plane state，而不是单纯 roster
```

---

### 第四部分：`sanitizeName()` / `sanitizeAgentName()` 揭示了 team 与 agent 在路径/ID 层面的两种不同约束

位置：

- `sanitizeName()` `src/utils/swarm/teamHelpers.ts:100`
- `sanitizeAgentName()` `src/utils/swarm/teamHelpers.ts:108`

这两个 sanitize 看似都在“清洗名字”，但语义不同。

#### `sanitizeName(name)`
- 替换所有非字母数字为 `-`
- 再转小写

用于：

- tmux window name
- worktree path
- file path
- team directory name

这是典型的：

```text
面向文件系统 / pane / worktree 的强约束规范化
```

#### `sanitizeAgentName(name)`
- 只把 `@` 替换成 `-`

注释写得很清楚：

```text
防止 agentName@teamName 这种 deterministic agentId 格式产生歧义
```

这说明 agent 的 sanitize 更偏：

```text
身份拼接格式层面的 disambiguation
```

所以这两个函数虽然名字相近，
但背后处理的是两种完全不同的问题：

- team 名偏路径/资源名规范化
- agent 名偏 identity encoding 规范化

---

### 第五部分：`getTeamDir()` / `getTeamFilePath()` 表明 team 是以独立目录 + `config.json` 的形式持久化

位置：

- `getTeamDir()` `src/utils/swarm/teamHelpers.ts:115`
- `getTeamFilePath()` `src/utils/swarm/teamHelpers.ts:122`

这里很直接：

- `getTeamDir(teamName)` -> `join(getTeamsDir(), sanitizeName(teamName))`
- `getTeamFilePath(teamName)` -> `join(getTeamDir(teamName), 'config.json')`

这说明一个 team 在本地落盘时的基本组织方式就是：

```text
~/.claude/teams/<sanitized-team-name>/config.json
```

所以“团队”在 swarm 里不是抽象概念，
而是明确对应一个本地目录树。

这会承接很多后面的行为：

- mailbox / inbox
- team file
- cleanup
- pane/worktree 关联

因此可以把这里记成：

```text
team directory = 团队控制面本地根目录
config.json = 团队元数据主文件
```

---

### 第六部分：`readTeamFile()` / `readTeamFileAsync()` 说明 team file 是一个在 sync UI 路径和 async handler 路径都要读的共享真源

位置：

- `readTeamFile()` `src/utils/swarm/teamHelpers.ts:131`
- `readTeamFileAsync()` `src/utils/swarm/teamHelpers.ts:147`

这里专门做了同步版和异步版，注释也写得很清楚：

- sync 版给 React render 等同步场景
- async 版给 tool handler 等异步场景

这说明 team file 在系统里地位很高，
高到：

```text
有些路径来不及异步 await，也仍然需要立刻拿到团队快照
```

而且两者的行为都统一成：

- 读成功 -> 返回 `TeamFile`
- 文件不存在 -> `null`
- 其他错误 -> log + `null`

这意味着 team file 的读取语义是：

```text
best-effort state lookup，而不是强异常控制流
```

也就是说很多 team 相关上层逻辑会把：

- team 不存在
- team 文件暂时读失败

都当成“当前没有可用 team state”来处理。

---

### 第七部分：同步/异步双写接口说明 team file 是跨多个执行上下文共享的基础设施

位置：

- `writeTeamFile()` `src/utils/swarm/teamHelpers.ts:166`
- `writeTeamFileAsync()` `src/utils/swarm/teamHelpers.ts:175`

这两者和读接口对应：

- sync path -> `mkdirSync` + `writeFileSync`
- async path -> `mkdir` + `writeFile`

这里最重要的信号不是实现复杂度，
而是：

```text
team metadata 的写入会发生在不同调用环境里，
系统明确接受“同步路径也会直接写 team file”
```

因此 team file 并不是只给某个后台 service 统一写，
而是一个被多个模块直接共享读写的状态文件。

这也解释了为什么很多 helper 都围绕它展开，
因为它就是：

```text
swarm 团队控制面的本地真源之一
```

---

### 第八部分：`removeTeammateFromTeamFile()` / `removeMemberFromTeam()` / `removeMemberByAgentId()` 暴露了三种不同的成员移除语义

位置：

- `removeTeammateFromTeamFile()` `src/utils/swarm/teamHelpers.ts:188`
- `removeMemberFromTeam()` `src/utils/swarm/teamHelpers.ts:285`
- `removeMemberByAgentId()` `src/utils/swarm/teamHelpers.ts:326`

这三者看起来都在“删成员”，但其实对应不同场景。

#### 1. `removeTeammateFromTeamFile(...)`
按：

- `agentId`
- 或 `name`

来删。

注释写的是：

```text
leader 处理 shutdown approval 时使用
```

这更像：

```text
高层 teammate 身份语义删除
```

#### 2. `removeMemberFromTeam(teamName, tmuxPaneId)`
按 pane ID 删除，且顺手从 `hiddenPaneIds` 里也删掉。

这对应的是：

```text
pane-backed teammate 的资源回收语义删除
```

#### 3. `removeMemberByAgentId(teamName, agentId)`
注释直接强调：

```text
给 in-process teammate 用
因为它们共享同一个 tmuxPaneId
```

这说明 in-process teammate 不能用 pane 作为唯一删除键，
必须按稳定身份键 `agentId` 删除。

所以这一组 helper 很清楚地揭示了：

```text
team member removal 不是单一动作，
而要按 backend / identity model 区分删除键
```

这也恰好和前面看到的：

- out-of-process teammate
- in-process teammate

两种后端模型对应起来了。

---

### 第九部分：`addHiddenPaneId()` / `removeHiddenPaneId()` 说明 pane 可见性是 team 级持久状态，而不是纯 UI 临时态

位置：

- `addHiddenPaneId()` `src/utils/swarm/teamHelpers.ts:235`
- `removeHiddenPaneId()` `src/utils/swarm/teamHelpers.ts:259`

这两个函数都很简单：

- 读 team file
- 更新 `hiddenPaneIds`
- 写回 team file

但架构意义很大。

因为如果 pane hide/show 只是当前 UI 层状态，
它根本不需要写入 team file。

现在它被持久化，说明系统要表达的是：

```text
某个 pane 是否显示，是团队会话的一部分共享事实
```

也就是说这类状态在：

- 下次刷新
- 其他 team UI 读取点
- 某些恢复路径

里都可能需要保持一致。

所以它不是 view-only toggle，
而是：

```text
team-level persisted UI coordination state
```

---

### 第十部分：`setMemberMode()` / `syncTeammateMode()` / `setMultipleMemberModes()` 暴露了 mode 同步的单点、当前 teammate、自批量三种更新路径

位置：

- `setMemberMode()` `src/utils/swarm/teamHelpers.ts:357`
- `syncTeammateMode()` `src/utils/swarm/teamHelpers.ts:397`
- `setMultipleMemberModes()` `src/utils/swarm/teamHelpers.ts:415`

这三者合起来非常有意思。

#### `setMemberMode(teamName, memberName, mode)`
这是 leader/控制面最直接的单成员更新口。

特点：

- 按 `member.name` 定位
- 只有实际变化才写文件
- 用 immutable members array 回写

说明它主要给：

```text
显式修改某个 teammate 当前 permission mode
```

#### `syncTeammateMode(mode, teamNameOverride?)`
逻辑是：

- 只有当前运行身份是 teammate 才执行
- 从当前 runtime 身份拿 `teamName` / `agentName`
- 然后调 `setMemberMode(...)`

这说明 mode 变化不一定总由 leader 主动回写，
teammate 自己也会把当前 mode 同步进 config.json，
让 leader/UI 能看到。

所以这里对应的是：

```text
runtime local truth -> team file visibility
```

#### `setMultipleMemberModes(...)`
注释明确说：

```text
单次原子批量更新，避免一次改多个 teammate 时竞争
```

这说明作者明确意识到：

```text
多次单成员写 config.json 会有 race / 覆盖问题
```

因此才做了批量口。

所以这组 API 最关键的理解是：

```text
team member mode 既是每个 teammate 的本地运行状态，
也是 team control plane 里要共享展示的持久状态
```

---

### 第十一部分：`setMemberActive()` 说明 active / idle 也会下沉到 team file，成为团队级可观察状态

位置：`src/utils/swarm/teamHelpers.ts:454`

这个函数做的就是：

- 找到 member
- 更新 `isActive`
- 异步写回 team file

注释明确说：

- idle -> `isActive=false`
- 新 turn 开始 -> `isActive=true`

这说明 “teammate 当前是不是正在工作” 不只是 task runtime 内部状态，
而是会被提升到团队控制面里。

也就是说：

```text
active/idle 是一种需要跨模块、跨界面共享的 team-visible state
```

这和前一站 `hasWorkingInProcessTeammates()` 的语义正好互补：

- `teammate.ts` 偏当前进程内的工作状态判断
- `teamHelpers.ts` 偏把这种状态同步进团队元数据文件

所以两者一起构成：

```text
local runtime activity state
  -> synced into team-wide persisted visibility state
```

---

### 第十二部分：`destroyWorktree()` 暴露了 team cleanup 不只是删目录，而是优先走 Git-level 反注册

位置：`src/utils/swarm/teamHelpers.ts:492`

这个函数很关键，因为它不是直接 `rm -rf`。

流程是：

1. 先读 `worktreePath/.git`
2. 从里面解析出真正的主仓库 `.git/worktrees/...` 路径
3. 反推出主 repo 根路径
4. 优先执行：
   - `git worktree remove --force <worktreePath>`
5. 如果失败，再 fallback：
   - `rm(worktreePath, { recursive: true, force: true })`

这说明作者明确知道：

```text
git worktree 不是普通目录，
只删文件夹可能留下主仓库里的 worktree 注册残骸
```

所以正确语义是：

```text
先做 Git 控制面层面的 worktree remove，
再做文件系统层面的兜底清理
```

这体现出这个模块并不只是“简单 housekeeping”，
而是在维护：

```text
team 相关外部资源的一致性
```

---

### 第十三部分：`registerTeamForSessionCleanup()` / `unregisterTeamForSessionCleanup()` 说明 session 级 team 生命周期还有一层“本会话创建登记簿”

位置：

- `registerTeamForSessionCleanup()` `src/utils/swarm/teamHelpers.ts:560`
- `unregisterTeamForSessionCleanup()` `src/utils/swarm/teamHelpers.ts:568`

这两个函数操作的是：

- `getSessionCreatedTeams()` 返回的 Set

注释写得很清楚：

- team 刚创建时登记进去
- 显式 TeamDelete 后要取消登记，避免 shutdown 时重复清理
- backing Set 放在 `bootstrap/state.ts`，测试 reset 时也会清掉

这说明系统并不把“所有存在的 team”都在 shutdown 时无脑清理，
而是只清理：

```text
本 session 创建、且尚未显式删除的 team
```

所以这里实际上维护的是：

```text
session-scoped cleanup ownership
```

这很重要，因为它避免了：

- 误删旧 team
- 重复清理已删 team
- 测试状态泄漏

---

### 第十四部分：`cleanupSessionTeams()` 说明异常退出时，系统会把“本 session 创造的团队资源”当作 orphan 统一回收

位置：`src/utils/swarm/teamHelpers.ts:576`

流程是：

1. 取 `getSessionCreatedTeams()`
2. 如果为空就返回
3. 先 `killOrphanedTeammatePanes(...)`
4. 再 `cleanupTeamDirectories(...)`
5. 最后清空 Set

注释强调的关键点是：

```text
在 SIGINT 这类场景里，teammate 进程可能还活着；
如果只删目录，不杀 pane，会在 tmux/iTerm2 里留下孤儿执行体
```

这说明 cleanup 的第一原则不是“删文件”，而是：

```text
先处理仍在运行的 teammate 宿主资源
```

所以这个函数真正表达的是：

```text
session 退出时，对 team 做的是 orphaned runtime + persisted state 的联合回收
```

而不是单纯的 filesystem cleanup。

---

### 第十五部分：`killOrphanedTeammatePanes()` 说明 pane-backed teammate 的紧急回收依赖 backend registry，而不是写死 tmux 逻辑

位置：`src/utils/swarm/teamHelpers.ts:598`

这个函数有几个关键点。

#### 1. 只杀 pane backend 的成员
过滤条件是：

- 不是 `TEAM_LEAD_NAME`
- 有 `tmuxPaneId`
- 有 `backendType`
- `isPaneBackend(backendType)`

这说明 team member backend 是多态的，
而 cleanup 只对 pane-based 后端走 kill pane。

#### 2. 动态 import backend registry / detection
注释写明：

```text
为了不把 registry/detection 静态挂进依赖图；
这逻辑只在 shutdown 跑，导入开销无所谓
```

这说明作者很在意：

- 常规路径的模块依赖轻量化
- shutdown path 可以接受懒加载

#### 3. 通过 backend 抽象 kill，而不是硬编码 tmux 命令
调用的是：

- `getBackendByType(m.backendType).killPane(...)`

这说明 swarm backend 已经抽象成统一 registry 模型，
team cleanup 只依赖后端接口，不关心具体 pane 技术细节。

所以这个函数揭示的是：

```text
team cleanup 是 backend-aware 的，
但仍通过统一 backend abstraction 实现
```

---

### 第十六部分：`cleanupTeamDirectories()` 把 worktree、team dir、tasks dir 三类资源统一纳入 team teardown

位置：`src/utils/swarm/teamHelpers.ts:641`

这个函数的流程非常完整：

1. 先 `sanitizeName(teamName)`
2. 先读 team file，提取所有 `member.worktreePath`
3. 逐个 `destroyWorktree(...)`
4. 删除 team directory
5. 删除 tasks directory
6. `notifyTasksUpdated()`

这里最重要的是它明确处理了三种资源层：

#### 1. worktree
对应每个 teammate 的 git 隔离工作副本

#### 2. team directory
对应 `~/.claude/teams/<team>/`
里的控制面元数据

#### 3. tasks directory
对应共享 task list / task 状态持久化

这说明“删除一个 team”并不是删一份配置文件，
而是一次：

```text
execution artifacts + control-plane metadata + task-state persistence
```

的联合 teardown。

而且它先收 worktree，再删 team dir，说明作者很清楚依赖顺序：

```text
worktree path 信息存在 team file 里，
必须先读出来，否则删完 team dir 就丢线索了
```

这也是一个很关键的正确性细节。

---

### 第十七部分：这份文件把“team metadata 管理”和“team teardown 管理”放在一起，说明它的真实定位是 team 生命周期门面

如果只看前半段，容易把它记成：

- team file schema
- read/write helper
- member mode 更新

但后半段又出现：

- worktree destroy
- session cleanup registry
- orphan pane kill
- tasks dir cleanup

这说明它真实做的不是“配置编辑”，而是：

```text
维护 team 作为一个持久化 swarm 实体，从存在到销毁的整个外围生命周期
```

所以更准确地说：

```text
teamHelpers.ts = 团队控制面状态 + 资源回收边界模块
```

它正好和前几站形成分层：

```text
teammateContext.ts
  -> in-process teammate 的 ALS 身份隔离
teammate.ts
  -> teammate 身份/状态统一门面
teamHelpers.ts
  -> team 元数据持久化与 cleanup 门面
```

---

### 读完这一站后，你应该抓住的 10 个事实

1. `teamHelpers.ts` 不是单纯的 team file 读写工具，而是 swarm 团队控制面的高层 helper 集合。
2. `TeamFile` 是团队的持久化控制面快照，记录的不只是成员名单，还包括 lead、backend、pane、worktree、mode、active 状态、hidden panes、allowed paths 等运行时关键信息。
3. `sanitizeName()` 与 `sanitizeAgentName()` 分别解决路径资源名规范化和 `agentName@teamName` 身份编码歧义两个不同问题。
4. team 的本地持久化组织方式是 `~/.claude/teams/<sanitized-team-name>/config.json`，其中 `config.json` 是团队元数据主文件。
5. `readTeamFile()` / `readTeamFileAsync()` 与对应写接口并存，说明 team file 会在同步 UI 路径和异步 handler 路径中都被直接当作共享真源访问。
6. 成员移除语义被拆成按身份、按 pane、按 agentId 三种 helper，分别照顾 shutdown、pane-backed teammate、in-process teammate 等不同后端模型。
7. `setMemberMode()` / `syncTeammateMode()` / `setMultipleMemberModes()` 说明 permission mode 既是本地运行状态，也是要同步进 team control plane 的共享可见状态。
8. `setMemberActive()` 把 active/idle 下沉到 team file，说明 teammate 工作状态也被建模成团队级可观察元数据。
9. `destroyWorktree()` 优先走 `git worktree remove`，说明 team cleanup 关注的不只是删目录，还要维护 Git 控制面的正确反注册。
10. `cleanupSessionTeams()` / `killOrphanedTeammatePanes()` / `cleanupTeamDirectories()` 一起实现了 team 的会话级孤儿资源回收：pane、worktree、team dir、tasks dir 都在 teardown 范围内。

---

### 现在把第 41-42 站串起来

```text
teammate.ts
  -> 把 ALS、dynamic context、teamContext/env 统一收口成 teammate 身份与状态 helper
teamHelpers.ts
  -> 把 team file、成员 mode/active、hidden panes、allowed paths、session cleanup/worktree teardown 统一收口成团队控制面 helper
```

所以现在 team / teammate 这条线可以先压缩成：

```text
runtime teammate identity
  -> teammateContext.ts / teammate.ts

persisted team control plane
  -> teamHelpers.ts / config.json
```

也就是说：

```text
“当前我是谁”靠 runtime context
“整个团队现在长什么样”靠 team file
```

---

### 下一站建议

下一站最顺的是：

```text
src/utils/swarm/backends/registry.ts
```

因为现在你已经看懂了：

- team file 里会记录 `backendType`
- cleanup 时会通过 `getBackendByType(...).killPane(...)` 做 backend-aware 回收
- pane-backed teammate 与 in-process teammate 的删除语义已经分开

下一步最自然就是继续往 backend 抽象层下钻：

**swarm 到底怎样注册不同 teammate backend、怎样按 type 分发 spawn/kill 能力、以及 pane backend / in-process backend / 其他后端是怎样被统一成同一套 team runtime 接口的。**
