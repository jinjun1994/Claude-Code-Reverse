# AgentTool 整体架构

## 结论先说

`AgentTool` 不是“调用一个外部智能体服务”的薄封装，而是 Claude Code 内部的一套**子代理装载 + 工具裁剪 + 上下文裁剪 + 提示词注入 + 生命周期管理**框架。

它的整体链路可以概括为：

1. 定义 agent
2. 注册 built-in / plugin / custom agent
3. 生成 Agent tool 的用户可见说明
4. 用户或主线程发起 `Agent(...)`
5. 运行时解析 agent 配置、裁剪上下文、解析工具集
6. 执行子代理并回收结果

---

## 1. 第一层：agent 定义模型

agent 定义类型在：`src/tools/AgentTool/loadAgentsDir.ts:110-143`

这里可以看到 agent definition 支持的字段很多，包括：

- `agentType`
- `whenToUse`
- `tools`
- `disallowedTools`
- `model`
- `permissionMode`
- `effort`
- `hooks`
- `skills`
- `isolation`
- `omitClaudeMd`

其中最关键的设计点是：

- built-in agent 用 `getSystemPrompt()` 动态生成提示词：`src/tools/AgentTool/loadAgentsDir.ts:135-143`
- `omitClaudeMd` 被显式定义成 agent 能力的一部分：`src/tools/AgentTool/loadAgentsDir.ts:128-132`

这说明 AgentTool 架构里的“agent”并不只是一个名字 + prompt，而是一整份运行配置。

---

## 2. 第二层：built-in agents 注册

内建 agent 的入口在：`src/tools/AgentTool/builtInAgents.ts:22-72`

`getBuiltInAgents()` 会按条件返回内建 agent 列表：

- 默认总有：
  - `GENERAL_PURPOSE_AGENT`
  - `STATUSLINE_SETUP_AGENT`
  - 见 `src/tools/AgentTool/builtInAgents.ts:45-48`
- feature gate 开启时追加：
  - `EXPLORE_AGENT`
  - `PLAN_AGENT`
  - 见 `src/tools/AgentTool/builtInAgents.ts:50-52`
- 非 SDK 入口时追加：
  - `CLAUDE_CODE_GUIDE_AGENT`
  - 见 `src/tools/AgentTool/builtInAgents.ts:54-62`
- verification gate 开启时追加：
  - `VERIFICATION_AGENT`
  - 见 `src/tools/AgentTool/builtInAgents.ts:64-68`

这里的反常识点是：

- built-in agents 不是固定全集
- 它们受 feature gate、入口模式、实验开关影响
- 所以“当前可用 agent 列表”是运行时产物，不是静态常量表

---

## 3. 第三层：built-in / plugin / custom 三路合流

统一装载逻辑在：`src/tools/AgentTool/loadAgentsDir.ts:296-393`

`getAgentDefinitionsWithOverrides(cwd)` 的流程：

1. 如果 `CLAUDE_CODE_SIMPLE` 打开，只返回 built-ins：`src/tools/AgentTool/loadAgentsDir.ts:298-305`
2. 从 markdown 子目录加载 custom agents：`src/tools/AgentTool/loadAgentsDir.ts:308-343`
3. 并发加载 plugin agents：`src/tools/AgentTool/loadAgentsDir.ts:344-355`
4. 获取 built-in agents：`src/tools/AgentTool/loadAgentsDir.ts:357`
5. 合并成 `allAgentsList`：`src/tools/AgentTool/loadAgentsDir.ts:359-363`
6. 再通过 `getActiveAgentsFromList(...)` 做最终激活版本选择：`src/tools/AgentTool/loadAgentsDir.ts:365`

这一层说明：

- AgentTool 不是“只认识内建 agent”
- 内建、插件、自定义 agent 在运行时会统一抽象成 `AgentDefinition`

这是整套架构扩展性的核心。

---

## 4. 第四层：给主线程生成 Agent tool 的使用说明

用户可见的 Agent tool 说明在：`src/tools/AgentTool/prompt.ts:201-287`

这里做的事情不是执行 agent，而是**塑造主线程如何使用 agent**。

关键点：

### 4.1 agent 列表可能不是直接内联
- 当开关打开时，agent 列表来自 attachment，而不是 prompt 本体：`src/tools/AgentTool/prompt.ts:190-199`

目的很反常识：
- 不是为了功能
- 是为了**保持 tool description 静态，避免 prompt cache 失效**

### 4.2 不指定 `subagent_type` 时默认走 `general-purpose`
- `src/tools/AgentTool/prompt.ts:208-212`

这点决定了 `general-purpose` 是整个系统的默认 agent 落点。

### 4.3 系统主动告诉主线程：很多搜索场景不该起 agent
- `src/tools/AgentTool/prompt.ts:235-239`

也就是：
- AgentTool 不是默认入口
- 直接 `Read/Grep/Glob` 在很多情况下优先级更高

### 4.4 每次 fresh invocation 都要完整 briefing
- `src/tools/AgentTool/prompt.ts:267`

这意味着 specialized subagent 并不是天然继承全部主线程上下文，它更像一个新开的 worker。

---

## 5. 第五层：工具解析模型

真实工具解析逻辑在：`src/tools/AgentTool/agentToolUtils.ts:122-225`

这是理解 AgentTool 的核心文件之一。

### 5.1 先过滤 agent 不该看到的工具
- `filterToolsForAgent(...)` 在更前面负责全局过滤
- `resolveAgentTools(...)` 在 `src/tools/AgentTool/agentToolUtils.ts:140-147` 调它

### 5.2 再应用 agent 自己的 `disallowedTools`
- `src/tools/AgentTool/agentToolUtils.ts:149-160`

### 5.3 如果 `tools` 未定义或是 `['*']`，都视为 wildcard
- `src/tools/AgentTool/agentToolUtils.ts:162-172`

这是全套架构里最大的反常识之一：

- 很多 agent 看起来像“没写 tools，所以工具更少”
- 实际上恰好相反：没写 `tools` 常常意味着“全部剩余工具都能用”

### 5.4 `Agent` 工具自身还有特殊元数据解析
- `src/tools/AgentTool/agentToolUtils.ts:190-204`

即使 subagent 本身通常拿不到 Agent tool，这里仍然保留了 `allowedAgentTypes` 的解析逻辑，说明系统把“工具权限”和“可再派生的 agent 类型限制”当成统一规则体系的一部分。

---

## 6. 第六层：真正执行 agent

执行主流程在：`src/tools/AgentTool/runAgent.ts:380-598`

这部分是 AgentTool 的运行时核心。

### 6.1 取基础上下文
- `getUserContext()` / `getSystemContext()`：`src/tools/AgentTool/runAgent.ts:380-383`

### 6.2 根据 agent 配置裁剪上下文
- 裁掉 `claudeMd`：`src/tools/AgentTool/runAgent.ts:385-398`
- 对 `Explore/Plan` 裁掉 `gitStatus`：`src/tools/AgentTool/runAgent.ts:400-410`

这里说明：
- 上下文并不是统一下发给所有 agent
- agent 可以有不同的“上下文密度”

### 6.3 调整权限模式与提示行为
- 权限模式覆盖：`src/tools/AgentTool/runAgent.ts:412-434`
- async agent 自动避免 permission prompts：`src/tools/AgentTool/runAgent.ts:436-451`
- background 场景等自动检查：`src/tools/AgentTool/runAgent.ts:453-463`
- `allowedTools` 作用域隔离：`src/tools/AgentTool/runAgent.ts:465-479`

这说明 AgentTool 并不是“prompt-based 小功能”，而是和权限系统深度耦合。

### 6.4 解析 agent 可用工具
- `resolveAgentTools(...)`：`src/tools/AgentTool/runAgent.ts:500-502`

### 6.5 生成 agent system prompt
- `getAgentSystemPrompt(...)`：`src/tools/AgentTool/runAgent.ts:508-518`

### 6.6 构造 agent 生命周期资源
- `AbortController`：`src/tools/AgentTool/runAgent.ts:520-528`
- `SubagentStart` hooks：`src/tools/AgentTool/runAgent.ts:530-555`
- frontmatter hooks 注册：`src/tools/AgentTool/runAgent.ts:557-575`
- 预加载 skills：`src/tools/AgentTool/runAgent.ts:577-598`

也就是说，一个 agent 启动时不仅拿到 prompt 和 tools，还能拿到：
- 生命周期 hook
- 前置技能
- 权限环境
- working directory 范围

---

## 7. 第七层：结果回收

agent 结果回收逻辑在：`src/tools/AgentTool/agentToolUtils.ts:276-357`

`finalizeAgentTool(...)` 负责：

- 找到最终 assistant 文本：`src/tools/AgentTool/agentToolUtils.ts:297-317`
- 统计 token / tool 使用次数：`src/tools/AgentTool/agentToolUtils.ts:319-355`
- 记录 analytics
- 返回结构化结果

这里也能看出一个设计取向：

- agent 返回的不是“原始会话”，而是一个被压缩后的结果对象
- 主线程只拿必要内容，再转述给用户

---

## 8. 这套架构的真正心智模型

如果把整个 AgentTool 看成一条流水线，可以写成：

### 8.1 定义阶段
`BuiltInAgentDefinition / CustomAgentDefinition / PluginAgentDefinition`

↓

### 8.2 装载阶段
`getBuiltInAgents()` + custom markdown + plugin agents

↓

### 8.3 暴露阶段
`prompt.ts` 生成主线程可见的 Agent tool 使用说明

↓

### 8.4 调用阶段
主线程调用 `Agent(...)`，传入：
- `subagent_type`
- `prompt`
- `run_in_background`
- `isolation`

↓

### 8.5 运行时准备
`runAgent.ts`：
- 取 context
- 裁 context
- 设 permission
- 解析 tools
- 生成 system prompt
- 注册 hooks / preload skills

↓

### 8.6 执行与回收
子代理运行，最后由 `finalizeAgentTool(...)` 回收结果

---

## 9. 几个最值得记住的反常识

### 反常识 1：AgentTool 的重点不是“多一个模型线程”，而是“多一层运行时配置壳”
agent 的差异很多都不是模型本身，而是：
- prompt
- tools
- context
- hooks
- permission mode

### 反常识 2：可用 agent 列表是运行时拼出来的，不是静态写死
证据：`src/tools/AgentTool/builtInAgents.ts:22-72` + `src/tools/AgentTool/loadAgentsDir.ts:296-393`

### 反常识 3：`tools` 不写，常常不是更少，而是更多
证据：`src/tools/AgentTool/agentToolUtils.ts:162-172`

### 反常识 4：prompt.ts 不只是文案文件，而是在做系统行为引导与缓存优化
证据：`src/tools/AgentTool/prompt.ts:190-199`

### 反常识 5：runAgent.ts 的关键职责不是“调用模型”，而是“准备一个 agent 专属执行环境”
证据：`src/tools/AgentTool/runAgent.ts:380-598`

### 反常识 6：很多 agent 的差异最终来自 token 成本设计
比如：
- `omitClaudeMd`
- 删除陈旧 `gitStatus`
- one-shot builtin

这说明 AgentTool 设计里，成本控制是一级目标，不是事后优化。

---

## 10. 一句话总结

> AgentTool 本质上是一套可扩展的子代理执行框架：先把 built-in / plugin / custom agent 统一成 `AgentDefinition`，再在调用时按 agent 配置为其裁剪上下文、解析工具、注入 prompt、挂接 hooks/skills，最后回收结构化结果；它的核心价值在“运行时装配”，不在“多开一个模型会话”本身。
