## 第 84 站：`src/utils/hooks/skillImprovement.ts`

### 这是什么文件

`src/utils/hooks/skillImprovement.ts` 是 hooks 子系统与 skill 系统的交叉反馈层：通过 post-sampling hook 每 N 轮对话用 LLM 分析用户在与 skill 交互中的偏好、请求和修正，生成 `SkillUpdate[]` 存入 app state，之后可由 `applySkillImprovement()` fire-and-forget 地调用另一个 LLM 把改进直接写回 skill 文件。

---

### 核心机制

#### 1. 检测：每 N 轮分析一次

位置：`skillImprovement.ts:75`

```ts
const TURN_BATCH_SIZE = 5

async shouldRun(context) {
  // 只在主 REPL 线程运行
  if (context.querySource !== 'repl_main_thread') return false

  // 需要有已调用的 project skill
  if (!findProjectSkill()) return false

  // 每 5 条 user message 才分析一次
  const userCount = count(context.messages, m => m.type === 'user')
  if (userCount - lastAnalyzedCount < TURN_BATCH_SIZE) return false

  lastAnalyzedCount = userCount
  return true
}
```

#### 2. 构建 prompt

位置：`skillImprovement.ts:94`

```ts
buildMessages(context) {
  const projectSkill = findProjectSkill()!
  const newMessages = context.messages.slice(lastAnalyzedIndex)
  lastAnalyzedIndex = context.messages.length

  return [createUserMessage({ content: `
You are analyzing a conversation where a user is executing a skill...
Look for:
- Requests to add, change, or remove steps
- Preferences about how steps should work
- Corrections
Ignore:
- Routine conversation
- Things the skill already does

Output JSON inside <updates> tags.
Each item: {"section": "...", "change": "...", "reason": "..."}.
` })]
}
```

#### 3. 解析结果

位置：`skillImprovement.ts:134`

```ts
parseResponse(content) {
  const updatesStr = extractTag(content, 'updates')
  if (!updatesStr) return []
  return jsonParse(updatesStr) as SkillUpdate[]
}
```

#### 4. 存储到 app state

位置：`skillImprovement.ts:146`

```ts
logResult(result, context) {
  if (result.type === 'success' && result.result.length > 0) {
    context.toolUseContext.setAppState(prev => ({
      ...prev,
      skillImprovement: {
        suggestion: { skillName, updates: result.result }
      }
    }))
  }
}
```

改进建议被存入 `appState.skillImprovement`，供 UI 或后续流程消费。

#### 5. 应用改进

位置：`skillImprovement.ts:188`

`applySkillImprovement(skillName, updates)` 是一个独立的 fire-and-forget 函数：

1. 读取 `.claude/skills/<name>/SKILL.md`
2. 把 `SkillUpdate[]` 格式化为改进列表
3. 调用 LLM（`temperatureOverride: 0` 确定性模式）要求改写 skill 文件
4. 从 `<updated_file>` 标签中提取新内容
5. 写回文件

---

### 初始化

位置：`skillImprovement.ts:175`

```ts
export function initSkillImprovement(): void {
  if (
    feature('SKILL_IMPROVEMENT') &&
    getFeatureValue_CACHED_MAY_BE_STALE('tengu_copper_panda', false)
  ) {
    registerPostSamplingHook(createSkillImprovementHook())
  }
}
```

这是一个 feature-flagged 功能。

---

### 架构值

这个文件展示了 hooks 系统的一种更高级用法：

```text
post-sampling hook (每 N 轮触发)
  -> LLM 分析用户交互中的 skill 偏好
  -> 生成 SkillUpdate[]
  -> 存入 app state
  -> （外部调用者）applySkillImprovement() -> LLM 改写 -> 写回文件
```

这是一个 **从用户交互中自学习 skill 定义** 的闭环。

---

### 读完这一站后，你应该抓住的 8 个事实

1. `skillImprovement.ts` 是一个 feature-flagged 的实验性功能，需要 `SKILL_IMPROVEMENT` 和 `tengu_copper_panda` 都启用。
2. 检测 hook 每 5 条 user message 分析一次，避免过于频繁的 LLM 调用。
3. 只分析主 REPL 线程且有 project skill 被调用的场景。
4. `buildMessages` 只分析自上次检查以来的新消息，而非完整历史，减少 token 消耗。
5. 改进建议存入 `appState.skillImprovement.suggestion`，供 UI 或其他流程消费。
6. `applySkillImprovement()` 是 fire-and-forget 的独立 LLM 调用，把改进建议写回 `.claude/skills/<name>/SKILL.md`。
7. 应用改进时用 `temperatureOverride: 0` 确保确定性输出，并保持 frontmatter 不变。
8. analytics 事件记录 `tengu_skill_improvement_detected`，包含 `skillName`（标记为 PII）。

---

最后剩余文件：`src/utils/hooks/apiQueryHookHelper.ts`。
