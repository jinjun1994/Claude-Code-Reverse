# 第 2 章：CLI 路由与命令装配

## 2.1 Commander 命令树构建

`main.tsx` 的 `run()` 函数构建了 Claude Code 的整个 Commander 命令树。这个大约 200 行的函数承担了双重职责：定义路由表和注册全局选项。

---

### Commander 实例初始化

```typescript
// main.tsx:894-903
function createSortedHelpConfig() {
  const getOptionSortKey = (opt: Option) =>
    opt.long?.replace(/^--/, '') ?? opt.short?.replace(/^-/, '') ?? '';
  return Object.assign({
    sortSubcommands: true,
    sortOptions: true,
  }, {
    compareOptions: (a, b) => getOptionSortKey(a).localeCompare(getOptionSortKey(b))
  });
}
const program = new CommanderCommand()
  .configureHelp(createSortedHelpConfig())
  .enablePositionalOptions();
```

**排序帮助文本的设计**——`compareOptions` 是 `@commander-js/extra-typings` 未暴露的类型属性。通过 `Object.assign` 注入，使得帮助输出按选项名排序而非注册顺序。这是面向用户体验的决策：60+ 个选项，无序输出将难以检索。

---

### preAction Hook：懒初始化模式

Commander 的 `preAction` hook 是 Claude Code 初始化序列的关键控制点。

```typescript
// main.tsx:907-967
program.hook('preAction', async thisCommand => {
  // 1. 等待并行预取完成 (MDM + Keychain)
  await Promise.all([
    ensureMdmSettingsLoaded(),
    ensureKeychainPrefetchCompleted()
  ]);
  // 2. 运行 init()
  await init();
  // 3. 设置进程标题
  if (!isEnvTruthy(process.env.CLAUDE_CODE_DISABLE_TERMINAL_TITLE)) {
    process.title = 'claude';
  }
  // 4. 注册日志 sink
  const { initSinks } = await import('./utils/sinks.js');
  initSinks();
  // 5. 行内 --plugin-dir
  const pluginDir = thisCommand.getOptionValue('pluginDir');
  if (Array.isArray(pluginDir) && pluginDir.length > 0) {
    setInlinePlugins(pluginDir);
    clearPluginCache('preAction: --plugin-dir inline plugins');
  }
  // 6. 运行数据库迁移
  runMigrations();
  // 7. 远程设置和策略 (非阻塞)
  void loadRemoteManagedSettings();
  void loadPolicyLimits();
  // 8. 设置同步 (非阻塞)
  if (feature('UPLOAD_USER_SETTINGS')) {
    void import('./services/settingsSync/index.js')
      .then(m => m.uploadUserSettingsInBackground());
  }
});
```

**为何用 hook 而非 action**——如果初始化放在 `.action()` handler 内部，`--help` 和 `--version` 等不需要初始化的操作也会触发 init。Commander 的 `preAction` 只在执行具体命令时触发，不在显示帮助时触发。这是干净的分离。

---

### `getCommands()`：延迟命令注册

`commands.ts` 维护了所有内置命令的列表。关键的设计决策是：`COMMANDS` 是一个 memoized function，而非模块级常量。

```typescript
// commands.ts:258-346
const COMMANDS = memoize((): Command[] => [
  addDir, advisor, agents, branch, /* ... */
  ...(!isUsing3PServices() ? [logout, login()] : []),
  passes,
  ...(peersCmd ? [peersCmd] : []),
  tasks,
  // ...
])
```

**为何 memoize 而非常量**——因为某些命令的条件注册依赖于 `isUsing3PServices()`，而这个函数需要从磁盘读取配置文件。如果在模块加载时执行，就会在不需要登录命令的场景（如 print 模式）下浪费 I/O。`memoize` 保证了第一次调用时读取，后续调用返回缓存结果。

---

### Feature Gate 与命令裁剪

许多命令根据 feature flag 有条件地包含或不包含：

```typescript
// commands.ts:225-254 (内置命令)
export const INTERNAL_ONLY_COMMANDS = [
  backfillSessions, breakCache, bughunter, commit, // ...
].filter(Boolean)  // 过滤 null 条目

// commands.ts:70-76 (条件导入)
const bridge = feature('BRIDGE_MODE')
  ? require('./commands/bridge/index.js').default
  : null
```

这构建了一个三层裁剪体系：

| 层级 | 机制 | 效果 |
|------|------|------|
| 编译期 | `feature()` | 整个命令从输出中消除 |
| 加载期 | memoized `COMMANDS()` | 延迟到第一次调用 |
| 运行期 | `isUsing3PServices()` | 基于当前环境条件过滤 |

---

## 2.2 多宿主启动模式

### Client Type 检测与分发

`main()` 在 `run()` 之前确定 `clientType`：

```typescript
// main.tsx:818-848
const clientType = (() => {
  if (isEnvTruthy(process.env.GITHUB_ACTIONS)) return 'github-action';
  if (process.env.CLAUDE_CODE_ENTRYPOINT === 'sdk-ts') return 'sdk-typescript';
  if (process.env.CLAUDE_CODE_ENTRYPOINT === 'sdk-py') return 'sdk-python';
  if (process.env.CLAUDE_CODE_ENTRYPOINT === 'sdk-cli') return 'sdk-cli';
  if (process.env.CLAUDE_CODE_ENTRYPOINT === 'claude-vscode') return 'claude-vscode';
  if (process.env.CLAUDE_CODE_ENTRYPOINT === 'local-agent') return 'local-agent';
  if (process.env.CLAUDE_CODE_ENTRYPOINT === 'claude-desktop') return 'claude-desktop';
  const hasSessionIngressToken =
    process.env.CLAUDE_CODE_SESSION_ACCESS_TOKEN ||
    process.env.CLAUDE_CODE_WEBSOCKET_AUTH_FILE_DESCRIPTOR;
  if (process.env.CLAUDE_CODE_ENTRYPOINT === 'remote' || hasSessionIngressToken) {
    return 'remote';
  }
  return 'cli';
})();
setClientType(clientType);
```

**为何在 `run()` 之前**——`clientType` 决定了 `questionPreviewFormat` 的默认值、`isNonInteractiveSession` 的初始状态，以及 `CLAUDE_CODE_ENTRYPOINT` 的遥测属性。`run()` 内部的 Commander action handler 依赖这些值。

### 交互 vs 非交互模式分流

```typescript
// main.tsx:800-808
const hasPrintFlag = cliArgs.includes('-p') || cliArgs.includes('--print');
const hasInitOnlyFlag = cliArgs.includes('--init-only');
const hasSdkUrl = cliArgs.some(arg => arg.startsWith('--sdk-url'));
const isNonInteractive = hasPrintFlag || hasInitOnlyFlag || hasSdkUrl || !process.stdout.isTTY;
```

`isNonInteractive` 是一个关键分歧点：
- 非交互模式：跳过 Early Input 捕获、不调用 TUI 渲染
- 交互模式（TTY）：需要 `showSetupScreens()` 显示引导、信任对话框等

---

## 2.3 命令分发与子命令系统

### Slash 命令与 Prompt 命令

Claude Code 的命令系统有两层：

1. **Commander 子命令**：`claude mcp add`, `claude session list` 等。在启动时通过 `.command()` 注册。

2. **Slash/Prompt 命令**：在 REPL 中以 `/` 开头调用，如 `/help`, `/clear`, `/resume`。这些命令通过 `COMMANDS` 数组注册，类型为 `Command`。

```typescript
// types/command.ts 简化视图
type Command = CommandBase & (PromptCommand | { type: 'local-jsx' });

interface PromptCommand {
  type: 'prompt';
  getPromptForCommand(args: string[], ctx): Promise<string>;
}
```

**为什么设计 Prompt Command 系统**——传统的 CLI 子命令需要独立的 action handler。Prompt Command 允许命令返回一个 prompt 字符串，由主循环发送给 LLM。这意味着新命令不需要新增 UI 组件——只需要返回合适的 prompt。`/cost`（成本查询）、`/summary`（对话总结）等命令都属于这种类型。

### 插件命令的动态发现

除内置命令外，Claude Code 还支持四种来源的动态命令：

```typescript
// commands.ts:353-378
async function getSkills(cwd) {
  const [skillDirCommands, pluginSkills] = await Promise.all([
    getSkillDirCommands(cwd),   // 从 .claude/skills/ 目录
    getPluginSkills(),          // 从插件目录
  ]);
  return {
    skillDirCommands,           // 用户自定义技能
    pluginSkills,               // 插件技能
    bundledSkills,              // 内置捆绑技能
    builtinPluginSkills,        // 内置插件技能
  };
}
```

动态命令的加载是并行的 (`Promise.all`)，并在失败时继续（`catch` 返回空数组）。一个损坏的技能目录不会阻止整个系统的启动。

---

## 2.4 选项注册与帮助文本

### 60+ 全局选项的组织

`run()` 注册的全局选项按功能分组：

| 分组 | 选项示例 |
|------|---------|
| 会话控制 | `--continue`, `--resume`, `--fork-session` |
| 模型选择 | `--model`, `--effort`, `--betas`, `--fallback-model` |
| 权限 | `--dangerously-skip-permissions`, `--permission-mode` |
| MCP | `--mcp-config`, `--strict-mcp-config` |
| 输出格式 | `--output-format`, `--input-format`, `--print`, `--bare` |
| 系统提示 | `--system-prompt`, `--system-prompt-file`, `--append-system-prompt` |
| 开发者调试 | `--debug`, `--debug-file`, `--verbose` |

### 排序帮助文本

```typescript
// main.tsx:894-903
function createSortedHelpConfig() {
  const getOptionSortKey = (opt: Option) =>
    opt.long?.replace(/^--/, '') ?? opt.short?.replace(/^-/, '') ?? ''
  return {
    sortSubcommands: true,
    sortOptions: true,
    compareOptions: (a, b) => getOptionSortKey(a).localeCompare(getOptionSortKey(b))
  }
}
```

`compareOptions` 是 Commander 未暴露的类型属性，通过 `Object.assign` 注入。60+ 个选项，无序输出将难以检索——排序是面向用户的决策。

---

## 2.5 条件命令注册

`COMMANDS` 是一个 memoized function，而非模块级常量：

```typescript
const COMMANDS = memoize((): Command[] => [
  addDir, advisor, agents, branch, ...
  ...(!isUsing3PServices() ? [logout, login()] : []),  // 条件注册
  ...
])
```

**为什么 memoize 而非常量**——某些命令的条件注册依赖于 `isUsing3PServices()`，而这个函数需要从磁盘读取配置文件。如果在模块加载时执行，就会在不需要登录命令的场景（如 print 模式）下浪费 I/O。`memoize` 保证了第一次调用时读取，后续调用返回缓存结果。

### Feature Gate 与命令裁剪

```typescript
// 编译期裁剪
const bridge = feature('BRIDGE_MODE')
  ? require('./commands/bridge/index.js').default
  : null
```

```typescript
// 运行期过滤
const INTERNAL_ONLY_COMMANDS = [...].filter(Boolean)  // 过滤 null 条目
```

三层裁剪：
1. **编译期**：`feature()` 裁剪，整命令从输出中消除
2. **加载期**：memoized `COMMANDS()`，延迟到第一次调用
3. **运行期**：`isUsing3PServices()`, 基于当前环境条件过滤

---

## 2.6 动态命令发现与 Skill 系统

`getSkills()` 通过 `Promise.all` 并行加载来自四个来源的命令：

```typescript
// commands.ts:353-378
async function getSkills(cwd) {
  const [skillDirCommands, pluginSkills] = await Promise.all([
    getSkillDirCommands(cwd),   // .claude/skills/ 用户自定义
    getPluginSkills(),          // 插件安装的技能
  ])
  return {
    skillDirCommands,           // 用户自定义技能
    pluginSkills,               // 插件技能
    bundledSkills,              // 内置捆绑技能
    builtinPluginSkills,        // 内置插件技能
  }
}
```

**为什么并行**：四个来源独立——文件系统读取、插件目录扫描等，没有依赖关系。`Promise.all` 使得总加载时间等于最慢来源的耗时。

**容错设计**——每个来源都有 `catch` 返回空数组。一个损坏的技能目录不会阻止整个系统的启动。

---

## 2.7 打印模式与特殊模式分流

`cli.tsx` 的快速路径中，有几个关键的打印/特殊模式：

```typescript
// --dump-system-prompt
if (args[0] === '--dump-system-prompt') {
  const config = await import('../utils/config.js')
  const prompts = await import('../utils/prompts.js')
  console.log(await prompts.buildSystemPrompt(...))
  return
}
```

**`--dump-system-prompt` 的延迟 import**——配置和提示模块在模块加载时不导入，只在需要时才加载。这是因为 `config.js` 可能需要从磁盘读取设置文件，而 `prompts.js` 可能需要加载所有工具定义——这些都是重操作。对于只需打印系统提示的场景，不需要初始化任何运行时状态。

---

## 2.8 选项预验证管道：argParser 与内联校验

Commander 的 `argParser` 是 Claude Code 的第一层选项验证——在 action handler 之前执行的类型转换与校验：

```typescript
// main.tsx:976-1000
.option('--max-budget-usd <amount>', '...', v => {
  const n = parseFloat(v);
  if (isNaN(n) || n <= 0) throw new Error('Invalid budget amount');
  return n;
})
.option('--task-budget <tokens>', '...', v => {
  const n = parseInt(v, 10);
  if (isNaN(n) || n <= 0) throw new Error('Invalid token budget');
  return n;
})
.addOption(new Option('--effort <level>', '...')
  .choices(['low', 'medium', 'high', 'max']))
```

**argParser 的三类验证**：

| 验证类型 | 选项 | 行为 |
|---------|------|------|
| 数值转换 | `--max-budget-usd` | `parseFloat` + 正数检查 |
| 选择限制 | `--effort` | `choices(['low', 'medium', 'high', 'max'])` |
| 标准化 | `--resume` | `value => value \|\| true`（无值时默认为 true） |

如果 `argParser` 抛出异常，Commander 会捕获并打印错误信息，action handler 不会执行。

### Action Handler 内联验证

在 action handler 的开始部分（main.tsx:1277-1340），有一系列内联校验：

```typescript
// 1. --session-id 与 --continue/--resume 冲突检查
if (sessionId && (continueFlag || resumeValue)) {
  if (!forkSession) {
    exitWithError('Cannot use --session-id with --continue/--resume without --fork-session');
  }
}

// 2. UUID 格式校验
if (sessionId) {
  validateUuid(sessionId);  // throw if invalid UUID format
}

// 3. sessionId 不重复检查
if (sessionId && sessionIdExists(sessionId)) {
  exitWithError(`Session ${sessionId} already exists`);
}

// 4. --file token 检查
if (fileFlag && !process.env.CLAUDE_CODE_SESSION_ACCESS_TOKEN) {
  exitWithError('--file requires CLAUDE_CODE_SESSION_ACCESS_TOKEN');
}

// 5. Fallback model 与主模型不能相同
if (fallbackModel && fallbackModel === model) {
  exitWithError('Fallback model cannot be the same as the main model');
}

// 6. --system-prompt 与 --system-prompt-file 互斥
if (systemPrompt && systemPromptFile) {
  exitWithError('Cannot use both --system-prompt and --system-prompt-file');
}
```

**为何不只依赖 argParser**——argParser 只能验证单个选项的值。action handler 中的校验需要访问多个选项的组合——这是跨选项一致性校验（如 `--session-id` 不能与 `--continue` 混用）。

---

## 2.9 --continue vs --resume：会话恢复的双重路径

`--continue` 和 `--resume` 都用于恢复之前的会话，但行为模型完全不同：

| 方面 | `--continue` | `--resume [value]` |
|------|-------------|-------------------|
| 目标会话 | 当前项目最新的会话 | 灵活多路分发 |
| 用户指定 | 无需 | 可选（UUID/文件路径/PR） |
| 回退行为 | 无会话时启动新的 | 无参数时启动交互式选择器 |
| 多路路径 | 单一 | 6 种恢复路径 |

### --continue 的恢复链

```typescript
// main.tsx:3101-3155
1. clearSessionCaches()            // 清除过期缓存
2. loadConversationForResume(undefined, undefined)
   → loadMessageLogs() 获取所有 LogOptions，按时间排序
   → 跳过 live --bg/daemon sessions (通过 UDS 查询)
   → 返回最近的非 live 日志
3. deserialize messages            // 反序列化 JSONL 到消息数组
   → filterUnresolvedToolUses()  // 移除孤立的结果
   → filterOrphanedThinkingOnlyMessages()  // 移除纯思考消息
   → filterWhitespaceOnlyAssistantMessages()  // 移除空助手消息
4. detectTurnInterruption()        // 检测会话是否在中途被中断
   → 如果中断，注入 "Continue from where you left off."
5. processResumedConversation()    // 恢复会话元数据、成本状态、工作树
6. launchRepl(initialMessages)     // 用加载的消息启动 REPL
```

**消息清洗的必要性**——转录文件（.jsonl）中可能包含 API 无效的消息类型（如孤立的 `tool_result` 没有对应的 `tool_use`）。如果不清洗，API 会拒绝。

### --resume 的六路分发

```typescript
// main.tsx:3355-3758
resume(value):
  1. 远程/teleport 路径
     → 检查 org policy allow_remote_sessions
     → teleportToRemoteWithErrorHandling()
  2. Teleport by ID
     → fetchSession(teleport) + validateSessionRepository()
     → teleportWithProgress() 加载消息
  3. ccshare URL 路径
     → parseCcshareId() + loadCcshare()
     → 加载共享的 JSONL 转录文件
  4. 文件路径
     → 加载 .jsonl 文件
  5. 会话 UUID
     → loadConversationForResume(sessionId)
  6. 交互式选择器（无参数时的回退）
     → launchResumeChooser() 模糊搜索历史会话
```

**`--fork-session` 的意义**——当结合 `--continue`/`--resume` 时，`--fork-session` 保留新的启动会话 ID，而不是切换到原始会话的 ID。这意味着 fork 是一个独立的新会话，继承了原始会话的内容，但有自己的 sessionId。

### 会话恢复的共享管道

无论通过哪条路径恢复，最终都经过 `processResumedConversation()`（sessionRestore.ts:409+）：

```
1. 匹配 coordinator/normal 模式
2. switchSession(sid)              // 重用或替换会话 ID
3. renameRecordingForSession()     // 重命名 asciicast 录制文件
4. resetSessionFilePointer()       // 重置会话文件指针
5. restoreCostStateForSession()    // 恢复成本追踪
6. restoreSessionMetadata()        // 恢复代理名、颜色、工作树状态
7. restoreWorktreeForResume()      // cd 回正确的工作树
8. adoptResumedSessionFile()       // 指向 sessionFile 到恢复的转录文件
9. contextCollapse restore         // 恢复折叠的上下文
10. restoreAgentFromSession()      // 恢复代理设置
11. restoreSessionStateFromLog()   // 恢复文件历史、跟踪、待办事项
```

11 个步骤涵盖了会话的完整状态恢复——不仅是消息，还包括成本、元数据、工作树、上下文状态等。

---

## 2.10 MCP 配置验证管道

`--mcp-config` 的验证是 CLI 中最复杂的选项处理链之一：

```
输入: --mcp-config file1.json '{"key":"value"}' file2.json
  ↓
  对每个配置:
  1. 尝试 JSON 字符串解析 (safeParseJSON)
     → 失败则尝试文件路径解析 (parseMcpConfigFromFilePath)
     → 都失败则累积错误
  2. 作用域注入 (scope: 'dynamic')
  3. 保留名称检查 (禁止使用内置服务器名)
  ↓
  如果有错误: exit(1) + 格式化的错误消息
  ↓
  filterMcpServersByPolicy(scopedConfigs)  // 企业策略过滤
```

**`--strict-mcp-config` 的语义**——当此 flag 设置时，清除所有其他 MCP 配置（用户配置、项目配置、插件配置），只使用 `--mcp-config` 传入的配置。这是零信任的 MCP 加载——外部提供的配置是唯一可信源。

**企业策略过滤**——`allowedMcpServers` 和 `deniedMcpServers` 组织策略在此应用。即使配置了某个服务器，如果策略不允许，也会被静默过滤。

---

## 2.11 权限模式映射：从 CLI 到运行时

```
--permission-mode <mode>  →  choices: ['default', 'acceptEdits', 'bypassPermissions', 'dontAsk', 'plan', 'auto']
  ↓
initialPermissionModeFromCLI():
  1. --dangerously-skip-permissions (最高优先级) → 'bypassPermissions'
  2. --permission-mode <mode> (CLI 显式) → permissionModeFromString()
  3. settings.permissions.defaultMode (设置文件)
  4. 回退到 'default'
  ↓
GrowthBook 闸门检查:
  - tengu_disable_bypass_permissions_mode 可阻止 bypass 模式
  - auto 模式的断路器检查 (getAutoModeEnabledStateIfCached())
```

**权限模式的废弃别名**：

| 废弃选项 | 替换 | 可见性 |
|---------|------|--------|
| `--delegate-permissions` | `--permission-mode auto` | `hideHelp()` |
| `--dangerously-skip-permissions-with-classifiers` | `--permission-mode auto` | `hideHelp()` |
| `--afk` | `--permission-mode auto` | `hideHelp()` |

所有这些使用 Commander 的 `.implies()` 方法自动映射到 `permissionMode: 'auto'`，并用 `.hideHelp()` 隐藏在公共帮助输出中。

---

## 2.12 Option 废弃与别名机制

Claude Code 使用多种方式实现选项别名和废弃：

| 机制 | 示例 | 效果 |
|------|------|------|
| Commander `.implies()` | `--afk` → `{ permissionMode: 'auto' }` | 自动转换值 |
| 短 flag | `-c` / `-r` / `-p` / `-d` | 标准短格式别名 |
| 双注册 | `--allowedTools, --allowed-tools` | camelCase + kebab-case |
| `.hideHelp()` | 所有内部选项 | 不在 `--help` 中显示 |
| `.hideHelp()` + 功能 gate | `INTERNAL_ONLY_COMMANDS` | 仅内部构建可用 |

**已废弃选项的处理策略**——不删除，只隐藏。这些选项继续功能正常，但不会出现在 `--help` 输出中。这是向后兼容的决策——依赖旧选项的脚本不会中断。

---

## 2.13 SSH 模式的标志转发

SSH 模式（`claude ssh <host>`）需要特殊的标志处理。由于 SSH 连接是远程执行，本地 CLI 需要提取某些标志并传递给远程 CLI：

```typescript
// main.tsx:725-733, 3193-3258
// 在 Commander 解析之前从 raw argv 中提取
const { flag, value, rest } = extractFlag(argv, '--permission-mode');
if (flag) {
  _pendingSSH.permissionMode = value;
  // 从本地 argv 中移除，推入 _pendingSSH.extraCliArgs
}
```

**支持的转发标志**：

| 本地 flag | 远程形式 |
|-----------|---------|
| `-c` | `--continue`（标准化） |
| `--continue` | `--continue` |
| `--resume <uuid>` | `--resume <uuid>` |
| `--model <model>` | `--model <model>` |
| `--permission-mode <mode>` | `--permission-mode <mode>` |

**为何需要标准化 `-c` → `--continue`**——远程 CLI 可能没有相同的短 flag 注册（或者注册顺序不同）。发送 `--continue` 而非 `-c` 保证远程行为一致。

提取同时支持空格分隔（`--flag value`）和等号形式（`--flag=value`）。提取后，标志被从本地 argv 中移除——这样 Commander 不会在本地端处理它们。

---

## 2.14 Commander.js 子命令树

Claude CLI 使用 Commander.js 作为路由框架，构建 **30+ 个子命令**层次结构。

**根命令配置**：
```typescript
const program = new Command()
  .name('claude')
  .description('Claude Code: Agent-assisted coding CLI')
  .version(getVersionString(), ...flags)
  .configureHelp({ ...customFormatting })
```

**子命令注册模式**——每个子命令通过链式 API 注册：
```typescript
program
  .command('codex [prompt...]')
  .description('...')
  .option('--continue', '...')
  .option('--model <model>', '...')
  .addCommand(subCommand)  // 级联子命令
```

---

## 2.15 选项解析与类型系统

**选项类型**：
- **布尔标志**：`--continue`, `--dangerously-skip-permission`
- **字符串参数**：`--model <model>`, `--output-format <format>`
- **多值选项**：`--allowedTool <tools...>`——`collectAllOptions()` 累加到数组
- **可否定选项**：`--[no-]install-mcp-servers`——自动处理 `--install-mcp-servers` 和 `--no-install-mcp-servers`
- **弃用选项**：`--mcp-configs`（`.implies()` 已被弃用）

**选项值处理**——`getPermissionModeFromOptions()` 将 CLI 选项映射为权限模式：
| 选项 | 权限模式 |
|------|---------|
| `--dangerously-skip-permission` | `yolo` |
| `--dangerously-skip-permissions` | `yolo`（别名） |
| 无额外标志 | `default`（由 settings 管线决定） |

**多值选项累加**——`collectAllOptions` 用于处理 `--allowedTool Read --allowedTool Bash` 等多值合并。

---

## 2.16 输出格式分支

`--output-format` 分支到三种格式化管线：

| 格式 | 描述 | 用例 |
|------|------|------|
| `text` | 默认人类可读输出 | 交互式终端 |
| `jsonl` | 每行一个 JSON 对象 | 结构化日志/管道 |
| `stream-json` | 增量 JSON 流 | 外部工具集成 |

**JSONL 格式**——每个事件是独立的 JSON 对象：
```json
{"type": "message", "content": "...", "timestamp": 1702500000000}
```

**输出初始化**——`getOutputFormat()` 在 `init()` 期间运行，设置全局的 stdout 编码和行结束符处理。

---

## 2.17 帮助系统自定义格式

`configureHelp()` 自定义 Commander 的帮助输出：

**自定义格式**：
- 选项按类别分组显示
- 子命令的描述被截断到终端宽度
- 弃用选项标记为 `[deprecated]`

**HelpV2 组件**——`HelpV2` 类扩展 Commander 的 `Help`：
- 自定义选项排序
- 自定义子命令排序（相关命令分组在一起）
- 自定义换行处理（长描述不破坏对齐）

**Shell 补全**——`registerCompletionCommand()` 注册 `claude completion` 子命令：
```bash
claude completion bash|zsh|fish|powershell
```
生成 shell 特定的补全脚本以安装到 `.bashrc`/`.zshrc` 等。

---

## 2.18 弃用处理机制

**弃用警告**——被弃用的选项在解析时发出警告：
```
⚠️  --mcp-configs is deprecated, use --mcp-config instead
```

**`.implies()` 弃用**——Commander 的 `.implies()` 方法在 v10/v11 中的行为不一致。Claude Code 使用手动 `checkOptionCombinations()` 替代。

**弃用时间线**——选项在弃用后保持 **2 个次要版本**，然后被移除。`DEPRECATED_OPTIONS` 数组追踪。

---

## 2.19 ReplBridge 传输协商

`replBridge` 处理 IDE 集成和子进程之间的路由：

**传输协商**——当 Claude Code 作为子进程启动时（如从 IDE 扩展）：
```
父进程设置 CLAUDE_IDE_MODE=vscode
子进程检测 env → 使用 stdin/stdout JSONRPC 传输
而非 TTY 交互模式
```

**消息格式**——stdin/stdout 上的 NDJSON（Newline Delimited JSON）：
```json
{"jsonrpc": "2.0", "method": "execute", "params": {"command": "...", "args": []}}
```

**权限桥接**——IDE 模式下，权限对话框通过 IDE 的原生 UI 显示而非 ncurses：
- `replBridge.requestPermission()` 发送请求到 IDE
- IDE 显示原生对话框
- 用户选择回传到 Claude Code

---

## 2.20 子命令分发器

**命令路由实现**——`cli.tsx` 中的路由逻辑：

```
if (no subcommand): run interactive REPL
else if subcommand === 'auth': run auth flow
else if subcommand === 'codex': run codex pipeline
else if subcommand === 'completion': output shell completions
else if subcommand === 'mcp': MCP server management
...
```

**Codex 管道**——`codex` 子命令有自包含的管线：
1. 加载凭据（keychain 或 env）
2. 解析 prompt 参数（stdio 或文件）
3. 初始化 API 客户端
4. 执行单次 API 调用
5. 格式化并输出响应

**Auth 管线**——`auth` 子命令处理 OAuth 流程：
- 登录流：打开浏览器，等待回调
- 登出流：清除本地 token
- 状态：显示当前认证状态

---

## 2.21 命令发现与动态加载

**懒加载子命令**——子命令模块**不**在启动时全部导入。它们使用动态 `import()`：
```typescript
program.command('codex').action(() => import('./codex').then(m => m.run()))
```

这防止了 30+ 子命令的同步模块评估影响启动性能。

**插件子命令**——用户安装的插件可注册额外子命令：
- 插件 manifest 声明 `commands` 字段
- 命令在 `loadPlugins()` 期间动态加载
- 插件命令与内置命令在同一命名空间中

---

---

## 2.26 预-Commander 快速路径路由

`entrypoints/cli.tsx`（302 行）——**预-Commander 调度层**，在 `main.tsx` 的 `run()` 之前拦截 CLI。12+ 个快速路径调度器，各用不同最小导入子集：

```
cli.tsx (引导) -> 早期 argv 扫描 -> 快速路径或透传 -> main.tsx run()
```

| 快速路径 | 导入量 | 说明 |
|----------|--------|------|
| `--version/-v` | 零导入 | 仅打印 `MACRO.VERSION` |
| `--dump-system-prompt` | 懒导 `config` + `model` + `prompts` | 渲染系统 prompt 后退出 |
| `--daemon-worker` | 轻量路径 | 无 `enableConfigs()`，无分析 sink |
| `remote-control/rc/bridge` | 全 auth+mcp+policy | 启动 `bridgeMain()` |
| `daemon ps\|logs\|attach\|kill` | Switch 分发 | `handleBgFlag` 回退 |
| `templates new\|list\|reply` | `process.exit(0)` | Ink TUI 可能留事件循环句柄 |
| `--worktree --tmux` | 失败时透传 | 回退到正常 CLI |
| `--update` -> `--upgrade` | `process.argv` 重写 | 拼写重定向 |

**关键约束**——"所有导入都是动态的以最小化快速路径模块评估。`--version` 快速路径除了本文件外零导入。" 整个文件是 302 行调度表，渐进式承诺（progressive commitment）。

---

## 2.27 早期输入捕获：字素簇原始模式 stdin 缓冲

`utils/earlyInput.ts`（192 行）——解决 REPL 初始化前用户已打字（丢失输入）的 UX 问题。`cli.tsx` 的 `void main()` 中最尽早将 stdin 设为 raw 模式并缓冲按键。

**`'readable'` 事件循环**（非 `'data'`）——匹配 Ink 后续处理 stdin 方式一致。

**`processChunk()` 状态机**（72-135 行）：
- Ctrl+C：退出码 130（标准 SIGINT）而非 1
- Ctrl+D：停止捕获不退出
- **退格键按*字素簇*（grapheme cluster）而非字符删除**——通过 `lastGrapheme()` 工具函数
- ESC 序列：扫描 ESC 字节（0x1B）然后跳过直到终止字节（0x40-0x7E 范围）
- 回车符统一为 `\n`

`seedEarlyInput()`（182 行）——可编程预填缓冲（deep-link 流程用于预填 prompt）。

---

## 2.28 Pending-State 全局对象用于早期 argv 提取

`main.tsx`（543-792 行）——三个可变全局对象在 Commander.js 之前通过**原始 argv 扫描**填充。***影子解析器***提取会话关键参数以选择正确后端：

**三个 pending 状态**：
| 状态 | 来源 | 用途 |
|------|------|------|
| `_pendingConnect` | `cc://` 或 `cc+unix://` deep-link URL | 重写为 open 子命令或剥离 |
| `_pendingAssistantChat` | `claude assistant [sessionId]` | 提取位置 sessionId |
| `_pendingSSH` | `--local`、`--dangerously-skip-permissions` 等 | 过滤并转发到远程 |

**`extractFlag` 闭包**——同时支持空格分隔和等号分隔语法，从原始数组中剥离并推入 `extraCliArgs` 转发到远程 CLI spawn：
```typescript
const extractFlag = (flag: string, opts = {}) => {
  const i = rawCliArgs.indexOf(flag)
  if (i !== -1) {
    _pendingSSH.extraCliArgs.push(opts.as ?? flag)
  }
}
extractFlag('-c', { as: '--continue' })  // 规范化短形式
```

---

## 2.29 入口点检测瀑布

`main.tsx`（517-540 行）——`initializeEntrypoint()` 在设置 `clientType` 之前检查 `cliArgs` 中的子命令模式和环境变量，设置 `CLAUDE_CODE_ENTRYPOINT`。**10+ 种运行模式识别**：

| 模式 | Entry point |
|------|-------------|
| `mcp serve` | `mcp` |
| `GITHUB_ACTION` | `claude-code-github-action` |
| 非交互式 | `sdk-cli` |
| 交互式 | `cli` |

运行时机：`cli.tsx` 初始快速路径分发之后、Commander action handler 之前。入口字符串驱动遥测归属、默认问题预览格式和 REPL 可用命令集。

---

## 2.30 命令可用性门控（Auth/Provider 层）

`commands.ts`（417-443 行）——命令有**第三层门控**：`availability`。声明命令需要的认证/环境条件：

```typescript
type CommandAvailability = 'claude-ai' | 'console'
```

`meetsAvailabilityRequirement()` 检查每个命令的 `availability` 数组：
- `'claude-ai'`：需 OAuth 订阅状态（`isClaudeAISubscriber()`）
- `'console'`：需直连 Console API key 用户

此门控每次调用都重新计算（非 memoized）因为认证状态可在会话中 `/login` 后变化。

---

## 2.31 命令类型系统与三个子类型

`types/command.ts`（217 行）——核心 `Command` 联合类型统一 slash 命令、本地 CLI 操作和 JSX 渲染命令：

```typescript
type Command = CommandBase & (PromptCommand | LocalCommand | LocalJSXCommand)
```

| 子类型 | 返回值 | 用途 |
|--------|--------|------|
| **PromptCommand** | `ContentBlockParam[]` | 返回 LLM prompt 文本 |
| **LocalCommand** | 文本输出 | `load()` 返回 `LocalCommandCall` |
| **LocalJSXCommand** | Ink 组件 | `load()` 返回 `LocalJSXCommandCall` + `onDone` 回调 |

**`CommandBase` 中的有趣字段**：
- `immediate?`：绕过调度队列，立即执行
- `isSensitive?`：参数在对话历史中被脱敏
- `paths?`：PromptCommand 专用——模型接触匹配文件后才可见
- `kind?: 'workflow'`：在自动补全中标记为工作流支持

---

## 2.32 Stdout JSON 流守卫

`utils/streamJsonStdoutGuard.ts`（123 行）——`--output-format=stream-json` 激活时模块猴子补丁 `process.stdout.write` 做逐行 JSON 验证器。非 JSON 行静默分流到 stderr，防止 NDJSON 客户端解析器崩溃。

**工作原理**：
1. 包装 `process.stdout.write` 为缓冲行分割器
2. 每行 `JSON.parse`——成功转真实 stdout；失败分流到 stderr 带 `[stdout-guard]` 标记
3. 空行视为有效（容忍 NDJSON 尾部换行）
4. 清理注册表在关闭时 flush 剩余缓冲并恢复原始 `stdout.write`

**核心价值**：一个来源依赖的 `console.log` 即可破坏整个 JSON 流。这层防御将崩溃转为可见警告。

---

## 2.33 精简输出转换器

`utils/streamlinedTransform.ts`（202 行）——有状态输出转换器，将冗长的工具调用周期折叠为摘要模式。**抗蒸馏模式**：

**状态机**——跨助消息累积工具计数（搜索、读取、写入、命令、其他）：
- 分类由工具名前缀匹配定义
- 出现文本消息时发出文本并重置所有计数器
- 文本消息间发出累积工具摘要：`"searched 3 patterns, read 2 files, wrote 1 file"`

工厂模式：`createStreamlinedTransformer()` 返回闭包捕获 `cumulativeCounts`，转换器在调用间保持状态。

---

## 2.34 优先级命令队列与多源输入

`utils/messageQueueManager.ts`（547 行）——统一命令队列接受三类输入：用户打字、任务通知、孤儿权限请求。全部汇入单一优先级排序的 FIFO/FEFO 结构。

**三个优先级**：`now`(0) > `next`(1) > `later`(2)

**出队算法**：
```typescript
export function dequeue(filter?: Filter) {
  let bestIdx = -1, bestPriority = Infinity
  for (let i = 0; i < commandQueue.length; i++) {
    const cmd = commandQueue[i]!
    if (filter && !filter(cmd)) continue
    const priority = PRIORITY_ORDER[cmd.priority ?? 'next']
    if (priority < bestPriority) { bestIdx = i; bestPriority = priority }
  }
  const [dequeued] = commandQueue.splice(bestIdx, 1)
}
```

扫描完整数组找最高优先级后 `splice` 删除。可选 `filter` 缩小候选范围但不影响非匹配条目——支持轮间排空限制为主线程命令（`cmd.agentId === undefined`）。

React 成使用 `Object.freeze()` 快照加`createSignal()` pub/sub 模式，兼容 `useSyncExternalStore`。

---

## 2.35 早期 Pre-Commander 标志解析器

`utils/cliArgs.ts`（61 行）——`eagerParseCliFlag()` 在 Commander.js 处理参数前扫描原始 `process.argv`，支持 `--flag value` 和 `--flag=value` 语法。

仅用于必须在 `init()` 运行前解析的标志：
- `--settings`：决定加载哪个配置文件
- `--setting-sources`：决定加载哪些来源

有意最小化——仅提取字符串值，无布尔值或多值累积。避免导入 Commander 以保持快速路径导入树清洁。

---

## 2.36 NDJSON 行终止符转义

`cli/ndjsonSafeStringify.ts`（32 行）——���义 U+2028（行分隔符）和 U+2029（段落分隔符）防止 NDJSON 流损坏。

**问题**：`JSON.stringify` 原样输出这些字符（ECMA-404 有效），但按行终止符分割的接收者（ECMA-262 定义包括 U+2028/U+2029）会在字符串中间切断。

**解决方案**：单一正则交替——每次匹配回调比两次全字符串扫描更便宜。转义后的 `\uXXXX` 形式 JSON 等价但永远不被误认为行终止符。

---

## 2.37 Stdin 超时与流竞态检测

`utils/process.ts`（48-68 行）——`peekForStdinData()` 竞测超时对抗 `data`/`end` 事件以检测管道 stdin 是否有数据：

```typescript
export function peekForStdinData(stream, ms): Promise<boolean> {
  return new Promise(resolve => {
    const onEnd = () => done(false)
    const peek = setTimeout(done, ms, true)
    stream.once('data', () => clearTimeout(peek))
  })
}
```

在 `getInputPrompt()` 中使用——stdin 被管道但父进程不写入时（如子进程未显式处理 stdin），3 秒超时防止静默挂起。无数据到达时 stderr 警告。

---

## 2.38 Bridge 安全命令白名单

`commands.ts`（618-676 行）——两个独立白名单定义可安全通过不同远程传输执行的 slash 命令：

| 白名单 | 传输类型 | 命令数 | 说明 |
|--------|----------|--------|------|
| `REMOTE_SAFE_COMMANDS` | `--remote` 模式 | 16 条 | 仅影响本地 TUI 状态 |
| `BRIDGE_SAFE_COMMANDS` | Remote Control 桥接（移动/web） | 6 条 | 更受限 |

类型化决策：
```typescript
if (cmd.type === 'local-jsx') return false     // 始终阻断（渲染 Ink UI）
if (cmd.type === 'prompt') return true         // 构造安全
return BRIDGE_SAFE_COMMANDS.has(cmd)           // 'local' 类型的显式允许列表
```

---

## 2.39 SIGINT 抢占式回退（Print 模式）

`main.tsx`（598-606 行）——SIGINT handler 竞争解决器：

```typescript
process.on('SIGINT', () => {
  // Print 模式中 print.ts 注册了自己的 SIGINT handler 中止进行中的查询并 gracefulShutdown
  if (process.argv.includes('-p') || process.argv.includes('--print')) return
  process.exit(0)
})
```

无检查的情况下 main.tsx 的 SIGINT handler 会抢占 print.ts 的更精细的优雅关闭（中止进行中的 API 查询、排空待定事件、然后关闭）。显式让位给更合适的 handler。

---
