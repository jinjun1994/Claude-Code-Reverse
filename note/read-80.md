## 第 80 站：`src/utils/hooks/registerFrontmatterHooks.ts`

### 这是什么文件

`src/utils/hooks/registerFrontmatterHooks.ts` 是 hooks 子系统里的 **frontmatter hook 注册器：负责把 CLAUDE.md / instruction files / skill 的 frontmatter 中定义的 hooks 注册为 session-scoped hooks，并在 agent 场景下自动将 Stop hooks 转换为 SubagentStop。**

所以最准确的一句话是：

```text
registerFrontmatterHooks.ts = frontmatter hooks 注册桥：把文件级指令中的 hooks 配置注册为 session-local 内存态 hooks，并处理 agent context 下的 Stop -> SubagentStop 事件映射
```

---

### 整体逻辑

位置：`src/utils/hooks/registerFrontmatterHooks.ts:18`

这个函数只做三件事：

1. 遍历 `HooksSettings`（event -> matchers -> hooks）
2. 对每个 hook 调用 `addSessionHook(...)` 注册到 session store
3. 如果是 agent 场景，把 `Stop` 事件自动转换为 `SubagentStop`

参数包括：

- `setAppState` — 状态更新函数
- `sessionId` — session/agent/skill ID
- `hooks` — frontmatter 中的 hooks 配置
- `sourceName` — 日志友好来源名
- `isAgent` — 是否 agent 场景（用于 Stop 转换）

---

### 最关键的点是 `Stop -> SubagentStop` 转换

位置：`src/utils/hooks/registerFrontmatterHooks.ts:38`

```ts
let targetEvent: HookEvent = event
if (isAgent && event === 'Stop') {
  targetEvent = 'SubagentStop'
  logForDebugging(
    `Converting Stop hook to SubagentStop for ${sourceName}
     (subagents trigger SubagentStop)`
  )
}
```

这说明：

- 用户在 agent 指令文件中写 `Stop` hook
- 实际触发的是 `SubagentStop` 事件
- 所以这里必须做事件映射

否则用户在 agent frontmatter 里写的 `Stop` hooks 永远不会被触发。

---

### 架构值

这个文件建立了：

```text
frontmatter hooks
  -> session hooks (scoped to session/agent/skill ID)
  -> cleaned up when session/agent ends
```

所以 hook 的持久性层级又多了一个：

```text
persisted in settings.json (permanent)
persisted in CLAUDE.md frontmatter (file-scoped)
in-memory session hooks (temporary)
in-memory function hooks (runtime-only)
plugin registered hooks (extension-scoped)
```

---

### 元

问题：**这一站真正想解决的架构问题是什么？**

回答：这一站关心的是，文件级 frontmatter 里的 hooks 为什么不能直接让 runtime 自己到处读，而要先注册成 session-scoped hooks。它把静态声明转成会话内可消费的内存态规则，并处理 agent 场景下的事件映射。

### 反

问题：**如果把这一站的设计反过来，会发生什么？**

回答：如果 frontmatter hooks 不经注册桥直接散入执行层，文件来源、session 生命周期和 event 变换都会混在一起。这样一来，agent 特有的 Stop/SubagentStop 语义也会越来越难维护。

### 空

问题：**跳出当前文件名，这一站背后更大的问题是什么？**

回答：更大的问题，是声明式配置怎样进入运行时而不泄漏来源细节。注册桥的作用，就是把“来自文件”转换成“属于当前会话”。

### 读完这一站后，你应该抓住的 4 个事实

1. `registerFrontmatterHooks.ts` 只有 67 行，纯粹的前端配置 -> session hooks 注册桥。
2. Agent 场景下 `Stop` 事件自动转换为 `SubagentStop`，确保 frontmatter 中的 Stop hooks 在 agent 完成时被触发。
3. Hooks 被注册为 session-scoped，随 session 结束自动清理。
4. `hookCount` 日志记录注册了多少个 frontmatter hooks。

---

### 下一站建议

下一个相关文件：

**`src/utils/hooks/registerSkillHooks.ts` — skill 的 hook 注册器。**
