## 第 83 站：`src/utils/hooks/postSamplingHooks.ts`

### 这是什么文件

`src/utils/hooks/postSamplingHooks.ts` 是 hooks 子系统里的 **model 采样后回调钩子：定义了一个简单的编程接口，在 model 采样完成后被顺序调用。这类 hooks 不走 `settings.json` 或 `schemas/hooks.ts`，只在代码内部编程注册，因此属于纯 runtime hook 层。**

文件只有 70 行，非常简单。

---

### 整体结构

#### 类型定义

```ts
REPLHookContext = {
  messages,        // 完整消息历史（含 assistant 响应）
  systemPrompt,
  userContext,
  systemContext,
  toolUseContext,
  querySource?
}

PostSamplingHook = (context: REPLHookContext) => Promise<void> | void
```

#### 注册

```ts
const postSamplingHooks: PostSamplingHook[] = []
registerPostSamplingHook(hook)  // 推入数组
```

#### 执行

```ts
executePostSamplingHooks(context)  // 顺序 await 每个 hook
```

---

### 关键特点

1. **不是配置驱动的** — 注释说 `not exposed in settings.json config (yet), only used programmatically`
2. **fire-and-forget 语义** — 错误只被 log，不阻断后续 hooks
3. **有完整的 REPL 上下文** — messages、systemPrompt、user/system context、toolUseContext
4. **顺序执行** — 按注册顺序 await

---

### REHLHookContext 的用途

`REPLHookContext` 这个类型名也用于其他 "repl hooks" 场景，
因为它包含完整的主循环上下文，包括：
- 消息历史
- 系统提示
- 查询来源

这说明 post-sampling hooks 能看到 model 采样后的完整状态。

---

### 读完这一站后，你应该抓住的 4 个事实

1. post-sampling hooks 是纯编程接口，不通过 settings 配置。
2. 注册方式是简单 `push` 到内部数组。
3. 执行时顺序 await 每个 hook，错误只被记录不阻断。
4. `REPLHookContext` 包含完整的 model 采样后上下文——消息、系统提示、user/system context、toolUseContext。

---

### 下一站建议

剩余文件：
- `src/utils/hooks/skillImprovement.ts` — skill 改进
- `src/utils/hooks/apiQueryHookHelper.ts` — API 查询辅助

继续读 `skillImprovement.ts`。
