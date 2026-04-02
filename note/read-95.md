## 第 95 站：`src/tools/AgentTool/prompt.ts`

### 这是什么文件

`prompt.ts`（288 行）是 AgentTool 的工具描述（prompt）构建器：生成给 model 看的 Agent 工具说明，包括 agent 列表、fork 指导、示例、使用注意事项。

---

### Agent 列表注入策略

#### 两种方式
1. **Inline mode**（默认）：在工具描述中嵌入完整 agent 列表：
   ```
   Available agent types and the tools they have access to:
   - Explore: Search the codebase... (Tools: GlobTool, GrepTool, FileReadTool)
   - Plan: Plan implementation... (Tools: *)
   ```

2. **Attachment mode**（`shouldInjectAgentListInMessages() === true`）：工具描述中只显示占位符：
   ```
   Available agent types are listed in <system-reminder> messages in the conversation.
   ```
   实际列表由 `attachments.ts` 作为 `agent_listing_delta` 附件消息注入。

#### 为什么要 attachment mode？
> 动态 agent 列表占 fleet cache_creation token 的 ~10.2%：MCP 异步连接、/reload-plugins、或权限模式变化都会 mutating 列表 → 工具描述变化 → prompt 缓存完全失效。

通过 attachment 方式，工具 schema 描述保持 static，缓存不会因 agent 列表变化而失效。

---

### Fork 启用时的 Prompt 变化

当 `isForkSubagentEnabled()` 时，prompt 中有四处 fork 相关变化：

#### 1. "When to fork" 指导
```
## When to fork
- Research: fork open-ended questions
- Implementation: prefer to fork implementation work requiring >couple edits
Forks are cheap because they share your prompt cache.
```

#### 2. 两条关键规则
- **Don't peek.** 不要读取 `output_file` 路径来获取子代理进展——相信通知。
- **Don't race.** 不要在通知到达前编造结果——给状态，不做猜测。

#### 3. 示例替换
- Fork 示例：展示省略 `subagent_type` 的 fork 调用、用户中途询问的响应方式
- 普通示例：展示 `subagent_type` 指定的专用 agent 调用

#### 4. "When NOT to use" 省略
Fork 路径中不展示 "When NOT to use ${AGENT_TOOL_NAME} tool"——因为 fork 是推荐的默认行为之一。

---

### Coordinator 模式 Prompt

Coordinator 模式只获取 slim prompt（`shared` 部分），因为 coordinator 的系统提示已经覆盖了使用注意事项和示例。

---

### `getToolsDescription()`

根据 agent 的 `tools` 和 `disallowedTools` 字段生成人类可读的工具描述：
- 两者都有 → 过滤掉 disallowed
- 只有 allowlist → 显示 allowlist
- 只有 denylist → "All tools except X, Y, Z"
- 都没有 → "All tools"

---

### 环境差异化

Prompt 中的搜索工具提示根据 `hasEmbeddedSearchTools()` 结果变化：
- **嵌入式搜索存在** → 指向 `find via Bash` 和 `grep via Bash`
- **嵌入式搜索不存在** → 指向 `GlobTool`

这是为了 ant 构建（别名为嵌入式 bfs/ugrep）的差异化提示。

---

### 并发通知

对于非 Pro 用户，显示并发提示：
```
Launch multiple agents concurrently whenever possible...
```

---

### 元

问题：**这一站真正想解决的架构问题是什么？**

回答：这一站真正要解释的是，Agent 工具为什么还需要一份专门的 prompt 构建器。agent 列表、fork 指导、示例和使用边界，决定了模型看到的工具说明不能只是一句简短描述。

### 反

问题：**如果把这一站的设计反过来，会发生什么？**

回答：如果没有这层提示构建，模型虽然能调用 Agent 工具，却不知道何时该选哪类 agent，也不知道 fork 该怎样受控使用。这样工具能力越强，误用就越频繁。

### 空

问题：**跳出当前文件名，这一站背后更大的问题是什么？**

回答：更大的问题，是复杂工具如何通过说明文本被模型正确理解。prompt 构建器就是工具产品面对模型时的“说明书系统”。

### 读完这一站后，你应该抓住的 5 个事实

1. Agent 列表的注入策略有两个版本：inline（默认）和 attachment（gate-enabled）。Attachment 模式解决了 ~10.2% 的缓存失效问题——当 agent 列表变化时不会 bust 工具 schema 的 prompt 缓存。
2. Fork 路径的 prompt 不展示 "When NOT to use Agent tool"，因为 fork 本身被鼓励为默认行为——模型应该在不需要结果的上下文时 fork。
3. Prompt 中的 "Don't peek" 和 "Don't race" 规则非常关键：模型不能中途 Read 子代理的 output 文件，也不能在通知到达前编造结果。这两条是 agent 委托范式中最常见的 model 行为错误。
4. 搜索工具提示根据 `hasEmbeddedSearchTools()` 动态切换——ant 构建指向 Bash 中的 find/grep，非 ant 指向 GlobTool。这是为了适配不同的部署配置。
5. Coordinator 模式获取 slim prompt，因为 coordinator 有自己的系统提示来覆盖使用细节。普通模式获取完整 prompt，包含示例、注意事项、fork 指导。
