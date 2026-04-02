## 第 75 站：`src/utils/hooks/execHttpHook.ts`

### 这是什么文件

`src/utils/hooks/execHttpHook.ts` 是 hooks 子系统里的 **HTTP hook 执行器与安全策略层：负责把 hook 输入 JSON POST 到配置的 URL，执行两层 URL 白名单策略（全局 HTTP hook 策略 + sandbox 代理域名白名单），通过 SSRF 防护和代理感知路由确保请求不会打到内网地址，同时 header 值支持环境变量插值但受白名单约束并带有 CRLF/NUL 清洗。**

这一站是三大特殊 hook 执行器中的第一站。此前 `hooks.ts` 已经把 `type: 'http'` 执行委托给了 `execHttpHook.ts`。

所以最准确的一句话是：

```text
execHttpHook.ts = HTTP hook 的安全执行层：负责 URL 白名单校验、header 环境变量安全插值、sandbox/env代理路由、SSRF 防护 POST 以及超时/取消/错误处理
```

---

### 先看它的整体定位

位置：`src/utils/hooks/execHttpHook.ts:1`

这个文件不是：

- hook schema 定义
- hook runtime 调度器
- hook 事件层
- generic HTTP client wrapper

它的职责是：

1. 在发请求前校验 URL 白名单
2. 构建 header 带环境变量插值
3. 选择代理路由（sandbox proxy / env proxy / 直连）
4. 配置 SSRF 防护 DNS lookup
5. POST hook 输入
6. 处理超时/取消/错误
7. 返回结构化结果

所以它本质上是一个：

```text
security-conscious HTTP hook POST executor
```

---

### 第一部分：`getSandboxProxyConfig()` 用动态 import 避免循环依赖，说明 HTTP hook 的沙箱代理初始化是个敏感路径——必须在 sandbox 网络就绪后才可用

位置：`src/utils/hooks/execHttpHook.ts:21`

```ts
const { SandboxManager } = await import('../sandbox/sandbox-adapter.js')
```

注释解释了原因：

```text
dynamic import to avoid a static import cycle
```

而且它不只是取端口，还会：

```ts
await SandboxManager.waitForNetworkInitialization()
```

这说明在 REPL 模式下 sandbox 网络初始化是异步的，
第一个 HTTP hook 可能在网络还没就绪时就触发。

所以这个文件采用了一种典型的"等到就绪"策略，
而不是"假设已经就绪"。

---

### 第二部分：`getHttpHookPolicy()` 读取的是全局安全策略层面的两层限制：哪些 URL 允许、哪些环境变量允许被注入 header

位置：`src/utils/hooks/execHttpHook.ts:49`

返回：

```ts
{
  allowedUrls: string[] | undefined,
  allowedEnvVars: string[] | undefined
}
```

注释特别重要：

```text
Follows the allowedMcpServers precedent: arrays concatenate across sources.
When allowManagedHooksOnly is set, only admin-defined hooks run anyway,
so no separate lock-down boolean is needed here.
```

这说明 HTTP hook 安全策略沿用了 MCP server 白名单的已有模式。

---

### 第三部分：`urlMatchesPattern(...)` 实现了一个简单的 `*` 通配符匹配，说明 URL 白名单策略的设计是"pattern-based"而不需要正则表达式

位置：`src/utils/hooks/execHttpHook.ts:64`

```ts
const escaped = pattern.replace(/[.+?^${}()|[\]\\]/g, '\\$&')
const regexStr = escaped.replace(/\*/g, '.*')
return new RegExp(`^${regexStr}$`).test(url)
```

注释：

```text
Same semantics as the MCP server allowlist patterns
```

这说明通配符模式是一套跨子系统复用的设计约定。

---

### 第四部分：`sanitizeHeaderValue(...)` 非常关键，它清洗掉 `CR`、`LF` 和 `NUL` 字节，专门防止 HTTP header 注入攻击

位置：`src/utils/hooks/execHttpHook.ts:76`

注释给出了具体攻击：

```text
A malicious env var like "token\r\nX-Evil: 1"
would otherwise inject a second header
```

这说明 header 值的来源不只是 hook 配置本身，
还可能来自环境变量——
恶意 env var 可以注入额外请求头。

所以清洗在这里是安全必须的。

---

### 第五部分：`interpolateEnvVars(...)` 是整份文件最核心的安全函数之一

位置：`src/utils/hooks/execHttpHook.ts:89`

它做了三件事：

1. 解析 `$VAR` 和 `${VAR}` 模式
2. 只在 `allowedEnvVars` 白名单里的变量才插值；不在白名单里替换为空字符串
3. 最后再调用 `sanitizeHeaderValue` 清洗结果

```ts
/\$\{([A-Z_][A-Z0-9_]*)\}|\$([A-Z_][A-Z0-9_]*)/g
```

正则只匹配大写变量名。

这说明 HTTP hook header 里的环境变量注入是：

```text
whitelist-required, sanitizing, fail-closed (empty)
```

而不是：

```text
passthrough all env
```

---

### 第六部分：主函数 `execHttpHook(...)` 的策略是"先校验、再构建、再 POST"——URL 白名单在所有 I/O 之前就被强制执行

位置：`src/utils/hooks/execHttpHook.ts:123`

执行序列是：

#### 1. URL 策略校验（在任何 I/O 之前）

```ts
if (policy.allowedUrls !== undefined) {
  const matched = policy.allowedUrls.some(p => urlMatchesPattern(hook.url, p))
  if (!matched) {
    return { ok: false, body: '', error: '...' }
  }
}
```

注释说：

```text
undefined → no restriction
[] → block all
non-empty → must match
```

这是"fail-closed"的 URL 策略。

#### 2. 计算超时

```ts
const timeoutMs = hook.timeout
  ? hook.timeout * 1000
  : DEFAULT_HTTP_HOOK_TIMEOUT_MS // 10 minutes
```

#### 3. 创建组合 abort signal

```ts
const { signal: combinedSignal, cleanup } = createCombinedAbortSignal(
  signal,
  { timeoutMs }
)
```

#### 4. 构建 header 带环境变量插值

```ts
const effectiveVars =
  policy.allowedEnvVars !== undefined
    ? hookVars.filter(v => policy.allowedEnvVars!.includes(v))
    : hookVars
```

这又是两层安全策略的交叉：
hook 配置的变量必须也在全局策略白名单里才被允许。

#### 5. 检测代理

- sandbox proxy
- env-var proxy (`HTTP_PROXY` / `HTTPS_PROXY` / `NO_PROXY`)

#### 6. axios POST

```ts
const response = await axios.post<string>(hook.url, jsonInput, {
  headers,
  signal: combinedSignal,
  responseType: 'text',
  validateStatus: () => true, // 接收所有状态码
  maxRedirects: 0,            // 不跟随重定向
  proxy: sandboxProxy ?? false,
  lookup: sandboxProxy || envProxyActive ? undefined : ssrfGuardedLookup,
})
```

#### 7. 结果

```ts
return {
  ok: 200 <= status < 300,
  statusCode,
  body
}
```

#### 8. catch

- 被中止 -> `{aborted: true}`
- 其他错误 -> 返回错误信息

---

### 第七部分：`validateStatus: () => true` 和 `maxRedirects: 0` 说明 HTTP hook 客户端选择"原始响应接收"而不是跟随 axios 默认行为

位置：`src/utils/hooks/execHttpHook.ts:205`

- `validateStatus: () => true` 意味着即使是 500 也不抛异常
- `maxRedirects: 0` 意味着不跟随任何重定向

这很重要，因为：

- 403 from sandbox proxy 也应该被正常处理
- 跟随重定向可能导致 SSRF 规避行为被旁路
- 调用方需要看到原始响应码做自己的逻辑

---

### 第八部分：代理感知 SSRF 防护策略

位置：`src/utils/hooks/execHttpHook.ts:216`

```ts
lookup: sandboxProxy || envProxyActive ? undefined : ssrfGuardedLookup
```

注释解释了关键安全逻辑：

```text
Skipped when any proxy is in use — the proxy performs DNS for the target,
and applying the guard would instead validate the proxy's own IP, breaking
connections to corporate proxies on private networks.
```

也就是说 SSRF guard 的逻辑是：

#### 无代理

```text
用 ssrfGuardedLookup 在 DNS 级别验证 IP
-> 只允许回环（本地开发）
-> 拒绝私有/链路本地地址
```

#### 有代理（sandbox 或 env）

```text
不用本地 ssrfGuardedLookup
-> 代理自身做域名白名单
-> 避免误拦企业代理的私有 IP
```

这表明系统设计了一个"SSRF 责任委托"模型：

- proxy 在 -> SSRF 职责交给 proxy
- proxy 不在 -> 系统自身用 guarded lookup

---

### 第九部分：环境变量代理检测的逻辑

位置：`src/utils/hooks/execHttpHook.ts:184`

```ts
const envProxyActive =
  !sandboxProxy &&
  getProxyUrl() !== undefined &&
  !shouldBypassProxy(hook.url)
```

这说明代理检测有三个约束：

1. 如果 sandbox proxy 已启用 -> 不用 env proxy
2. 必须设置了 `HTTP_PROXY` / `HTTPS_PROXY`
3. 目标 URL 不在 `NO_PROXY` 白名单中

这种代理优先级是典型的：

```text
sandbox proxy > env proxy > direct
```

---

### 第十部分：`createCombinedAbortSignal(signal, { timeoutMs })` 说明 HTTP hook 的超时和外部取消信号被合并成一个 signal

位置：`src/utils/hooks/execHttpHook.ts:151`

这意味着：

- 如果调用方的 signal abort -> axios 取消
- 如果超时到了 -> axios 取消
- 任何一个触发就取消

这比"只用超时信号或只用外部信号"更安全。

---

### 第十一部分：两层环境变量插值策略的交叉

位置：`src/utils/hooks/execHttpHook.ts:162`

```ts
const hookVars = hook.allowedEnvVars ?? []
const effectiveVars =
  policy.allowedEnvVars !== undefined
    ? hookVars.filter(v => policy.allowedEnvVars!.includes(v))
    : hookVars
```

这说明即使 hook 配置了 `allowedEnvVars: ["SECRET_TOKEN"]`，
如果全局策略说 `allowedEnvVars: ["NONSECRET"]`，
那结果是交集：空。

这又是一种"安全策略必须同时满足两层约束"的模型。

---

### 第十二部分：整份文件最核心的架构价值，是把 HTTP hook 的执行从通用 HTTP 请求中完全分离出来，成为一个自带 URL 策略、header 注入防护、代理路由和 SSRF 防护的安全执行层

所以最准确的压缩表达是：

```text
execHttpHook.ts = HTTP hook 的安全执行层：在 POST 前校验 URL 白名单，header 值使用白名单环境变量插值并清洗注入字符，通过 sandbox/env 代理路由选择路径，在无代理时用 SSRF-guarded lookup 防止内网请求，并处理超时/取消/错误
```

---

### 读完这一站后，你应该抓住的 10 个事实

1. `src/utils/hooks/execHttpHook.ts` 不是通用 HTTP 客户端，而是专门用于 HTTP hooks 的安全执行层。
2. URL 白名单在所有 I/O 之前被强制校验：`undefined` = 无限制，`[]` = 全部屏蔽，非空 = 必须匹配通配符 pattern。
3. Header 值支持 `$VAR` 和 `${VAR}` 环境变量插值，但只在 hook 配置的全局策略双重白名单交叉后才允许；不在白名单的变量替换为空字符串。
4. `sanitizeHeaderValue(...)` 清洗掉 CR/LF/NUL 字节，防止 `token\r\nX-Evil: 1` 这类 CRLF 注入攻击通过恶意 env var 注入额外 header。
5. 代理路由采用三层优先级：sandbox proxy > env-var proxy > 直连。
6. SSRF 防护是代理感知的：无代理时用 `ssrfGuardedLookup` 在 DNS 层验证 IP 不是私有/链路本地地址；有代理时跳过 SSRF guard（代理自身做 DNS 和白名单）。
7. `validateStatus: () => true` 和 `maxRedirects: 0` 意味着 axios 从不因错误状态码或重定向而抛异常——调用方始终看到原始响应。
8. 超时默认 10 分钟，可通过 hook 配置覆盖；超时与外部取消信号被合并成单个 abort signal。
9. URL 通配符匹配沿用了 MCP server allowlist 的 pattern 语义，确保 hook 和其他安全白名单机制的模式一致。
10. 这个文件是 hooks 子系统中唯一直接执行 SSRF 防护的地方——command/prompt/agent hooks 不通过 HTTP 客户端访问外部网络。

---

### 下一站建议

下一站是三大特殊执行器中的第二站：

**`src/utils/hooks/execPromptHook.ts` 怎样通过 LLM 评估 prompt hooks。**

> [!note] 注意: 我在写这份笔记时注意到这份文件只有 242 行，所以整站已经分析完毕。
