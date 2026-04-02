# Claude Code 架构导读

本文档是 142 篇架构文档的索引和总览，覆盖 198 个架构站点。

## 阅读顺序建议

按以下 4 个阶段阅读，每阶段 2-4 篇文档：

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

| 子系统 | 覆盖文档 | 关键站点 |
|--------|----------|----------|
| **启动 + CLI** | read-28~35, read-142 | 34-41, 195-198 |
| **Bootstrap 状态** | read-131 | 150-151 |
| **类型系统** | read-132 | 152 |
| **工具系统** | read-42~60 | 42-60 |
| **权限系统** | read-61~70, read-138 | 61-70, 175 |
| **Hook 系统** | read-136, read-138 | 162, 175 |
| **Query 循环** | read-137 | 168 |
| **Compact 系统** | read-130 | 149 |
| **Forked Agent** | read-131 | 150 |
| **API 层** | read-133 | 153 |
| **MCP** | read-71~80 | 71-80 |
| **Plugin/Marketplace** | read-81~85 | 81-85 |
| **Memory 系统** | read-86~90, read-138 | 86-90, 173 |
| **Agent Team** | read-91~100, read-138 | 91-100, 174 |
| **Auth + Config** | read-135, read-141 | 157, 189-194 |
| **Settings Sync** | read-134 | 154 |
| **Bridge/Remote** | read-135 | 155 |
| **UI/Ink** | read-135, read-142 | 156, 195-198 |
| **Tips/Prompt Suggestion** | read-134 | 154 |
| **Voice** | read-135 | 158 |
| **Diagnostics** | read-135 | 158 |
| **Analytics** | read-137 | 170 |
| **Keybindings** | read-136 | 166 |
| **VCR/Testing** | read-137 | 171 |
| **Scheduler/Cron** | read-139 | 178 |
| **Graceful Shutdown** | read-139 | 179 |
| **Shell 引擎** | read-139 | 181 |
| **文件缓存** | read-139 | 182-183 |
| **Collapse/Streaming** | read-140 | 185 |
| **Profiling** | read-140 | 187 |

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
