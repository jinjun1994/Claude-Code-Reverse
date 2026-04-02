## 第 79 站：`src/utils/hooks/ssrfGuard.ts`

### 这是什么文件

`src/utils/hooks/ssrfGuard.ts` 是 HTTP hooks 的 **SSRF 防护底层：在 DNS 解析层面阻止 HTTP hook 请求打到私有/链路本地/云元数据地址，同时明确允许回环地址（本地开发），通过自定义 axios `lookup` 函数在 socket 连接前就拦截危险 IP，消除验证到连接之间的 TOCTOU 窗口。**

这个文件是 `execHttpHook.ts` 中提到的 `ssrfGuardedLookup` 的具体实现位置。

所以最准确的一句话是：

```text
ssrfGuard.ts = HTTP hook 的 DNS 层 SSRF 防护：通过自定义 dns lookup 函数在解析阶段拦截私有 IP、IPv4-mapped IPv6、CGNAT 等危险地址范围，只允许回环和公共地址
```

---

### 先看它的整体定位

位置：`src/utils/hooks/ssrfGuard.ts:1`

这个文件不是：

- HTTP 客户端
- hook 执行器
- 通用安全中间件

它的职责非常集中：

1. 定义 `isBlockedAddress(address)` 判断 IP 是否在拦截范围内
2. 实现 IPv4 私有/危险地址范围检测
3. 实现 IPv6 私有/危险地址范围检测（含 IPv4-mapped 提取）
4. 导出 `ssrfGuardedLookup(hostname, options, callback)` 作为 axios `lookup` 选项
5. 拦截的地址抛出自定义错误码 `ERR_HTTP_HOOK_BLOCKED_ADDRESS`

所以它本质上是一个：

```text
dns-level SSRF guard for HTTP hook requests
```

---

### 第一部分：设计意图注释

位置：`src/utils/hooks/ssrfGuard.ts:5`

文件开头的注释说明：

```text
Blocks private, link-local, and other non-routable address ranges
to prevent project-configured HTTP hooks from reaching
cloud metadata endpoints (169.254.169.254) or internal infrastructure.

Loopback (127.0.0.0/8, ::1) is intentionally ALLOWED
— local dev policy servers are a primary HTTP hook use case.
```

这说明：

- 主要攻击面：项目配置的 HTTP hooks 被恶意指向 `169.254.169.254`
  云元数据 endpoint
- 允许回环，因为本地开发场景下 hooks 经常需要调用本地服务
- 代理模式下 guard 会被绕过（因为代理自己做 DNS 和白名单）

---

### 第二部分：`isBlockedAddress(address)` 入口

位置：`src/utils/hooks/ssrfGuard.ts:42`

```ts
export function isBlockedAddress(address: string): boolean {
  const v = isIP(address)
  if (v === 4) return isBlockedV4(address)
  if (v === 6) return isBlockedV6(address)
  return false
}
```

注意非 IP 地址（域名等）返回 `false`——
这个函数只处理已经是 IP 字面量的地址。

---

### 第三部分：IPv4 拦截规则

位置：`src/utils/hooks/ssrfGuard.ts:55`

#### Blocked
| 范围 | CIDR | 原因 |
|---|---|---|
| 0.0.0.0/8 | `a === 0` | "this" network |
| 10.0.0.0/8 | `a === 10` | private |
| 100.64.0.0/10 | `a === 100 && 64 <= b <= 127` | CGNAT（部分云元数据，如阿里云 100.100.100.200） |
| 169.254.0.0/16 | `a === 169 && b === 254` | link-local / cloud metadata |
| 172.16.0.0/12 | `a === 172 && 16 <= b <= 31` | private |
| 192.168.0.0/16 | `a === 192 && b === 168` | private |

#### Allowed
| 范围 | CIDR | 原因 |
|---|---|---|
| 127.0.0.0/8 | `a === 127` | 回环（本地开发用） |

注意 127.x.x.x 整个回环范围都被放行，
不是只放行 127.0.0.1。

---

### 第四部分：IPv6 拦截规则

位置：`src/utils/hooks/ssrfGuard.ts:88`

#### 阻止
- `::` — unspecified
- `fc00::/7` — unique local（`fc` 或 `fd` 开头）
- `fe80::/10` — link-local（`fe80` 到 `febf`）
- `::ffff:<v4>` — IPv4-mapped IPv6 地址

#### 允许
- `::1` — loopback

---

### 第五部分：IPv4-mapped IPv6 地址的处理

位置：`src/utils/hooks/ssrfGuard.ts:97-104`

```ts
const mappedV4 = extractMappedIPv4(lower)
if (mappedV4 !== null) {
  return isBlockedV4(mappedV4)
}
```

注释：

```text
Without this, hex-form mapped addresses
(e.g. ::ffff:a9fe:a9fe = 169.254.169.254)
bypass the guard.
```

这说明如果只检查前缀，
`::ffff:a9fe:a9fe` 会被放过，
但展开后就是 `169.254.169.254`（AWS 元数据）。

所以这个文件必须实现一个完整的 IPv6 展开函数。

---

### 第六部分：`expandIPv6Groups` 是完整的 IPv6 解压缩函数

位置：`src/utils/hooks/ssrfGuard.ts:133`

它处理：
1. 尾部点分十进制 IPv4 地址（`::ffff:169.254.169.254`）
2. 双冒号扩展（`::` -> 适当数量的零组）
3. 填充到 8 个组
4. 验证每个组在 0-0xffff 范围内

这说明作者不是简单检查前缀字符串，
而是完全展开 IPv6 地址。

---

### 第七部分：`ssrfGuardedLookup` 是核心导出函数

位置：`src/utils/hooks/ssrfGuard.ts:216`

```ts
export function ssrfGuardedLookup(
  hostname: string,
  options: object,
  callback: (...) => void,
): void
```

这个函数的工作流是：

1. 检测 `options.all` 是否需要返回所有地址
2. 如果 hostname 已经是 IP 字面量 -> 直接验证
3. 否则调用 `dnsLookup(hostname, { all: true }, ...)`
4. 对所有返回的地址逐一验证 `isBlockedAddress`
5. 如果任何一个在拦截范围内 -> 回调错误
6. 否则正常返回

注释中的关键设计：

```text
Used as the lookup option in axios request config
so that the validated IP is the one the socket
connects to — no rebinding window between
validation and connection.
```

这说明防护发生在 DNS 层，
不是先解析再验证。
DNS 返回的结果就已经被清洗过。

---

### 第八部分：错误类型

位置：`src/utils/hooks/ssrfGuard.ts:285`

```ts
const err = new Error(
  `HTTP hook blocked: ${hostname} resolves to ${address}
   (private/link-local address).
   Loopback (127.0.0.1, ::1) is allowed for local dev.`
)
Object.assign(err, {
  code: 'ERR_HTTP_HOOK_BLOCKED_ADDRESS',
  hostname,
  address,
})
```

错误消息包含：
- hostname
- 解析出的危险 IP
- 说明回环仍然允许

---

### 读完这一站后，你应该抓住的 6 个事实

1. `ssrfGuard.ts` 定义了 DNS 级别的 SSRF 防护，用于 HTTP hook 请求。
2. IPv4 拦截范围包括：0.0.0.0/8、10.0.0.0/8、100.64.0.0/10 (CGNAT)、169.254.0.0/16 (link-local)、172.16.0.0/12、192.168.0.0/16。
3. 127.0.0.0/8 整个回环范围被明确允许，因为本地开发是 HTTP hooks 的主要使用场景。
4. IPv6 拦截范围包括：`::`、`fc00::/7` (ULA)、`fe80::/10` (link-local)、以及 IPv4-mapped IPv6（通过完整的 IPv6 解压缩验证）。
5. `ssrfGuardedLookup` 是 axios `lookup` 选项的直接替代品，确保 DNS 解析结果在 socket 连接前就被验证，消除 TOCTOU 窗口。
6. 当代理启用时（sandbox 或 env-var proxy），`execHttpHook.ts` 跳过 SSRF guard，因为代理自己负责 DNS 和白名单。

---

### 下一站建议

接下来还有几个相关文件可以读：

- `src/utils/hooks/registerFrontmatterHooks.ts` — frontmatter hook 注册器
- `src/utils/hooks/registerSkillHooks.ts` — skill hook 注册器
- `src/utils/hooks/hooksConfigSnapshot.ts` — 配置快照
- `src/utils/hooks/fileChangedWatcher.ts` — 文件变更 watcher
- `src/utils/hooks/postSamplingHooks.ts` — 后采样 hooks
- `src/utils/hooks/skillImprovement.ts` — skill 改进
- `src/utils/hooks/apiQueryHookHelper.ts` — API 查询辅助

最自然的下一站是：

**`src/utils/hooks/registerFrontmatterHooks.ts` — 负责从 CLAUDE.md / instruction files 的 frontmatter 中注册 hooks。**
