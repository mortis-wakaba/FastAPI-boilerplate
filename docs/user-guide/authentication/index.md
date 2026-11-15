<a id="authentication-security"></a>
<a id="authentication--security"></a>

# 身份认证与安全

模板使用 **HttpOnly Cookie 配合服务端会话**，默认不使用 JWT。认证由 [`crudauth`](https://pypi.org/project/crudauth/) 提供，会话存储在 Redis 或可配置的内存中，具有 CSRF 防护和登录锁定限流。组装入口是 `infrastructure/auth/setup.py` 中的 `auth = CRUDAuth(...)` 单例，见[会话 → 认证架构](sessions.md#auth-architecture)。

机器间调用使用 **API 密钥**，支持独立权限和用量记录。

<a id="what-youll-learn"></a>

## 内容概览

- **[会话](sessions.md)**：服务端会话、Cookie 和 CSRF。
- **[用户管理](user-management.md)**：注册、登录和资料操作。
- **[权限](permissions.md)**：基于角色的访问控制与资源归属。
- **[迁移密码](migrating-passwords.md)**：导入其他系统的密码哈希。

<a id="why-sessions-not-jwt"></a>

## 为什么使用会话而非 JWT

原模板使用 JWT、刷新令牌和令牌黑名单，后来改为会话，原因如下：

- **退出简单**：删除会话记录即可，无需维护黑名单。
- **凭据轮换简单**：更新会话记录，无需等待令牌到期。
- **内置 CSRF**：服务端会话可配合双重提交 CSRF 令牌。
- **服务端存储**：避免长期令牌因 XSS 从客户端存储泄露。
- **符合常见认证思路**：“是否已登录”通过存储查询判断，而非仅依赖密码学验证。

需要客户端视角的无会话凭据，例如无法共享会话存储的服务间调用，可使用 **API 密钥**，由服务端认证。

<a id="need-jwt-for-mobile-or-native-apps"></a>

### 移动端或原生应用需要 JWT？

Cookie 和 CSRF 对移动端、原生客户端及 CLI 不方便。crudauth 提供可与会话并行的 **bearer（JWT）传输**，模板默认未启用。两者都解析为同一个 `Principal`，因此 `CurrentUserDep`、`get_current_user` 等路由保护无需改变，只改变客户端认证方式。

启用时，在 `infrastructure/auth/setup.py` 的 `transports` 列表加入 `BearerTransport`，并挂载 crudauth bearer 路由，提供 `POST /token` 登录和 `POST /refresh` 刷新访问令牌：

```python
# infrastructure/auth/setup.py
from crudauth import BearerTransport, CookieConfig, CRUDAuth, SessionTransport

auth = CRUDAuth(
    session=async_session,
    user_model=User,
    SECRET_KEY=settings.SECRET_KEY,
    cookies=CookieConfig(secure=settings.SESSION_SECURE_COOKIES),
    transports=[
        SessionTransport(...),                       # browsers (unchanged)
        BearerTransport(access_ttl=900, refresh="body"),  # mobile / API clients
    ],
    ...
)
```

```python
# wherever the auth router is included (e.g. interfaces/api/v1)
app.include_router(auth.bearer_router, prefix="/api/v1/auth")
```

移动端通常选择 `refresh="body"`，将刷新令牌放在 JSON 响应中自行存储，而非 Cookie。访问令牌通过 `Authorization: Bearer <token>` 发送。同时存在会话 Cookie 和 bearer 令牌时，**列表中排在前面的传输优先**。

令牌生命周期、刷新策略、scope 以及两种传输共存，见 crudauth 的 [Bearer tokens](https://benavlabs.github.io/crudauth/guides/auth/bearer/) 和 [Multiple transports](https://benavlabs.github.io/crudauth/guides/auth/multiple-transports/)。

<a id="authentication-mechanisms"></a>

## 认证机制

模板提供三条可共存的认证路径，按接口选择。

<a id="1-sessions-browser-clients"></a>

### 1. 会话（浏览器客户端）

```bash
# Log in — server sets the session cookie and returns a CSRF token
curl -X POST "http://localhost:8000/api/v1/auth/login" \
  -H "Content-Type: application/x-www-form-urlencoded" \
  -d "username=admin&password=your_admin_password" \
  -c cookies.txt
# → { "id": 1, "username": "admin", "csrf_token": "..." }

# Subsequent requests — send the cookie back
curl http://localhost:8000/api/v1/users/me -b cookies.txt

# Log out
curl -X POST http://localhost:8000/api/v1/auth/logout -b cookies.txt
```

路由通过 `Depends(get_current_user)` 要求有效会话。

<a id="2-oauth-google-and-github"></a>

### 2. OAuth（Google 和 GitHub）

使用带 PKCE 的 OAuth 2.0 社交登录。浏览器前往提供方登录，返回回调后创建会话并跳转到应用。

```text
# Link or redirect the browser to (redirect_to is optional, same-origin paths only):
GET /api/v1/auth/oauth/google?redirect_to=/dashboard
# → 307 to https://accounts.google.com/...

# Google sends the browser back to:
GET /api/v1/auth/oauth/callback/google?code=...&state=...
# → session + CSRF cookies set, 307 to /dashboard (or to OAUTH_REDIRECT_BASE_URL)
```

GitHub 路径为 `/api/v1/auth/oauth/github` 和 `/api/v1/auth/oauth/callback/github`。

向各提供方注册 `{OAUTH_REDIRECT_BASE_URL}/api/v1/auth/oauth/callback/<provider>`。Google 在其控制台配置；GitHub 在 Settings → Developer settings → OAuth Apps → New OAuth App 的 **Authorization callback URL** 配置。`OAUTH_REDIRECT_BASE_URL` 是 API 不含路径的公开源地址。以上路径遵循 `API_PREFIX`，修改前缀后需注册新回调。

登录失败，例如用户拒绝或邮箱超过列长度，会跳转至 `OAUTH_REDIRECT_BASE_URL?error=<code>`。回调 state 与启动时 Cookie 不匹配、已使用或已过期时，返回 `error=invalid_state`，不创建会话，因为可能是登录 CSRF。常见原因是登录耗时过长或在另一浏览器完成，应允许重新开始。

**登录写回的内容**：每次 OAuth 登录活跃账户时都会执行 `infrastructure/auth/oauth_profile.py` 的 `refresh_the_provider_profile`，作为 crudauth 的 `on_oauth_login` 钩子注册。

| 列 | 写入时机 |
|----|----------|
| `name` | 创建账户时，从提供方资料初始化一次 |
| `profile_image_url` | 账户仍使用初始占位头像时，采用提供方头像 |
| `oauth_updated_at` | 创建之后的每次登录，由 crudauth 写入 |
| `google_id` / `github_id`、`oauth_provider`、`email_verified` | 创建或关联账户时由 crudauth 写入 |

提供方初始化头像和显示名后，由账户自行拥有，用户修改后不会被后续登录覆盖。希望始终同步头像，可移除钩子中的占位判断；需要当前登录名（GitHub 改名会改变 `info.username`）可在钩子写入，完整提供方数据在 `info.raw_data`。钩子只接收 crudauth 自有字段，自定义字段应从数据库记录读取。

提供方必须同时设置两项配置才启用：`OAUTH_GOOGLE_CLIENT_ID` / `OAUTH_GOOGLE_CLIENT_SECRET` 或 `OAUTH_GITHUB_CLIENT_ID` / `OAUTH_GITHUB_CLIENT_SECRET`。可以启用任意一个、两个或都不启用。只有 client ID 没有 secret 不会生效，没有提供方时不挂载 OAuth 路由。

GitHub 通过 `GET /user/emails` 单独提供邮箱，crudauth 选择已验证的主邮箱。未验证邮箱返回 `?error=email_unverified`，不创建账户和会话，避免冒用地址。没有公开姓名的 GitHub 用户以登录名作为显示名。

路由由 crudauth 提供，包含 PKCE、与浏览器绑定的一次性 state、安全的同源重定向。新增提供方时，在 `infrastructure/auth/setup.py` 的 `OAUTH_PROVIDERS` 列出名称，添加 `OAUTH_<NAME>_CLIENT_ID` / `_CLIENT_SECRET`，前提是 crudauth 已注册同名提供方类。

<a id="3-api-keys-machine-to-machine"></a>

### 3. API 密钥（机器间调用）

适用于服务、程序、脚本及集成。密钥属于账户，有独立范围，可单独撤销：

```bash
# Create one (while signed in; the full key is shown once)
curl -X POST "http://localhost:8000/api/v1/api-keys/" \
  -H "Content-Type: application/json" -H "X-CSRF-Token: <token>" \
  -b cookies.txt \
  -d '{"name": "Integration Key", "permissions": ["user.read"], "usage_limits": {"requests_per_minute": 60}}'

# Call with it, in the X-API-Key header
curl http://localhost:8000/api/v1/auth/me -H "X-API-Key: fai_..."
# → { "user_id": 1, "username": "admin", ..., "via": "apikey" }
```

密钥请求以所有者身份执行，但权限被密钥 scope 收窄，且不继承超级用户标记。无需 CSRF，不返回 Cookie；密码、邮箱、会话、关闭账户和密钥管理等要求账户登录的路由会直接拒绝密钥。每次调用计入密钥自身限制并记录用量。

**[API 密钥](api-keys.md)** 介绍创建、范围、调用、限制、用量、撤销、轮换和存储方式。

<a id="recovery-flows-email"></a>

## 恢复流程（邮件）

crudauth 的恢复路由挂载在同一前缀，提供邮箱验证、密码重置和确认式邮箱变更：

```text
POST /api/v1/auth/email/verify-request    {"email": "..."}
POST /api/v1/auth/email/verify-confirm    {"token": "..."}
POST /api/v1/auth/password/reset-request  {"email": "..."}
POST /api/v1/auth/password/reset-confirm  {"token": "...", "new_password": "..."}
POST /api/v1/auth/email/change-request    {"new_email": "...", "password": "..."}   # authenticated
POST /api/v1/auth/email/change-confirm    {"token": "..."}
```

`-request` 路由无论地址是否属于账户都返回相同结果，避免枚举注册邮箱，并按客户端地址和目标地址限流。投递使用 [`EMAIL_BACKEND`](../configuration/environment-variables.md#email) 指定的发送器。

`email/change-request` 校验当前密码，因此也计入账户的改密预算：每小时 5 次，密码正确也计数，与 `POST /api/v1/auth/change-password` 共享。超限直接返回 `429`，不再检查，避免持有会话的人通过不断更换目标邮箱猜密码。

crudauth 保证且项目测试验证：

- 令牌一次性使用，密码、邮箱或 `token_version` 变化后提前失效。
- 完成重置会终止账户的**全部会话**，使被盗会话失效。
- 邮箱变更确认后向**旧地址**发送 `email_changed` 通知。项目还通过 `infrastructure/auth/evictions.py` 结束全部会话，因为恢复地址变更属于凭据变更。确认不附带会话，所有者需重新登录。
- 不向非活跃账户发送邮件，已有链接也失效。`User.is_active` 为 `not is_deleted`，因此覆盖软删除账户。

库拥有的请求模型以普通 `str` 接收字段，因此此处所有 crudauth 路由都携带 `infrastructure/auth/deps.py` 的 `body_a_database_can_hold`。JSON 任意位置（键和值）存在数据库无法存储的孤立代理字符或 NUL 时，返回标明字段的 `422`。它按 FastAPI 方式识别内容类型，也将 `application/merge-patch+json` 作为 JSON。否则 `{"token": "\ud800"}` 会进入 `jwt.decode`，在 UTF-8 编码时异常并返回 500。

同时使用 `a_redirect_a_link_can_carry`，拒绝超过 2048 字符的 `redirect_to`，返回 `422`。目标会签入令牌，过长路径会形成同样巨大的邮件链接和回传令牌。OAuth 查询参数 `?redirect_to=` 也应用此限制，目标在回调前保存在 state 存储中。crudauth 已过滤非同源路径，此处只限制长度。

<a id="where-the-links-point"></a>

### 链接指向哪里

模板不提供页面，链接指向**你的前端**：

```env
FRONTEND_URL=https://app.example.com
```

```text
{FRONTEND_URL}/verify-email?token=...
{FRONTEND_URL}/reset-password?token=...
{FRONTEND_URL}/confirm-email-change?token=...
```

页面从查询字符串读取 `token`，POST 到对应 `-confirm` 路由。生产必须设置 `FRONTEND_URL`，空值或 `localhost` 会在启动时被拒绝，否则生成的链接无法访问。

开发默认 `console` 后端记录完整邮件和链接，无需运行前端，可从日志复制令牌手动确认：

```bash
curl -X POST http://localhost:8000/api/v1/auth/password/reset-request \
    -H "Content-Type: application/json" -d '{"email": "you@example.com"}'
# the API log prints: ... Subject: Reset your password
#                     ... http://localhost:3000/reset-password?token=eyJhbGciOi...

curl -X POST http://localhost:8000/api/v1/auth/password/reset-confirm \
    -H "Content-Type: application/json" \
    -d '{"token": "eyJhbGciOi...", "new_password": "An0therPassword!"}'
```

<a id="key-features"></a>

## 主要功能

<a id="server-side-sessions"></a>

### 服务端会话

- **存储**：默认 Redis，也可用项目数据库或内存，通过 `SESSION_BACKEND` 配置。
- **HttpOnly Cookie**：JavaScript 无法读取 `session_id`。
- **CSRF 令牌**：登录返回并设置 Cookie，变更状态请求需携带 `X-CSRF-Token`。
- **超时**：`SESSION_TIMEOUT_MINUTES`。
- **用户上限**：`MAX_SESSIONS_PER_USER` 限制同时会话数。
- **清理**：`SESSION_CLEANUP_INTERVAL_MINUTES` 控制过期扫描。

<a id="user-management"></a>

### 用户管理

- 同一 `/api/v1/auth/login` 接受**用户名或邮箱**。
- **bcrypt** 密码哈希。
- **软删除**：通过 `is_deleted` 停用账户，而非销毁。
- **GDPR / LGPD 匿名化**：`DELETE /api/v1/users/db/{username}` 硬清除个人信息。
- 用户模型提供 `google_id`、`github_id`、`oauth_provider` 等 **OAuth 标记**。

<a id="permission-system"></a>

### 权限系统

- **角色**：`modules/role/` 的 `resource.action` 权限，`require_permissions("user.read")` 保护路由，超级用户绕过。
- **超级用户标记**：`User.is_superuser` 用于仅管理员路由。
- **等级**：用户归属 `Tier`，按等级和路径配置限流。
- **资源归属**：在服务而非路由检查。

<a id="login-lockout"></a>

### 登录锁定

crudauth 按 IP 和登录标识逐级锁定，阈值为 `LOGIN_*` 配置，见[认证与安全](../configuration/environment-variables.md#authentication--security)。达到限制后，`POST /api/v1/auth/login` 返回带 `Retry-After` 的 `429 Too Many Requests`。反向代理后设置 `TRUSTED_PROXY_HOPS`，以真实客户端 IP 为键。

登录标识维度在检查密码前汇总所有地址的失败，因此知道用户名的人可通过达到 `LOGIN_MAX_ATTEMPTS` 持续锁定该账户，管理员也不例外。这是按账户锁定的取舍：只按 IP 则僵尸网络每个地址都能对每个账户猜同样次数，但按账户也给持有标识的人提供了影响可用性的手段。

缓解方式有两个：锁定自动到期，从 `LOGIN_LOCKOUT_BASE_SECONDS` 开始逐轮翻倍至 `LOGIN_LOCKOUT_MAX_SECONDS`；运维也可立即清除单个账户的锁：

```bash
uv run --no-sync python scripts/unlock_account.py <username-or-email>
```

脚本清除账户用户名和邮箱下的失败次数、锁和递增轮次，因为登录接受两者，锁可能落在用户平时不用的标识上。来源 IP 的计数不变，因为属于触发它的人。计数位于限流器，只有共享后端（`RATE_LIMITER_BACKEND=redis` 或数据库存储）才能影响运行中的 API；`memory` 每 Worker 独立，需要重启清除。

脚本清除后重新读取，仍被锁定或无法连接则非零退出，因为底层清除方法只记录后端错误并继续。退出码 `0` 表示账户确实可再次登录。

<a id="authentication-patterns"></a>

## 认证模式

依赖均位于 `src/infrastructure/auth/dependencies.py`，封装 crudauth 的 `auth` 单例。

<a id="required-authentication"></a>

### 必须认证

```python
from ...infrastructure.auth.dependencies import get_current_user

@router.get("/me", response_model=UserRead)
async def me(
    current_user: Annotated[dict[str, Any], Depends(get_current_user)],
) -> dict[str, Any]:
    return current_user
```

会话 Cookie 缺失或无效返回 401。

<a id="optional-authentication"></a>

### 可选认证

```python
from ...infrastructure.auth.dependencies import get_optional_user

@router.get("/")
async def list_things(
    user: Annotated[dict[str, Any] | None, Depends(get_optional_user)],
):
    # Logged-in users see extras; anonymous users still get a response
    if user is not None:
        return {"premium": True}
    return {"premium": False}
```

<a id="superuser-only"></a>

### 仅超级用户

```python
from ...infrastructure.auth.dependencies import get_current_superuser

@router.delete("/{username}/permanent")
async def gdpr_delete_user(
    username: str,
    db: Annotated[AsyncSession, Depends(async_session)],
    user_service: Annotated[UserService, Depends(get_user_service)],
    _: Annotated[dict[str, Any], Depends(get_current_superuser)],
) -> dict[str, str]:
    ...
```

仅触发依赖的参数以前导下划线命名。

<a id="permission-required"></a>

### 要求权限

```python
from ...infrastructure.auth.authorization import require_permissions

@router.get("/", dependencies=[require_permissions("user.read")])
async def get_users(
    db: AsyncSessionDep,
    user_service: UserServiceDep,
) -> dict[str, Any]:
    ...
```

调用者角色未提供全部指定权限则返回 403，超级用户始终通过。`infrastructure/auth/deps.py` 通过 `CurrentPermissionsDep` 提供权限集合，通过 `CurrentPrincipalDep` 提供 crudauth 的 `Principal`，见[权限](permissions.md#role-based-permissions)。

<a id="resource-ownership"></a>

### 资源归属

在服务层检查，而非路由：

```python
# modules/user/service.py
async def verify_user_permission(
    self,
    current_user: dict[str, Any],
    target_username: str,
    action: str,
) -> None:
    if current_user["username"] != target_username and not current_user["is_superuser"]:
        raise PermissionDeniedError(f"Cannot {action} for another user")
```

路由委派后由服务抛 `PermissionDeniedError`，自动映射为 403，见[异常处理](../api/exceptions.md)。

<a id="security-features"></a>

## 安全功能

<a id="session-security"></a>

### 会话安全

- HttpOnly `session_id` 无法由 JavaScript 读取；原文将其称为防 XSS 的 Cookie 保护。
- 非开发环境使用 `Secure` Cookie，`SESSION_SECURE_COOKIES=true`。
- 状态变更请求校验 CSRF，`CSRF_ENABLED=true`。
- 会话记录 IP 和 User-Agent。
- `MAX_SESSIONS_PER_USER` 限制会话数。

<a id="password-security"></a>

### 密码安全

- bcrypt 自动加盐。
- Pydantic 在 `UserCreate.password` 校验最小长度和复杂度。
- 明文密码不存储、不写日志。
- 登录限流防止撞库。

<a id="production-validator"></a>

### 生产校验器

`ENVIRONMENT=production` 且 `PRODUCTION_SECURITY_VALIDATION_ENABLED=true` 时，以下问题会拒绝启动：

- 不安全或占位 `SECRET_KEY`。
- 默认或空数据库密码。
- 启用后台却缺少 `ADMIN_USERNAME` / `ADMIN_PASSWORD`。
- `CORS_ORIGINS` 包含 `*`。

空 `CORS_ORIGINS` 表示完全禁止跨域，不是错误。生产拒绝 `*`；其他环境允许时会关闭 `CORS_ALLOW_CREDENTIALS`，使通配来源不支持凭据响应。

<a id="configuration"></a>

## 配置

完整参考见[环境变量](../configuration/environment-variables.md)，主要配置为：

```env
# Sessions
SESSION_TIMEOUT_MINUTES=30
SESSION_CLEANUP_INTERVAL_MINUTES=15
MAX_SESSIONS_PER_USER=5
SESSION_SECURE_COOKIES=true
SESSION_BACKEND=redis             # redis | database | memory

# CSRF
CSRF_ENABLED=true                  # set false for dev/test

# Trusted reverse proxies in front of the app (real client IP for login lockout)
TRUSTED_PROXY_HOPS=0

# OAuth
OAUTH_REDIRECT_BASE_URL=http://localhost:8000
OAUTH_GOOGLE_CLIENT_ID=
OAUTH_GOOGLE_CLIENT_SECRET=
OAUTH_GITHUB_CLIENT_ID=            # data model anticipates GitHub; no provider/routes wired
OAUTH_GITHUB_CLIENT_SECRET=

# Security
SECRET_KEY=<openssl rand -hex 32>
PRODUCTION_SECURITY_VALIDATION_ENABLED=true
```

<a id="quick-examples"></a>

## 快速示例

<a id="frontend-login-flow-javascript"></a>

### 前端登录（JavaScript）

```javascript
class AuthClient {
    async login(username, password) {
        const res = await fetch('/api/v1/auth/login', {
            method: 'POST',
            credentials: 'include',                   // important — accept cookies
            headers: { 'Content-Type': 'application/x-www-form-urlencoded' },
            body: new URLSearchParams({ username, password }),
        });
        if (!res.ok) throw new Error('login failed');
        const { csrf_token } = await res.json();
        // Store the CSRF token in memory; cookie is set automatically
        this.csrfToken = csrf_token;
        return csrf_token;
    }

    async post(url, body) {
        return fetch(url, {
            method: 'POST',
            credentials: 'include',
            headers: {
                'Content-Type': 'application/json',
                'X-CSRF-Token': this.csrfToken,       // required for state-changing requests
            },
            body: JSON.stringify(body),
        });
    }

    async logout() {
        await fetch('/api/v1/auth/logout', {
            method: 'POST',
            credentials: 'include',
            headers: { 'X-CSRF-Token': this.csrfToken },
        });
        this.csrfToken = null;
    }
}
```

`credentials: 'include'` 使浏览器在跨源请求中发送 Cookie，服务端还需正确配置 CORS，包括 `CORS_ALLOW_CREDENTIALS=true`。

<a id="custom-tier-based-dependency"></a>

### 自定义等级依赖

组合内置依赖检查等级：

```python
from typing import Annotated, Any
from fastapi import Depends, HTTPException

from ...infrastructure.auth.dependencies import get_current_user


async def require_tier(
    tier_name: str,
    user: Annotated[dict[str, Any], Depends(get_current_user)],
) -> dict[str, Any]:
    user_tier = user.get("tier") or {}
    if user_tier.get("name") != tier_name:
        raise HTTPException(status_code=403, detail=f"Requires {tier_name} tier")
    return user


# Usage with a Pro tier
@router.get("/premium")
async def premium_feature(
    user: Annotated[dict[str, Any], Depends(lambda u=Depends(get_current_user): require_tier("pro", u))],
):
    return {"data": "premium content"}
```

实际使用优先在服务中抛出 `PermissionDeniedError`，由映射层一致转换，见[异常处理](../api/exceptions.md)。

<a id="getting-started"></a>

## 阅读顺序

1. **[会话](sessions.md)**：会话原理、Cookie 和 CSRF。
2. **[用户管理](user-management.md)**：注册、登录和资料。
3. **[权限](permissions.md)**：角色与资源访问控制。

<a id="whats-next"></a>

## 后续步骤

- **[环境变量](../configuration/environment-variables.md)**：全部认证配置。
- **[异常处理](../api/exceptions.md)**：领域异常如何映射为 403 / 401。
- **[API 接口](../api/endpoints.md)**：路由保护模式。
