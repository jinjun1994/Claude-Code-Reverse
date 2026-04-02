# Claude Code 架构导读

本文档是 142 篇架构文档的索引和总览，覆盖 198 个架构站点。

## 顶层模块地图

为了避免只按站点号线性阅读，建议先把整套笔记理解成 4 个顶层模块：

| 模块 | 核心问题 | 优先文档 |
|------|----------|----------|
| **启动与入口** | Claude Code 如何从进程启动进入可运行会话？ | `read.md`、`read-28~35.md`、`read-142.md` |
| **控制面与运行时** | 配置、权限、工具执行、Query 主循环如何拼成一次真实调用？ | `read-98.md`、`read-126.md`、`read-130.md`、`read-137.md`、`read-139.md`、`read-141.md` |
| **扩展面与协作** | MCP、Plugin、Memory、Agent Team、Bridge、Transport 如何接入并协同？ | `read-71~80.md`、`read-81~85.md`、`read-86~100.md`、`read-135.md`、`read-136.md`、`read-146.md` |
| **架构总览与阅读路径** | 不按文件顺推时，怎样建立整套系统脑图？ | `read.md`、`read-138.md`、`read-142.md`、`read-143.md`、`read-146.md` |

`read.md` 现在负责“总入口导读”，而本文件负责“完整架构索引与覆盖矩阵”。两者配合使用：

- `read.md`：适合第一次进入语料时建立主问题意识
- `read-143.md`：适合回头做模块化索引、交叉跳读与架构收束

## 阅读顺序建议

### 先按模块读

#### 模块 1：启动与入口

先建立全局术语、启动链与主入口理解。

| 文档 | 站点 | 核心内容 |
|------|------|----------|
| `read.md` | 1-27 | 早期总站笔记，建立 main/init/commands/query 主链路 |
| `read-28` 到 `read-35` | 34-41 | CLI 入口、main.tsx 编排、init 初始化 |
| `read-142` | 195-198 | Entrypoints、commands、constants、UI 组件完整图 |
| `read-135 (156)` | 156 | Ink 框架、Store 架构、REPL 组件 |

#### 模块 2：控制面与运行时

理解一次 API 调用从配置、权限、工具到 query 收束的完整生命周期。

| 文档 | 站点 | 核心内容 |
|------|------|----------|
| `read-98` | 98 | Bash 安全与校验链 |
| `read-126` | 135-142 | 工具执行管线、权限运行时、Hook、Turn Loop |
| `read-130` | 149 | Compact 系统（4 种压缩层级） |
| `read-137` | 168-171 | Query 模块、Context Providers、Analytics、VCR/Testing |
| `read-139` | 178-183 | Scheduler/Cron、Shutdown、Shell、文件缓存 |
| `read-141` | 189-194 | Betas、Fast Mode、Thinking/Effort、Session State |

#### 模块 3：扩展面与协作

理解 Claude Code 如何向外扩展，并支持多智能体与远程协同。

| 文档 | 站点 | 核心内容 |
|------|------|----------|
| `read-71` 到 `read-80` | 71-80 | MCP 集成架构 |
| `read-81` 到 `read-85` | 81-85 | Plugin 和 Marketplace |
| `read-86` 到 `read-100` | 86-100 | Memory + Agent Team |
| `read-135` | 155-158 | Bridge、Remote、UI、Voice、Diagnostics |
| `read-136` | 159-167 | Buddy、Coordinator、Skill、Migrations、Keybindings |
| `read-146` | 201 | Transport 层完整架构 |

#### 模块 4：架构总览与阅读路径

适合回头做跨模块收束，把系统从“很多站”还原成“几条骨架”。

| 文档 | 站点 | 核心内容 |
|------|------|----------|
| `read.md` | 1-27 | 新总入口：启动链与阅读方式 |
| `read-138` | 173-177 | 中后段系统综合与最终总结 |
| `read-142` | 195-198 | 入口与 UI 完整图 |
| `read-143` | 索引文档 | 阅读顺序、覆盖矩阵、关键设计决策 |
| `read-146` | 201 | Transport 收尾补图 |

### 再按阶段读

如果你更习惯时间线式推进，可以继续按以下 4 个阶段阅读。

### 阶段 1：总览 + 启动链

先建立全局术语和主链路理解。

| 文档 | 站点 | 核心内容 |
|------|------|----------|
| read-28 到 read-35 | 34-41 | CLI 入口、main.tsx 编排、init 初始化 |
| read-142 | 195-198 | Entrypoints、commands、constants、UI 组件完整图 |
| read-135 (156) | 156 | Ink 框架、Store 架构、REPL 组件 |

### 阶段 2：控制面（配置、认证、设置）

理解 session 如何启动、设置如何合并。

| 文档 | 站点 | 核心内容 |
|------|------|----------|
| read-131 | 150-151 | Bootstrap 状态、Schema 系统 |
| read-133 (157) | 157 | Utils（context/model/config/attachments/claudemd） |
| read-134 | 154 | Settings Sync、Policy Limits |
| read-141 | 189-194 | Betas、Fast Mode、Thinking/Effort、Session State |

### 阶段 3：运行时循环（Query + Tool + Permission）

理解一次 API 调用完整生命周期。

| 文档 | 站点 | 核心内容 |
|------|------|----------|
| read-137 | 168 | Query 模块（config/deps/stopHooks/tokenBudget） |
| read-130 | 149 | Compact 系统（4 种压缩层级） |
| 之前文档 | 42-60 | 工具系统全览（所有内置工具实现） |
| read-138 (175) | 175 | Hook 实现系统（4-way permission racer） |
| 之前文档 | 61-70 | 权限系统核心 |

### 阶段 4：扩展面（MCP、Plugin、Memory、Team）

| 文档 | 站点 | 核心内容 |
|------|------|----------|
| read-133 | 153 | API 层（重试、多后端、流式） |
| 之前文档 | 71-80 | MCP 集成架构 |
| read-136 | 162 | Skill 加载与发现系统 |
| 之前文档 | 81-85 | Plugin 和 Marketplace |
| read-138 | 173 | Memdir 内存目录系统 |
| 之前文档 | 86-100 | Agent Team 架构 |
| read-135 | 155 | Bridge 远程控制系统 |

---

## 子系统覆盖矩阵

| 子系统 | 模块归属 | 覆盖文档 | 关键站点 |
|--------|----------|----------|----------|
| **总入口导读** | 架构总览与阅读路径 | `read.md`, `read-143` | 1-27, 索引 |
| **启动 + CLI** | 启动与入口 | read-28~35, read-142 | 34-41, 195-198 |
| **Bootstrap 状态** | 控制面与运行时 | read-131 | 150-151 |
| **类型系统** | 控制面与运行时 | read-132 | 152 |
| **工具系统** | 控制面与运行时 | read-42~60 | 42-60 |
| **权限系统** | 控制面与运行时 | read-61~70, read-138 | 61-70, 175 |
| **Hook 系统** | 控制面与运行时 | read-136, read-138 | 162, 175 |
| **Query 循环** | 控制面与运行时 | read-137 | 168 |
| **Compact 系统** | 控制面与运行时 | read-130 | 149 |
| **Forked Agent** | 扩展面与协作 | read-131 | 150 |
| **API 层** | 控制面与运行时 | read-133 | 153 |
| **MCP** | 扩展面与协作 | read-71~80 | 71-80 |
| **Plugin/Marketplace** | 扩展面与协作 | read-81~85 | 81-85 |
| **Memory 系统** | 扩展面与协作 | read-86~90, read-138 | 86-90, 173 |
| **Agent Team** | 扩展面与协作 | read-91~100, read-138 | 91-100, 174 |
| **Auth + Config** | 控制面与运行时 | read-135, read-141 | 157, 189-194 |
| **Settings Sync** | 控制面与运行时 | read-134 | 154 |
| **Bridge/Remote** | 扩展面与协作 | read-135 | 155 |
| **UI/Ink** | 启动与入口 | read-135, read-142 | 156, 195-198 |
| **Tips/Prompt Suggestion** | 控制面与运行时 | read-134 | 154 |
| **Voice** | 扩展面与协作 | read-135 | 158 |
| **Diagnostics** | 扩展面与协作 | read-135 | 158 |
| **Analytics** | 控制面与运行时 | read-137 | 170 |
| **Keybindings** | 扩展面与协作 | read-136 | 166 |
| **VCR/Testing** | 控制面与运行时 | read-137 | 171 |
| **Scheduler/Cron** | 控制面与运行时 | read-139 | 178 |
| **Graceful Shutdown** | 控制面与运行时 | read-139 | 179 |
| **Shell 引擎** | 控制面与运行时 | read-139 | 181 |
| **文件缓存** | 控制面与运行时 | read-139 | 182-183 |
| **Collapse/Streaming** | 控制面与运行时 | read-140 | 185 |
| **Profiling** | 控制面与运行时 | read-140 | 187 |
| **Transport** | 扩展面与协作 | read-146 | 201 |

---

## 最关键的 10 个架构决策

这些是贯穿整个代码库的核心设计：

1. **Prompt Cache 五字段匹配**——`CacheSafeParams`（systemPrompt/userContext/systemContext/toolUseContext/forkContextMessages）必须在父子请求间完全相同。任何不同都导致 cache miss。这几乎影响了系统的每个部分。

2. **隔离-默认共享**——forked agent 隔离所有 mutable 回调，只有三个 opt-in 标志可共享（shareSetAppState/shareAbortController/shareSetResponseLength）。这是安全设计。

3. **三优先级消息队列**——`now/next/later` 控制并发用户输入和系统事件。`now` 中断 in-flight 工具；`next` 等当前工具完成后插队；`later` 等 turn 完全结束后作为新请求。

4. **4-way Permission Racer**——权限决定由 bridge/channel/hook/classifier 四个来源竞争。`claim()` 原子守卫确保 race-free 决定。200ms 用户交互宽限期。

5. **后台调用不重试 529**——防止拥塞级联时 3-10x 网关放大。来自真实生产事件的设计。

6. **连续 3 个 529 触发模型回退**——Opus → Sonnet。生产级降级设计。

7. **Tool Result 三路分区预算执行**——mustReapply/frozen/fresh 三分区。一旦结果命运决定被冻结。这是提示缓存稳定性设计。引用"768GB 事件"。

8. **Context Collapse "marble-origami"**——通过 UUID 稳定的边界归档消息跨度，替换为摘要，resume 时回放。Commit + Snapshot 双模型。

9. **Query Guard 三状态机**——idle → dispatching → running。中间 dispatching 状态防止队列处理器在异步间隙期间出列。Generation 号检测陈旧完成。

10. **Conversation Recovery 的合成消息注入**——恢复被中断会话时注入合成消息使 transcript 对 API 有效。不是简单复制——智能路径映射、CLAUDE.md 重新解析。

---

## 生产设计模式汇总

### 性能优化
| 模式 | 位置 | 效果 |
|------|------|------|
| Prompt cache 五字段匹配 | forkedAgent.ts | cache hit 率从 2% → 98% |
| Tool schema 首次渲染锁定 | toolSchemaCache.ts | 中段 flag 变化不 bust cache |
| Markdown 稳定前缀 memoized | Markdown.tsx | 流式渲染减少 50%+ 重新渲染 |
| Stats Store reservoir sampling | stats.tsx | 百分位计算 1024 样本 vs 无限 |
| Capped max_tokens 8K | context.ts | 减少 8-16x over-reservation |
| Lazy require 打破循环 | lazySchema, lockfile | 启动 <200ms |
| 并行系统提示部分获取 | queryContext.ts | 减少串行等待 |
| 两阶段词法分析 | Markdown.tsx | 简单文本跳过 marked.lexer |

### 可靠性
| 模式 | 位置 | 效果 |
|------|------|------|
| 后台不重试 529 | withRetry.ts | 防止 3-10x 网关放大 |
| 连续 529 模型回退 | withRetry.ts | 拥塞时自动降级 |
| Compact PTL 自截断重试 | compact.ts | compact 自身不会 fail |
| 迁移从不抛出 | migrations/ | 迁移失败不阻塞启动 |
| Hook 失败吞噬 | attachments maybe() | 单附件失败不阻止轮次 |
| 死进程检测 | lockfile/autoDream | PID 不存活回收锁 |
| Graceful shutdown 超时 | gracefulShutdown.ts | 不会卡死关闭 |
| 防磁盘填满看门狗 | ShellCommand.ts | 引用 768GB 事件 |

---

## 代码库统计

| 指标 | 值 |
|------|-----|
| 源文件读取 | ~850+ |
| 架构站点 | 198 |
| 文档数 | 142 |
| 覆盖子系统 | 30+ |
| 关键设计决策 | 10 个 |
| 生产事件修复 | 5+ (768GB, cache miss, gateway amplification, etc.) |
