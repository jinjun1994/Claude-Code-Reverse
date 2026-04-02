## 第 149 站：Compact 系统（compact/ 目录，~4K）

### 这是什么

Compact 是对话上下文压缩系统——当 token 数接近上下文窗口限制时，调用一个分叉的 agent 对整个对话生成摘要，替换旧消息。 Claude Code 有四种压缩层：microcompact、reactive compact、auto compact、session memory compaction、manual compact。

---

### 四种压缩层级

| 类型 | 触发 | 代价 | 位置 |
|---|---|---|---|
| Microcompact | 每次 turn Phase 2 | 缓存编辑（无 API） | `microCompact.ts` |
| Auto Compact | token >= threshold（有效窗口 - 13K） | 1 次 API 调用 | `autoCompact.ts` |
| Reactive Compact | 413 prompt-too-long | API 调用 + 截断 | query.ts error recovery |
| Manual Compact | `/compact` 命令 | API 调用 + 更大的 buffer（3K token reserved） | `compact.ts` |

---

### compact.ts —— 完整对话压缩

**核心流程 `compactConversation()`**：
```
1. Pre-compact token 计数
2. 执行 PreCompact hooks（`trigger: auto | manual`）
3. forked-agent 压缩（复用主对话的 prompt cache）
4. 如果 compact 自己 prompt-too-long → 截断最老 API 轮次 → 重试（最多 MAX_PTL_RETRIES）
5. 清除 readFileState 缓存 + loadedNestedMemoryPaths
6. 并行生成 post-compact 附件
7. 替换消息为压缩摘要
8. 执行 PostCompact hooks
```

**PTL 重试机制**：
```ts
for (;;) {
  summaryResponse = await streamCompactSummary(...)
  summary = getAssistantMessageText(summaryResponse)
  if (!summary?.startsWith(PROMPT_TOO_LONG_ERROR_MESSAGE)) break

  // 自截断重试
  ptlAttempts++
  if (ptlAttempts <= MAX_PTL_RETRIES) {
    messagesToSummarize = truncateHeadForPTLRetry(...)
  } else {
    throw new Error(ERROR_MESSAGE_PROMPT_TOO_LONG)
  }
}
```

---

### 分组策略：按 API 轮次

```ts
export function groupMessagesByApiRound(messages: Message[]): Message[][]
```

替代了先前的人类轮次分组——现在每次 API 往返为一组。
这使 reactive compact 可以在单 prompt 的 agent 会话（SDK/CCR/eval）中运行。

注释中的设计权衡：
> 跟踪未解决的 tool_use ID 只在对话格式错误时有用——但在那种情况下它永远钉住 gate 关闭。我们让 API 轮次的边界触发；summarizer fork 的 `ensureToolResultPairing` 在 API 时间修复悬挂的 tu。

---

### Microcompact（无 API 缓存编辑）

**`NO_TOOLS_PREAMBLE` 是关键约束**：
```
CRITICAL: Respond with TEXT ONLY. Do NOT call any tools.
Tool calls will be REJECTED and will waste your only turn — you will fail the task.
```

放在 prompt 最前面——在 `4.6+` adaptive-thinking 模型上，模型有时尽管有更弱的 trailer instruction 仍然尝试工具调用。

**两种模式**：
- `BASE`——范围是"整个对话"
- `PARTIAL`——范围是"最近的消息"

`<analysis>` 块是草稿板，`formatCompactSummary()` 在摘要到达上下文前剥离它。

**COMPACTABLE_TOOLS**——只压缩这些工具的结果：
```ts
const COMPACTABLE_TOOLS = new Set([
  FILE_READ_TOOL_NAME,      // 文件读取
  ...SHELL_TOOL_NAMES,      // Bash/PowerShell
  GREP_TOOL_NAME, GLOB_TOOL_NAME,
  WEB_SEARCH_TOOL_NAME, WEB_FETCH_TOOL_NAME,
  FILE_EDIT_TOOL_NAME, FILE_WRITE_TOOL_NAME,
])
```

**`[Old tool result content cleared]`**——旧的工具结果被替换为占位文本，不是删除。保留消息结构（tool_use + tool_result 配对）但清除内容 token。

---

### Post-Compact 附件

Compact 后需要恢复一些上下文，通过附件而不是摘要文本：

```ts
POST_COMPACT_MAX_FILES_TO_RESTORE = 5         // 最多恢复 5 个文件
POST_COMPACT_TOKEN_BUDGET = 50_000            // 总 token 预算
POST_COMPACT_MAX_TOKENS_PER_FILE = 5_000      // 单文件上限
POST_COMPACT_MAX_TOKENS_PER_SKILL = 5_000     // 单 skill 上限
POST_COMPACT_SKILLS_TOKEN_BUDGET = 25_000     // skills 总预算
```

**Intentionally NOT resetting `sentSkillNames`**——skill 注入在 compact 后被截断。注释：
> 重新注入完整的 skill_listing（~4K tokens）在 compact 后是纯 cache_creation 收益微乎其微。模型的 SkillTool schema 里仍有，invoked_skills 附件保留了已用 skill 内容。

---

### Prompt Cache 共享

```ts
const promptCacheSharingEnabled = getFeatureValue_CACHED_MAY_BE_STALE(
  'tengu_compact_cache_prefix', true  // 3P 默认 true
)
```

Forked-agent 路径复用主对话的 prompt cache。

实验证明：`false` 路径 98% cache miss，成本约 0.76% 的 fleet cache_creation（~38B tok/day），集中在 ephemeral 环境（CCR/GHA/SDK）。

---

### Compact 提示词结构

**两变体**：
- `BASE`——"分析整个对话"
- `PARTIAL`——"分析最近的消息"

**`<analysis>` + `<summary>` 双块格式**——analysis 是思维草稿，最终只取 summary 块。

**Partial compact direction**——可以是"从头压缩"或"从尾压缩"。从头压缩保留最近上下文；从尾压缩保留早期上下文。

---

### 元

问题：**这一站真正想解决的架构问题是什么？**

回答：这一站把上下文压缩正式分成 microcompact、auto compact、reactive compact 与 manual compact，多层次地应对 token 窗口压力。重点不只是“压短”，而是何时压、压哪部分、失败时如何自截断重试。

### 反

问题：**如果把这一站的设计反过来，会发生什么？**

回答：如果压缩只有一种粗暴摘要方式，一到 prompt-too-long 就只能整体退让，既贵又容易丢结构。更糟的是 compact 自己也可能太长，反过来卡死在救火流程里。

### 空

问题：**跳出当前文件名，这一站背后更大的问题是什么？**

回答：更大的问题是，长对话系统怎样在不失忆的前提下持续变轻。149 站的答案是承认压缩本身需要分层治理，而不是一次总结打天下。

### 读完后应抓住的 5 个事实

1. **Compact PTL 自截断重试**——Compact 请求自己可能 prompt-too-long。不报错，而是截断最老的 API 轮次并重试（最多 MAX_PTL_RETRIES 次）。这是防御性的：compact 本身不应该失败挂住用户。

2. **按 API 轮次分组 vs 按人类轮次**——API 轮次是 API 安全的分割点——API 合约保证每个 tool_use 在下一个 assistant turn 之前已解决。这使 reactive compact 能在单 prompt agent 会话（SDK/CCR/eval）中运行，而不仅是多用户 prompt。

3. **Post-compact 附件不是全量**——最多 5 个文件、50K token 总预算、单文件 5K——不是全部恢复。Skill 注入被截断，不重注入完整 4K+ skill_listing（纯 cache_creation）。

4. **prompt cache 共享是默认开启**——forked-agent 复用主对话 prompt cache。`false` 路径 98% cache miss 浪费 38B tokens/day——这是一个真实的成本优化事件修复的。

5. **COMPACTABLE_TOOLS 白名单**——compact 只压缩"读取/搜索/写入"类工具的结果（Bash/Read/Grep/Glob/WebFetch/Edit/Write）。Agent/Team/SendMessage 等编排工具的结果不压缩——它们携带结构化信息。
