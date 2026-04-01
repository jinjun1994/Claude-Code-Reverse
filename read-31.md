## 第 31 站：`src/hooks/toolPermission/handlers/coordinatorHandler.ts`

### 这是什么文件

`src/hooks/toolPermission/handlers/coordinatorHandler.ts` 是权限系统里的**coordinator worker 自动化前置处理器**。

上一站 `interactiveHandler.ts` 看到的是：

```text
ask -> 先进入交互式审批，再让用户 / bridge / channel / hooks / classifier 并发竞态
```

而这一站处理的是另一种调度策略：

```text
ask -> 先顺序等待自动化检查
   -> 只有自动化都没解决时，才回落到 interactive dialog
```

所以最准确的一句话是：

```text
coordinatorHandler.ts = “先自动化、后打扰用户”的 ask 预处理器
```

---

### 文件虽然短，但它表达的是产品策略

位置：`src/hooks/toolPermission/handlers/coordinatorHandler.ts:16`

这个文件的注释很短，但很重要：

- 针对 coordinator worker
- hooks 和 classifier 先顺序 await
- 自动化若能解决，直接返回最终 PermissionDecision
- 否则返回 `null`，交给调用方继续落到 interactive dialog

也就是说，这个文件的重要性不在逻辑量，而在它明确表达了一条策略：

```text
coordinator worker 应尽量先靠自动化解决 ask，
只有自动化不能决定时，才真正打扰用户
```

---

### 第一部分：它的输入说明它站在 `PermissionContext` 之上工作

位置：`src/hooks/toolPermission/handlers/coordinatorHandler.ts:8`

参数包括：

- `ctx`
- `pendingClassifierCheck`
- `updatedInput`
- `suggestions`
- `permissionMode`

这说明它本身不重新发明：

- hooks 逻辑
- classifier 逻辑
- allow / deny 结果构造
- persist / logging / abort 等副作用

它只是决定：

```text
在 coordinator worker 这个角色下，自动化检查的执行顺序应该是什么
```

所以它更像调度器，而不是能力提供者。

---

### 第二部分：先跑 hooks，而且是同步前置等待

位置：`src/hooks/toolPermission/handlers/coordinatorHandler.ts:31`

第一步是：

- `await ctx.runHooks(permissionMode, suggestions, updatedInput)`

注释直接写了：

- hooks first
- fast
- local

这说明在 coordinator 流里，hooks 被视为：

```text
最优先、最低成本的自动化审批来源
```

这里和 interactive 分支的最大区别在于：

- interactive：dialog 已出现，hook 在后台竞态
- coordinator：先等 hook 结果，没解决再往后走

所以你应该把它记成：

```text
在 coordinator ask 中，hook 不是后台参与者，而是同步前置检查器
```

---

### 第三部分：再跑 classifier，作为第二层自动化

位置：`src/hooks/toolPermission/handlers/coordinatorHandler.ts:40`

如果 hook 没有解决，第二步才是 classifier：

- 只有开启 `BASH_CLASSIFIER` 才会尝试
- 调 `ctx.tryClassifier?.(pendingClassifierCheck, updatedInput)`

注释已经点明了它的定位：

- slow
- inference
- bash only

所以 coordinator 路径的成本分层非常清楚：

```text
local hook
  -> classifier inference
  -> interactive dialog
```

这就是典型的：

```text
能便宜解决就先便宜解决；
不能再升级到更贵的自动化；
都不行才打断用户
```

---

### 第四部分：返回值 `null` 的语义很关键

位置：`src/hooks/toolPermission/handlers/coordinatorHandler.ts:22`

这个函数返回的是：

- `PermissionDecision`
- 或 `null`

这里的 `null` 不是 deny，也不是 error。

它的真实语义是：

```text
自动化检查没有给出最终决定；
调用方请继续回落到 interactive dialog
```

所以它本质上是一个 gate：

```text
resolved -> 直接返回最终裁决
unresolved -> 继续走人工审批链
```

这就是为什么 `useCanUseTool.tsx` 里会先 await 它，再决定是否继续往 interactiveHandler 走。

---

### 第五部分：自动化失败也不会卡死 ask 流

位置：`src/hooks/toolPermission/handlers/coordinatorHandler.ts:47`

如果 hooks / classifier 抛异常，它会：

- `logError(...)`
- 然后继续返回 `null`

也就是说：

```text
自动化失败 ≠ 权限失败
自动化失败 = 放弃自动化，交给用户手动决定
```

这个策略很成熟，因为它明确把自动化定义成：

- 优化层
- 非唯一通道
- 绝不能成为单点阻塞

所以这一段最该记住的是：

```text
coordinatorHandler 的目标不是“必须自动化成功”，
而是“尽量少打扰用户，但绝不因为自动化异常而卡死权限流”
```

---

### 第六部分：它和 interactiveHandler 的本质区别是什么

现在把这两个文件并排看，差异会很清楚：

#### `interactiveHandler.ts`
```text
先展示审批入口
本地用户 / bridge / channel / hooks / classifier 并发竞态
```

#### `coordinatorHandler.ts`
```text
先顺序等待 hook -> classifier
都没结果再回落 interactive dialog
```

所以两者不是“谁更复杂”而已，而是两套不同的权限产品策略：

- main agent：更强调实时交互 + 多终端并发审批
- coordinator worker：更强调后台自动化优先，减少用户打断

---

### 读完这一站后，你应该抓住的 6 个事实

1. `coordinatorHandler.ts` 不是交互处理器，而是 coordinator worker 的自动化前置处理器。
2. 它的顺序是：先 hook，再 classifier，最后才回落人工 dialog。
3. 在 coordinator 路径里，hooks 是同步前置检查器，不是后台竞态者。
4. classifier 是第二层自动化，只在 hook 无法解决时才尝试。
5. 返回 `null` 的意思不是拒绝，而是“请继续走 interactive dialog”。
6. 自动化报错不会中断 ask 流，只会记录日志后回落人工决策。

---

### 现在把第 30-31 站串起来

```text
interactiveHandler.ts
  -> main agent 的 interactive ask 竞态协调器
coordinatorHandler.ts
  -> coordinator worker 的 automated-first ask 预处理器
```

所以你应该把它们记成一对对照组：

```text
interactive = 先给交互入口，再并发竞态
coordinator = 先顺序自动化，再决定是否需要交互
```

---

### 下一站建议

下一站最顺的是：

```text
src/hooks/toolPermission/handlers/swarmWorkerHandler.ts
```

因为现在你已经看懂了：

- main agent 的 interactive ask
- coordinator worker 的 automated-first ask

下一步最自然就是补齐第三种分支：

**当 ask 发生在 swarm / teammate worker 身上时，权限请求如何上送给 leader，并等待 leader 回应。**
