## 第 81 站：`src/utils/hooks/registerSkillHooks.ts`

### 这是什么文件

`src/utils/hooks/registerSkillHooks.ts` 是 hooks 子系统里的 **skill hook 注册器：负责把 skill frontmatter 中定义的 hooks 注册为 session-scoped hooks，并支持 `once: true` 一次性 hook（执行成功后自动移除）和 `skillRoot` 来源追踪。**

这个文件比 frontmatter hooks 更简单，但多了一个关键特性：

```text
once: true -> 执行成功后通过 onHookSuccess 回调自动移除
```

---

### 整体逻辑

位置：`src/utils/hooks/registerSkillHooks.ts:20`

函数参数包括：

- `setAppState` — 状态更新
- `sessionId` — 当前 session ID
- `hooks` — skill frontmatter 中的 hooks 配置
- `skillName` — skill 名称（日志用）
- `skillRoot` — skill 目录（设置 `CLAUDE_PLUGIN_ROOT` 环境变量）

工作流程：

1. 遍历 hooks (event -> matchers -> hooks)
2. 对 `once: true` 的 hook 构造 `onHookSuccess` 回调来执行后移除
3. 调用 `addSessionHook(...)` 注册，传入 `skillRoot`

---

### 最关键的特性：`once: true` 的自动清理

位置：`src/utils/hooks/registerSkillHooks.ts:35`

```ts
const onHookSuccess = hook.once
  ? () => {
      removeSessionHook(setAppState, sessionId, eventName, hook)
    }
  : undefined
```

这说明：

- `once: true` 的 hook 只会被执行一次
- 执行成功后通过 `onHookSuccess` 回调从 session store 中移除
- 这避免了 hook 在后续每个匹配事件中重复触发

这与 `sessionHooks.ts` 中看到的 `onHookSuccess` 机制直接配合。

---

### 与 frontmatter hooks 注册器的对比

| | frontmatter hooks | skill hooks |
|---|---|---|
| 来源 | CLAUDE.md / agent frontmatter | skill frontmatter |
| `once: true` | 不支持 | 支持（执行后自动移除） |
| `skillRoot` | 不传 | 传入（设置 env 变量） |
| Stop -> SubagentStop | 支持 | 不支持 |

这说明 skill hooks 被设计为：

- 可以是临时的一次性 hook
- 需要知道 skill 根目录位置
- 不需要 agent 场景的事件转换

---

### 读完这一站后，你应该抓住的 3 个事实

1. skill hooks 支持 `once: true`，执行成功后通过 `onHookSuccess` 回调自动从 session store 移除。
2. skill hooks 传入 `skillRoot`，用于运行时设置 `CLAUDE_PLUGIN_ROOT` 环境变量。
3. skill hooks 不做 Stop -> SubagentStop 转换，因为 skill 本身不是 agent。

---

### 下一站建议

剩余未读的 hook 相关文件：

- `src/utils/hooks/hooksConfigSnapshot.ts` — 配置快照
- `src/utils/hooks/fileChangedWatcher.ts` — 文件变更 watcher
- `src/utils/hooks/postSamplingHooks.ts` — 后采样 hooks
- `src/utils/hooks/skillImprovement.ts` — skill 改进
- `src/utils/hooks/apiQueryHookHelper.ts` — API 查询辅助

下一步应该读：

**`src/utils/hooks/hooksConfigSnapshot.ts` — hook 配置快照，在 hook 执行期间捕获当前配置状态用于比较和决策。**
