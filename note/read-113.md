## 第 113 站：`src/tools/AskUserQuestionTool/`（2 个文件，~44K）

### 这是什么

AskUserQuestionTool 是模型向用户提问的工具——支持多选选择题（2-4 个选项）、单选或多选、1-4 个问题批量询问。这是模型主动征求用户意见的通道，与用户通过 TUI 输入不同。

---

### 输入 Schema
```ts
{
  questions: [            // 1-4 个问题
    {
      question: string    // 完整的问题文本（应以 ? 结尾）
      header: string      // 显示为芯片/标签的短标签（最大宽度的字符）
      options: [          // 2-4 个选项
        {
          label: string       // 选项显示文本（1-5 字）
          description: string // 选项说明/后果
          preview?: string    // 可选：预览内容（mockup/代码片段/对比图）
        }
      ]
      multiSelect: boolean    // 默认 false：是否允许多选
    }
  ]
  answers?: Record<string, string>   // 用户回答（由权限组件收集）
  annotations?: Record<string, { preview?: string, notes?: string }>  // 每问题的注解
  metadata?: { source?: string }     // 分析元数据（如 "remember" 用于 /remember）
}
```

**唯一性约束**：问题文本必须在所有问题中唯一，选项标签必须在问题内唯一。

---

### 输出 Schema
```ts
{
  questions: [...]         // 原样回显问题
  answers: Record<string, string>   // 用户回答
  annotations?: {...}      // 用户注解
}
```

---

### 关键特征

#### 始终需要用户交互
```ts
requiresUserInteraction() { return true }
```

这是唯一显式要求 `requiresUserInteraction` 的工具。它不能在没有键盘/界面的环境中运行。

#### Channel 模式自动禁用
```ts
isEnabled() {
  if ((feature('KAIROS') || feature('KAIROS_CHANNELS')) && getAllowedChannels().length > 0) {
    return false
  }
  return true
}
```

当通过 Telegram/Discord 等频道运行时（用户不在 TUI 前），多选对话框会挂起——没有人在键盘前。因此工具直接被禁用。

#### 权限始终 ask
```ts
async checkPermissions() {
  return { behavior: 'ask', message: 'Answer questions?', updatedInput: input }
}
```

所有 AskUserQuestion 调用都需要用户明确确认，没有 auto-allow 路径。

---

### Preview 机制

每个选项可以有 `preview` 字段——当用户聚焦该选项时显示的预览内容：
- ASCII mockups
- 代码片段
- 图表对比

预览在 UI 中以 monospace 框渲染，允许用户视觉比较不同选项。

---

### HTML Preview 验证

当 `getQuestionPreviewFormat() === 'html'` 时：
```ts
validateHtmlPreview(opt.preview)
```

HTML 预览有安全验证——防止注入恶意 HTML 到 TUI 渲染中。验证在 `validateInput()` 中执行。

---

### React Compiler 编译

文件已被 React Compiler 预编译（`import { c as _c } from "react/compiler-runtime"`）——这是 Bun 编译链的结果。`$[0] === Symbol.for("react.memo_cache_sentinel")` 模式是 React.memo 的缓存实现。

---

### 自动化分类输入

```ts
toAutoClassifierInput(input) {
  return input.questions.map(q => q.question).join(' | ')
}
```

所有问题文本用 ` | ` 连接作为分类器输入——用于分析模型在什么场景下问用户问题。

---

### 结果渲染

```tsx
function AskUserQuestionResultMessage({ answers }) {
  return <Box flexDirection="column">
    <Text>User answered Claude's questions:</Text>
    {Object.entries(answers).map(([questionText, answer]) => (
      <Text>· {questionText} → {answer}</Text>
    ))}
  </Box>
}
```

结果以 "问题 → 回答" 格式显示。

---

### Prompt 动态注入

```ts
async prompt() {
  const format = getQuestionPreviewFormat()
  if (format === undefined) return ASK_USER_QUESTION_TOOL_PROMPT
  return ASK_USER_QUESTION_TOOL_PROMPT + PREVIEW_FEATURE_PROMPT[format]
}
```

当 preview 格式可用时，额外追加 prompt 指导来告诉模型如何使用 preview 字段。

---

### 读完后应抓住的 4 个事实

1. **AskUserQuestion 是唯一需要 `requiresUserInteraction()` 的工具**——没有键盘或界面的环境（Telegram、Discord 频道）工具直接被禁用。这意味着 auto 模式下模型不能通过 AskUserQuestion 提问——必须有用户在屏幕前。

2. **权限始终 ask**——没有 auto-allow 路径。每道问题都需要用户确认才能展示。这是 UX 安全措施：模型不能自动弹出对话框而不经用户同意，防止对话轰炸。

3. **Preview 是可比的 UI 组件**——每个选项可以有一个 `preview` 字段，聚焦时以 monospace 模式渲染。这让模型展示 ASCII mockups、代码差异、渲染图表等——让用户能视觉比较选项。HTML 预览有安全验证防止 XSS。

4. **批量提问（最多 4 个问题）**——输入支持 1-4 个问题数组，用户可以一次性回答所有问题。这减少了多轮交互的往返次数——模型可以一次问多个选择题而不是逐个问。
