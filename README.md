# dsh-client-connection-secure

**安全加固版** 的 DeepSeek Harness `client-connection` 包。

- 原始来源：`@deepseek-ai/dsh-client-connection`（deepseek-harness `packages/client/connection`）
- 本仓库：在官方包基础上，**修复了一个允许公网攻击者绕过登录门禁、直接调用 `/api` 读取敏感凭证的安全漏洞**。

---

## ⚠️ 修复的安全漏洞

### 漏洞描述

DeepSeek Harness 自身的 `/api` 信任围栏 `isTrustedApiRequest` 只做 **DNS-rebinding 防护**，**不做认证**（官方注释原话：`trustedHosts is a DNS-rebinding fence, explicitly not authentication`）。

当通过公网入口（如 `dsh.chunbosama.xyz`，被加入 `--trusted-host`）访问时，攻击者只要把请求的 `Host` 头写成受信任域名，就能**无认证**直接调用后端 `/api`，包括：

- `POST /api/session.prompt` — 给 AI Agent 发指令
- `/api/credentials.*` — 读取 / 修改凭证
- `/api/settings.*` — 读取 / 修改配置
- `/api/session.*` — 创建 / 读取会话

**实际攻击**：攻击者利用该缺口，通过 `POST /api/session.prompt` 让 Agent 执行 `cat $HOME/.dsh/.credentials.yaml`，成功读取并泄露了 API Key（DeepSeek / BigModel / Anthropic 网关）。

---

## 🔧 本仓库的修复内容

### 修复重点：`lib/index.js`

在 `isTrustedApiRequest` 中，对**非回环来源**（即公网请求）的 `/api` 访问，**强制要求携带一个真实有效的 dsh-internet-access 认证 cookie**（访问口令或用户名+密码会话），否则返回 403：

```javascript
// 原逻辑：只要 Host 是 trustedHosts 且来源不是 cross-site 就放行
if (!isLoopbackHostname(hostUrl.hostname) && !isTrustedAuthority(hostUrl, trustedHosts)) return false;
if (header(request.headers, "sec-fetch-site") === "cross-site") return false;
const origin = header(request.headers, "origin");
if (origin === undefined) return true;   // ← 无认证直接放行（漏洞）

// 修复后：非回环来源必须通过真实 cookie 校验
if (!isLoopbackHostname(hostUrl.hostname) && !isTrustedAuthority(hostUrl, trustedHosts)) return false;
if (!isLoopbackHostname(hostUrl.hostname)) {
  if (!iaValidAuthCookie(request)) return false;   // ← 无有效 cookie 拒绝
}
```

新增的 `iaValidAuthCookie` 会**真实对照** `$DSH_HOME/public-access-token.json` 和 `$DSH_HOME/internet-access-sessions.json`，用 **timing-safe 比较**校验 cookie 值（不是只检查 cookie 是否存在）：

- ✅ 访问口令 cookie（`dsh_public_access=<真实token>`）通过
- ✅ 用户名+密码会话 cookie（未过期）通过
- ❌ 无 cookie、伪造 cookie、过期会话 → 403 拒绝
- ✅ 本机回环访问（`localhost` / `127.0.0.1`）**不受影响**，正常放行

### 附带清理：`lib/client.js`

移除了此前遗留的浏览器端硬编码域名信任补丁：

```javascript
// 已移除
const LOOPBACK_TRUSTED_HOSTS = ["dsh.chunbosama.xyz"];
```

浏览器端 `isLoopback` 判定恢复到官方基线（仅 `localhost` / 回环地址视为本机）。公网访问的认证交由服务端 `index.js` 的 cookie 校验统一处理。

---

## 📦 安装 / 使用

本包需要以「覆盖」或「替换」原 `@deepseek-ai/dsh-client-connection` 的方式接入 DSH 环境。典型场景：在 DeepSeek Harness 公网部署中，将其作为本地 patch 覆盖安装，配合 `dsh-internet-access` 插件实现真正的公网认证门禁。

> 依赖：`ws`、`@deepseek-ai/schemastery`（peer: 一系列 `@deepseek-ai/dsh-*` 包）。

---

## ✅ 安全验证

对修复后的版本做过完整验证（`Host: dsh.chunbosama.xyz`）：

| 场景 | 预期 | 结果 |
|------|------|------|
| 无 cookie 调 `/api/session.list` | 403 | ✅ |
| 伪造 cookie 调 `/api/session.list` | 403 | ✅ |
| 空 cookie 调 `/api/session.list` | 403 | ✅ |
| 前缀近似 cookie | 403 | ✅ |
| 接近真值 cookie | 403 | ✅ |
| **真实口令 cookie** | **200** | ✅ |
| 本机回环 `localhost` | 200 | ✅ |
| 本机回环 `127.0.0.1` | 200 | ✅ |

---

## 📄 License

MIT（保留自官方原包）

## 🔗 源仓库

DeepSeek Harness 官方：`git+https://github.com/deepseek-ai/deepseek-harness.git`（`packages/client/connection`）
