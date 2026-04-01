## 第 48 站：`src/utils/swarm/spawnUtils.ts`

### 这是什么文件

`src/utils/swarm/spawnUtils.ts` 是 swarm 体系里的 **process-based teammate 启动参数拼装器**。

上一站 `inProcessRunner.ts` 已经看到：

```text
in-process teammate 不需要外部 CLI bootstrap，
直接在当前进程里挂 ALS context 然后 runAgent 即可
```

但 pane/process-based teammate 完全不同。

那条链路需要回答：

- 用哪个可执行命令去启动 teammate
- leader 当前 CLI flags 哪些应该继承
- 权限模式该怎样传递
- model/settings/plugin/chrome 这些启动态配置怎样下发
- 哪些环境变量必须显式透传给 tmux / 外部 session 里的子进程

所以这一站回答的是：

```text
process-based teammate 在被 spawn 到新 pane / 新进程时，
启动命令、继承 flags、继承 env 是怎样统一拼装的
```

所以最准确的一句话是：

```text
spawnUtils.ts = process teammate 启动继承策略的公共拼装层
```

---

### 先看它的整体定位

位置：`src/utils/swarm/spawnUtils.ts:1`

这个文件很短，但职责非常集中，主要只有三件事：

1. `getTeammateCommand()`
2. `buildInheritedCliFlags()`
3. `buildInheritedEnvVars()`

这说明它不负责：

- 选 backend
- 建 pane
- 发 mailbox
- 注册 task state
- 跑 agent loop

它只负责：

```text
把“该怎样启动一个 process-based teammate”压缩成一份统一的命令拼装策略
```

也就是说它是：

```text
spawn command construction layer
```

这正好处在：

- `PaneBackendExecutor` 的 shell command 注入
- teammate CLI 真正启动

之间。

---

### 第一部分：`getTeammateCommand()` 说明 process teammate 默认复用当前 Claude Code 可执行入口，但允许显式覆盖

位置：`src/utils/swarm/spawnUtils.ts:23`

逻辑非常简单：

1. 如果设置了 `TEAMMATE_COMMAND_ENV_VAR`
   - 用这个 env var 指定的命令
2. 否则：
   - bundled mode -> `process.execPath`
   - 非 bundled mode -> `process.argv[1]`

这说明系统对“spawn teammate 用哪个命令”采用的是：

```text
default = 复用当前这份 Claude Code 可执行入口
override = 用专门 env var 强制指定
```

也就是说 teammate 默认不是另一套独立 binary，
而是：

```text
当前 leader 所在的 Claude Code 启动入口的派生实例
```

这与 swarm 整体“派生子执行体”的模型非常一致。

---

### 第二部分：`TEAMMATE_COMMAND_ENV_VAR` 的存在说明系统明确支持“teammate 启动命令与当前进程入口不同”的高级部署场景

位置：`src/utils/swarm/spawnUtils.ts:16`

虽然正常情况下会直接用：

- `process.execPath`
- 或 `process.argv[1]`

但这里仍然提供了一个环境变量覆盖口。

这说明作者考虑过一些更复杂的场景，比如：

- 包装启动器
- 特殊分发渠道
- 调试/开发环境下的替代入口
- 某些容器/远程执行场景里需要显式指向另一个命令

所以它表达的是：

```text
teammate command resolution 既有合理默认，也保留了部署级 override 能力
```

这是一种很稳妥的 spawn 基础设施设计。

---

### 第三部分：`buildInheritedCliFlags()` 揭示了 process teammate 并不是从零启动，而是 leader CLI 启动策略的受控继承体

位置：`src/utils/swarm/spawnUtils.ts:38`

这个函数是整个文件最核心的地方。

它会决定哪些 CLI flags 要从 leader 传给 teammate，
包括：

- permission mode
- model override
- settings path
- inline plugin dirs
- teammate mode snapshot
- chrome flag override

这说明 teammate 启动时不是“只传 identity flags 就完事”，
而是要尽可能继承 leader 当前启动策略中的重要部分。

所以这里真正编码的是：

```text
spawned teammate 应该继承 leader 哪些 session policy
```

而不是简单的参数拷贝。

---

### 第四部分：权限相关 flags 的继承逻辑很谨慎——`planModeRequired` 会压过 bypass permissions

位置：`src/utils/swarm/spawnUtils.ts:45`

这里的优先级很值得注意：

- 如果 `planModeRequired` -> 不继承 bypass permissions
- 否则如果当前 permissionMode 是 `bypassPermissions` 或 session bypass 打开
  - 传 `--dangerously-skip-permissions`
- 否则如果 permissionMode 是 `acceptEdits`
  - 传 `--permission-mode acceptEdits`

注释明确写了：

```text
Plan mode takes precedence over bypass permissions for safety
```

这说明 teammate 启动策略不是机械继承 leader 权限状态，
而是带有一个明确的安全规则：

```text
如果这名 teammate 被要求先 plan，
那就不能同时把“完全跳过权限”这种更危险的模式继承过去
```

所以 `buildInheritedCliFlags()` 不是单纯的 flag serializer，
而是在做：

```text
policy-aware inheritance
```

这是非常关键的系统语义。

---

### 第五部分：它既看 `permissionMode` 参数，也看 `getSessionBypassPermissionsMode()`，说明权限继承同时兼容“当前运行时上下文”与“session 启动快照”两条来源

位置：`src/utils/swarm/spawnUtils.ts:49`

这里判断 bypass permissions 时同时参考：

- 函数参数里的 `permissionMode`
- `getSessionBypassPermissionsMode()`

这说明权限继承可能来自两类来源：

#### 1. 当前调用点显式传入的 mode
比如当前 AppState / tool permission context

#### 2. 会话级 CLI/启动状态
即 session 一开始就处在 bypass permissions 模式

所以这里真正要保证的是：

```text
不管 bypass 权限是从当前运行态还是会话启动态来的，
spawn 出来的 teammate 都应该在允许的情况下继承到同样语义
```

这进一步说明 spawn helper 处在“静态启动参数”和“动态运行时状态”之间的桥位。

---

### 第六部分：`--model` 的传播只在显式 CLI override 存在时发生，说明 spawn helper 区分“用户明确指定的启动策略”与“系统内部默认模型选择”

位置：`src/utils/swarm/spawnUtils.ts:58`

这里不是无脑传递当前主模型，
而是只在：

- `getMainLoopModelOverride()` 有值

时才加：

- `--model ...`

这说明作者不想把所有模型选择都固化到 spawn command 上，
而只传播：

```text
用户/启动入口显式覆盖过的模型选择
```

这背后很重要的语义是：

```text
teammate 默认应沿用系统正常模型解析逻辑；
只有当 leader session 明确被 CLI override 改过时，才需要显式继承
```

这避免了过度序列化隐式默认值。

---

### 第七部分：`--settings` 与 `--plugin-dir` 的继承说明 teammate process 需要共享 leader 的配置视图与扩展视图

位置：`src/utils/swarm/spawnUtils.ts:64`

这里会传播：

- `--settings <path>`
- 多个 `--plugin-dir <dir>`

这非常关键，因为它说明 process-based teammate 不只是要共享“模型和权限”，
还要共享：

#### 配置视图
同一份 settings 文件

#### 扩展视图
同一批 inline plugins

所以 process teammate 启动时实际上是在尽量保持：

```text
leader 和 teammate 对“当前 Claude 环境长什么样”的理解一致
```

否则会出现：

- leader 有某个 plugin，teammate 没有
- leader 的 settings 改了路径/行为，teammate 看不到

因此这些 flags 的继承是非常必要的。

---

### 第八部分：`--teammate-mode` 一定会传播，说明 teammate mode 被当成 session 级稳定策略，而不是只对 leader 本地有效的偏好

位置：`src/utils/swarm/spawnUtils.ts:76`

这里不带条件，直接：

- `const sessionMode = getTeammateModeFromSnapshot()`
- `flags.push(--teammate-mode ${sessionMode})`

这说明上一站 registry 里那个“session snapshot mode”不仅影响 leader 本地 backend 选择，
还会成为所有 process teammates 启动时的显式继承项。

也就是说：

```text
teammate mode 是整个 swarm session 的共享启动策略
```

而不是 leader 单机上的临时决策。

这也解释了为什么它取的是 snapshot，
而不是实时 settings：

```text
要保证整个 session 派生出来的 teammate 使用同一套稳定模式解释
```

---

### 第九部分：`--chrome` / `--no-chrome` 的继承说明 browser/computer-use 相关能力也被视为 teammate 启动环境的一部分

位置：`src/utils/swarm/spawnUtils.ts:80`

这里会根据 `getChromeFlagOverride()` 传播：

- `--chrome`
- 或 `--no-chrome`

这说明 Chrome / computer-use 能力开关不是只对 leader 有意义。

系统明确把它看成：

```text
spawned teammate 的运行环境能力之一
```

换句话说，如果 leader 是在某种明确的 browser integration 策略下运行，
teammate process 也应该看到同样的显式模式。

这对于 computer-use / Claude in Chrome 一类功能的一致性很重要。

---

### 第十部分：`TEAMMATE_ENV_VARS` 才是最重要的环境变量继承白名单——它不是“全量继承环境”，而是有意识选择一小批必须跨 tmux/session 保真的变量

位置：`src/utils/swarm/spawnUtils.ts:96`

这里定义了一组显式白名单，包括：

- API provider 选择：
  - `CLAUDE_CODE_USE_BEDROCK`
  - `CLAUDE_CODE_USE_VERTEX`
  - `CLAUDE_CODE_USE_FOUNDRY`
- `ANTHROPIC_BASE_URL`
- `CLAUDE_CONFIG_DIR`
- `CLAUDE_CODE_REMOTE`
- `CLAUDE_CODE_REMOTE_MEMORY_DIR`
- proxy / cert 相关变量：
  - `HTTPS_PROXY`
  - `HTTP_PROXY`
  - `NO_PROXY`
  - `SSL_CERT_FILE`
  - `NODE_EXTRA_CA_CERTS`
  - `REQUESTS_CA_BUNDLE`
  - `CURL_CA_BUNDLE`

这说明作者明确没有选择：

```text
把 leader 进程所有环境变量无脑传给 teammate
```

而是做了：

```text
最小必要环境继承白名单
```

这是一种非常稳妥的做法，
既减少不可控污染，
也明确表达哪些环境变量对 teammate 是关键且必须保真。

---

### 第十一部分：env 白名单里的注释非常有价值，它们揭示了 spawn helper 是被真实线上问题打磨出来的

位置：`src/utils/swarm/spawnUtils.ts:97`

这些注释不是泛泛而谈，而是在记录非常具体的失败模式。

#### API provider env
注释说：

```text
不传这些，teammates 会默认 firstParty，导致请求打错 endpoint
```

甚至还提到了具体 issue：

- `GitHub issue #23561`

#### `CLAUDE_CODE_REMOTE`
注释说明：

```text
CCR-aware code path 需要这个标记；
auth 自己会从 remote oauth token 走，不靠 FD env 传递
```

#### `CLAUDE_CODE_REMOTE_MEMORY_DIR`
注释说明：

```text
只传 REMOTE 不传 MEMORY_DIR，会把 teammate 错误切到 memory-off
```

#### proxy vars
注释说明：

```text
如果不传 proxy/cert vars，teammates 会绕过 parent 的 MITM relay / 上游代理配置
```

这说明 `spawnUtils.ts` 不是抽象美学产物，
而是：

```text
被真实部署、远程运行、代理网络、memory 兼容问题反复修出来的兼容层
```

这个信号非常重要。

---

### 第十二部分：`buildInheritedEnvVars()` 说明 env 继承的目标不是完整复制 parent env，而是确保 tmux/new shell 场景里的关键配置不丢失

位置：`src/utils/swarm/spawnUtils.ts:135`

这里会构造：

- 固定加：
  - `CLAUDECODE=1`
  - `CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS=1`
- 然后对白名单里的 env：
  - 如果当前进程里有值，就 `KEY=quoted(value)` 加进去

注释明确说：

```text
tmux 可能会起新 login shell，不继承 parent env，
所以这些关键 env 必须显式转发
```

这说明这层 helper 主要解决的是：

```text
pane/process teammate 启动时的 shell environment discontinuity
```

也就是 leader 当前进程的环境上下文，
在 tmux 或外部 shell 里默认并不可靠延续，
必须手工 bridge。

---

### 第十三部分：固定注入 `CLAUDECODE=1` 和 `CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS=1`，说明 teammate process 会被显式标记为 Claude Code / experimental teams 运行环境

位置：`src/utils/swarm/spawnUtils.ts:136`

即便别的环境变量都不传，
这里也一定会加：

- `CLAUDECODE=1`
- `CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS=1`

这说明 process teammate 不是在“普通 shell 环境”里自己摸索身份，
而是会被明确告知：

```text
你是一个运行在 Claude Code 中、且 agent teams 功能已启用的派生进程
```

所以这两个 env 可以看成 process-based swarm teammate 的最基础环境标记。

---

### 第十四部分：整个文件最关键的设计不是“拼字符串”，而是把 teammate 启动继承策略制度化

如果只看代码表面，感觉它只是：

- 拼 command
- 拼 flags
- 拼 env

但真正重要的是它把很多隐含规则都集中显式化了：

- 哪些 CLI 参数该继承
- 哪些参数需要条件继承
- plan mode 和 bypass permissions 谁优先
- 哪些 env 必须透传
- teammate mode 该从 session snapshot 来
- 哪些浏览器/插件/settings 视图应该和 leader 一致

所以它真实做的是：

```text
把“派生 teammate process 应该继承 leader 的哪些运行策略”收口成一套统一制度
```

这比单纯命令拼接重要得多。

---

### 第十五部分：这份文件和 `PaneBackendExecutor` 的关系是“命令承载器”和“继承策略提供器”的分工

把第 45 站和这一站对照起来很清楚：

#### `PaneBackendExecutor.ts`
负责：
- 建 pane
- 调 `sendCommandToPane(...)`
- 发初始 prompt mailbox
- 管 terminate/kill

#### `spawnUtils.ts`
负责：
- 决定 executable 是谁
- 决定传哪些 CLI flags
- 决定传哪些 env vars

也就是说：

```text
PaneBackendExecutor 负责“把启动命令送进宿主”
spawnUtils 负责“这条启动命令里到底该包含什么”
```

这是一个非常干净的分层。

---

### 读完这一站后，你应该抓住的 9 个事实

1. `spawnUtils.ts` 不是 backend 选择器或 runner，而是 process-based teammate 启动命令的公共拼装层。
2. `getTeammateCommand()` 默认复用当前 Claude Code 启动入口，但允许通过 `TEAMMATE_COMMAND_ENV_VAR` 做部署级覆盖。
3. `buildInheritedCliFlags()` 的核心职责是把 leader 当前 session 中需要保真的 CLI 启动策略，选择性地继承给 spawned teammate。
4. 权限相关 flag 的继承是 policy-aware 的：如果 `planModeRequired` 为真，就不会继承 bypass permissions，这体现了 plan mode 对危险权限模式的安全优先级。
5. `--model`、`--settings`、`--plugin-dir`、`--teammate-mode`、`--chrome/--no-chrome` 都可能被传播，说明 teammate process 需要尽量共享 leader 的配置视图与能力视图。
6. teammate mode 使用 `getTeammateModeFromSnapshot()` 传播，说明它被视为整个 session 的稳定派生策略，而不是实时热更新选项。
7. `TEAMMATE_ENV_VARS` 是一个最小必要环境变量白名单，而不是全量环境复制，体现了“关键兼容性保真 + 控制污染范围”的折中设计。
8. env 白名单中的注释揭示了这层代码是被真实生产问题打磨出来的，涵盖 provider endpoint、CCR、memory gating、proxy/cert 等具体兼容场景。
9. `buildInheritedEnvVars()` 主要解决的是 tmux/new shell 场景下环境变量继承不可靠的问题，确保 spawned teammate 不会丢失关键运行环境信息。

---

### 现在把第 47-48 站串起来

```text
inProcessRunner.ts
  -> 负责同进程 teammate 的长期执行循环、idle wait、权限桥接与 mailbox 协调
spawnUtils.ts
  -> 负责进程型 teammate 启动时的 command / flags / env 继承策略拼装
```

所以现在两条 teammate 执行路线可以先压缩成：

```text
in-process route
  -> shared process + ALS + local task/runtime loop

process-based route
  -> pane host + spawned CLI process + inherited command/flags/env + mailbox protocol
```

也就是说：

```text
inProcessRunner.ts 回答“同进程 teammate 如何活着并持续工作”
spawnUtils.ts 回答“进程型 teammate 启动时该继承哪些 leader 运行策略”
```

---

### 下一站建议

下一站最顺的是：

```text
src/utils/swarm/backends/detection.ts
```

因为现在你已经看懂了：

- teammate backend 如何在 registry 中统一
- pane backend 怎样被适配成 `TeammateExecutor`
- process teammate 启动命令如何继承 leader 的 flags/env

下一步最自然就是把 backend 选择前的环境探测细节补齐：

**系统到底怎样判断自己是不是在 tmux / iTerm2 里、tmux/it2 CLI 是否可用、以及这些底层环境探测结果是怎样支撑 `registry.ts` 那条 backend 决策树的。**
