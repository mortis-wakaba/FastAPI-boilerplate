<a id="sessions"></a>

# 会话

会话是模板默认的认证机制，内置 API 路由使用会话认证。

<a id="auth-architecture"></a>

## 认证架构

认证由 [`crudauth`](https://pypi.org/project/crudauth/) 提供，组装入口是 `infrastructure/auth/setup.py` 中的单例：

```python
# infrastructure/auth/setup.py
auth = CRUDAuth(session=async_session, user_model=User, SECRET_KEY=settings.SECRET_KEY, ...)
```

路由和依赖在导入时引用 `auth`。应用生命周期通过 `app_factory` 在启动时调用 `auth.initialize()`，关闭时调用 `auth.shutdown()`，建立和释放会话后端连接。下文的依赖、登录、CSRF、锁定和存储都由该单例提供，模板只负责组装及路由处理。

<a id="protecting-routes"></a>

## 保护路由

导入会话依赖并添加到路由：

```python
from typing import Annotated, Any
from fastapi import APIRouter, Depends

from ...infrastructure.auth.dependencies import get_current_user

router = APIRouter()


@router.get("/my-profile")
async def get_profile(
    current_user: Annotated[dict[str, Any], Depends(get_current_user)],
) -> dict[str, Any]:
    return {"user_id": current_user["id"], "email": current_user["email"]}
```

无有效会话时返回 `401 Unauthorized`。

<a id="available-dependencies"></a>

### 可用依赖

均位于 `src/infrastructure/auth/dependencies.py`，封装 crudauth 单例。Cookie 校验、CSRF 和锁定由库完成，处理器继续使用普通用户字典。

**`get_current_user`**：返回已认证用户字典，未认证抛出 401。

```python
@router.get("/dashboard")
async def dashboard(
    current_user: Annotated[dict[str, Any], Depends(get_current_user)],
) -> dict[str, Any]:
    return {"welcome": current_user["username"]}
```

**`get_current_superuser`**：额外检查 `is_superuser=True`，非超级用户返回 403。

```python
@router.delete("/users/{user_id}")
async def delete_user(
    user_id: int,
    current_user: Annotated[dict[str, Any], Depends(get_current_superuser)],
) -> None:
    # Only superusers reach this code
    ...
```

**`get_optional_user`**：已认证返回用户字典，否则 None。原文将其描述为不抛异常的变体。

```python
@router.get("/products")
async def list_products(
    current_user: Annotated[dict[str, Any] | None, Depends(get_optional_user)],
) -> list[dict[str, Any]]:
    if current_user:
        # Personalize for logged-in users
        ...
```

**`get_current_principal`**：返回经过会话和 CSRF 校验的 crudauth `Principal`。需要 `principal.metadata["session_id"]` 或直接使用 `user_id` 时使用。`get_optional_principal` 是可选变体。

**`get_current_permissions`**：以 `frozenset[str]` 返回用户角色赋予的有效权限。FastAPI 在请求内缓存，多次保护只查一次。超级用户拥有全部已注册权限。

```python
@router.patch("/{username}")
async def update_user_profile(
    username: str,
    permissions: Annotated[frozenset[str], Depends(get_current_permissions)],
) -> dict[str, str]:
    if "user.update" in permissions:
        ...
```

**`require_permissions(*names)`**：缺少任意权限时返回 403，超级用户通过。不注入值，应放在路由 `dependencies` 中：

```python
from ...infrastructure.auth.authorization import require_permissions


@router.get("/", dependencies=[require_permissions("user.read")])
async def get_users(...) -> dict[str, Any]:
    ...
```

权限声明见[权限](permissions.md#role-based-permissions)。

<a id="protecting-entire-routers"></a>

### 保护整个路由器

为路由器全部接口应用认证：

```python
router = APIRouter(
    prefix="/admin",
    dependencies=[Depends(get_current_superuser)],
)


@router.get("/stats")
async def stats() -> dict[str, Any]:
    # Already authenticated at the router level
    ...
```

路由器级依赖不向处理器注入值。处理器需要用户对象时，仍在该路由添加 `Depends(get_current_user)`。

<a id="how-sessions-work"></a>

## 会话工作方式

`POST /api/v1/auth/login` 委托给 crudauth 单例（见[认证架构](#auth-architecture)），依次：

1. 检查 IP 和登录标识锁定，超限返回 `429` 和 `Retry-After`。
2. 校验用户凭据，拒绝 `is_active == False` 的软删除用户。
3. 在默认 Redis 等配置后端写会话，使用会话 ID 的 HMAC 而非原始 ID 作为键。
4. 生成绑定会话的 CSRF 令牌。
5. 设置两个 Cookie：HttpOnly 的 `session_id`，以及 JavaScript 可读、与 JSON 响应一致的 `csrf_token`。

后续请求中，crudauth 依赖：

1. 从 Cookie 读取 `session_id`。
2. 用同样 HMAC 查询后端，拒绝缺失或过期会话。
3. `CSRF_ENABLED=true` 时，对 POST / PUT / DELETE / PATCH 校验 CSRF。
4. 返回 `Principal`，`get_current_user` 再加载完整用户行，并通过 `lazy="selectin"` 加载 `Tier`。

重新登录会结束浏览器原先携带的会话，被复制的旧 Cookie 在账户下次登录后失效。

存储不保留原始会话 ID 和 CSRF 令牌，两者使用以 `SECRET_KEY` 为密钥的 HMAC。只读访问存储无法获得可登录凭据，更改 `SECRET_KEY` 会使所有人退出。

crudauth 管理路由与项目路由并列挂载：`GET /sessions`、`DELETE /sessions/{handle}`、`POST /logout-all`、`POST /csrf/refresh`，见[设备管理](#manage-devices)。

`POST /api/v1/auth/logout` 调用会话传输的 `complete_logout`，删除会话、清除各配置传输的 Cookie，并以结束会话的 handle 调用 `on_after_logout`。通过 `AuthHooks` 注册的审计可同时看到登录和退出。怀疑泄露时，使用 `POST /api/v1/auth/logout-all` 结束全部设备会话，见[退出全部会话](#logout-all-sessions)。

密码重置完成和邮箱变更确认会自动结束全部会话，后者由 `infrastructure/auth/evictions.py` 实现，因为邮箱是账户恢复凭据。这两种操作通过邮件链接确认，不保留发起会话。`change-password` 则保留当前会话，因为它携带当前密码。

<a id="two-timeouts"></a>

### 两种超时

`SESSION_TIMEOUT_MINUTES` 是空闲超时，认证请求会续期，持续使用的会话不会因此结束。`SESSION_ABSOLUTE_TIMEOUT_HOURS` 限制自登录起的总时长，可要求定期重新登录，默认未设置，仅使用空闲超时。小于 1 的值在启动拒绝，避免新会话立即过期。达到上限会删除会话，下次请求返回 `401`。

<a id="asking-for-a-fresh-sign-in"></a>

### 要求近期重新登录

敏感操作可能要求持有 Cookie 的人再次证明身份。设置 `FRESH_SIGN_IN_SECONDS` 后，要求新鲜会话的路由会对登录时间过久的会话返回 `403 Sign in again to continue.`，客户端重新走 `/api/v1/auth/login` 后重试。只有会话可以满足，API 密钥等其他凭据不能。

未设置时，关闭账户不要求重新登录；crudauth 的 `/set-password` 和 MFA 注册路由在挂载时仍使用 10 分钟默认值，本项目未挂载这两者。设置后所有相关路由使用同一窗口。

本项目的 `DELETE /api/v1/users/{username}` 受此限制。自定义路由可用 `FreshSessionPrincipalDep` 替代 `SessionPrincipalDep`：

```python
from src.infrastructure.auth.deps import FreshSessionPrincipalDep


@router.post("/billing/close")
async def close_billing(principal: FreshSessionPrincipalDep) -> dict[str, str]: ...
```

<a id="csrf-protection"></a>

## CSRF 防护

会话认证内置 CSRF。原文说明非 GET 请求可通过以下方式提交：

- 浏览器自动发送的 `csrf_token` Cookie。
- JavaScript 客户端常用的 `X-CSRF-Token` 请求头。

```javascript
const csrfToken = getCookie('csrf_token');

await fetch('/api/v1/users/', {
    method: 'POST',
    credentials: 'include',          // include cookies cross-origin
    headers: {
        'X-CSRF-Token': csrfToken,
        'Content-Type': 'application/json',
    },
    body: JSON.stringify(data),
});
```

会话中需要刷新令牌时调用 `POST /api/v1/auth/csrf/refresh`，有效令牌原样返回，仅在缺失或过期时新建，并设置 Cookie。

<a id="passwords"></a>

## 密码

crudauth 账户路由挂载在 `/api/v1/auth`：

| 路由 | 行为 |
|------|------|
| `POST /api/v1/auth/change-password` | 校验 `current_password`，设置 `new_password`，推进 `token_version`，撤销其他会话并保留当前会话；每账户每小时最多 5 次，成功也计数 |
| `GET /api/v1/auth/me` | 返回解析的身份：ID、用户名、邮箱、超级用户、scope、传输方式 |

```bash
curl -X POST http://localhost:8000/api/v1/auth/change-password \
  -b cookies.txt \
  -H "Content-Type: application/json" \
  -H "X-CSRF-Token: <token>" \
  -d '{"current_password": "<current>", "new_password": "<new>"}'
```

当前密码错误返回 `401`，缺少 `X-CSRF-Token` 返回 `403`，新密码不符合策略返回 `422`。改密与邮箱变更共用每账户每小时 5 次预算，所有校验密码的调用都计数，超过后返回 `429`。

crudauth 还有面向无密码账户的 `POST /set-password`，本应用**未挂载**，避免持有借用会话的人给纯 OAuth 账户添加密码。首次设置密码应使用重置链接，先证明邮箱归属：

```bash
curl -X POST http://localhost:8000/api/v1/auth/password/reset-request \
  -H "Content-Type: application/json" -d '{"email": "you@example.com"}'
# then POST the token from the link to /api/v1/auth/password/reset-confirm
```

之后可用提供方或密码登录，见[恢复流程](index.md#recovery-flows-email)。

旧系统哈希可通过 `legacy_verifiers` 登录，并在首次成功时转为 crudauth 格式，见[迁移密码](migrating-passwords.md)。

开发测试中可设置 `CSRF_ENABLED=false`。

<a id="device-tracking"></a>

## 设备跟踪

crudauth 在会话内记录 IP、User-Agent 和时间戳。本节原文称模板没有设备列表路由或 `SessionData` Schema，需要活动会话界面时使用 `auth.sessions` 构建。

> 译注：此描述与本页后文已有设备管理接口的说明不一致。

<a id="login-lockout"></a>

## 登录锁定

crudauth 按 IP 和登录标识**逐级锁定**。达到阈值，`/api/v1/auth/login` 返回带 `Retry-After` 的 `429 Too Many Requests`。四个配置通过 `infrastructure/auth/setup.py` 的 `LockoutConfig` 传入：

```env
LOGIN_MAX_ATTEMPTS=5             # failures allowed inside the window, per address and per account
LOGIN_ATTEMPT_WINDOW_SECONDS=900 # how long failures keep counting
LOGIN_LOCKOUT_BASE_SECONDS=300   # first lockout, doubling each round
LOGIN_LOCKOUT_MAX_SECONDS=3600   # ceiling for the doubling
```

时间窗口对缓慢攻击尤其重要：crudauth 默认只计 60 秒，因此每分钟五次不会累积。成功登录清除账户失败记录及其对来源地址造成的失败压力。管理后台使用[自己的两套策略](../admin-panel/configuration.md#login-authentication)，阈值相同，但两个入口互不累计。代理后设置 `TRUSTED_PROXY_HOPS` 以识别真实 IP。

`RATE_LIMITER_BACKEND=redis` 时计数在 Redis，crudauth 锁定配置为 `fail_open=False`。Redis 不可达时，**所有登录都被拒绝**，返回基础锁定窗口的 `429`，而非绕过限制。把限流 Redis 视为登录必要依赖并监控：不可达时 `GET /health/ready` 返回 `503`，日志标记 `rate_limiter`，与数据库和会话存储并列。限流使用独立 `RATE_LIMITER_REDIS_*` 连接，缓存使用 `CACHE_REDIS_*`；多个配置以相同凭据指向同一服务器时，就绪探针只查询一次。`memory` 在进程内计数，只适合单 Worker；`database` 使用共享 `crudauth_counters`，由数据库就绪检查覆盖，`rate_limiter` 不再报告额外网络目标。

<a id="what-a-refusal-records"></a>

### 拒绝记录什么

每次密码登录拒绝都通过 crudauth 的 `on_login_failed` 和 `on_lockout` 在 `src.infrastructure.auth.audit` 记录一条 `WARNING`：

```
login refused at the login api: 'ada' from 203.0.113.7 (curl/8.4.0); invalid_credentials
login refused at the login api: an identifier no account has from 203.0.113.7 (curl/8.4.0); invalid_credentials
login locked out at the login api for 300s, from 203.0.113.7 (curl/8.4.0)
```

原因是 `invalid_credentials` 或 `inactive`。响应不区分，均为 `401`，日志才区分停用账户和密码错误。地址由 `TRUSTED_PROXY_HOPS` 解析，为客户端而非代理。

审计标记入口，因为 crudauth 不是唯一登录入口。后台有独立登录，不触发 crudauth 钩子，拒绝时自行记录相同两类日志：

```
login refused at the admin panel: 'admin' from 203.0.113.7 (curl/8.4.0); invalid_credentials
login locked out at the admin panel for 300s, from 203.0.113.7 (curl/8.4.0)
login locked out at the admin panel (every address) for 300s, from 198.51.100.9 (curl/8.4.0)
```

第三类表示全地址共用预算耗尽，响应与单地址锁定相同，只有日志区分。

**只有识别到真实账户才记录标识**。用户名输入可能是拼写错误、探测、他人地址甚至误填密码，未识别账户时只记录未解析状态、地址和 User-Agent。锁定发生在解析前，不记录标识；此前消耗预算的每次尝试已单独记录。

钩子在拒绝结果确定后执行，不能改变结果。钩子异常由 crudauth 记录，请求仍正常响应。项目可在 `infrastructure/auth/audit.py` 改为写审计表或外部系统。

<a id="running-a-second-crudauth-app-beside-this-one"></a>

## 与另一个 crudauth 应用并存

两个应用指向同一存储时会共享数据，crudauth 不会自动按应用命名空间隔离：

| 共享内容 | 后果 | 隔离方式 |
|----------|------|----------|
| 同 Redis DB 的会话和 CSRF 键空间 | 用户索引按用户 ID 而非 HMAC；同 ID 即使代表不同人，设备列表、全部退出和会话上限也会互相影响；`SECRET_KEY` 相同时 Cookie 可跨应用登录 | 独立 `SESSION_REDIS_DB` / `SESSION_REDIS_URL`，或 `SessionTransport(storage_prefix="other:session:", csrf_storage_prefix="other:csrf:")` |
| 浏览器 Cookie 名称 | 默认 `session_id` 和 `csrf_token` 位于 `/`，第二个登录覆盖第一个 | 设置 `SessionTransport(cookie_name="other_session", csrf_cookie_name="other_csrf")`，两者名称也必须互不相同 |
| 登录锁定和节流计数 | 一个应用锁定的地址或标识也在另一个锁定，失败累计 | 独立 `RATE_LIMITER_REDIS_DB`，或键前缀 |
| 同数据库的 `crudauth_store` / `crudauth_counters` | 行级同样冲突，会话按 `user_id` 索引，计数共用名称 | 独立数据库或 schema，或使用存储和限流前缀 |

最简单的隔离方式是与项目内部相同：独立 Redis DB。缓存 0、限流 1、会话 2、Taskiq 3，第二个应用可从 4 开始。

必须共用一个 DB 时，为所有内容添加前缀：

```python
# the other app's setup.py — not this one's
auth = CRUDAuth(
    ...,
    transports=[
        SessionTransport(
            cookie_name="other_session",
            csrf_cookie_name="other_csrf",
            storage_prefix="other:session:",
            csrf_storage_prefix="other:csrf:",
        )
    ],
    rate_limit_prefix="other:rl:",
)
```

`rate_limit_prefix` 仅适用于 crudauth 自建限流器，不能与 `rate_limiter=` 同时使用。本项目通过 `infrastructure/auth/limiter.py` 的 `build_rate_limiter()` 传入自己的限流器，因此应在那里设置 `redis_rate_limiter(client=..., prefix="other:rl:")`，数据库后端使用 `database_store.rate_limiter(prefix=...)`。

一次性令牌、OAuth state、MFA 存储使用服务端随机值作为键，共用空间不会冲突；除非共用 `SECRET_KEY`，另一应用也不会接受签名令牌。

!!! note "模板管理后台不属于第二个 crudauth 应用"
    后台是 [SQLAdmin](../admin-panel/index.md)，使用 `ADMIN_USERNAME` / `ADMIN_PASSWORD`，以及独立 Starlette Cookie `admin_session`，由 `SECRET_KEY` 签名且路径限定在后台挂载点。后台使用自己的前缀和两套锁定：`admin-panel:<address>` 及全部地址共享预算。因此 API 登录无法锁住后台，但在 API 失败的地址到后台仍有独立预算。上述前缀不适用于它。

<a id="session-limits"></a>

## 会话上限

`MAX_SESSIONS_PER_USER` 默认 5，超过时结束最旧会话。

<a id="storage-backends"></a>

## 存储后端

通过 `SESSION_BACKEND` 配置服务端存储：

| 值 | 用途 |
|----|------|
| `redis`，默认 | 生产；支持过期、模式扫描清理，进程重启后保留 |
| `memory` | 仅测试；重启清空，不适合多进程 |

后端实现在 crudauth 内，`setup.py` 按配置选择 `redis`、`database` 或 `memory`，未知值在启动拒绝。Memcached 已不支持会话，原文称仍可用于一般缓存和限流器。

`database` 将会话、CSRF、邮件一次性令牌和 OAuth state 放在项目数据库的 `crudauth_store`、`crudauth_counters` 两张表，多 Worker 无需 Redis 即可共享。表注册到 `Base.metadata`，Alembic 自动生成迁移，开发时可自动建表。切换 database 后端属于结构变更，需要迁移。存储写入时删除过期行，也可定期调用 `DatabaseStore.purge_expired()` 全量清理。

<a id="configuration"></a>

## 配置

```env
# Backend
SESSION_BACKEND=redis                # redis | database | memory
SESSION_REDIS_DB=2                   # on the cache Redis; isolated from cache (0), rate limiter (1), and taskiq (3)
# SESSION_REDIS_URL=                 # optional dedicated session Redis, e.g. rediss://user:password@host:6380/0

# Lifetime
SESSION_TIMEOUT_MINUTES=30           # inactive sessions expire
# SESSION_ABSOLUTE_TIMEOUT_HOURS=12  # the most a session may live from sign-in; unset for no cap
# FRESH_SIGN_IN_SECONDS=600            # how recent a sign-in must be for the routes that ask; unset, closing an account asks nothing
SESSION_CLEANUP_INTERVAL_MINUTES=15  # how often the storage backend sweeps expired entries

# Per-user cap
MAX_SESSIONS_PER_USER=5

# Cookie security (HTTPS only)
SESSION_SECURE_COOKIES=true

# CSRF
CSRF_ENABLED=true

# Trusted reverse proxies in front of the app (real client IP for login lockout)
TRUSTED_PROXY_HOPS=0
```

开发通常设置 `SESSION_SECURE_COOKIES=false` 和 `CSRF_ENABLED=false`，便于 HTTP 和 curl / Postman；预发布与生产重新开启。

<a id="login-logout-flow"></a>
<a id="login--logout-flow"></a>

## 登录与退出流程

<a id="login"></a>

### 登录

```bash
curl -X POST http://localhost:8000/api/v1/auth/login \
  -H "Content-Type: application/x-www-form-urlencoded" \
  -d "username=admin&password=your_admin_password" \
  -c cookies.txt
```

响应：

```json
{ "id": 1, "username": "admin", "csrf_token": "..." }
```

HttpOnly `session_id` 已保存在 `cookies.txt`。CSRF 令牌既设置 Cookie，也在响应体返回，方便 JavaScript 保存，因为 HttpOnly 会话 Cookie 不可读。

表单增加 `remember_me=true` 可持久化 Cookie，否则随浏览器会话结束。浏览器标记 `Sec-Fetch-Site: cross-site` 的登录返回 `403`，防止其他站点将访问者登录到攻击者账户。不发该头的 curl、移动端不受影响。

<a id="authenticated-request"></a>

### 认证请求

```bash
curl http://localhost:8000/api/v1/users/me -b cookies.txt
```

变更状态请求添加 CSRF 头：

```bash
curl -X POST http://localhost:8000/api/v1/users/ \
  -b cookies.txt \
  -H "Content-Type: application/json" \
  -H "X-CSRF-Token: <token-from-login-response>" \
  -d '{"name": "...", "username": "...", "email": "...", "password": "..."}'
```

<a id="refresh-csrf-token"></a>

### 刷新 CSRF

```bash
curl -X POST http://localhost:8000/api/v1/auth/csrf/refresh -b cookies.txt
```

有效令牌不轮换，可在每次页面加载调用。按地址限流，crudauth 的 `csrf_refresh` 默认每小时 30 次。

<a id="logout"></a>

### 退出

```bash
curl -X POST http://localhost:8000/api/v1/auth/logout -b cookies.txt
```

结束会话并清除 Cookie。

<a id="logout-all-sessions"></a>

### 退出全部会话

```bash
curl -X POST http://localhost:8000/api/v1/auth/logout-all \
  -b cookies.txt \
  -H "X-CSRF-Token: <token-from-login-response>"
```

结束当前用户所有设备的**全部会话**，包括当前会话，并清除 Cookie：

```json
{ "detail": "Signed out of all sessions.", "revoked": 3 }
```

只退出其他设备，保留当前登录时，传 `keep_current=true`：

```bash
curl -X POST "http://localhost:8000/api/v1/auth/logout-all?keep_current=true" \
  -b cookies.txt \
  -H "X-CSRF-Token: <token-from-login-response>"
```

无需再次认证，因为这是用户怀疑当前会话不可信时需要的操作。按用户限流，`logout_all` 默认每小时 10 次。

<a id="manage-devices"></a>

### 设备管理

```bash
curl http://localhost:8000/api/v1/auth/sessions -b cookies.txt
```

```json
[
  {"id": "7f3c…", "device": "Chrome on macOS", "ip": "203.0.113.7",
   "created_at": "2026-10-08T09:12:44Z", "last_activity": "2026-10-08T11:03:02Z", "current": true}
]
```

列出全部会话，以 `current` 标记调用者。`id` 是会话 **handle** 而非凭据 ID，可安全展示，不能用于认证。按 handle 退出设备：

```bash
curl -X DELETE http://localhost:8000/api/v1/auth/sessions/7f3c… \
  -b cookies.txt \
  -H "X-CSRF-Token: <token-from-login-response>"
```

检查归属，其他用户的 handle 与不存在的一样返回 `404`，避免泄露。撤销自身 handle 会清 Cookie。所有后端均支持列表：memory 扫描条目，Redis 使用用户索引，database 使用 `user_id` 列。

<a id="key-files"></a>

## 关键文件

| 组件 | 位置 |
|------|------|
| `auth = CRUDAuth(...)` 单例 | `backend/src/infrastructure/auth/setup.py` |
| 依赖 | `backend/src/infrastructure/auth/dependencies.py` |
| OAuth 配置 | `backend/src/infrastructure/auth/setup.py` |
| 登录、退出与 OAuth 路由 | `backend/src/infrastructure/auth/routes.py` |
| 登录拒绝与锁定钩子 | `backend/src/infrastructure/auth/audit.py` |
| 会话管理路由 | crudauth 的 `build_session_management_router`，在 `routes.py` 挂载 |
| HTTP 异常 | `backend/src/infrastructure/http_exceptions.py` |
| 认证配置 | `backend/src/infrastructure/config/settings.py` 的 `AuthSettings` |

会话存储、CSRF 和锁定实现在 crudauth 中，而非模板本身。

---

[← 认证概览](index.md){ .md-button } [用户管理 →](user-management.md){ .md-button .md-button--primary }
