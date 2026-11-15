<a id="environment-variables-reference"></a>

# 环境变量参考

本页列出模板读取的全部环境变量。`backend/.env.example` 提供新项目通常需要设置的变量，默认值与此处一致；其余变量在本页说明。未显式设置的变量使用下文默认值。

应用启动时，通过 `src/infrastructure/config/settings.py` 中的 Pydantic `BaseSettings` 类从 `backend/.env` 加载变量。

<a id="environment"></a>

## 环境

```env
# Options: development, staging, production, local
ENVIRONMENT=development
```

| 变量 | 默认值 | 用途 |
|------|--------|------|
| `ENVIRONMENT` | `development` | 控制日志样式、文档可见性和安全校验，见[分环境配置](environment-specific.md) |

<a id="database"></a>

## 数据库

```env
POSTGRES_USER=postgres
POSTGRES_PASSWORD=postgres
POSTGRES_DB=postgres
POSTGRES_SERVER=postgres    # use "localhost" without Docker
POSTGRES_PORT=5432
POSTGRES_ASYNC_PREFIX=postgresql+asyncpg://
CREATE_TABLES_ON_STARTUP=true   # on by default in local and development only
```

| 变量 | 默认值 | 用途 |
|------|--------|------|
| `POSTGRES_USER` | `postgres` | 数据库用户 |
| `POSTGRES_PASSWORD` | `postgres` | 数据库密码 |
| `POSTGRES_DB` | `postgres` | 数据库名 |
| `POSTGRES_SERVER` | `localhost` | 主机名，Compose 中使用服务名 `postgres` |
| `POSTGRES_PORT` | `5432` | TCP 端口 |
| `POSTGRES_ASYNC_PREFIX` | `postgresql+asyncpg://` | 应用异步代码的驱动前缀 |
| `CREATE_TABLES_ON_STARTUP` | local 和 development 为 `true`，其他为 `false` | 启动时按模型建表，生产环境拒绝 `true` |
| `POSTGRES_POOL_SIZE` | `20` | SQLAlchemy 连接池大小 |
| `POSTGRES_MAX_OVERFLOW` | `0` | 连接池允许的额外连接数 |
| `POSTGRES_POOL_PRE_PING` | `true` | 使用池内连接前检查，替换已被服务端关闭的连接 |
| `POSTGRES_POOL_RECYCLE` | `-1` | 丢弃存在超过指定秒数的连接，`-1` 表示禁用 |

直接设置 `DATABASE_URL` 会覆盖自动构造的 URL。需要主机、端口和凭据之外的选项，例如托管数据库要求 TLS 时，应使用它：

```env
DATABASE_URL=postgresql+asyncpg://user:password@host.example.com/dbname?ssl=require
```

URL 必须使用 `postgresql+asyncpg://` 前缀，查询参数会传给 `asyncpg`，其 TLS 参数是 `ssl=require`，不是 libpq 的 `sslmode=require`。[生产校验器](../production.md#the-production-validator)从 URL 提取凭据，因此 `POSTGRES_*` 可以保持默认值。无服务器数据库的完整示例见 [Neon](../database/neon.md)。

<a id="cache"></a>

## 缓存

```env
CACHE_ENABLED=true
CACHE_BACKEND=redis           # or "memcached"

# Client-side cache (Cache-Control headers)
CLIENT_CACHE_ENABLED=true
CLIENT_CACHE_MAX_AGE=60
```

<a id="redis-backend"></a>

### Redis 后端

```env
CACHE_REDIS_HOST=redis        # use "localhost" without Docker
CACHE_REDIS_PORT=6379
CACHE_REDIS_DB=0
CACHE_REDIS_PASSWORD=
CACHE_REDIS_CONNECT_TIMEOUT=5
CACHE_REDIS_POOL_SIZE=10
```

<a id="memcached-backend"></a>

### Memcached 后端

```env
CACHE_MEMCACHED_HOST=localhost
CACHE_MEMCACHED_PORT=11211
CACHE_MEMCACHED_POOL_SIZE=10
CACHE_MEMCACHED_CONNECT_TIMEOUT=5
```

<a id="rate-limiting"></a>

## 限流

由 `crudauth` 提供，使用 Redis 或内存。每个请求根据用户等级和路径解析限制，找不到规则则使用以下默认值。各路径独立计数。

```env
RATE_LIMITER_ENABLED=true
RATE_LIMITER_BACKEND=redis        # or database (shared), or memory (per process, single worker only)
DEFAULT_RATE_LIMIT_LIMIT=100
DEFAULT_RATE_LIMIT_PERIOD=60
```

<a id="redis-backend_1"></a>

### Redis 后端

```env
RATE_LIMITER_REDIS_HOST=redis
RATE_LIMITER_REDIS_PORT=6379
RATE_LIMITER_REDIS_DB=1        # separate DB from cache (DB 0)
RATE_LIMITER_REDIS_PASSWORD=
RATE_LIMITER_REDIS_CONNECT_TIMEOUT=5
RATE_LIMITER_REDIS_POOL_SIZE=10
```

<a id="email"></a>

## 邮件

```env
EMAIL_BACKEND=console           # or "smtp"
EMAIL_FROM=no-reply@localhost
EMAIL_FROM_NAME=
FRONTEND_URL=http://localhost:3000
```

| 变量 | 默认值 | 用途 |
|------|--------|------|
| `EMAIL_BACKEND` | `console` | `console` 只记录邮件而不发送，`smtp` 实际投递；生产环境拒绝 `console` |
| `EMAIL_FROM` | `no-reply@localhost` | 账户邮件的发件地址 |
| `EMAIL_FROM_NAME` | 空 | 发件人显示名称 |
| `FRONTEND_URL` | `http://localhost:3000` | 找回链接指向的前端地址；生产环境拒绝空值和 `localhost` |

<a id="smtp"></a>

### SMTP

```env
EMAIL_SMTP_HOST=
EMAIL_SMTP_PORT=587
EMAIL_SMTP_USER=
EMAIL_SMTP_PASSWORD=
EMAIL_SMTP_STARTTLS=true
EMAIL_SMTP_TIMEOUT_SECONDS=10
```

`EMAIL_BACKEND=smtp` 但未设置 `EMAIL_SMTP_HOST` 时，构造发送器会明确报错。`EMAIL_SMTP_STARTTLS` 校验服务器证书和主机名，因此拒绝自签名证书；只应在自己控制的网络中、连接可信中继时关闭。

启用 Taskiq 功能后，邮件通过 `email:send` 任务入队，由 Worker 发送，请求无需等待邮件服务器。消息代理不可达时，由发送器移交的任务直接投递邮件，避免丢失，仍不阻塞请求。找回接口如果仅在地址存在时等待邮件，就会泄露哪些地址已注册。

<a id="background-tasks-taskiq"></a>

## 后台任务（Taskiq）

```env
TASKIQ_BROKER_TYPE=redis        # or "rabbitmq"
TASKIQ_DEFAULT_RETRY_COUNT=3    # runs in all for a task labelled retry_on_error=True; 0 turns retries off
TASKIQ_RETRY_DELAY_SECONDS=0    # seconds a retry waits; 0 retries immediately, above 0 needs the scheduler
```

<a id="redis-broker"></a>

### Redis 消息代理

```env
TASKIQ_REDIS_HOST=redis
TASKIQ_REDIS_PORT=6379
TASKIQ_REDIS_DB=3               # separate DB from cache and rate limiter
TASKIQ_REDIS_PASSWORD=
```

<a id="rabbitmq-broker"></a>

### RabbitMQ 消息代理

```env
TASKIQ_RABBITMQ_HOST=localhost
TASKIQ_RABBITMQ_PORT=5672
TASKIQ_RABBITMQ_USER=guest
TASKIQ_RABBITMQ_PASSWORD=guest
TASKIQ_RABBITMQ_VHOST=/
```

<a id="web-server"></a>

## Web 服务

<a id="cors"></a>

### CORS

```env
CORS_ENABLED=true
CORS_ORIGINS=http://localhost:3000,http://localhost:5173  # comma-separated list of origins
CORS_ALLOW_CREDENTIALS=true
CORS_ALLOW_METHODS=*
CORS_ALLOW_HEADERS=*
```

!!! danger "生产环境中的 CORS"
    生产环境的 `CORS_ORIGINS` 不应使用 `*`，否则任意网站都可以从用户浏览器调用 API。存在 `*` 时应用会关闭 `CORS_ALLOW_CREDENTIALS`，使其他来源页面无法读取携带用户 Cookie 的调用响应，因此通配来源也无法支持需要登录的前端。跨站简单请求仍会送达，在 `SameSite` 允许时还会携带 Cookie；浏览器只是阻止读取响应。生产安全校验器会拒绝启动，应指定确切域名：

    ```env
    CORS_ORIGINS=https://yourapp.com,https://www.yourapp.com
    CORS_ALLOW_METHODS=GET,POST,PUT,DELETE,PATCH
    CORS_ALLOW_HEADERS=Authorization,Content-Type
    ```

<a id="compression"></a>

### 压缩

```env
GZIP_ENABLED=true
GZIP_MINIMUM_SIZE=1000
```

<a id="api-docs"></a>

### API 文档

```env
ENABLE_DOCS_IN_PRODUCTION=false  # serve /docs even when ENVIRONMENT=production (superuser-only)
OPENAPI_PREFIX=                   # path prefix for the OpenAPI schema
```

非开发环境提供文档时，即 staging 或设置了 `ENABLE_DOCS_IN_PRODUCTION=true` 的 production，不注册 FastAPI 内置文档路由。`/docs`、`/redoc` 和 `/openapi.json` 仅通过应用自己的路由提供，并要求超级用户认证。

<a id="authentication-security"></a>
<a id="authentication--security"></a>

## 认证与安全

```env
SECRET_KEY=insecure-secret-key-change-this-in-production

# Production security validation (enabled by default in production)
PRODUCTION_SECURITY_VALIDATION_ENABLED=true

# Response security headers: HSTS outside local and development, plus the frame,
# content-type and referrer headers
SECURITY_HEADERS_ENABLED=true
```

生成强密钥：

```bash
python -c "import secrets; print(secrets.token_urlsafe(64))"
```

<a id="password-policy"></a>

### 密码策略

crudauth 在注册、重置和修改密码等所有设置密码的位置执行策略。初始数据脚本会拒绝不符合要求的 `ADMIN_PASSWORD`，并说明失败原因。

```env
PASSWORD_MIN_LENGTH=8
PASSWORD_REQUIRE_UPPERCASE=true
PASSWORD_REQUIRE_LOWERCASE=true
PASSWORD_REQUIRE_DIGIT=true
PASSWORD_REQUIRE_SPECIAL=true
```

<a id="sessions"></a>

### 会话

```env
SESSION_TIMEOUT_MINUTES=30
SESSION_CLEANUP_INTERVAL_MINUTES=15
MAX_SESSIONS_PER_USER=5
SESSION_SECURE_COOKIES=true

LOGIN_MAX_ATTEMPTS=5             # failures allowed inside the window, per address and per account
LOGIN_ATTEMPT_WINDOW_SECONDS=900 # how long failures keep counting
LOGIN_LOCKOUT_BASE_SECONDS=300   # first lockout, doubling each round
LOGIN_LOCKOUT_MAX_SECONDS=3600   # ceiling for the doubling
SESSION_BACKEND=redis            # redis | database | memory
SESSION_REDIS_DB=2               # on the cache Redis, apart from the cache DB so a flush won't log users out
# SESSION_REDIS_URL=             # optional dedicated session Redis, e.g. rediss://user:password@host:6380/0

# Number of trusted reverse proxies in front of the app. crudauth uses this to
# resolve the real client IP (from X-Forwarded-For) for login lockout.
# 0 = no proxy; set 1 behind a single nginx/Caddy, 2 if Cloudflare is also in front.
TRUSTED_PROXY_HOPS=0

# SESSION_ABSOLUTE_TIMEOUT_HOURS=   # cap on a session's total life, however active it stays; unset means no cap
# FRESH_SIGN_IN_SECONDS=            # how recently a caller must have signed in to close an account,
                                    # and to reach crudauth's set-password and MFA routes where a project
                                    # mounts them; unset asks nothing, and a value below 1 is refused
```

专用会话 Redis 可通过 URL 或逐项主机配置指定。设置了 `SESSION_REDIS_URL` 时优先使用它，否则以下变量与 `SESSION_REDIS_DB` 一起使用：

```env
SESSION_REDIS_HOST=redis
SESSION_REDIS_PORT=6379
SESSION_REDIS_PASSWORD=
```

<a id="forwarded-headers"></a>

### 转发头

```env
# Which peers uvicorn accepts X-Forwarded-For and X-Forwarded-Proto from.
FORWARDED_ALLOW_IPS=172.31.240.0/24
```

`FORWARDED_ALLOW_IPS` 由 **uvicorn** 读取，而非应用读取，决定 `request.client` 和访问日志显示的地址。uvicorn 从自身进程环境读取，因此需要通过 Compose 的 `env_file` 或启动环境传入，而不经过应用配置。只有 `backend/.env` 同时也是容器的 `env_file` 时，其中的值才生效。`bp deploy generate nginx` 会生成该配置，值取自 `--internal-subnet`。

应用前方没有代理时保持未设置。通配符 `*` 允许任意客户端声明任意地址，此时 `TRUSTED_PROXY_HOPS` 统计的是客户端可控请求头中的跳数。

<a id="csrf"></a>

### CSRF

```env
# Set false to disable CSRF validation in dev/test
CSRF_ENABLED=true
```

<a id="login-lockout"></a>

### 登录锁定

原文称登录节流没有环境变量：`crudauth` 内部按 IP 和登录标识执行逐级锁定，达到阈值后返回带 `Retry-After` 的 `429 Too Many Requests`。在代理后部署时，设置上文的 `TRUSTED_PROXY_HOPS`，让锁定使用真实客户端 IP，而非代理 IP。

> 译注：此处与入门配置指南列出的 `LOGIN_*` 变量不一致，使用时请核对当前代码。

<a id="oauth"></a>

### OAuth

```env
OAUTH_REDIRECT_BASE_URL=http://localhost:8000

# Google OAuth (leave empty to disable)
OAUTH_GOOGLE_CLIENT_ID=
OAUTH_GOOGLE_CLIENT_SECRET=

# GitHub OAuth (leave empty to disable)
OAUTH_GITHUB_CLIENT_ID=
OAUTH_GITHUB_CLIENT_SECRET=
```

<a id="admin-interface-sqladmin"></a>

## 管理界面（SQLAdmin）

```env
ADMIN_ENABLED=true              # enables the panel
ADMIN_BASE_URL=/admin           # where it is mounted, and the admin cookie's path
```

<a id="application-metadata"></a>

## 应用元数据

```env
DEBUG=false
APP_NAME=FastAPI Boilerplate
APP_DESCRIPTION=Modular FastAPI starter
VERSION=0.1.0
API_CONTACT_NAME=Support
API_CONTACT_EMAIL=support@example.com
API_CONTACT_URL=
API_LICENSE_NAME=MIT
API_LICENSE_IDENTIFIER=
API_LICENSE_URL=
API_SUMMARY=
API_TERMS_OF_SERVICE=
API_TAGS_METADATA=[]
```

`API_SUMMARY` 是描述旁的一行摘要；`API_TERMS_OF_SERVICE`、`API_CONTACT_URL` 和 `API_LICENSE_URL` 接收 URL；`API_LICENSE_IDENTIFIER` 接收 `MIT` 等 SPDX 标识；`API_TAGS_METADATA` 接收 OpenAPI 标签对象的 JSON 数组，无法解析时不加入文档。

`API_TITLE`、`API_DESCRIPTION` 和 `API_VERSION` 覆盖 OpenAPI 文档中的 `APP_NAME`、`APP_DESCRIPTION` 和 `VERSION`。字段为空时不输出，因此新项目在自行配置前不会声明联系人或许可证。

<a id="api-settings-optional-overrides"></a>

### API 设置（可选覆盖）

```env
# API_PREFIX=/api
# DOCS_URL=/docs
# REDOC_URL=/redoc
# OPENAPI_URL=/openapi.json   # set empty to serve no schema, which also takes /docs and /redoc with it
```

`API_PREFIX` 必须以 `/` 开头，且不能以 `/` 结尾，否则加载配置时拒绝。它会同时移动所有 API 路由、OAuth 路由，以及 `no-store` 缓存头的作用路径。

<a id="api-keys"></a>

## API 密钥

```env
KEY_USAGE_RETENTION_DAYS=90      # days of key usage rows to keep; 0 keeps every row, below 0 is refused
```

每次请求的记录包含调用者地址和 User-Agent，因此保留窗口决定表中个人数据的存储范围。分析接口不会读取保留范围之外的数据。清理方式见[用量记录保留时间](../authentication/api-keys.md#how-long-the-usage-rows-are-kept)。

<a id="initial-setup"></a>

## 初始设置

由 `python -m scripts.setup_initial_data` 读取：

```env
ADMIN_NAME=Admin User
ADMIN_EMAIL=admin@example.com
ADMIN_USERNAME=admin
ADMIN_PASSWORD=Change-Me-Now-1!
```

默认等级名称也可配置，默认为 `free`：

```env
DEFAULT_TIER_NAME=free
```

<a id="logging"></a>

## 日志

```env
LOG_LEVEL=INFO
LOG_FORMAT=                     # console only: simple | detailed | structured | json; empty = the environment's default
LOG_CONSOLE_ENABLED=true
LOG_FILE_ENABLED=false
LOG_FILE_PATH=logs/app.log
LOG_FILE_MAX_SIZE=10485760      # 10 MB
LOG_FILE_BACKUP_COUNT=5
LOG_DEVELOPMENT_VERBOSE=true
LOG_PRODUCTION_OPTIMIZE=true
```

`LOG_FORMAT` 只指定控制台格式。日志文件使用所在环境对应的格式，以符合收集器预期。配置加载时会拒绝没有对应格式化器的值。

<a id="production-security-checklist"></a>

## 生产安全清单

部署生产环境前：

1. 生成强 `SECRET_KEY`，至少包含 64 字节熵。
2. 数据库和每个 Redis 实例使用独立密码。
3. 各服务使用独立 Redis DB：`CACHE_REDIS_DB=0`、`RATE_LIMITER_REDIS_DB=1`、`TASKIQ_REDIS_DB=3`。
4. `CORS_ORIGINS` 限定为真实域名，不使用 `*`。
5. 设置强管理员凭据：`ADMIN_USERNAME`、`ADMIN_PASSWORD`。
6. 根据安全要求检查会话超时。
7. 设置 `ENVIRONMENT=production`，启用安全校验器。
8. 使用 RabbitMQ 时替换 `guest/guest` 默认凭据。

<a id="troubleshooting"></a>

## 故障排查

<a id="variables-not-loading"></a>

### 变量未加载

```bash
# Check the file location
ls -la backend/.env

# Make sure there are no spaces around =
grep "=" backend/.env | head -5

# Verify what Python sees
cd backend
uv run --no-sync python -c "from src.infrastructure.config.settings import get_settings; s = get_settings(); print(s.APP_NAME, s.ENVIRONMENT)"
```

<a id="database-connection-failed"></a>

### 数据库连接失败

```bash
# Linux
sudo systemctl status postgresql
psql -h localhost -U postgres -d postgres

# macOS
brew services list | grep postgresql
```

<a id="redis-connection-failed"></a>

### Redis 连接失败

```bash
redis-cli -h localhost -p 6379 ping  # should print PONG

# Linux
sudo systemctl status redis-server

# macOS
brew services list | grep redis
```

<a id="see-also"></a>

## 另请参阅

- **[配置类](settings-classes.md)**：环境变量如何转换为 Python 配置。
- **[Docker 配置](docker-setup.md)**：Compose 配置。
- **[分环境配置](environment-specific.md)**：各环境推荐值。
