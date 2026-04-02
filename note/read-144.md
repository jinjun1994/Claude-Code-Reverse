## 第 199 站：剩余重要 Utils 模块

### Workload Context（workloadContext.ts）——AsyncLocalStorage 标签

**Turn-scoped 工作负载标签**——通过 `AsyncLocalStorage`。从 `bootstrap/state.ts` 分离因为浏览器 bundle 不能导入 Node 的 `async_hooks`。

**为什么用 ALS 而非全局变量**——void-detached 后台代理在执行第一个 await 时 yield。父 turn 的同步继续（包括 `finally` 块）在分离闭包恢复前完成运行。在闭包顶部设置 `setWorkload('cron')` 会被确定性覆盖。ALS 在调用时捕获上下文，在链中每个 await 都存活，与父隔离。

**泄漏修复**——先前实现在 `undefined` 时短路 `return fn()`——但那是直通，不是边界。如果调用者已经在泄漏的 cron 上下文中，直通让 `getWorkload()` 返回泄漏的标签。始终调用 `.run()` 保证 `fn` 内部返回完全匹配调用者传入的内容——包括 `undefined`。

### Session Ingress Auth（sessionIngressAuth.ts）——远程会话认证

**三层优先级的 Token 读取**：
1. 环境变量（`CLAUDE_CODE_SESSION_ACCESS_TOKEN`）——生成时设置，通过 `update_environment_variables` stdin 消息从父 bridge 进程更新
2. 文件描述符（遗留路径）—— `CLAUDE_CODE_WEBSOCKET_AUTH_FILE_DESCRIPTOR`，读一次并缓存
3. 已知文件——覆盖 subprocesses 不能继承 FD 的情况

**两种 Auth 格式**——`sk-ant-sid` 的 Cookie 认证 + `X-Organization-Uuid`；JWT 的 Bearer 认证。

---

### Undercover 模式（undercover.ts）——安全贡献工具

**目的**——为 Anthropic 员工在公共/开源仓库贡献时的安全工具。

**激活**：
- `CLAUDE_CODE_UNDERCOVER=1`——强制开启
- 否则自动：除非仓库远程匹配内部允许列表（`INTERNAL_MODEL_REPOS`），否则活跃
- **没有强制关闭**。这防止模型 codename 泄漏——如果我们不能确定在内部仓库，保持潜伏。

**安全指令**——向 commit/PR 提示添加安全说明：
- 禁止内部模型 codenames（Capybara, Tengu 等）
- 禁止未发布版本号（opus-4-7, sonnet-4-8）
- 禁止内部仓库/项目名称
- 禁止内部工具、Slack 频道或短链
- 禁止 "Claude Code" 或任何 AI 提及
- 禁止 Co-Authored-By 行

**构建时剥离**——所有路径都门控在 `process.env.USER_TYPE === 'ant'`。由于 USER_TYPE 是构建时 `--define`，捆绑器常折叠这些检查，在外部构建中死代码消除。

---

### Agent Context（agentContext.ts）

与 `workloadContext.ts` 相同模式——使用 `AsyncLocalStorage` 而不是全局槽，因为 void-detached 后台代理在第一个 await 时 yield。

---

### 其余 170 个 Utils 文件的分类总结

通过大探索代理的完整扫描，这些文件大多数是小型工具函数。按类别分组：

#### 字符串/文本处理
| 文件 | 用途 |
|------|------|
| `stringUtils.ts` | 通用字符串工具 |
| `words.ts` | 词/字符操作 |
| `sliceAnsi.ts` | ANSI-aware 字符串切片 |
| `truncate.ts` | 截断函数 |
| `format.ts` | 格式化函数 |
| `highlightMatch.tsx` | 匹配高亮 |
| `textHighlighting.ts` | 文本高亮 |

#### 平台/路径
| 文件 | 用途 |
|------|------|
| `platform.ts` | 平台检测 |
| `path.ts` | 路径操作 |
| `windowsPaths.ts` | Windows 特殊路径 |
| `xdg.ts` | XDG 目录 |
| `systemDirectories.ts` | 系统目录常量 |
| `cwd.ts` | 当前工作目录管理 |
| `getWorktreePaths.ts` | Worktree 路径解析 |
| `which.ts` | 可执行文件定位 |
| `findExecutable.ts` | 二进制定位 |

#### JSON/YAML/XML
| 文件 | 用途 |
|------|------|
| `json.ts` | JSON 解析（安全模式，容忍截断） |
| `jsonRead.ts` | JSON 文件读取 |
| `yaml.ts` | YAML 解析 |
| `xml.ts` | XML 常量 |
| `markdown.ts` | Markdown 工具 |
| `markdownConfigLoader.ts` | Markdown 配置加载 |
| `zodToJsonSchema.ts` | Zod schema → JSON schema |

#### 文件 I/O
| 文件 | 用途 |
|------|------|
| `file.ts` | 文件操作 |
| `fsOperations.ts` | 跨平台 FS 操作 |
| `fileRead.ts` | 文件读取 |
| `glob.ts` | Glob 模式匹配 |
| `ripgrep.ts` | Ripgrep 集成 |
| `pdf.ts` + `pdfUtils.ts` | PDF 处理 |
| `notebook.ts` | Jupyter notebook 处理 |
| `readFileInRange.ts` | 范围读取用于 diff |
| `readEditContext.ts` | Read/Edit 操作上下文 |
| `tempfile.ts` | 临时文件（内容寻址标识符防 cache bust） |

#### Git/Worktree
| 文件 | 用途 |
|------|------|
| `gitDiff.ts` | Git diff 解析 |
| `gitSettings.ts` | Git 相关设置 |
| `githubRepoPathMapping.ts` | GitHub 仓库路径映射 |
| `worktree.ts` | Git worktree 操作 |
| `worktreeModeEnabled.ts` | Worktree 模式检测 |

#### 终端/Ink
| 文件 | 用途 |
|------|------|
| `ink.ts` | Ink 工具函数 |
| `fullscreen.ts` | 全屏模式 |
| `horizontalScroll.ts` | 水平滚动 |
| `terminalPanel.ts` | 终端面板 |

#### IDE 集成
| 文件 | 用途 |
|------|------|
| `ide.ts` | IDE 交互 |
| `idePathConversion.ts` | IDE 路径转换 |
| `jetbrains.ts` | JetBrains IDE 集成 |
| `iTermBackup.ts` | iTerm2 标签/窗口备份 |
| `appleTerminalBackup.ts` | Apple Terminal 备份 |

#### 图片处理
| 文件 | 用途 |
|------|------|
| `imagePaste.ts` | 图片粘贴功能 |
| `imageResizer.ts` | 图片压缩 |
| `imageStore.ts` | 图片存储 |
| `imageValidation.ts` | 图片验证 |
| `screenshotClipboard.ts` | 截图剪贴板 |

#### 网络/HTTP
| 文件 | 用途 |
|------|------|
| `http.ts` | HTTP 客户端工具 |
| `proxy.ts` | 代理配置 |
| `mtls.ts` | mTLS 证书 |
| `caCerts.ts` + `caCertsConfig.ts` | CA 证书管理 |
| `stream.ts` | 流处理 |
| `streamJsonStdoutGuard.ts` | JSON stdout 防护 |

#### 身份验证
| 文件 | 用途 |
|------|------|
| `authFileDescriptor.ts` | 文件描述符认证 |
| `authPortable.ts` | 便携认证 |
| `sessionIngressAuth.ts` | 远程会话认证 |
| `billing.ts` | 计费信息 |
| `peerAddress.ts` | 对等方地址 |
| `userAgent.ts` | User agent 构建 |

#### 错误/日志
| 文件 | 用途 |
|------|------|
| `errors.ts` | 错误工具和类 |
| `log.ts` | 轻量日志接口（排队直到 sink 附加） |
| `diagLogs.ts` | 诊断日志 |
| `unaryLogging.ts` | 单次日志 |
| `warningHandler.ts` | Node 警告处理 |
| `debug.ts` + `debugFilter.ts` | 调试工具 |

#### 通用工具
| 文件 | 用途 |
|------|------|
| `uuid.ts` | UUID 生成 |
| `hash.ts` | 哈希函数 |
| `crypto.ts` | 加密工具 |
| `fingerprint.ts` | 环境指纹 |
| `memoize.ts` | 自定义 memoize |
| `withResolvers.ts` | Promise.withResolvers 垫片 |
| `signal.ts` | 信号/事件发射器 |
| `sinks.ts` | 输出槽 |
| `sequential.ts` | 顺序执行器 |
| `generators.ts` | 生成器工具 |
| `set.ts` | Set 工具 |
| `array.ts` / `contentArray.ts` | 数组工具 |
| `objectGroupBy.ts` | 对象分组 |
| `treeify.ts` | 树形结构构建 |
| `semver.ts` | 语义版本 |
| `semanticBoolean.ts` | 语义布尔解析 |
| `semanticNumber.ts` | 语义数字解析 |

---

### 读完后应抓住的 2 个事实

1. **AsyncLocalStorage 而非全局状态用于 Turn-scoped 标签**——`workloadContext.ts` 和 `agentContext.ts` 都用 `AsyncLocalStorage` 而不是全局槽。这因为 void-detached 后台代理在首个 await 时 yield，且父 turn 的同步继续在子恢复前完成。还有一个泄漏修复：先前实现在 `undefined` 时直通（`return fn()`），但泄漏的 ambient 上下文会永远传播。现在始终 `.run()` 建立边界。

2. **Undercover 模式的构建时死代码消除**——所有 undercover 路径门控在 `process.env.USER_TYPE === 'ant'`。由于这是构建时 `--define`，捆绑器将这些检查折叠到常量，在外部构建中完全消除反分支。没有 force-OFF——如果我们不能确定在内部仓库，保持潜伏。这是安全优先设计。
