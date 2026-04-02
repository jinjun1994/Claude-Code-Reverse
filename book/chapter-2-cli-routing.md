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
