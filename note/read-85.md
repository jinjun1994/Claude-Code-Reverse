# 第 85 站：`src/utils/hooks/apiQueryHookHelper.ts`

## 这是什么文件

`src/utils/hooks/apiQueryHookHelper.ts` 是 hooks 子系统里的 **通用 LLM 查询 hook 工厂：定义了一个可复用的配置驱动模式，把 post-sampling hook 的常见套路（判是否运行 -> 构建消息 -> 调用模型 -> 解析响应 -> 记录结果）抽象为 `ApiQueryHookConfig<TResult>` + `createApiQueryHook(config)`。**

这个文件被 `skillImprovement.ts` 用作其 skill 改进检测的底层执行引擎。

---

## 核心抽象

### `ApiQueryHookConfig<TResult>`

位置：`apiQueryHookHelper.ts:16`

配置对象定义了 LLM 查询 hook 的所有可变部分：

| 字段 | 说明 |
|---|---|
| `name` | querySource 标识符 |
| `shouldRun(context)` | 判是否执行 |
| `buildMessages(context)` | 构建请求消息列表 |
| `systemPrompt?` | 可选系统提示覆盖 |
| `useTools?` | 是否使用上下文中工具（默认 true） |
| `parseResponse(content, context)` | 解析模型响应 |
| `logResult(result, context)` | 记录结果/副作用 |
| `getModel(context)` | 选择模型（lazy function 避免循环依赖） |

### `createApiQueryHook<TResult>(config)`

位置：`apiQueryHookHelper.ts:56`

返回 `async (context) => void`，正好符合 `PostSamplingHook` 签名。

执行流：

1. `shouldRun(context)` -> false 则跳过
2. `buildMessages(context)` -> 消息列表
3. `queryModelWithoutStreaming({ messages, systemPrompt, tools, ... })`
4. `extractTextContent(response.message.content)`
5. `parseResponse(content, context)` -> `TResult`
6. `logResult({ type: 'success', result, ... }, context)`
   或 `logResult({ type: 'error', error, ... }, context)`

---

## `ApiQueryResult<TResult>`

位置：`apiQueryHookHelper.ts:40`

两种结果：

```ts
{ type: 'success', queryName, result, messageId, model, uuid }
{ type: 'error', queryName, error, uuid }
```

每种都有 `uuid` 用于 analytics 跟踪。

---

## 为什么 `getModel` 是 function

位置：`apiQueryHookHelper.ts:37`

注释说：

```ts
// Must be a function to ensure lazy loading
// (config is accessed before allowed)
```

`getSmallFastModel()` 可能在模块加载时尚未初始化，
所以 `getModel` 必须延迟到运行时调用。

---

## `temperatureOverride: 0`

位置：`apiQueryHookHelper.ts:102`

所有通过此框架的 LLM 查询都使用 `temperatureOverride: 0`，
确保分析性/验证性任务的确定性输出。

---

## 架构值

这个文件是 hooks 子系统中的一种 "higher-level pattern"：

```text
createApiQueryHook(config)
  -> returns PostSamplingHook
  -> registerPostSamplingHook(hook)
  -> executed after model sampling
```

`skillImprovement.ts` 正是这个模式的使用者。

---

## 读完这一站后，你应该抓住的 6 个事实

1. `apiQueryHookHelper.ts` 定义了一个配置驱动的 LLM 查询 hook 工厂。
2. `ApiQueryHookConfig<TResult>` 把 LLM 查询的可变部分抽象为 8 个策略点。
3. `createApiQueryHook` 返回符合 `PostSamplingHook` 签名的函数，可以直接注册。
4. 所有结果都有 `uuid` 用于 analytics 跟踪。
5. `getModel` 必须是函数形式以避免模块加载时的循环依赖/未初始化问题。
6. 所有查询都使用 `temperatureOverride: 0` 确保确定性。

---

## Hooks 子系统完整阅读完成

至此，已经阅读完 `src/utils/hooks/` 下的全部 18 个文件：

| 站号 | 文件 | 一句话 |
|---|---|---|
| 67 | `hooks.ts` | hook 运行时执行中枢 |
| 68 | `types/hooks.ts` | hook 运行时协议类型 |
| 69 | `schemas/hooks.ts` | hook 持久化配置 schema |
| 70 | `hooksConfigManager.ts` | 配置管理视图与 event/matcher 分组 |
| 71 | `hooksSettings.ts` | 多来源 hooks 标准化与展示辅助 |
| 72 | `sessionHooks.ts` | session-scoped 内存态 hook 注册与投影 |
| 73 | `AsyncHookRegistry.ts` | async hook 进程跟踪与轮询 |
| 74 | `hookEvents.ts` | 生命周期事件广播 |
| 75 | `execHttpHook.ts` | HTTP hook 安全 POST 执行 |
| 76 | `execPromptHook.ts` | LLM prompt hook 单次判定 |
| 77 | `execAgentHook.ts` | agentic verifier 多轮验证 |
| 78 | `hookHelpers.ts` | 共享辅助函数 |
| 79 | `ssrfGuard.ts` | DNS 级 SSRF 防护 |
| 80 | `registerFrontmatterHooks.ts` | frontmatter hook 注册 |
| 81 | `registerSkillHooks.ts` | skill hook 注册 |
| 82 | `hooksConfigSnapshot.ts` | 配置快照与治理策略 |
| 83 | `fileChangedWatcher.ts` | 动态文件变更监控 |
| 84 | `skillImprovement.ts` | skill 自学习闭环 |
| 85 | `apiQueryHookHelper.ts` | 通用 LLM 查询 hook 工厂 |
| 83 | `postSamplingHooks.ts` | 采样后回调钩子（注：更早读的） |
