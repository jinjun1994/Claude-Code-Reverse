# general-purpose agent 与 Explore / Plan 对比

## 结论先说

`general-purpose` 才是 AgentTool 的**默认子代理**，而 `Explore` / `Plan` 更像是在同一套 subagent runtime 之上，额外包出来的两种“高频只读角色”。

三者最核心的区别，不在底层执行框架，而在：

- system prompt
- 是否裁剪上下文
- 是否 one-shot
- 默认模型策略
- 对用户的推荐使用场景

---

## 1. general-purpose 是默认兜底 agent

定义在：`src/tools/AgentTool/built-in/generalPurposeAgent.ts:25-34`

关键字段：

- `agentType: 'general-purpose'`：`src/tools/AgentTool/built-in/generalPurposeAgent.ts:25-27`
- `whenToUse`：`src/tools/AgentTool/built-in/generalPurposeAgent.ts:27-28`
- `tools: ['*']`：`src/tools/AgentTool/built-in/generalPurposeAgent.ts:29`
- `getSystemPrompt()`：`src/tools/AgentTool/built-in/generalPurposeAgent.ts:19-23`

Agent tool 的提示里明确写了：

- 如果没指定 `subagent_type`，默认就是 `general-purpose`
- 见 `src/tools/AgentTool/prompt.ts:208-212`

这意味着：

- `general-purpose` 不是普通 built-in 之一而已
- 它是整个 subagent 机制的默认落点

---

## 2. 它的 system prompt 更像“通用执行者”

内容在：`src/tools/AgentTool/built-in/generalPurposeAgent.ts:3-23`

提示词给它的定位是：

- 搜索代码
- 理解架构
- 调查复杂问题
- 执行 multi-step task

但它没有 Explore/Plan 那种强制只读声明。相反，它只是说：

- 非必要不要创建文件：`src/tools/AgentTool/built-in/generalPurposeAgent.ts:15-16`
- 文档文件不要主动创建，除非明确要求：`src/tools/AgentTool/built-in/generalPurposeAgent.ts:16`

也就是说：

- `general-purpose` 是“能研究，也能动手”的 worker
- `Explore` / `Plan` 是“只研究，不动手”的角色化变体

---

## 3. 三者底层 runtime 其实是同一套

运行时公共主流程在：`src/tools/AgentTool/runAgent.ts:380-518`

无论是：
- `general-purpose`
- `Explore`
- `Plan`

最终都要经过：

1. 获取 user/system context：`src/tools/AgentTool/runAgent.ts:380-383`
2. 处理上下文裁剪：`src/tools/AgentTool/runAgent.ts:385-410`
3. 解析工具集：`src/tools/AgentTool/runAgent.ts:500-502`
4. 生成 agent 专属 system prompt：`src/tools/AgentTool/runAgent.ts:508-518`

所以三者不是三套不同系统，而是一套框架下三种 profile。

---

## 4. 工具模型差异：表面不同，底层没想象中那么大

### general-purpose
- 显式写了 `tools: ['*']`：`src/tools/AgentTool/built-in/generalPurposeAgent.ts:29`

### Explore
- 不写 `tools`
- 只写 `disallowedTools`：`src/tools/AgentTool/built-in/exploreAgent.ts:67-73`

### Plan
- 写了 `tools: EXPLORE_AGENT.tools`：`src/tools/AgentTool/built-in/planAgent.ts:85`
- 但 Explore 实际没有 `tools` 字段，所以这里本质上也是 `undefined`

真正的解析逻辑在：`src/tools/AgentTool/agentToolUtils.ts:162-172`

- `tools === undefined` 或 `tools === ['*']`，都按 wildcard 处理

所以从底层结果看：

- `general-purpose`：全部工具
- `Explore`：几乎全部工具，减去 denylist
- `Plan`：几乎全部工具，减去 denylist

这就带来一个反常识结论：

> 三者在“底层可解析工具范围”上的差距，没有提示词读起来那么大；真正把它们区分开的，更多是行为提示和运行时上下文策略。

---

## 5. 只读约束主要只发生在 Explore / Plan

在 `runAgent.ts` 中，只有 `Explore` / `Plan` 会被特殊裁剪上下文：

### 裁剪 `claudeMd`
- `src/tools/AgentTool/runAgent.ts:385-398`

### 裁剪 `gitStatus`
- `src/tools/AgentTool/runAgent.ts:400-410`

条件明确写成：
- `agentType === 'Explore' || agentType === 'Plan'`

说明：
- `general-purpose` 保留更完整的上下文
- `Explore` / `Plan` 是刻意瘦身的高频只读 agent

这点非常关键。

---

## 6. 对用户的使用建议也不同

在 Agent tool prompt 中：`src/tools/AgentTool/prompt.ts:235-239`

系统先明确说：
- 读具体文件，不要起 agent
- 搜某个 class/function，不要起 agent
- 搜 2~3 个文件里的代码，不要起 agent

进一步，在 session guidance 里又明确说：
- 简单定向搜索，直接用搜索工具
- 更广的代码库探索和深度研究，才用 `Explore`
- 见 `src/constants/prompts.ts:374-380`

同时：
- 若不写 `subagent_type`，默认用 `general-purpose`：`src/tools/AgentTool/prompt.ts:208-212`

所以官方推荐心智模型其实是：

1. 最简单问题：直接工具
2. 开放式复杂问题：`general-purpose`
3. 明确是“代码探索/调研”：`Explore`
4. 明确是“实施方案设计”：`Plan`

---

## 7. 模型策略也不同

### general-purpose
- 未显式指定 model：`src/tools/AgentTool/built-in/generalPurposeAgent.ts:32-33`
- 因此走默认 subagent model，即 `inherit`：`src/utils/model/agent.ts:21-27`

### Explore
- 内部用户：`inherit`
- 外部用户：`haiku`
- `src/tools/AgentTool/built-in/exploreAgent.ts:76-78`

### Plan
- 显式 `model: 'inherit'`：`src/tools/AgentTool/built-in/planAgent.ts:87`

这说明：
- `Explore` 被设计成更强调速度和成本
- `Plan` 与 `general-purpose` 更强调继承主线程推理能力

---

## 8. one-shot 行为只属于 Explore / Plan

定义在：`src/tools/AgentTool/constants.ts:6-12`

- `ONE_SHOT_BUILTIN_AGENT_TYPES = { Explore, Plan }`

含义：
- 它们执行一次就返回报告
- 通常不会继续 `SendMessage`
- 系统还会为了省 token 省掉结果尾巴的一些内容

而 `general-purpose` 不在这个集合里。

这意味着：
- `general-purpose` 更适合持续交互、继续追问、扩展任务
- `Explore` / `Plan` 更像一次性咨询结果

---

## 9. 三者对比表

| 维度 | general-purpose | Explore | Plan |
|---|---|---|---|
| 默认性 | 是默认 subagent | 否 | 否 |
| 角色定位 | 通用执行者 | 代码搜索/调研专家 | 架构规划师 |
| 是否只读人设 | 否，偏灵活 | 是 | 是 |
| 明确禁止写文件 | 否，只是不鼓励 | 是 | 是 |
| 工具声明 | `['*']` | `tools` 未定义 + denylist | 实际等价于未定义 + denylist |
| 上下文裁剪 | 无特殊裁剪 | 裁剪 `claudeMd` / `gitStatus` | 裁剪 `claudeMd` / `gitStatus` |
| 默认模型 | inherit | 外部更偏 haiku | inherit |
| 是否 one-shot | 否 | 是 | 是 |
| 典型产出 | 执行结果或研究结果 | 搜索/调研结果 | 实施方案 + 关键文件 |

---

## 10. 最大的几个反常识

### 反常识 1：默认 agent 其实不是 Explore，而是 general-purpose
证据：`src/tools/AgentTool/prompt.ts:208-212`

很多人会以为不指定类型会更“聪明地挑一个专用 agent”，但实现不是。默认就是 `general-purpose`。

### 反常识 2：Explore / Plan 的特殊性主要不在底层权限，而在提示词和上下文瘦身
证据：
- prompt 定义：`src/tools/AgentTool/built-in/exploreAgent.ts:24-56`、`src/tools/AgentTool/built-in/planAgent.ts:21-70`
- context 裁剪：`src/tools/AgentTool/runAgent.ts:385-410`

### 反常识 3：Plan 看起来比 general-purpose “更高级”，但本质只是另一个角色模板
证据：`src/tools/AgentTool/runAgent.ts:500-518`

它没有独立执行引擎，没有独立 planner pipeline，本质还是同一 subagent runtime。

### 反常识 4：Explore / Plan 都比 direct tool 更重
证据：
- `src/tools/AgentTool/prompt.ts:235-239`
- `src/constants/prompts.ts:374-380`

系统明确鼓励：
- 简单搜索不用 agent
- 只有复杂、开放式研究才起 agent

### 反常识 5：看起来最“受限”的 Explore / Plan，底层并不是最窄白名单
证据：`src/tools/AgentTool/agentToolUtils.ts:162-172`

所以如果只看 UX 描述，很容易误判它们的真实技术边界。

---

## 11. 一句话理解三者关系

> `general-purpose` 是默认通用 worker，`Explore` 是瘦上下文的一次性调研员，`Plan` 是瘦上下文的一次性架构规划师；三者共用同一执行框架，差别主要来自 prompt、上下文裁剪和输出预期，而不是完全不同的技术实现。
