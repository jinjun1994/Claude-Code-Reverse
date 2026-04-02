## 第 116 站：`src/tools/ScheduleCronTool/`（5 个文件，~28K）

### 这是什么

ScheduleCronTool 是定时任务调度系统——包含三个工具（CronCreate、CronDelete、CronList）。与 AgentTool 的同步 fork 不同，Cron 是一个事件驱动的定时器：当 REPL 空闲时，将预定义的 prompt 重新排队到查询循环。这是模型主动设置定时提醒、定期检查任务的机制。

---

### 三个工具：Create / Delete / List

### CronCreate — 创建定时任务

```ts
{
  cron: string       // 5 字段 cron: "M H DoM Mon DoW"（本地时间）
  prompt: string     // 定时触发的 prompt
  recurring?: boolean  // false = 一次性（触发后自动删除）
  durable?: boolean    // true = 持久化到 .claude/scheduled_tasks.json
}
```

### CronDelete — 删除定时任务

```ts
{ id: string }   // CronCreate 返回的 job ID
```

### CronList — 列出所有定时任务

```ts
{}   // 无输入
```

---

### 验证链

#### CronCreate

```ts
// 1. Cron 表达式语法验证
if (!parseCronExpression(input.cron)) → error

// 2. 未来一年内是否有匹配日期？
if (nextCronRunMs(input.cron, Date.now()) === null) → error

// 3. 最大任务数限制
const tasks = await listAllCronTasks()
if (tasks.length >= MAX_JOBS) → error  // MAX_JOBS = 50

// 4. Te 不允许
// 5. 队友不能创建 durable cron
if (input.durable && getTeammateContext()) → error
```

#### CronDelete

```ts
// 队友只能删除自己的 cron
if (ctx && task.agentId !== ctx.agentId) → error
```

#### CronList

```ts
// 队友只看自己的 cron，team lead 看所有
const tasks = ctx ? allTasks.filter(t => t.agentId === ctx.agentId) : allTasks
```

---

### 持久化机制

```ts
const effectiveDurable = durable && isDurableCronEnabled()
const id = await addCronTask(cron, prompt, recurring, effectiveDurable, agentId)
setScheduledTasksEnabled(true)
```

调用后启用调度器——`setScheduledTasksEnabled(true)` 设置全局 flag，`useScheduledTasks` hook 会在下次 tick 时开始轮询。

**In-memory vs 持久化**：
- **Session-only（默认）**：任务只在内存中，CLaude 退出时消失
- **Durable**：任务写入 `.claude/scheduled_tasks.json`，下次启动自动恢复

---

### 运行时行为

#### REPL 空闲时执行

```
Jobs only fire while the REPL is idle (not mid-query).
```

Cron 不会中断正在进行的模型响应——只有 REPL 空闲（等待用户输入或模型完成）时才触发。这防止定时任务打断长推理。

#### 确定性 Jitter

```
Recurring tasks fire up to 10% of their period late (max 15 min)
One-shot tasks landing on :00 or :30 fire up to 90s early
```

每次触发有小的确定性偏移——这是为了避免所有用户在同一时刻触发（集群效应）。模型被引导选择非 `:00`/`:30` 的分钟来进一步分散负载。

#### 自动过期

```
Recurring tasks auto-expire after 7 days — they fire one final time, then are deleted.
```

重复任务最多运行 7 天——这是防止无限循环任务永远运行的安全保护。

---

### 特性门

```ts
// 主调度
isKairosCronEnabled() {
  return feature('AGENT_TRIGGERS')
    ? !isEnvTruthy(process.env.CLAUDE_CODE_DISABLE_CRON) &&
        getFeatureValue_CACHED_WITH_REFRESH('tengu_kairos_cron', true, 5min)
    : false
}

// Durable 持久化
isDurableCronEnabled() {
  return getFeatureValue_CACHED_WITH_REFRESH('tengu_kairos_cron_durable', true, 5min)
}
```

默认 `true`——`/loop` 技能是 GA 的（已在 changelog 中宣布）。GrowthBook gate 是 fleet-wide kill switch，不是针对单个用户。`CLAUDE_CODE_DISABLE_CRON` 是本地开发覆盖。

---

### Prompt 设计：引导模型避开 :00/:30

Prompt 中明确引导模型：

```
Every user who asks for "9am" gets `0 9`, every user who asks for "hourly" gets `0 *`
— requests from across the planet land on the API at the same instant.
When the user's request is approximate, pick a minute that is NOT 0 or 30:
  "every morning around 9" → "57 8 * * *" or "3 9 * * *" (not "0 9 * * *")
```

这是 Anthropic API 调用量分散策略——通过引导模型使用随机分钟，减少所有用户同一时刻同时触发 API 的峰值。

---

### 读完后应抓住的 5 个事实

1. **三个工具形成 CRUD 接口**：CronCreate（创建）、CronDelete（取消）、CronList（查看）。这是完整的定时任务管理——模型可以自由创建和取消定时任务。

2. **REPL 空闲时执行**：Cron 不会中断正在进行的对话——只有 REPL 空闲时才触发。Jitter 机制添加确定性移（最多 10% 周期，最长 15 分钟）来分散 API 调用。

3. **Durable vs Session 持久化**：Durable 模式任务写入 `.claude/scheduled_tasks.json`，下次启动自动恢复。Session-only 任务只在内存中，进程退出消失。默是 session-only——需要用户明确要求持久化时才用 durable。

4. **7 天自动过期**：重复任务在 7 次后自动删除。这是防止无限循环的边界控制——模型和用户在 7 天内会注意到并决定是否继续。

5. **Prompt 引导避开 :00/:30**：这是一个精细的 API 成本优化——通过引导模型选择非整点的分钟，Anthropic 可以减少所有用户同时触发的并发峰值。这不只是 UX 指导，而是基础设施保护机制。
