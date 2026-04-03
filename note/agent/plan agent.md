# Plan agent 如何实现

## 结论先说

`Plan` 不是“进入计划模式的主线程代理”，而是一个**只负责调研和产出实施方案的只读子代理**。它和 `Explore` 共享很多基础设计：都走同一套 subagent 框架、都裁剪上下文、都 one-shot、都主要靠 prompt 约束。但 `Plan` 比 `Explore` 多了一层“架构师视角”的输出结构要求。

---

## 1. 定义在哪里

核心定义在：`src/tools/AgentTool/built-in/planAgent.ts:14-92`

关键字段：

- `agentType: 'Plan'`：`src/tools/AgentTool/built-in/planAgent.ts:73-74`
- `whenToUse`：`src/tools/AgentTool/built-in/planAgent.ts:75-76`
- `disallowedTools`：`src/tools/AgentTool/built-in/planAgent.ts:77-83`
- `tools: EXPLORE_AGENT.tools`：`src/tools/AgentTool/built-in/planAgent.ts:85`
- `model: 'inherit'`：`src/tools/AgentTool/built-in/planAgent.ts:87`
- `omitClaudeMd: true`：`src/tools/AgentTool/built-in/planAgent.ts:88-90`
- `getSystemPrompt()`：`src/tools/AgentTool/built-in/planAgent.ts:91`

它的 system prompt 把自己定义成：

- `software architect and planning specialist`
- 只读
- 不允许创建/修改/删除文件
- 任务目标是“explore the codebase and design implementation plans”
- 最终必须输出 `### Critical Files for Implementation`

对应提示词在：`src/tools/AgentTool/built-in/planAgent.ts:21-70`

---

## 2. 它如何被注册

注册逻辑和 Explore 一样，在：`src/tools/AgentTool/builtInAgents.ts:22-72`

关键点：

1. `getBuiltInAgents()` 中，只有 `areExplorePlanAgentsEnabled()` 为真时，才一起加入 `EXPLORE_AGENT, PLAN_AGENT`：`src/tools/AgentTool/builtInAgents.ts:50-52`
2. 这个开关本身又受 feature gate 控制：`src/tools/AgentTool/builtInAgents.ts:13-19`

也就是说：

- `Plan` 不是永远可用的核心能力
- 而是和 `Explore` 绑定投放的内建子代理

这是第一个反常识点。

---

## 3. 它不是 EnterPlanMode / ExitPlanMode 本身

这个很容易误解。

`Plan` 虽然名字叫 Plan，但它并不等于主线程的“计划模���”。

证据：
- `Plan` 自己的 `disallowedTools` 明确禁止 `ExitPlanMode`：`src/tools/AgentTool/built-in/planAgent.ts:77-83`
- 同时也禁止 `Edit/Write/NotebookEdit`

这说明：

- `Plan` 不能自己结束 plan mode
- 不能直接改代码
- 不能自己把计划写进主线程的计划文件

它的角色只是：

1. 帮主线程调研
2. 给出实施方案
3. 列出关键文件
4. 返回结果给主线程

真正的 Enter/Exit plan mode 是主线程工具层职责，不是这个子代理的职责。

这是一个很重要的“名实不符”反常识：

> `Plan agent` 不是“plan mode 的执行者”，而是“给主线程提供计划草案的只读架构师”。

---

## 4. system prompt 如何塑造它的行为

提示词在：`src/tools/AgentTool/built-in/planAgent.ts:21-70`

它要求的流程是：

1. **Understand Requirements**：理解需求：`src/tools/AgentTool/built-in/planAgent.ts:39`
2. **Explore Thoroughly**：读文件、找模式、理解架构、找相似实现、追踪代码路径：`src/tools/AgentTool/built-in/planAgent.ts:41-48`
3. **Design Solution**：设计方案、考虑 trade-off、遵循已有模式：`src/tools/AgentTool/built-in/planAgent.ts:50-53`
4. **Detail the Plan**：给出步骤、依赖、顺序、风险：`src/tools/AgentTool/built-in/planAgent.ts:55-58`

最后强制它输出：
- `### Critical Files for Implementation`：`src/tools/AgentTool/built-in/planAgent.ts:60-69`

这意味着 `Plan` 不只是“搜到什么报什么”，而是要做一层抽象：

- 从代码观察提升到实施策略
- 从文件事实提升到架构建议

和 `Explore` 相比，`Plan` 的核心附加值不是搜索，而是**规划结构化输出**。

---

## 5. 运行时怎么执行

运行时主流程在：`src/tools/AgentTool/runAgent.ts:380-518`

`Plan` 和 `Explore` 共用同一条执行路径。

### 5.1 裁剪 CLAUDE.md
- `omitClaudeMd` 为真时，会从 userContext 中去掉 `claudeMd`：`src/tools/AgentTool/runAgent.ts:390-398`

### 5.2 裁剪 gitStatus
- 对 `agentType === 'Explore' || agentType === 'Plan'`，会去掉 session-start 的 `gitStatus`：`src/tools/AgentTool/runAgent.ts:404-410`

### 5.3 解析工具
- `resolveAgentTools(...)`：`src/tools/AgentTool/runAgent.ts:500-502`

### 5.4 生成 agent 专属 system prompt
- `getAgentSystemPrompt(...)`：`src/tools/AgentTool/runAgent.ts:508-518`

说明 `Plan` 并没有特殊执行引擎；它只是：

- 用同一套 Agent runtime
- 换了一份更偏设计/规划的提示词
- 再叠加和 Explore 一样的上下文瘦身策略

---

## 6. 它和 Explore 的关系非常近

有一个很微妙的实现细节：

- `tools: EXPLORE_AGENT.tools`：`src/tools/AgentTool/built-in/planAgent.ts:85`

但 `EXPLORE_AGENT.tools` 本身其实是未定义的，因为 Explore 定义里并没有 `tools` 字段，只有 `disallowedTools`：`src/tools/AgentTool/built-in/exploreAgent.ts:64-83`

这就导致：

- `Plan.tools` 实际上也是 `undefined`
- 而 `resolveAgentTools()` 会把 `tools === undefined` 当作 wildcard：`src/tools/AgentTool/agentToolUtils.ts:162-172`

所以 `Plan` 的真实工具模型和 Explore 一样：

- 不是硬白名单
- 是“全部可用工具”减去 denylist

这非常反常识，因为代码表面看起来像是在“复用 Explore 的工具配置”，但实际上复用到的是一个 `undefined`。

可以理解为：

- 这是在表达一种语义上的“Plan 和 Explore 同类”
- 但不是技术上真正显式列出同一组白名单工具

---

## 7. 真正的工具权限模型

核心逻辑在：`src/tools/AgentTool/agentToolUtils.ts:122-172`

### 7.1 先做通用 agent 过滤
`filterToolsForAgent(...)`：`src/tools/AgentTool/agentToolUtils.ts:70-116`

先去掉：
- 全局 agent 禁用工具
- 某些自定义 agent 禁止工具
- async agent 不允许的工具

### 7.2 再应用 Plan 自己的 denylist
Plan 禁掉的是：
- `Agent`
- `ExitPlanMode`
- `Edit`
- `Write`
- `NotebookEdit`

来源：`src/tools/AgentTool/built-in/planAgent.ts:77-83`

### 7.3 `tools` 未定义就按 wildcard 处理
实现：`src/tools/AgentTool/agentToolUtils.ts:162-172`

所以真实结论是：

- `Plan` 并不是“只允许 Read/Grep/Glob/Bash”
- 而是“几乎所有 agent 可见工具都可用，只是删掉一小部分写和流程控制工具”

和它只读规划师的人设相比，这个权限模型明显更宽。

---

## 8. 它和 general-purpose agent 的差异

普通 agent 在：`src/tools/AgentTool/built-in/generalPurposeAgent.ts:25-34`

### general-purpose
- `tools: ['*']`：`src/tools/AgentTool/built-in/generalPurposeAgent.ts:29`
- 默认模型未写死，走 subagent 默认 `inherit`：`src/tools/AgentTool/built-in/generalPurposeAgent.ts:32-33` + `src/utils/model/agent.ts:21-27`
- 提示词允许执行任务，不只是研究：`src/tools/AgentTool/built-in/generalPurposeAgent.ts:3-23`

### Plan
- 只读，禁止写文件：`src/tools/AgentTool/built-in/planAgent.ts:23-33`
- 专注设计和实施顺序：`src/tools/AgentTool/built-in/planAgent.ts:37-58`
- 必须列关键实现文件：`src/tools/AgentTool/built-in/planAgent.ts:60-69`
- `model: 'inherit'`：`src/tools/AgentTool/built-in/planAgent.ts:87`

所以 Plan 和 general-purpose 的差异不在于“底层引擎不同”，而在于：

- 提示词目标不同
- denylist 不同
- 上下文瘦身不同
- 产出格式要求不同

---

## 9. 它也是 one-shot built-in agent

定义在：`src/tools/AgentTool/constants.ts:6-12`

`ONE_SHOT_BUILTIN_AGENT_TYPES` 包含：
- `Explore`
- `Plan`

注释说明：
- 这种 agent 跑一次、回一次报告
- 父线程通常不会继续 `SendMessage`
- 为了省 token，会省掉结果里的某些尾部提示

这意味着 `Plan` 的产品定位不是持续对话式顾问，而是：

- 一次性生成计划草案
- 结果交给主线程消化、落地、向用户展示

---

## 10. 它也是 fresh-context subagent

Agent tool 的提示里写得很清楚：
- 每次 fresh Agent invocation 都要提供完整任务描述：`src/tools/AgentTool/prompt.ts:267`
- 指定 `subagent_type` 的 specialized agent 是 fresh start：`src/tools/AgentTool/prompt.ts:145-152`

所以 `Plan` 并不会自动继承主线程全部上下文认知。

这带来两个效果：

1. 主线程必须把需求讲清楚
2. `Plan` 更像“受控咨询器”，而不是主线程思维的延伸副本

这也解释了为什么运行时还会继续删 `claudeMd` 和 `gitStatus`：

- 它本来就不是来吃满上下文的
- 它是来低成本地产出规划结论的

---

## 11. embedded search 环境下，Plan 的搜索建议也会变

在 `getPlanV2SystemPrompt()` 里：`src/tools/AgentTool/built-in/planAgent.ts:14-20`

如果是 embedded search tools 环境：
- 它被引导使用 `find` / `grep` / `Read`

否则：
- 它被引导使用 `Glob` / `Grep` / `Read`

并且 Bash 只允许只读命令：`src/tools/AgentTool/built-in/planAgent.ts:47-48`

这说明 Plan 也不是绑定固定工具接口，而是一个随运行环境动态切换“推荐搜索手段”的抽象规划角色。

---

## 12. 反常识总结

### 反常识 1：Plan agent 不等于 plan mode
证据：`src/tools/AgentTool/built-in/planAgent.ts:77-83`

- 它甚至不能用 `ExitPlanMode`
- 也不能改文件
- 它只是主线程的只读规划顾问

### 反常识 2：它表面像有 Explore 同款工具配置，实际上拿到的是 `undefined`
证据：
- `tools: EXPLORE_AGENT.tools`：`src/tools/AgentTool/built-in/planAgent.ts:85`
- Explore 没定义 `tools`：`src/tools/AgentTool/built-in/exploreAgent.ts:64-83`

这是一个很隐蔽的实现细节。

### 反常识 3：`tools === undefined` 被当成 wildcard
证据：`src/tools/AgentTool/agentToolUtils.ts:162-172`

所以它不是精确白名单，而是宽权限 + denylist + prompt 约束。

### 反常识 4：它明明是规划 agent，却主动丢掉 `CLAUDE.md`
证据：
- 定义：`src/tools/AgentTool/built-in/planAgent.ts:88-90`
- 运行时：`src/tools/AgentTool/runAgent.ts:390-398`

注释里甚至写了：
- 如果需要 conventions，它可以自己读 `CLAUDE.md`
- 没必要默认塞进上下文

这说明系统优先级是 token 成本，而不是让它默认拿满规则。

### 反常识 5：它也会丢弃系统上下文里的旧 gitStatus
证据：`src/tools/AgentTool/runAgent.ts:400-410`

也就是说，系统认为：
- “主线程开局那份 gitStatus” 对 Plan 没价值，甚至可能误导
- 真要 git 信息，自己现场查

### 反常识 6：它不是一个“更聪明的大模型规划器”，而是普通 subagent runtime 上的一层角色包装
证据：`src/tools/AgentTool/runAgent.ts:500-518`

Plan 没有独立执行架构，核心差异主要来自：
- prompt
- denylist
- 上下文裁剪
- 输出模板

### 反常识 7：它和 Explore 一样是高频 one-shot 基础设施
证据：`src/tools/AgentTool/constants.ts:6-12`

这说明系统把它当作：
- 一次性产出报告的低成本组件
- 而不是持久协作线程

---

## 13. 一句话理解 Plan

如果只记一句话：

> Plan agent 本质上不是“计划模式本身”，而是一个一次性、只读、瘦上下文的架构规划子代理；它的能力边界主要由 prompt 和 denylist 塑造，而不是严格白名单权限模型。
