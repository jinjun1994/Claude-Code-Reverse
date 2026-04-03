# Explore agent 如何实现

## 结论先说

`Explore` 不是一个“硬编码只允许 Read/Grep/Glob 的只读搜索器”，而是一个**靠系统提示词 + 少量 denylist + 运行时上下文瘦身**塑造出来的“快速研究型子代理”。这点非常反常识。

---

## 1. 定义在哪里

核心定义在：`src/tools/AgentTool/built-in/exploreAgent.ts:13`

关键字段：

- `agentType: 'Explore'`：`src/tools/AgentTool/built-in/exploreAgent.ts:64`
- `disallowedTools`：`src/tools/AgentTool/built-in/exploreAgent.ts:67-73`
- `model`：`src/tools/AgentTool/built-in/exploreAgent.ts:76-78`
- `omitClaudeMd: true`：`src/tools/AgentTool/built-in/exploreAgent.ts:79-81`
- `getSystemPrompt()`：`src/tools/AgentTool/built-in/exploreAgent.ts:82`

它的系统提示词把自己定义为：

- “file search specialist”
- 只读
- 禁止创建/修改/删除文件
- Bash 只能用于只读命令
- 尽量并行 Grep/Read
- 最终直接回报，不允许自己落文件

对应内容在：`src/tools/AgentTool/built-in/exploreAgent.ts:24-56`

---

## 2. 怎么注册进系统

注册入口在：`src/tools/AgentTool/builtInAgents.ts:22-72`

流程：

1. `getBuiltInAgents()` 先加入通用内建 agent：`src/tools/AgentTool/builtInAgents.ts:45-48`
2. 如果 `areExplorePlanAgentsEnabled()` 为真，则把 `EXPLORE_AGENT` 和 `PLAN_AGENT` 一起加入：`src/tools/AgentTool/builtInAgents.ts:50-52`
3. `areExplorePlanAgentsEnabled()` 又受 feature gate 控制：`src/tools/AgentTool/builtInAgents.ts:13-19`

这里有个非直觉点：

- `Explore` 不是永远存在的；它是**功能开关控制的内建 agent**，不是静态必然可用的。

---

## 3. 怎么与其他 agent 合并

所有 agent 的统一装载逻辑在：`src/tools/AgentTool/loadAgentsDir.ts`

其中 `omitClaudeMd` 的类型注释非常关键：`src/tools/AgentTool/loadAgentsDir.ts:128-132`

这里明确写了：

- `Explore/Plan` 这类只读 agent 不需要把 `CLAUDE.md` 层级规则放进上下文
- 这样可以节省大量 token

也就是说，`Explore` 在定义层就已经被视为一种**高频、低上下文、读多写零**的 agent 类型。

---

## 4. 运行时如何启动

运行时主流程在：`src/tools/AgentTool/runAgent.ts:380-518`

关键步骤：

### 4.1 获取基础上下文
- `getUserContext()` / `getSystemContext()`：`src/tools/AgentTool/runAgent.ts:380-383`

### 4.2 根据 agent 配置裁剪用户上下文
- 如果 `agentDefinition.omitClaudeMd` 为真，并且没有 caller 显式覆盖 userContext，则删掉 `claudeMd`：`src/tools/AgentTool/runAgent.ts:385-398`

注释直接写明：
- 这是为了给 Explore/Plan 节省 token
- 主 agent 才是理解和解释结果的那个

### 4.3 裁剪系统上下文中的 gitStatus
- 对 `Explore/Plan`，会把 session-start 的 `gitStatus` 去掉：`src/tools/AgentTool/runAgent.ts:400-410`

原因也写得很直白：
- 这份 `gitStatus` 可能是陈旧的
- 如果真需要 git 信息，agent 自己跑一次 `git status` 更准确

### 4.4 解析真正可用的工具集
- `resolveAgentTools(...)`：`src/tools/AgentTool/runAgent.ts:500-502`

### 4.5 注入 Explore 专属 system prompt
- `getAgentSystemPrompt(...)`：`src/tools/AgentTool/runAgent.ts:508-518`

所以 Explore 运行时并不是“一个独立实现的搜索引擎”，而是：

- 同一套 agent 执行框架
- 加上 Explore 自己的配置
- 再加上更激进的上下文裁剪

---

## 5. 它和普通 general-purpose agent 的差异

普通 agent 定义在：`src/tools/AgentTool/built-in/generalPurposeAgent.ts:25-34`

### general-purpose 的特点
- `tools: ['*']`：`src/tools/AgentTool/built-in/generalPurposeAgent.ts:29`
- 默认模型继承父线程：`src/tools/AgentTool/built-in/generalPurposeAgent.ts:32`
- 提示词更宽松，只说“尽量别创建文件”：`src/tools/AgentTool/built-in/generalPurposeAgent.ts:15-16`

### Explore 的特点
- 没有显式 `tools: ['*']`
- 但有 `disallowedTools`
- 系统提示词极其严格，只读且禁止一切修改
- `omitClaudeMd: true`
- 对外部用户强制更偏向 `haiku`

---

## 6. 真正的工具权限模型

最关键逻辑在：`src/tools/AgentTool/agentToolUtils.ts:122-172`

### 6.1 先做全局过滤
`filterToolsForAgent(...)` 在这里：`src/tools/AgentTool/agentToolUtils.ts:70-116`

它会先排掉：
- 所有 agent 统一禁止的工具
- 某些自定义 agent 禁止工具
- async agent 不允许的工具

### 6.2 再应用 agent 自己的 disallowedTools
- `disallowedToolSet` 构造：`src/tools/AgentTool/agentToolUtils.ts:149-155`
- 实际过滤：`src/tools/AgentTool/agentToolUtils.ts:157-160`

### 6.3 如果 `tools` 未定义，则视为 wildcard
最反常识的一段在：`src/tools/AgentTool/agentToolUtils.ts:162-172`

代码语义是：

- 如果 `tools === undefined`
- 或者 `tools === ['*']`
- 那就视为“允许全部剩余工具”

而 Explore 恰好就是：
- **没有写 `tools`**
- 只写了 `disallowedTools`

这意味着 Explore 的真实权限模型不是：

- “只允许 Read/Grep/Glob/Bash”

而是：

- “先拿到 agent 可用工具全集”
- “再减去 `Agent / ExitPlanMode / Edit / Write / NotebookEdit`”
- “剩下的都可用”

对应 denylist：`src/tools/AgentTool/built-in/exploreAgent.ts:67-73`

---

## 7. 为什么它看起来像白名单，其实不是

因为系统提示词在强烈暗示：

- 用 `Glob`
- 用 `Grep`
- 用 `Read`
- Bash 只能用于只读命令

见：`src/tools/AgentTool/built-in/exploreAgent.ts:43-56`

所以表面上你会以为 Explore 是一个“专用搜索工具包 agent”。

但实现并不是靠精确白名单保证的，而是靠：

1. prompt 约束
2. denylist
3. 运行时框架的通用过滤

这就是它最值得记住的“反常识”。

---

## 8. 为什么系统还说它比直接 Grep/Glob 更慢

主提示词中有明确建议：

- 简单、定向搜索，直接用 `Glob/Grep/Read`
- 更广的代码探索，才用 `Agent(subagent_type=Explore)`

位置：`src/constants/prompts.ts:374-380`

尤其这句最重要：

- Explore **比直接工具更慢**

并且阈值还被写成常量：
- `EXPLORE_AGENT_MIN_QUERIES = 3`：`src/tools/AgentTool/built-in/exploreAgent.ts:59`

也就是：

- 如果预计 1~2 次搜索就能解决，别用 Explore
- 只有当问题需要多轮开放式探索时，Explore 才值得

这也很反常识：

- 一个“专门探索代码库”的 agent，系统竟然明确说它不是默认最佳搜索手段
- 它本质上是**复杂研究时的上下文隔离器**，不是更快的 grep 封装

---

## 9. 模型选择也不是统一的

Explore 的模型策略：`src/tools/AgentTool/built-in/exploreAgent.ts:76-78`

- 内部用户（`USER_TYPE === 'ant'`）：`inherit`
- 外部用户：`haiku`

注释还提到：
- 对内部用户，`getAgentModel()` 运行时还会看 `tengu_explore_agent` 实验开关

而普通 subagent 的默认模型是 `inherit`：`src/utils/model/agent.ts:21-27`

这说明：

- Explore 不是一个语义完全固定的 agent
- 它在不同部署环境下，实际模型策略不一样

反常识点：

- “同一个内建 agent” 对内外部用户表现并不完全一致

---

## 10. Explore 还是 one-shot agent

定义在：`src/tools/AgentTool/constants.ts:6-12`

`ONE_SHOT_BUILTIN_AGENT_TYPES` 里只有：
- `Explore`
- `Plan`

注释说得很明确：
- 这类 agent 跑一次就返回报告
- 父线程通常不会再 `SendMessage` 续聊
- 因此可以省掉结果尾部一些元数据提示，进一步节省 token

这说明 Explore 的产品定位并不是“持续协作的副驾驶”，而是：

- 一次性调查员
- 一次性吐出结论
- 快速回收

---

## 11. embedded search 环境下，行为会变

`hasEmbeddedSearchTools()` 会影响提示词内容：`src/tools/AgentTool/built-in/exploreAgent.ts:14-22`

如果是 embedded 环境：
- 不强调专用 `Glob/Grep`
- 改成让它通过 Bash 调 `find/grep`

也就是说，Explore 并不是绑定某组固定工具名字；它是一个**抽象的“探索角色”**，根据运行环境动态换推荐工具。

这也很容易误判。

---

## 12. 反常识总结

### 反常识 1：Explore 不是硬白名单 agent
实现真相：`src/tools/AgentTool/agentToolUtils.ts:162-172`

- `tools` 缺省 = wildcard
- 所以它不是“只有搜索工具”
- 它只是减掉少数写操作工具

### 反常识 2：它主要靠 prompt 管，不主要靠权限管
证据：`src/tools/AgentTool/built-in/exploreAgent.ts:24-56`

- 强只读行为更多来自提示词
- 权限层只是做兜底，不是精细封死

### 反常识 3：官方自己说它比直接 Glob/Grep 慢
证据：`src/constants/prompts.ts:374-380`

- 它不是默认搜索路径
- 是复杂调查场景的升级选项

### 反常识 4：它刻意被做成“瘦上下文”
证据：
- `omitClaudeMd`：`src/tools/AgentTool/built-in/exploreAgent.ts:79-81`
- 运行时删 `claudeMd`：`src/tools/AgentTool/runAgent.ts:385-398`
- 运行时删 `gitStatus`：`src/tools/AgentTool/runAgent.ts:400-410`

说明它的目标不是“理解全部上下文”，而是“低成本快速探索”。

### 反常识 5：它是高频基础设施，不只是一个功能点
证据都写在注释里：
- `runAgent.ts:387`
- `runAgent.ts:403`
- `constants.ts:8`

注释直接按周统计 token 节省量，说明它是超高频路径，微优化都值得做。

### 反常识 6：同名 agent 不同用户群体行为并不完全一样
证据：`src/tools/AgentTool/built-in/exploreAgent.ts:76-78`

- 内部 / 外部模型不同
- 实际体验并非严格同构

---

## 13. 一句话理解 Explore

如果用一句话概括：

> Explore agent 本质上不是“更强的 grep”，而是一个被精简上下文、强调只读、一次性返回结果的高频研究型子代理；它的约束更多来自 prompt 和运行时瘦身，而不是严格白名单权限模型。
