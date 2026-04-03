# 内建 agents 全景图

## 结论先说

Claude Code 的 built-in agents 不是一组“平铺的工具人格”，而是几类非常不同的运行时角色：

- 默认执行者：`general-purpose`
- 只读研究员：`Explore`
- 只读规划师：`Plan`
- 文档顾问：`claude-code-guide`
- 定向配置器：`statusline-setup`
- 对抗式验收员：`verification`

它们共用同一套 AgentTool runtime，但在下面几件事上差别很大：

- 是否默认启用
- 是否 one-shot
- 是否只读
- 工具白名单/黑名单策略
- 是否裁剪上下文
- 模型策略
- 是否允许后台运行
- 产出形式

---

## 1. built-in agents 从哪里来

统一注册入口在：`src/tools/AgentTool/builtInAgents.ts:22-72`

默认加入：
- `general-purpose`：`src/tools/AgentTool/builtInAgents.ts:45-48`
- `statusline-setup`：`src/tools/AgentTool/builtInAgents.ts:45-48`

条件加入：
- `Explore` + `Plan`：feature gate 控制，`src/tools/AgentTool/builtInAgents.ts:13-19,50-52`
- `claude-code-guide`：仅非 SDK 入口，`src/tools/AgentTool/builtInAgents.ts:54-62`
- `verification`：feature gate 控制，`src/tools/AgentTool/builtInAgents.ts:64-68`

第一层反常识：

> 内建 agent 列表不是静态常量，而是运行时按入口、实验和开关拼出来的。

---

## 2. 总览表

| agent | 角色 | 默认启用 | 只读 | one-shot | 工具策略 | 模型 | 特别之处 |
|---|---|---:|---:|---:|---|---|---|
| `general-purpose` | 通用执行者 | 是 | 否 | 否 | `tools: ['*']` | 默认继承 | 默认 subagent |
| `Explore` | 代码探索员 | 条件启用 | 是 | 是 | denylist + prompt | 外部偏 `haiku` | 裁剪 `claudeMd` / `gitStatus` |
| `Plan` | 架构规划师 | 条件启用 | 是 | 是 | denylist + prompt | `inherit` | 强制输出关键文件 |
| `claude-code-guide` | Claude 文档顾问 | 条件启用 | 实质只读 | 否 | 显式白名单 | `haiku` | `dontAsk`，会抓官方文档 |
| `statusline-setup` | 状态栏配置器 | 是 | 否 | 否 | 仅 `Read`/`Edit` | `sonnet` | 极窄白名单，定向修改设置 |
| `verification` | 对抗式验证员 | 条件启用 | 项目内只读 | 否 | denylist + prompt | `inherit` | 默认后台，必须给 verdict |

---

## 3. general-purpose：默认 worker

定义在：`src/tools/AgentTool/built-in/generalPurposeAgent.ts:25-34`

核心特征：
- `tools: ['*']`：`src/tools/AgentTool/built-in/generalPurposeAgent.ts:29`
- 模型未显式指定，走默认 subagent model：`src/tools/AgentTool/built-in/generalPurposeAgent.ts:32-33`
- prompt 定位是研究 + 执行 multi-step task：`src/tools/AgentTool/built-in/generalPurposeAgent.ts:3-23`

更关键的是：
- Agent tool prompt 明确规定，不指定 `subagent_type` 时就用它：`src/tools/AgentTool/prompt.ts:208-212`

所以它的真实地位不是“其中一个内建 agent”，而是：

- AgentTool 的默认落点
- 默认 worker
- 默认兜底执行者

反常识：

> 大多数人会以为系统会自动帮你挑最专业的 agent；实际上没指定类型时，直接落到 `general-purpose`。

---

## 4. Explore：高频、只读、瘦上下文研究员

定义在：`src/tools/AgentTool/built-in/exploreAgent.ts:64-83`

核心特征：
- 只读 prompt：`src/tools/AgentTool/built-in/exploreAgent.ts:24-56`
- denylist 禁掉 `Agent / ExitPlanMode / Edit / Write / NotebookEdit`：`src/tools/AgentTool/built-in/exploreAgent.ts:67-73`
- 外部用户模型偏 `haiku`：`src/tools/AgentTool/built-in/exploreAgent.ts:76-78`
- `omitClaudeMd: true`：`src/tools/AgentTool/built-in/exploreAgent.ts:79-81`

运行时特殊处理：
- 删 `claudeMd`：`src/tools/AgentTool/runAgent.ts:385-398`
- 删 `gitStatus`：`src/tools/AgentTool/runAgent.ts:400-410`
- 是 one-shot：`src/tools/AgentTool/constants.ts:6-12`

官方使用建议也很关键：
- 简单搜索别用它，直接工具更快：`src/constants/prompts.ts:374-380`

反常识：

1. 它看起来像白名单搜索 agent，其实不是严格白名单
2. 它不是默认搜索入口，官方反而说它比 direct tool 更慢
3. 它被极度优化 token 成本，说明是超高频路径

---

## 5. Plan：一次性架构规划师，不等于 plan mode

定义在：`src/tools/AgentTool/built-in/planAgent.ts:73-92`

核心特征：
- prompt 明确要求只读调研 + 设计实施方案：`src/tools/AgentTool/built-in/planAgent.ts:21-70`
- denylist 与 Explore 基本一致：`src/tools/AgentTool/built-in/planAgent.ts:77-83`
- `tools: EXPLORE_AGENT.tools`：`src/tools/AgentTool/built-in/planAgent.ts:85`
- `model: 'inherit'`：`src/tools/AgentTool/built-in/planAgent.ts:87`
- `omitClaudeMd: true`：`src/tools/AgentTool/built-in/planAgent.ts:88-90`

输出要求：
- 必须给 `### Critical Files for Implementation`：`src/tools/AgentTool/built-in/planAgent.ts:60-69`

运行时：
- 和 Explore 一样裁剪 `claudeMd` / `gitStatus`：`src/tools/AgentTool/runAgent.ts:385-410`
- 也是 one-shot：`src/tools/AgentTool/constants.ts:6-12`

最大反常识：

> `Plan agent` 不是 plan mode 本身。

因为它自己就禁止 `ExitPlanMode`：`src/tools/AgentTool/built-in/planAgent.ts:77-83`

所以它只是：
- 只读规划顾问
- 给主线程一份计划草案
- 真正的 Enter/Exit plan mode 仍然是主线程工具职责

---

## 6. claude-code-guide：文档型顾问，不是代码库搜索 agent

定义在：`src/tools/AgentTool/built-in/claudeCodeGuideAgent.ts:98-205`

核心特征：
- 专门回答三类问题：Claude Code / Claude Agent SDK / Claude API：`src/tools/AgentTool/built-in/claudeCodeGuideAgent.ts:30-39,98-100`
- 工具不是 wildcard，而是显式白名单：
  - 本地搜索：`Read + Glob/Grep` 或 embedded 环境下 `Read + Bash`
  - 网络抓取：`WebFetch + WebSearch`
  - 见 `src/tools/AgentTool/built-in/claudeCodeGuideAgent.ts:103-116`
- `model: 'haiku'`：`src/tools/AgentTool/built-in/claudeCodeGuideAgent.ts:119`
- `permissionMode: 'dontAsk'`：`src/tools/AgentTool/built-in/claudeCodeGuideAgent.ts:120`

它还有一个很特别的地方：
- `getSystemPrompt({ toolUseContext })` 会动态注入用户当前配置：
  - custom skills
  - custom agents
  - MCP servers
  - plugin skills
  - settings.json
  - 见 `src/tools/AgentTool/built-in/claudeCodeGuideAgent.ts:121-204`

这说明它不是单纯“去网上搜文档”，而是：
- 把官方文档和用户当前本地配置拼起来回答问题

反常识：

1. 它虽然是 built-in agent，但更像“文档问答路由器”
2. 它比 Explore/Plan 更依赖显式白名单，而不是 denylist
3. 它的上下文丰富度来自动态配置注入，而不是大范围代码库上下文

---

## 7. statusline-setup：最窄、最定向的 built-in agent

定义在：`src/tools/AgentTool/built-in/statuslineSetup.ts:134-144`

核心特征：
- `tools: ['Read', 'Edit']`：`src/tools/AgentTool/built-in/statuslineSetup.ts:138`
- `model: 'sonnet'`：`src/tools/AgentTool/built-in/statuslineSetup.ts:141`
- 专门改 `~/.claude/settings.json` 中的 `statusLine` 配置：`src/tools/AgentTool/built-in/statuslineSetup.ts:116-125`

它的 system prompt 非常长，但职责极窄：
- 读 shell 配置
- 转换 PS1
- 写入 Claude Code statusLine command
- 保留现有 settings
- 如果设置文件是 symlink，要改目标文件

反常识：

> 它不是“研究型子代理”，而是一个狭义任务机器人。

也就是说，built-in agent 里并不全是“智能研究员”风格，有的就是强约束、强流程的定向配置器。

---

## 8. verification：最像“对抗 QA”的 built-in agent

定义在：`src/tools/AgentTool/built-in/verificationAgent.ts:134-152`

核心特征：
- `background: true`：`src/tools/AgentTool/built-in/verificationAgent.ts:138`
- denylist 禁掉写工具和 `Agent`：`src/tools/AgentTool/built-in/verificationAgent.ts:139-145`
- `model: 'inherit'`：`src/tools/AgentTool/built-in/verificationAgent.ts:148`
- 有 `criticalSystemReminder_EXPERIMENTAL`：`src/tools/AgentTool/built-in/verificationAgent.ts:150-151`

最特殊的 prompt 特征：
- 明确要求“不是确认实现工作，而是去试着搞坏它”：`src/tools/AgentTool/built-in/verificationAgent.ts:10-13`
- 允许在 `/tmp` 写临时测试脚本，但**禁止改项目目录**：`src/tools/AgentTool/built-in/verificationAgent.ts:14-21`
- 必须输出标准化检查块和最终 `VERDICT: PASS/FAIL/PARTIAL`：`src/tools/AgentTool/built-in/verificationAgent.ts:81-129`

反常识：

1. 它不是普通“跑测试” agent，而是有明显 adversarial persona
2. 它不是完全只读——允许在临时目录写验证脚本
3. 它默认后台运行，这和其他几个 built-in 很不一样

---

## 9. one-shot 内建 agent 其实只有两个

定义在：`src/tools/AgentTool/constants.ts:6-12`

只有：
- `Explore`
- `Plan`

这说明 built-in agents 其实分成两大类：

### 一次性报告型
- `Explore`
- `Plan`

特点：
- 跑一次
- 回报告
- 通常不续聊
- 更重视 token 成本

### 持续协作型 / 任务型
- `general-purpose`
- `claude-code-guide`
- `statusline-setup`
- `verification`

特点：
- 可以继续跟进
- 更像专门 worker
- 不一定追求极致瘦上下文

---

## 10. 再看一遍：哪些是真白名单，哪些是假白名单

### 真正显式白名单风格
- `general-purpose`：`['*']`
- `claude-code-guide`：显式列出工具
- `statusline-setup`：显式 `['Read', 'Edit']`

### 表面像专用 agent，实际是 denylist + prompt 约束
- `Explore`
- `Plan`
- `verification`

这里又有个反常识：

> 最“像专用工具人”的 Explore / Plan，反而不是最严格白名单；最严格的反而是 statusline-setup 和 claude-code-guide 这种狭义任务 agent。

---

## 11. 整体心智模型

如果从产品角色看，这些 built-in agents 可以分成三层：

### A. 默认劳动力层
- `general-purpose`

### B. 认知增强层
- `Explore`
- `Plan`
- `claude-code-guide`

它们主要帮主线程：
- 调研
- 规划
- 查询官方知识

### C. 定向执行 / 验收层
- `statusline-setup`
- `verification`

它们主要帮主线程：
- 改特定配置
- 做独立验收

这比“按文件名理解 agent”更接近真实系统结构。

---

## 12. 一句话总结

> Claude Code 的 built-in agents 不是一排平等的小助手，而是一个分层体系：`general-purpose` 负责默认执行，`Explore/Plan/claude-code-guide` 负责认知外包，`statusline-setup/verification` 负责强约束任务；它们共用同一运行时框架，但在工具策略、上下文密度、输出契约和产品定位上差异极大。
