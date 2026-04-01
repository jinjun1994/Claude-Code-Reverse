# Claude Code Tool Call Loop 架构图 / Tool Call Loop Architecture

基于 `outputs/claude-cli-clean.js` 中与 `tool_use`、`tool_result`、`tools/call`、权限请求、结果回注、以及流式回合继续执行相关实现整理。

## 1. 架构图 / Architecture Diagram

```mermaid
flowchart TD
    ToolUse(["API returns tool_use / API 返回 tool_use"]) --> AddTool["Register tool call / 登记工具调用"]

    AddTool --> FindTool{"Find tool definition / 查找工具定义"}
    FindTool -->|not found / 未找到| NotFound["Return error / 返回错误<br/>No such tool available"]
    FindTool -->|found / 找到| ValidateInput["Validate input / 验证输入<br/>inputSchema.safeParse(input)"]

    ValidateInput --> CheckConcurrency["Check concurrency safety / 检查并发安全性<br/>isConcurrencySafe()"]
    CheckConcurrency --> Enqueue["Queue tool call / 加入执行队列<br/>queued"]
    Enqueue --> ProcessQueue["Process execution queue / 处理执行队列"]

    ProcessQueue --> CanExec{"Can execute now / 现在可执行吗"}
    CanExec -->|busy and unsafe / 忙且非并发安全| Wait["Wait / 等待"]
    Wait --> ProcessQueue
    CanExec -->|ready / 可执行| PermCheck

    subgraph PermCheck[Permission check / 权限检查]
        PC1["canUseTool()"]
        PC2{"Permission mode / 权限模式"}
        PC3["classifier auto decision / 分类器自动判断"]
        PC4["ask user / 用户确认"]
        PC5["bypass / 直接通过"]
        PC6["allow rule match / allow 规则匹配"]
        PC7["deny rule match / deny 规则匹配"]
        PC1 --> PC2
        PC2 --> PC3
        PC2 --> PC4
        PC2 --> PC5
        PC2 --> PC6
        PC2 --> PC7
    end

    PermCheck -->|deny / 拒绝| Rejected["Return rejected tool_result / 返回拒绝结果"]
    PermCheck -->|allow / 允许| PreHook

    subgraph PreHook[PreToolUse hooks / PreToolUse 钩子]
        PH1["Match hook matcher / 匹配 hook matcher"]
        PH2["Run hook command / 执行 hook command"]
        PH3{"Hook decision / hook 决策"}
        PH4["approve / 继续"]
        PH5["deny / 阻止"]
        PH6["modify input / 修改输入"]
        PH1 --> PH2 --> PH3
        PH3 --> PH4
        PH3 --> PH5
        PH3 --> PH6
    end

    PreHook -->|deny / 阻止| Rejected
    PreHook -->|approve or modify / 继续或修改| Execute["Execute tool / 执行工具<br/>tool.call(input, context)"]

    Execute --> PostHook

    subgraph PostHook[PostToolUse hooks / PostToolUse 钩子]
        POH1["Match hook matcher / 匹配 hook matcher"]
        POH2["Run hook command / 执行 hook command"]
        POH3{"Hook decision / hook 决策"}
        POH4["continue / 继续"]
        POH5["stop / 停止会话"]
        POH6["block or annotate / 阻止或附加结果"]
        POH1 --> POH2 --> POH3
        POH3 --> POH4
        POH3 --> POH5
        POH3 --> POH6
    end

    PostHook --> Result["Return tool_result / 返回 tool_result"]
    Result --> BackToLoop(["Back to main loop / 返回对话主循环"])

    Execute -->|exception / 异常| ErrorHook["PostToolUseFailure hook / 失败后钩子"]
    ErrorHook --> ErrorResult["Return error tool_result / 返回错��� tool_result"]
    ErrorResult --> BackToLoop

    classDef loop fill:#dcfce7,stroke:#16a34a,color:#111827;
    classDef control fill:#fef3c7,stroke:#d97706,color:#111827;
    classDef state fill:#f3f4f6,stroke:#6b7280,color:#111827;
    classDef error fill:#fee2e2,stroke:#dc2626,color:#111827;

    class ToolUse,AddTool,ValidateInput,CheckConcurrency,Enqueue,ProcessQueue,Execute,Result,BackToLoop loop;
    class FindTool,CanExec,PermCheck,PC1,PC2,PC3,PC4,PC5,PC6,PC7,PreHook,PH1,PH2,PH3,PH4,PH5,PH6,PostHook,POH1,POH2,POH3,POH4,POH5,POH6,Wait control;
    class Rejected state;
    class NotFound,ErrorHook,ErrorResult error;
```

## 2. 架构图详细说明 / Detailed Explanation

### 2.1 tool call loop 的起点 / Start of the tool call loop

tool call loop 的起点不是用户输入本身，而是模型在 assistant content 中产出 `tool_use` block。

The tool call loop does not start from raw user input itself. It starts when the model emits a `tool_use` block inside assistant content.

这意味着工具调用是 turn loop 的中途分支，而不是完全独立的子系统。

That means tool execution is a mid-turn branch of the larger turn loop, not a completely separate subsystem.

### 2.2 每个工具调用都带有 tool_use_id / Every tool call carries a `tool_use_id`

在 bridge 调用实现中，系统会先生成一个唯一 id：`outputs/claude-cli-clean.js:19078-19086`。

- `let w = crypto.randomUUID()`
- 之后把它写入 `tool_use_id`

这个 id 是整条链路的主键：

- 发起 `tool_call` 时带上它
- 权限请求时用它回溯 pending call
- 返回 `tool_result` 时用它对上原始调用

This id is the primary key of the whole loop:

- attached to the outgoing `tool_call`
- used to find the pending call during permission handling
- used again to match the returning `tool_result`

### 2.3 工具执行主链不止是 permission 和 tool_result / The execution path includes more than permission and tool_result

你指出得对，原图过度强调了 `permission_request -> tool_result -> transcript` 这一段，但少了真正的工具执行主链中间层。

You were right. The previous diagram emphasized the `permission_request -> tool_result -> transcript` slice, but it underrepresented the internal execution pipeline in the middle.

补全后，主链应当包括这些阶段：

1. 查找工具定义 / find tool definition
2. `inputSchema.safeParse(input)` 输入校验
3. `isConcurrencySafe()` 并发安全检查
4. queued / process queue 队列调度
5. `canUseTool()` 权限判定
6. `PreToolUse` hooks
7. `tool.call(input, context)` 实际执行
8. `PostToolUse` hooks
9. 异常时进入 `PostToolUseFailure`
10. 返回 `tool_result` 并回到主对话循环

So the tool call loop is not only a permission bridge. It is a longer execution pipeline with validation, scheduling, hooks, execution, failure handling, and then transcript reintegration.


### 2.4 输入校验与工具查找 / Tool lookup and input validation

在真正执行前，系统首先需要把 `tool_use.name` 映射到具体工具定义，然后用该工具的 schema 校验输入。

Before execution, the system must resolve `tool_use.name` to a concrete tool definition and then validate the input against that tool's schema.

代码依据：

- `inputSchema.safeParse(...)`：`outputs/claude-cli-clean.js:137573-137579`
- 另一处 tool input 校验：`outputs/claude-cli-clean.js:152353-152361`
- 另一处 tool input 校验：`outputs/claude-cli-clean.js:174898-174898`

这一步对应图中的：

- `Find tool definition / 查找工具定义`
- `Validate input / 验证输入`

如果工具不存在或输入不合法，tool call loop 会在真正执行前被截断。

### 2.5 并发安全与队列 / Concurrency safety and queueing

工具并不是一律立刻执行。代码里大量工具实现暴露了 `isConcurrencySafe()`，说明运行时会根据工具是否可并发来调度执行。

Tools are not always executed immediately. Many tool implementations expose `isConcurrencySafe()`, which indicates the runtime schedules execution according to concurrency safety.

代码依据：

- 多处 `isConcurrencySafe()`：例如 `outputs/claude-cli-clean.js:117633`, `121538`, `147472`
- queued message 渲染：`outputs/claude-cli-clean.js:152393-152399`

因此图里补上了：

- `Check concurrency safety / 检查并发安全性`
- `Queue tool call / 加入执行队列`
- `Process execution queue / 处理执行队列`
- `Wait / 等待`

这里表达的是概念性主链：先判断并发属性，再决定立即执行还是排队。

### 2.6 权限检查是执行前闸门 / Permission check is the gate before execution

权限检查层仍然重要，但它位于查找、校验、排队之后、执行之前。

The permission layer is still important, but it sits after lookup/validation/queueing and before the actual tool call.

代码依据：

- tool decision 记录与埋点：`outputs/claude-cli-clean.js:137656-137691`
- `canUseTool()` 相关调用主链：`outputs/claude-cli-clean.js:201346-201346`

因此 flowchart 里把权限层展开成：

- `canUseTool()`
- classifier auto decision
- ask user
- bypass
- allow rule match
- deny rule match

这更符合真实实现的控制分层。

### 2.7 PreToolUse hooks 在真正执行前运行 / `PreToolUse` hooks run before execution

代码里明确存在这些 hook 事件：

- `PreToolUse`
- `PostToolUse`
- `PostToolUseFailure`

对应依据：`outputs/claude-cli-clean.js:36092`, `135304-135306`, `109893-109916`。

此外，工具执行进度渲染里也明确标记了 `PreToolUse`：`outputs/claude-cli-clean.js:152381-152387`。

所以图里补上了 PreHook 子图，表示在真正 `tool.call(...)` 之前，系统还会：

1. 匹配 hook matcher
2. 执行 hook command
3. 根据 hook 决策继续、拒绝或修改输入

### 2.8 真正执行点是 `tool.call(input, context)` / The real execution point is `tool.call(input, context)`

权限与 pre-hook 都通过之后，才进入真正的工具执行点。

Only after permission and pre-hook checks pass does the system reach the actual execution point.

文档中的 `tool.call(input, context)` 是架构层表达，用来表示“进入具体工具实现”。它和上游的 dispatch/permission/hook 是不同层次。

### 2.9 PostToolUse hooks 与失败后钩子 / `PostToolUse` and failure hooks

工具执行成功后，还不会立刻结束，而是会进入 `PostToolUse` hooks。对应代码：`outputs/claude-cli-clean.js:201341-201439`。

当工具执行异常时，会进入 `PostToolUseFailure` hooks。对应代码：`outputs/claude-cli-clean.js:201440-201519`。

这也是你指出原图缺失的重要部分。因为少了这一层，图就看起来像“执行完直接回写结果”，而真实实现中间还有 hook 后处理。

### 2.10 返回 `tool_result` 后才回到主循环 / Return to the main loop only after `tool_result`

无论是正常路径还是异常路径，最终都会落到：

- `Return tool_result / 返回 tool_result`
- `Back to main loop / 返回对话主循环`

也就是把工具执行结果重新并入 transcript，然后交回上层 turn loop 继续采样。


## 3. sequenceDiagram 时序图版 / Sequence Diagram

```mermaid
sequenceDiagram
    autonumber
    participant Model as Assistant model / assistant 模型
    participant Runtime as Tool runtime / bridge / 工具运行时
    participant Queue as Queue scheduler / 队列调度器
    participant Perm as Permission handler / 权限处理器
    participant PreHook as PreToolUse hooks / PreToolUse 钩子
    participant Tool as Tool implementation / 工具实现
    participant PostHook as PostToolUse hooks / PostToolUse 钩子
    participant FailHook as PostToolUseFailure hooks / 失败后钩子
    participant Transcript as Transcript validator / 会话校验器
    participant Sampler as sampling/createMessage / 继续采样

    Model-->>Runtime: 发出 tool_use / Emit tool_use
    Runtime->>Runtime: 查找工具并校验输入 / Resolve tool and validate input
    Runtime->>Queue: 检查并发安全并入队 / Check concurrency safety and enqueue
    Queue-->>Runtime: 允许执行 / Ready to execute

    alt 需要权限 / permission required
        Runtime->>Perm: 请求权限 / Request permission
        Perm-->>Runtime: 返回决定 / Return decision
    else 无需权限 / no permission needed
        Runtime->>Runtime: 直接继续 / Continue directly
    end

    alt PreToolUse 阻止 / PreToolUse denies
        Runtime->>PreHook: 执行 PreToolUse / Run PreToolUse
        PreHook-->>Runtime: deny or modify / 拒绝或修改
        Runtime->>Transcript: 写入拒绝结果 / Append rejected result
    else 允许执行 / allowed to execute
        Runtime->>PreHook: 执行 PreToolUse / Run PreToolUse
        PreHook-->>Runtime: approve or modify / 允许或修改
        Runtime->>Tool: 执行工具 / Execute tool

        alt 执行成功 / execution succeeds
            Tool-->>Runtime: 返回结果 / Return result
            Runtime->>PostHook: 执行 PostToolUse / Run PostToolUse
            PostHook-->>Runtime: continue stop or annotate / 继续 停止 或附加结果
            Runtime->>Transcript: 写入 tool_result / Append tool_result
        else 执行异常 / execution fails
            Tool-->>Runtime: 抛出错误 / Throw error
            Runtime->>FailHook: 执行 PostToolUseFailure / Run failure hooks
            FailHook-->>Runtime: 返回失败后上下文 / Return failure context
            Runtime->>Transcript: 写入错误 tool_result / Append error tool_result
        end
    end

    Transcript->>Transcript: 校验 id 匹配 / Validate ids match
    Transcript-->>Sampler: 返回更新后的消息 / Return updated messages
    Sampler-->>Model: 继续生成 / Continue generation
```

## 4. 时序图详细说明 / Sequence Explanation

### 4.1 时序图现在覆盖了完整执行流水线 / The sequence diagram now covers the full execution pipeline

更新后的时序图不再只描述 `permission_request -> tool_result -> sampling/createMessage` 这条窄路径，而是补全为：

1. assistant 发出 `tool_use`
2. Runtime 查找工具并校验输入
3. Queue scheduler 按并发安全规则调度
4. Permission handler 做执行前审批
5. `PreToolUse` hooks 在执行前介入
6. Tool implementation 真正执行
7. 成功时进入 `PostToolUse`
8. 失败时进入 `PostToolUseFailure`
9. Transcript validator 校验 `tool_result` 与上一条 `tool_use` 的 id 对应关系
10. Sampler 用更新后的消息继续生成

The updated sequence diagram no longer shows only the narrow `permission_request -> tool_result -> sampling/createMessage` path. It now covers the full execution pipeline from tool emission to reintegration into sampling.


### 4.2 Runtime 负责把“模型意图”变成“真实调用” / Runtime turns model intent into a real invocation

运行时会登记 pending call、设置 timeout、附加权限模式，然后才真正发起调用。

The runtime registers a pending call, sets timeout and permission-related fields, and only then performs the actual invocation.

### 4.3 Permission handler 是执行前闸门 / The permission handler is a gate before execution

如果需要审批，工具不会直接执行，而是先走 `permission_request` / `permission_response`。

If approval is required, execution pauses until the permission loop completes.

### 4.4 Transcript validator 保证消息结构合法 / The transcript validator guarantees structural correctness

系统不允许随意拼接 `tool_result`，而是强制要求它们和上一条 assistant 消息里的 `tool_use` 成对出现且 id 一致。

The system does not allow arbitrary tool-result insertion. It enforces one-to-one structural pairing with the immediately preceding `tool_use` blocks.

### 4.5 Sampler 让回合继续闭环 / The sampler closes the loop and continues the turn

一旦 transcript 合法，系统就会再次请求 `sampling/createMessage`，让模型基于新上下文继续回答。

Once the transcript is valid, the system samples again so the assistant can continue with the tool output now included in context.

## 5. 代码依据 / Code References

- tool call 发起与 `tool_use_id`、`pendingCalls`：`outputs/claude-cli-clean.js:19063-19143`
- 权限请求与响应：`outputs/claude-cli-clean.js:19544-19586`
- tool result 处理与归一化：`outputs/claude-cli-clean.js:19588-19657`
- `requestStream(...)` 任务轮询：`outputs/claude-cli-clean.js:25261-25339`
- `tool_result` 与上一条 `tool_use` 的严格配对校验：`outputs/claude-cli-clean.js:25806-25825`, `outputs/claude-cli-clean.js:26150-26169`
- `tools/call` 请求与能力校验：`outputs/claude-cli-clean.js:25985-26011`, `outputs/claude-cli-clean.js:26092-26096`
