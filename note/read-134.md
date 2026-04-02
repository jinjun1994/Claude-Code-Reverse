## 第 154 站：Prompt Suggestion / Tips / Settings Sync / Policy Limits

### Prompt Suggestion 与 Speculation

**Ghost text建议**——forked agent 预测用户下一步输入，显示为输入行的 ghost 文本。

**Speculation（推测执行）**——如果用户接受，forked agent 在沙盒 overlay 文件系统中预先执行预测工作：
```
overlay: CLAUDE_TEMP_DIR/speculation/${pid}/${id}/
```

**工具权限模型（speculation）**：
- Write 工具：检查权限模式，不允许时 abort 在 "edit boundary"
- 文件路径重写：写入 overlay，从 overlay 读取
- 只读工具：始终允许
- Bash：仅 `checkReadOnlyConstraints` 通过时
- 其他：拒绝，abort 在 "denied_tool boundary"

**4 种边界类型**：`complete`, `bash`, `edit`, `denied_tool`——每种记录什么停止了 speculation。

**Accept 流程**：copy overlay → main cwd，合并消息到历史，合并文件状态缓存。

**Pipelining**——speculation 完成后生成下一条建议，接受后如果 speculation 完整，立即 promotion + 新一轮 speculation。

**11+ 建议过滤器**——过滤 "done", "silence", Claude voice（"Let me...", "I'll..."）, eval 短语（"thanks", "looks good"）等。

---

### Tips 系统

**62 个生产力提示**，在 spinner 等待期间轮换显示。

**选择算法**——选择最久未被显示过的 tip：
```ts
tipsWithSessions.sort((a, b) => b.sessions - a.sessions)
```

**9 种 Tip 类别**：Onboarding、Feature awareness、IDE integration、Platform、Plugin promotion、GrowthBook-gated、Referral/growth、Custom tips、Internal-only。

**关联性检查**——每个 tip 有 `isRelevant()` 谓词，检查：全局配置（使用计数、上次时间戳）、终端检测、当前上下文（CLI 工具、已读文件）。

---

### Settings Sync

**双向同步**——通过云端 API 在多环境之间同步用户设置和 memory 文件。

**4 个同步键**：
- `~/.claude/settings.json`（全局设置）
- `~/.claude/CLAUDE.md`（全局 memory）
- `projects/{id}/.claude/settings.local.json`（项目设置）
- `projects/{id}/CLAUDE.local.md`（项目 memory）

**增量 push**：只上传变化的条目——`pickBy(local, (v, k) => remote[k] !== v)`。

**Pull 路径**：成功后 `applyRemoteEntriesToLocal` 写入磁盘，`markInternalWrite` 防止假文件变更检测事件，重置 settings 和 memory 缓存。

**500KB 文件大小限制**——空文件被跳过。

---

### Policy Limits

**API 拉取组织级策略限制**——fail-open 设计（获取失败 → 功能保持启用）。

**响应格式**：`{"allow_remote_sessions": {allowed: true/false}, ...}`——缺失的键表示允许。

**HIPAA fail-closed**：`isEssentialTrafficOnly()` 模式下，`allow_product_feedback` 等关键策略在缓存未命中时 fail-closed（拒绝）。

**"Waiting for loading" 模式**——早期创建 deferred promise，其他系统 `await` 策略限制即使 `loadPolicyLimits()` 还没调用。30 秒超时防护。

---

### 读完后应抓住的 4 个事实

1. **Speculation overlay 文件系统**——Speculation 不是直接写主目录，而是在 `CLAUDE_TEMP_DIR/speculation/${pid}/${id}/` overlay 中。Accept 时才复制 overlay 文件到 main cwd。这是安全的推测执行——如果用户不接受，overlay 被丢弃。

2. **Tips 按 session 计数轮换**——不是按时间，而是按启动次数选择最久未显示的 tip。这确保每个 session 只看到一个 tip，且轮换均匀。

3. **Settings Sync 的 markInternalWrite**——从云端写入本地设置时，标记 "internal write" 防止文件变更检测器误触发自己的变更检测。这是一个常见陷阱。

4. **Policy Limits HIPAA fail-closed 例外**——整体是 fail-open（API 失败 = 允许功能），但 `allow_product_feedback` 在 `isEssentialTrafficOnly()` 时 fail-closed（API 失败 = 禁止）。这是合规需求。
