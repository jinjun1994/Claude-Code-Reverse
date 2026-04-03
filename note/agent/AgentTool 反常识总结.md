# AgentTool 反常识总结

## 结论先说

如果只看 Claude Code 对外表现，你很容易把 AgentTool 理解成“起一个专用小 agent 去帮忙”。但从实现看，它真正的核心不是“多开一个模型线程”，而是：

- 用统一 runtime 装配不同 agent profile
- 用 prompt、上下文裁剪、权限策略和生命周期机制，塑造出不同 agent 的行为边界

下面只保留最容易看错、最值得记住的点。

---

## 1. 不写 `tools`，往往不是更少，而是更多

最反直觉的实现：`src/tools/AgentTool/agentToolUtils.ts:162-172`

规则是：
- `tools === undefined` → wildcard
- `tools === ['*']` → wildcard

所以：
- 你以为“这个 agent 没写 tools，应该很受限”
- 实际却可能是“它拿到所有剩余可用工具”

这直接影响对 `Explore` 和 `Plan` 的理解。

---

## 2. Explore / Plan 看起来像严格白名单，实际上主要靠 prompt 管

`Explore` 和 `Plan` 都有很强的只读人设：
- `src/tools/AgentTool/built-in/exploreAgent.ts:24-56`
- `src/tools/AgentTool/built-in/planAgent.ts:21-70`

但它们底层更接近：
- wildcard 工具集合
- 再减去一小部分 denylist

也就是说，它们的真正约束主要来自：
1. system prompt
2. `disallowedTools`
3. 运行时通用过滤

而不是“严格白名单硬封死”。

---

## 3. `Plan agent` 不等于 plan mode

这点名字最容易误导。

证据：`src/tools/AgentTool/built-in/planAgent.ts:77-83`

`Plan` 自己就禁用了：
- `ExitPlanMode`
- `Edit`
- `Write`

所以它不是“进入计划模式后负责执行计划流程的 agent”，而只是：
- 只读调研者
- 方案生成器
- 主线程的规划顾问

真正的 plan mode 控制权还在主线程工具层。

---

## 4. 默认 agent 不是最专用的，而是 `general-purpose`

证据：`src/tools/AgentTool/prompt.ts:208-212`

当你不写 `subagent_type` 时，默认并不会：
- 自动挑最适合任务的 built-in agent
- 自动路由到 Explore / Plan

而是直接用：
- `general-purpose`

这说明系统的默认哲学是：
- 先给一个通用 worker
- 专用 agent 由主线程显式选择

---

## 5. 官方自己明确说：很多搜索场景不该起 agent

证据：
- `src/tools/AgentTool/prompt.ts:235-239`
- `src/constants/prompts.ts:374-380`

系统明确建议：
- 读具体文件：直接 `Read`
- 定向查代码：直接 `Glob/Grep`
- 简单搜索：不要起 `Explore`

也就是说，AgentTool 不是默认工作入口；很多时候直接工具更优。

这很反常识，因为从产品名上看，人们往往会以为 agent 更“高级”，应该优先用。

---

## 6. `Explore` 不是“更快的 grep”，而是“更贵的上下文隔离器”

官方明确说 `Explore` 比 direct tool 慢：`src/constants/prompts.ts:374-380`

这意味着它真正的价值不是：
- 替代 `Grep`
- 替代 `Read`

而是：
- 把开放式探索任务外包出去
- 隔离上下文污染
- 用一次性子线程回收研究结果

所以 `Explore` 更像“复杂调研容器”，不是“更聪明的搜索命令”。

---

## 7. built-in agents 不是固定全集，而是运行时拼出来的

证据：`src/tools/AgentTool/builtInAgents.ts:22-72`

built-in agents 会受这些条件影响：
- feature gate
- entrypoint 类型
- 是否 SDK 模式
- 实验开关

所以“有哪些 built-in agents”不是静态答案，而是环境相关答案。

---

## 8. prompt.ts 不只是文案，它在做缓存优化和行为引导

证据：`src/tools/AgentTool/prompt.ts:190-199`

这里有个很容易忽略的点：
- agent 列表有时不直接内联在 tool description 里
- 而是放进 attachment message

原因不是业务逻辑，而是：
- 避免 prompt cache 因 agent / MCP / permission 变化频繁失效

所以 `prompt.ts` 不是简单的帮助文案，它同时承担：
- 主线程行为引导
- prompt cache 稳定化

---

## 9. runAgent.ts 的重点不是“发请求给模型”，而是“拼出 agent 专属执行环境”

证据：`src/tools/AgentTool/runAgent.ts:380-598`

它做的大量事情不是推理本身，而是运行时装配：
- 取 user/system context
- 按 agent 裁剪 context
- 覆盖 permission mode
- 处理 prompt 弹窗策略
- 解析工具集
- 注入 system prompt
- 注册 hooks
- 预加载 skills

这说明 agent 差异很多都不是模型推理差异，而是环境差异。

---

## 10. `omitClaudeMd` 暴露了一个核心原则：不是所有 agent 都值得吃满上下文

证据：
- 定义：`src/tools/AgentTool/loadAgentsDir.ts:128-132`
- 运行时：`src/tools/AgentTool/runAgent.ts:385-398`

`Explore` / `Plan` 会默认丢掉 `CLAUDE.md` 上下文。

这背后反映的是一个设计取向：
- 不是“信息越多越好”
- 而是“为任务匹配最合适的上下文密度”

这和很多人对 LLM 系统“尽量塞满上下文”的直觉正好相反。

---

## 11. 系统宁愿让 agent 现场重查，也不愿把陈旧状态塞给它

证据：`src/tools/AgentTool/runAgent.ts:400-410`

对 `Explore/Plan`，系统会主动丢掉 session-start 的 `gitStatus`。

原因写得很直白：
- 那份状态可能已经 stale
- 真需要的话，agent 自己跑一次 `git status`

反常识点在于：
- 系统不是尽量多传上下文
- 而是宁可减少上下文，让 agent 去拿“更新鲜但更贵”的信息

---

## 12. one-shot built-in 只有 Explore / Plan，说明系统把它们当高频一次性组件

证据：`src/tools/AgentTool/constants.ts:6-12`

只有两个 one-shot built-in：
- `Explore`
- `Plan`

这说明它们的产品定位是：
- 一次性研究/规划结果生成器
- 不是持续协作 worker

也说明系统对它们做的 token 优化，不是偶然补丁，而是基础定位决定的。

---

## 13. 最严格白名单的，反而不是 Explore / Plan

从直觉上你会以为：
- 越“专用”的 agent，工具越严格

但实际最窄白名单是：
- `statusline-setup`：`src/tools/AgentTool/built-in/statuslineSetup.ts:138`
- `claude-code-guide`：`src/tools/AgentTool/built-in/claudeCodeGuideAgent.ts:103-116`

反而：
- `Explore`
- `Plan`
- `verification`

更多是 denylist + prompt 约束。

这说明“专用 agent”内部其实分两类：
- 流程型 / 配置型：真白名单
- 研究型 / 判断型：宽工具面 + 强提示词

---

## 14. verification 不是“跑测试 agent”，而是“反驳实现者”的 agent

证据：`src/tools/AgentTool/built-in/verificationAgent.ts:10-13`

它的定位不是：
- 替你执行几条检查
- 顺便确认一下没问题

而是：
- 专门寻找最后 20% 的失败点
- 对 happy path 保持怀疑
- 必须给结构化证据和最终 verdict

这说明 built-in agents 里有些是“能力 agent”，有些已经是“组织流程里的制度角色”。

---

## 15. 一句话压缩全部反常识

> AgentTool 的真实设计不是“每个 agent = 一份 prompt + 一组工具”，而是“每个 agent = 一种运行时 profile”；很多看起来像权限边界的东西，实际是 prompt 边界，很多看起来像静态配置的东西，实际是运行时装配结果。
