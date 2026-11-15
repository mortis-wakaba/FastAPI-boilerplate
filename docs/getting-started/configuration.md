<a id="configuration"></a>

# 配置指南

本指南介绍启动 FastAPI 应用所需的基本配置。

<a id="quick-setup"></a>

## 快速配置

复制环境变量示例文件，并修改其中的配置值：

```bash
cd backend
cp .env.example .env
```

全部变量见 `backend/.env.example`。以下介绍最常需要调整的选项。

<a id="essential-configuration"></a>

## 必要配置

打开 `backend/.env`，设置以下必要选项。

<a id="application-settings"></a>

### 应用设置

```env
APP_NAME=Your app name here
APP_DESCRIPTION=Your app description here
VERSION=0.1.0
API_CONTACT_NAME=Your name
API_CONTACT_EMAIL=your@email.com
API_LICENSE_NAME=The license you picked
```

<a id="environment-type"></a>

### 环境类型

```env
# Options: development, staging, production, local
ENVIRONMENT=development
```

- **development**：在 `/docs`、`/redoc` 提供 API 文档，启用详细日志。
- **staging**：结构化日志，启用日志文件输出。
- **production**：JSON 日志、安全校验，通过 `ENABLE_DOCS_IN_PRODUCTION` 控制文档开放。
- **local**：默认设置与 development 相同，适用于测试。

<a id="database"></a>

### 数据库

```env
POSTGRES_USER=postgres
POSTGRES_PASSWORD=changeme
POSTGRES_DB=postgres
POSTGRES_SERVER=postgres    # use "localhost" without Docker
POSTGRES_PORT=5432
CREATE_TABLES_ON_STARTUP=true   # the default in local and development
```

<a id="security"></a>

### 安全设置

生成强随机 `SECRET_KEY`：

```bash
python -c "import secrets; print(secrets.token_urlsafe(64))"
```

```env
SECRET_KEY=your-generated-secret-key-here

# Production security validation (enabled by default in production)
PRODUCTION_SECURITY_VALIDATION_ENABLED=true
```

<a id="sessions"></a>

### 会话

```env
SESSION_TIMEOUT_MINUTES=30
# SESSION_ABSOLUTE_TIMEOUT_HOURS=12  # unset: the idle timeout is the only one
# FRESH_SIGN_IN_SECONDS=600          # unset: closing an account asks nothing; crudauth's own routes keep their 10-minute default
SESSION_CLEANUP_INTERVAL_MINUTES=15
MAX_SESSIONS_PER_USER=5
SESSION_SECURE_COOKIES=true
SESSION_BACKEND=redis            # redis | memory
SESSION_REDIS_DB=2               # on the cache Redis, apart from the cache DB so a flush won't log users out
# SESSION_REDIS_URL=             # optional dedicated session Redis, e.g. rediss://user:password@host:6380/0

# CSRF protection (set false to disable in dev/test)
CSRF_ENABLED=true

# Trusted reverse proxies in front of the app (used to resolve the real client
# IP for login lockout). 0 = none; set 1 behind a single nginx/Caddy.
TRUSTED_PROXY_HOPS=0

# Password policy (enforced by crudauth on registration and password changes)
PASSWORD_MIN_LENGTH=8
PASSWORD_REQUIRE_UPPERCASE=true
PASSWORD_REQUIRE_LOWERCASE=true
PASSWORD_REQUIRE_DIGIT=true
PASSWORD_REQUIRE_SPECIAL=true
```

登录锁定由 `crudauth` 实现，同时按客户端 IP 和提交的登录标识统计失败次数；触发限制时返回 `429 Too Many Requests`，并附带 `Retry-After` 响应头。以下四项配置会在 `infrastructure/auth/setup.py` 中传入 crudauth 的 `LockoutConfig`：

```env
LOGIN_MAX_ATTEMPTS=5              # failures allowed inside the window
LOGIN_ATTEMPT_WINDOW_SECONDS=900  # how long failures keep counting
LOGIN_LOCKOUT_BASE_SECONDS=300    # first lockout, doubling each round
LOGIN_LOCKOUT_MAX_SECONDS=3600    # cap on the escalation
```

由于同时统计两个维度，从同一地址不断更换登录标识会耗尽该地址的尝试次数；从不同地址尝试同一标识则会耗尽该标识的次数。

<a id="first-admin-user"></a>

### 首个管理员用户

`setup_initial_data` 脚本首次运行时读取以下配置：

```env
ADMIN_NAME=Admin User
ADMIN_EMAIL=admin@example.com
ADMIN_USERNAME=admin
ADMIN_PASSWORD=Change-Me-Now-1!
```

然后运行：

```bash
uv run --no-sync python -m scripts.setup_initial_data
```

<a id="cache-redis-or-memcached"></a>

### 缓存（Redis 或 Memcached）

```env
CACHE_ENABLED=true
CACHE_BACKEND=redis             # or "memcached"

# Client-side cache (Cache-Control headers)
CLIENT_CACHE_ENABLED=true
CLIENT_CACHE_MAX_AGE=60

# Redis settings
CACHE_REDIS_HOST=redis          # use "localhost" without Docker
CACHE_REDIS_PORT=6379
CACHE_REDIS_DB=0
CACHE_REDIS_PASSWORD=
```

<a id="rate-limiting"></a>

### 限流

```env
RATE_LIMITER_ENABLED=true
RATE_LIMITER_BACKEND=redis        # or memory (per process, single worker only)
DEFAULT_RATE_LIMIT_LIMIT=100
DEFAULT_RATE_LIMIT_PERIOD=60

# Redis (uses DB 1 by default to separate from cache)
RATE_LIMITER_REDIS_HOST=redis   # use "localhost" without Docker
RATE_LIMITER_REDIS_PORT=6379
RATE_LIMITER_REDIS_DB=1
RATE_LIMITER_REDIS_PASSWORD=
```

`crudauth` 根据用户等级和请求路径，为每个请求确定 API 限流规则。已认证请求以用户 ID 为键；匿名请求以客户端 IP 为键，并遵循 `TRUSTED_PROXY_HOPS` 设置。

<a id="background-tasks-taskiq"></a>

### 后台任务（Taskiq）

```env
TASKIQ_BROKER_TYPE=redis        # or "rabbitmq"

# Redis broker (uses DB 3 by default)
TASKIQ_REDIS_HOST=redis         # use "localhost" without Docker
TASKIQ_REDIS_PORT=6379
TASKIQ_REDIS_DB=3
TASKIQ_REDIS_PASSWORD=
```

<a id="cors"></a>

### CORS

```env
CORS_ENABLED=true
CORS_ORIGINS=http://localhost:3000,http://localhost:5173  # comma-separated origins
CORS_ALLOW_CREDENTIALS=true
CORS_ALLOW_METHODS=*
CORS_ALLOW_HEADERS=*
```

!!! warning "生产环境中的 CORS"
    生产环境的 `CORS_ORIGINS` 不应使用 `*`。请指定确切域名，并明确允许的方法和请求头：

    ```env
    CORS_ORIGINS=https://yourapp.com,https://www.yourapp.com
    CORS_ALLOW_METHODS=GET,POST,PUT,DELETE,PATCH
    CORS_ALLOW_HEADERS=Authorization,Content-Type
    ```

<a id="oauth-optional"></a>

### OAuth（可选）

Google / GitHub 登录配置：

```env
OAUTH_REDIRECT_BASE_URL=http://localhost:8000

# Google OAuth
OAUTH_GOOGLE_CLIENT_ID=
OAUTH_GOOGLE_CLIENT_SECRET=

# GitHub OAuth
OAUTH_GITHUB_CLIENT_ID=
OAUTH_GITHUB_CLIENT_SECRET=
```

当某个提供方的两项设置均已填写时，才会接入该提供方。因此可以仅启用 Google、仅启用 GitHub、同时启用两者或全部关闭；将对应的一对配置留空即可禁用该提供方。详细流程见[身份认证](../user-guide/authentication/index.md)。

crudauth 内置 OAuth 路由保留现有路径 `/api/v1/auth/oauth/{provider}` 和 `/api/v1/auth/oauth/callback/{provider}`，返回 JSON，设置会话 Cookie，将 state 绑定到浏览器，并校验登录后的重定向地址必须是同源相对路径。

<a id="admin-interface"></a>

### 管理界面

```env
ADMIN_ENABLED=true              # enables SQLAdmin at /admin
```

<a id="docker-compose-settings"></a>

## Docker Compose 配置

使用 Docker Compose 时，服务之间通过服务名访问。在 `.env` 中使用以下主机名：

```env
POSTGRES_SERVER=postgres
CACHE_REDIS_HOST=redis
RATE_LIMITER_REDIS_HOST=redis
TASKIQ_REDIS_HOST=redis
```

<a id="thats-it"></a>

## 启动应用

完成上述设置后，启动应用：

=== "使用 uv 在本地运行"

    ```bash
    uv run --no-sync fastapi dev src/interfaces/main.py
    ```

=== "Docker Compose"

    ```bash
    docker compose up
    ```

完整参考和高级配置见[用户指南 → 配置](../user-guide/configuration/index.md)。
