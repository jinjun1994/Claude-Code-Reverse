## 第 96 站：`src/tools/AgentTool/builtInAgents.ts`

### 这是什么文件

`builtInAgents.ts`（73 行）是内置 agent 列表的加载入口——根据 feature flag、运行模式和 entrypoint，动态返回当前会话可用的内置 agent 定义列表。

---

### 内置 Agent 列表

| Agent | 触发条件 |
|---|---|
| `GENERAL_PURPOSE_AGENT` | 始终启用 |
| `STATUSLINE_SETUP_AGENT` | 始终启用 |
| `EXPLORE_AGENT` | `BUILTIN_EXPLORE_PLAN_AGENTS` && `tengu_amber_stoat` |
| `PLAN_AGENT` | 同上 |
| `CLAUDE_CODE_GUIDE_AGENT` | 非 SDK entrypoint |
| `VERIFICATION_AGENT` | `VERIFICATION_AGENT` && `tengu_hive_evidence` |

---

### 优先级顺序

```ts
1. CLAUDE_AGENT_SDK_DISABLE_BUILTIN_AGENTS + non-interactive → 返回空列表
2. COORDINATOR_MODE → 返回 getCoordinatorAgents()（覆盖所有内置 agent）
3. 默认列表：
   3a. GENERAL_PURPOSE_AGENT + STATUSLINE_SETUP_AGENT（始终）
   3b. EXPLORE + PLAN（feature gated）
   3c. CLAUDE_CODE_GUIDE（非 SDK）
   3d. VERIFICATION（feature gated）
```

---

### Explore/Plan Agent 的实验设计

```ts
areExplorePlanAgentsEnabled() {
  if (feature('BUILTIN_EXPLORE_PLAN_AGENTS')) {
    // 3P default: true — A/B test treatment sets false to measure impact
    return getFeatureValue_CACHED_MAY_BE_STALE('tengu_amber_stoat', true)
  }
  return false
}
```

这是一个 A/B 测试——control 组默认启用 Explore/Plan agent（与 pre-experiment 行为一致），treatment 组设置为 false 以测量移除这些 agent 的影响。

---

### Coordinator 模式覆盖

在 coordinator 模式下，`getBuiltInAgents()` 不返回内置列表，而是返回 coordinator 专属的 worker agent 列表：

```ts
if (feature('COORDINATOR_MODE') && isEnvTruthy(CLAUDE_CODE_COORDINATOR_MODE)) {
  return getCoordinatorAgents()  // 通过 dynamic require 打破循环依赖
}
```

使用 `require()` 延迟加载而非顶层 import，因为这会引入 `coordinator/workerAgent.js` 的依赖，否则会创建 `tools.ts → AgentTool → builtInAgents → coordinator → tools.ts` 的循环。

---

### SDK Entry point 过滤

`CLAUDE_CODE_GUIDE_AGENT` 通过 entrypoint 环境变量控制：

```ts
const isNonSdkEntrypoint =
  process.env.CLAUDE_CODE_ENTRYPOINT !== 'sdk-ts' &&
  process.env.CLAUDE_CODE_ENTRYPOINT !== 'sdk-py' &&
  process.env.CLAUDE_CODE_ENTRYPOINT !== 'sdk-cli'
```

这个 agent 只在 CLI/interactive 模式下可用——SDK 用户不需要通过代码引导来学习 Claude Code 的使用。

---

### 元

问题：**这一站真正想解决的架构问题是什么？**

回答：这一站关心的是，内置 agent 列表为什么要按 feature flag、运行模式和入口动态生成。它说明“有哪些内置 agent”不是固定常量，而是会话条件的一部分。

### 反

问题：**如果把这一站的设计反过来，会发生什么？**

回答：如果内置 agents 被写死，那么多仅在特定模式或入口有效的 agent 会在错误场景暴露出来。那样表面上是列表更完整，实际上是能力边界更含混。

### 空

问题：**跳出当前文件名，这一站背后更大的问题是什么？**

回答：更大的问题，是平台能力如何随运行环境裁剪而不破坏整体心智。动态内置列表正是这种条件化暴露的体现。
