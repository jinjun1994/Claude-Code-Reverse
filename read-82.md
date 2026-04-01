## 第 82 站：`src/utils/hooks/hooksConfigSnapshot.ts`

### 这是什么文件

`src/utils/hooks/hooksConfigSnapshot.ts` 是 hooks 子系统里的 **配置快照与 hook 治理策略层：负责捕获/更新 hooks 配置快照，并集中定义三类 hook 治理策略（`disableAllHooks`、`allowManagedHooksOnly`、`isRestrictedToPluginOnly`）的求值和优先级。**

这个文件回答了三个关键问题：

1. 哪些来源的 hooks 被允许在当前 session 使用
2. 配置变更时怎样刷新快照
3. 治理策略之间的优先级和交叉关系

---

### `getHooksFromAllowedSources()` 是核心治理逻辑

位置：`hooksConfigSnapshot.ts:18`

决策顺序：

1. `policySettings.disableAllHooks === true` -> `{}` 全部禁用
2. `policySettings.allowManagedHooksOnly === true` -> 仅 policy 的 hooks
3. `isRestrictedToPluginOnly('hooks')` -> 仅 policy 的 hooks（plugin-only 模式下 user/project/local settings 被屏蔽）
4. `mergedSettings.disableAllHooks === true` -> 仅 policy 的 hooks
5. 否则：正常合并所有来源

---

### `shouldAllowManagedHooksOnly()`

返回 `true` 当：
- policy 设置 `allowManagedHooksOnly: true`
- 或非 policy 设置 `disableAllHooks: true`（非 Managed 的 hook 被禁用，managed 仍然运行，等效 managed-only）

---

### `shouldDisableAllHooksIncludingManaged()`

仅当 `policySettings.disableAllHooks === true` 时才返回 `true`。

这说明：

```text
禁用所有 hooks（包括 managed）的权限只有 policy settings 有
```

---

### 快照 API

- `captureHooksConfigSnapshot()` — 启动时调用
- `updateHooksConfigSnapshot()` — 配置修改时调用，先 `resetSettingsCache()` 再重新捕获
- `getHooksConfigFromSnapshot()` — 获取，未捕获则先自动捕获
- `resetHooksConfigSnapshot()` — 测试清桩

---

### 读完这一站后，你应该抓住的 5 个事实

1. `policySettings.disableAllHooks` 是唯一能禁用所有 hooks（包括 managed）的方式。
2. `disableAllHooks` 在非 policy settings 中只影响 user/project/local，managed hooks 仍运行 —— 非 managed 设置不能覆盖 managed hooks。
3. `updateHooksConfigSnapshot()` 先 `resetSettingsCache()` 再重新捕获，避免 file watcher 稳定性阈值未到时使用 stale 缓存。
4. `isRestrictedToPluginOnly('hooks')` 屏蔽 user/project/local settings hooks，但不影响 plugin registered hooks（走另一条路径）。
5. 快照文件只 133 行，核心就是一个全局 `initialHooksConfig` 变量加上治理策略求值。

---

继续下一站 `fileChangedWatcher.ts`。
