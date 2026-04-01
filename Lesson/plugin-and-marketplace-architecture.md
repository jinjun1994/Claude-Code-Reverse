# Claude Code 插件与市场架构图

基于 `outputs/claude-cli-clean.js` 中与 plugin manifest、marketplace schema、plugin CLI、plugin configs、plugin hooks/skills/agents/LSP/MCP 扩展能力相关实现整理。

## 1. 架构图

```mermaid
flowchart TD
    USER[User or admin<br/>用户或管理员] --> MARKET[Marketplace sources<br/>市场源]
    USER --> CLI[plugin subcommands<br/>plugin 子命令]

    MARKET --> MANIFEST[Plugin and marketplace manifests<br/>插件与市场清单]
    MANIFEST --> VALIDATE[Schema validation<br/>Schema 校验]
    VALIDATE --> REG[Plugin registry<br/>插件注册表]

    CLI --> INSTALL[install uninstall enable disable update<br/>安装 卸载 启用 禁用 更新]
    INSTALL --> REG

    REG --> PLUGIN[Installed plugin package<br/>已安装插件包]

    subgraph PluginSurfaces[插件扩展面 / Plugin Surfaces]
        PSROOT[Plugin capability surfaces<br/>插件能力扩展面]
        HOOKS[hooks<br/>钩子]
        CMDS[commands and skills<br/>命令与技能]
        AGENTS[agents<br/>代理定义]
        STYLES[outputStyles<br/>输出风格]
        MCP[mcpServers or MCPB<br/>MCP 服务或 MCPB]
        LSP[lspServers<br/>LSP 服务]
        OPTIONS[userConfig and options<br/>用户配置项]
    end

    PSROOT --> HOOKS
    PSROOT --> CMDS
    PSROOT --> AGENTS
    PSROOT --> STYLES
    PSROOT --> MCP
    PSROOT --> LSP
    PSROOT --> OPTIONS

    PLUGIN --> PSROOT
    OPTIONS --> CFG[pluginConfigs and secure storage<br/>pluginConfigs 与安全存储]
    MCP --> TOOLSYS[Tool and runtime integration<br/>工具与运行时集成]
    HOOKS --> TOOLSYS
    CMDS --> TOOLSYS
    AGENTS --> TOOLSYS
```

## 2. 架构图详细说明

### 2.1 插件不是单一命令，而是一组可注册能力

插件 manifest 中不只描述名字和版本，还能携带：

- hooks
- commands
- skills
- agents
- outputStyles
- MCP servers 或 MCPB
- LSP servers
- userConfig

因此插件系统不是“装一个脚本”，而是“把一组可扩展能力注入 Claude Code 运行时”。对应：`outputs/claude-cli-clean.js:36360-36447`。

### 2.2 marketplace 与 installed plugin 是两层结构

源码中同时定义了：

- marketplace source
- installedPlugins
- installedPluginsHistory
- additionalMarketplaces
- strictKnownMarketplaces
- blockedMarketplaces

这说明插件系统在架构上分为：

1. 市场源管理
2. manifest 拉取与校验
3. 本地安装与启停
4. 历史与策略控制

对应：`outputs/claude-cli-clean.js:36333-36603`, `36763-36765`。

### 2.3 plugin CLI 是插件控制面

插件相关 CLI 命令包括：

- `plugin validate`
- `plugin list`
- `plugin marketplace add/list/remove/update`
- `plugin install`
- `plugin uninstall`
- `plugin enable`
- `plugin disable`
- `plugin update`

因此插件系统既有运行时层，也有完整 CLI 控制面。对应：`outputs/claude-cli-clean.js:376755-376847`。

### 2.4 pluginConfigs 把插件运行期配置接回 settings 层

`pluginConfigs` 允许为每个插件保存：

- `mcpServers` 用户配置值
- `options` 非敏感配置值
- 敏感配置转入 secure storage

这意味着插件系统并不是独立文件夹，而是和 settings/persistence 系统深度耦合。对应：`outputs/claude-cli-clean.js:36794-36797`, `36420-36421`。

## 3. 时序图

```mermaid
sequenceDiagram
    autonumber
    actor User as User / 用户
    participant CLI as plugin command / plugin 命令
    participant Market as Marketplace source / 市场源
    participant Schema as Manifest schema / Manifest Schema
    participant Registry as Plugin registry / 插件注册表
    participant Runtime as Claude runtime / Claude 运行时

    User->>CLI: 执行 plugin install 或 marketplace add
    CLI->>Market: 读取 marketplace 或 plugin source
    Market->>Schema: 返回 manifest
    Schema-->>CLI: 校验 manifest
    CLI->>Registry: 安装并登记插件
    Registry->>Runtime: 注册 hooks skills agents MCP LSP 等能力
    Runtime-->>User: 新能力可被调用
```

## 4. 时序图详细说明

这条时序强调两件事：

1. 先有 marketplace 或 plugin source
2. 再经 schema 校验后进入 registry
3. 最后才把 hooks/skills/agents/MCP/LSP 等能力暴露给运行时

所以插件安装本质上是“受控能力注入流程”。

## 5. 代码依据

- 官方市场名与 manifest 规则：`outputs/claude-cli-clean.js:36333-36354`
- plugin manifest 主体：`outputs/claude-cli-clean.js:36360-36447`
- userConfig 与敏感配置：`outputs/claude-cli-clean.js:36420-36421`
- marketplace 与 installed plugin 结构：`outputs/claude-cli-clean.js:36505-36603`
- additionalMarketplaces 与 pluginConfigs：`outputs/claude-cli-clean.js:36763-36797`
- plugin CLI：`outputs/claude-cli-clean.js:376755-376847`
