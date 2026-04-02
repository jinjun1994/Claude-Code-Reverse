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

### 元

问题：**这一站真正想解决的架构问题是什么？**

回答：这一站真正要说明的是，为什么模型采样后还需要一条纯代码注册的后处理 hook 通道。它为那些不适合放进 settings 的运行态逻辑，提供了一个顺序执行、错误不阻断的插口。

### 反

问题：**如果把这一站的设计反过来，会发生什么？**

回答：如果没有这种轻量 post-sampling 接口，所有采样后分析都只能硬塞回主循环或别的 hook 体系。那样既破坏边界，也让简单的后处理变得过度配置化。

### 空

问题：**跳出当前文件名，这一站背后更大的问题是什么？**

回答：更大的问题，是系统怎样给“内部编程扩展”留出位置，而不强迫一切都走用户配置。这个文件体现的是一条更低摩擦的 runtime 扩展路径。

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
