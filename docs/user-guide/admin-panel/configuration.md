<a id="admin-panel-configuration"></a>

# 管理后台配置

后台有意保持简单：一个 [SQLAdmin](https://github.com/smithyhq/sqladmin) 实例，通过环境变量中的用户名和密码保护，只需少量 `.env` 配置。

<a id="environment-variables"></a>

## 环境变量

```env
# Toggle the admin panel (default: false)
ADMIN_ENABLED=true

# Admin login credentials — must BOTH be set, otherwise the panel is not mounted
ADMIN_USERNAME=admin
ADMIN_PASSWORD=Change-Me-Now-1!

# Used for admin session encryption (same SECRET_KEY as the rest of the app)
SECRET_KEY=<openssl rand -hex 32>
```

以上为后台专属配置。本节原文称引擎、模型和挂载路径在 `src/interfaces/admin/initialize.py` 写死；后文又说明路径可由 `ADMIN_BASE_URL` 配置。

<a id="backing-settings-classes"></a>

### 对应配置类

`src/infrastructure/config/settings.py` 中：

- **`AdminSettings`**：`ADMIN_NAME`、`ADMIN_EMAIL`、`ADMIN_USERNAME`、`ADMIN_PASSWORD`、`DEFAULT_TIER_NAME`，供后台登录和初始化首个超级用户脚本共用。
- **`SQLAdminSettings`**：`ADMIN_ENABLED`，后台总开关。

<a id="what-happens-at-startup"></a>

## 启动流程

1. 应用工厂调用 wiring 安装器列表中的 admin `install(app)`。
2. 未设置或关闭 `ADMIN_ENABLED` 时返回 None，**不挂载**。
3. 用户名或密码为空时记录警告并返回 None，避免挂载只有登录表单却无法使用的后台。
4. 以 `SECRET_KEY` 构造 `AdminAuth`。
5. 用应用现有数据库引擎创建 SQLAdmin `Admin`。
6. 注册 `src/wiring/admin.py` 列出的 UserAdmin、TierAdmin 和三个角色视图。
7. 挂载 `/admin`。

<a id="login-authentication"></a>

## 登录认证

`interfaces/admin/auth.py` 流程：

```python
class AdminAuth(AuthenticationBackend):
    async def login(self, request: Request) -> bool:
        settings = get_settings()
        if not settings.ADMIN_USERNAME or not settings.ADMIN_PASSWORD:
            return False

        address = get_client_ip(request, settings.TRUSTED_PROXY_HOPS)
        counted_against = f"{LOCKOUT_PREFIX}:{address}"
        # panel_lockout counts this address, every_address_lockout counts them all;
        # either one locked refuses before the password is read
        ...
```

说明：

- 凭据来自环境变量，**不是数据库**。修改需重启，旧凭据创建的所有会话随之失效。
- 用户名和密码必须同时设置，空默认值不能登录。
- 只支持一套后台登录，没有多管理员表。
- **后台使用独立锁定**，按客户端地址和 `admin-panel:<address>` 标识计数，阈值使用 `LOGIN_MAX_ATTEMPTS`、`LOGIN_ATTEMPT_WINDOW_SECONDS`、`LOGIN_LOCKOUT_BASE_SECONDS`、`LOGIN_LOCKOUT_MAX_SECONDS`。锁定后不比较密码。前缀独立，因此 API 登录失败不影响后台，代价是 API 上已失败的地址到后台仍有完整预算。代理后设置 `TRUSTED_PROXY_HOPS`，否则全算代理地址。
- **另有全地址共享预算**，同窗口为 `LOGIN_MAX_ATTEMPTS` 五倍，默认 25。后台只有一个密码，若只按地址计数，换地址可无限猜。成功登录退还本次尝试，但不清除其他来源失败。
- **共享预算耗尽后拒绝所有地址**，基础锁定默认 300 秒。失败仍在默认 900 秒窗口累计，首次到期后再尝试会重新锁定并翻倍，最高 3600 秒。递增轮次保留一小时，每次锁定重置保留时间。因此约 26 次攻击就可能使后台关闭近一小时，锁定中的重试（包括操作员）还会延长；操作员五次尝试也会锁住自身地址。这是限制共享密码猜测与可用性的取舍，也是[不向公网开放后台](#production-hardening)的理由。
- **提前解锁**可清除共享计数。其前缀为 `admin-panel:rl:all:`，单地址前缀为 `admin-panel:rl:`，以下模式覆盖两者：

    ```bash
    # RATE_LIMITER_BACKEND=redis, on RATE_LIMITER_REDIS_DB (1 by default)
    redis-cli -n 1 --scan --pattern 'admin-panel:rl:*' | xargs redis-cli -n 1 del

    # RATE_LIMITER_BACKEND=database
    psql -c "delete from crudauth_counters where key like 'admin-panel:rl:%'"
    ```

    `memory` 后端重启即可清除。清除也恢复攻击者的 25 次机会，因攻击触发时应更换 `ADMIN_PASSWORD`。

- **每次拒绝写审计日志** `src.infrastructure.auth.audit`，包含入口、地址、User-Agent，识别到后台用户名时记录用户名，例如 `login refused at the admin panel: 'admin' from 203.0.113.7 (curl/8.4.0); invalid_credentials`，见[拒绝记录](../authentication/sessions.md#what-a-refusal-records)。
- 两项凭据均用 `hmac.compare_digest` 比较，避免首字节差异影响耗时。
- 后台专属 `SessionMiddleware` 通过 `SECRET_KEY` 签名。
- **会话从登录起最多八小时**，与活跃程度无关。它携带签发时间和 `ADMIN_USERNAME:ADMIN_PASSWORD` 的指纹；指纹以 `SECRET_KEY` 加盐、scrypt 派生。超过时限或指纹不匹配即拒绝。Cookie 可读，工作因子保护低熵操作员密码，避免同时得到密钥的人轻易跑字典。派生在构建后台时执行并复用，不阻塞每请求。拒绝会清会话 Cookie，缺少这两个字段的旧版 Cookie 也拒绝。
- 退出通过 POST，执行 `request.session.clear()`。

需要多管理员时，见[用户管理](user-management.md)。

<a id="requests-from-another-site"></a>

### 来自其他站点的请求

SQLAdmin 没有 CSRF 令牌，使用两层保护：

- **`SameSite=Strict` Cookie** 不随其他站点发起的请求发送，顶层链接也不例外，所以外站链接会落到登录页；从地址栏打开仍可使用原会话。兄弟子域如 `other.example.com` 与 `admin.example.com` 属于同一 site，Strict 不阻止，因此还需下一层。
- **`SameOriginOnly`** 位于 `interfaces/admin/origin.py`，在后台会话前拒绝可能修改数据的跨源请求，返回 `403`。覆盖 POST / PUT / PATCH / DELETE，也包括 `/{identity}/action/{slug}` 的 GET，因为 SQLAdmin 的 `@action` 通过 GET 提供。只有 `Sec-Fetch-Site` 为 `same-origin` / `none`，或 `Origin` 等于接收主机时通过。其他 GET 不变。无这两个头的 curl 和脚本允许，因为站点无法让浏览器完全省去来源信息。

**退出使用 POST** 也是同一理由。原 SQLAdmin 用 GET，页面链接、外站请求都可能使操作员退出。现在 GET `/admin/logout` 返回单按钮表单，只有同源 POST 清会话。侧栏、书签和外链只进入确认表单。

<a id="mount-path"></a>

## 挂载路径

`ADMIN_BASE_URL` 默认 `/admin`：

```env
ADMIN_BASE_URL=/management
```

同时控制 `Admin(base_url=...)` 和 Cookie path。`management`、`/management/` 会规范为相同路径，无有效路径时启动拒绝。

代理使用 `--root-path /svc` 时，Cookie 自动跟随实际前缀，如 `/svc/admin`，无需额外配置。修改后应同步前端和运维文档链接。

!!! warning "从 Cookie 尚未限定路径的版本升级"
    旧 `path=/` Cookie 在八小时内仍发送到所有路径，新路径退出只清新 Cookie。浏览器先发长路径 Cookie，Starlette 保留最后值，旧根路径 Cookie 可能在退出后继续有效。升级后手动清一次 `admin_session`，或更换 `SECRET_KEY` 统一失效；后者也使 crudauth API 用户全部退出。

<a id="database-connection"></a>

## 数据库连接

后台复用 API 的 **SQLAlchemy 引擎**，无需独立连接或连接池。

<a id="session-cookies"></a>

## 会话 Cookie

`src/interfaces/admin/auth.py` 的 `AdminAuth` 只为后台挂载 Starlette `SessionMiddleware`，API 不解码后台 Cookie：

```python
self.middlewares = [
    Middleware(SameOriginOnly),
    Middleware(
        RootPathSessionMiddleware,
        base_url=base_url or admin_base_url(),
        secret_key=secret_key,
        session_cookie="admin_session",
        max_age=SESSION_MAX_AGE_SECONDS,
        same_site="strict",
        https_only=not local,
    ),
]
```

行为：

- HttpOnly，名 `admin_session`，`SECRET_KEY` 签名。
- 路径为 `ADMIN_BASE_URL` 加 `root_path`，不发送到 API。
- `SameSite=strict`。
- local / development 外启用 Secure。
- 八小时有效期，常量 `SESSION_MAX_AGE_SECONDS`。

写会话（如[闪现消息](adding-models.md#bulk-actions)）会重签 Cookie 并重置 Max-Age，因此真正时限不是 Cookie，而是登录记录的时间，超过八小时仍拒绝。更换任一凭据使旧会话全部失效，这是后台的撤销方式。

生产 HTTPS 部署通常在代理终止 TLS，并从公网路由移除 `/admin`，见[生产加固](#production-hardening)。

<a id="development-vs-production"></a>

## 开发与生产

<a id="development"></a>

### 开发

`.env.example` 已提供开发配置：

```env
ENVIRONMENT=development
ADMIN_ENABLED=true
ADMIN_USERNAME=admin
ADMIN_PASSWORD=Change-Me-Now-1!
SECRET_KEY=insecure-secret-key-change-this-in-production
```

访问 <http://localhost:8000/admin>，登录后管理 Users 和 Tiers。

<a id="production-hardening"></a>

### 生产加固

三种方案：

1. **完全禁用**：

    ```env
    ADMIN_ENABLED=false
    ```

    不挂载后台，改用 `uv run --no-sync python -m scripts.setup_initial_data`、一次性脚本或临时覆盖。

2. **代理或负载均衡限制**：保持启用，只允许 VPN CIDR 或指定 IP 访问 `/admin`。
3. **强独立密码**：无法网络限制时，从密钥管理器部署注入，不提交，定期轮换，使用长高熵密码。生产校验器拒绝占位 `SECRET_KEY` 或启用后台却缺凭据，弱管理员密码仅告警。

`infrastructure/security/` 在生产启动检查后台凭据等设置，应有意配置。

<a id="environment-detection"></a>

## 环境检测

后台核心仍是同一 SQLAdmin，环境差异包括：

- **Cookie 安全**：`https_only` 在 local / development 关闭，其余开启。
- **日志**：使用 `infrastructure/logging/` 的相同配置。
- **存储**：后台 Cookie 自包含且签名，不使用 API 的 Redis / memory 会话后端。原文认为后台操作员重新登录成本较低。

<a id="troubleshooting"></a>

## 故障排查

<a id="admin-returns-404"></a>

### `/admin` 返回 404

检查 `ADMIN_ENABLED` 及两项凭据。默认关闭，任一凭据为空也不挂载，启动日志有说明：

```bash
cd backend
uv run --no-sync python -c "from src.infrastructure.config.settings import get_settings; print(get_settings().ADMIN_ENABLED)"
```

<a id="login-form-keeps-rejecting-credentials"></a>

### 登录持续失败

- 核对 `backend/.env` 的用户名密码。
- 修改后重启。
- Docker 内检查变量是否传入：`docker compose exec api env | grep ADMIN_`。

<a id="admin-session-keeps-logging-out"></a>

### 会话频繁退出

默认登录八小时后失效。需修改时编辑 `src/interfaces/admin/auth.py` 的 `SESSION_MAX_AGE_SECONDS`：

```python
SESSION_MAX_AGE_SECONDS = 60 * 60 * 8
```

<a id="wrong-engine-connection-no-such-table"></a>
<a id="wrong-engine-connection--no-such-table"></a>

### 引擎错误或 “no such table”

后台复用 API 引擎，需要 local / development 默认的自动建表，或已应用 Alembic 迁移。视图空白或报错时检查：

```bash
cd backend
uv run --no-sync alembic current
```

<a id="next-steps"></a>

## 后续步骤

- **[添加模型](adding-models.md)**：注册自定义视图。
- **[用户管理](user-management.md)**：扩展后台认证。
- **[生产环境](../production.md)**：加固清单。
