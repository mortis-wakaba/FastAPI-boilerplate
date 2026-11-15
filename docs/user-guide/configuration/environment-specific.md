<a id="environment-specific-configuration"></a>

# 分环境配置

模板根据 `ENVIRONMENT` 调整行为。本页介绍各环境的推荐设置，以及代码已自动完成的调整。

<a id="supported-environments"></a>

## 支持的环境

```env
# Options: production, staging, development, local
ENVIRONMENT=development
```

| 值 | 用途 |
|----|------|
| `development` | 本地开发，详细日志和 DEBUG 级别输出 |
| `local` | 默认配置与 development 相同，用于测试和 CI |
| `staging` | 预生产测试，结构化日志、INFO 级别、部分安全校验 |
| `production` | 线上部署，JSON 日志、启用安全校验、受控开放文档 |

<a id="what-the-codebase-does-for-you"></a>

## 模板自动调整的行为

以下行为已根据 `ENVIRONMENT` 切换，无需另外编写条件逻辑：

| 行为 | development / local | staging | production |
|------|---------------------|---------|------------|
| **日志样式** | 详细文本、彩色控制台 | 结构化键值 | JSON |
| **默认日志级别** | `LOG_DEVELOPMENT_VERBOSE=true` 时为 DEBUG | INFO | `LOG_PRODUCTION_OPTIMIZE=true` 时为 WARNING |
| **噪声较多的库日志** | 正常级别 | 正常级别 | 降低 urllib3、sqlalchemy、redis 等日志量 |
| **安全校验** | 跳过 | 检查密钥、邮件后端和 `CREATE_TABLES_ON_STARTUP` | `PRODUCTION_SECURITY_VALIDATION_ENABLED=true`（默认）时，启动执行全部检查 |
| **`/docs` 文档** | 可用 | 可用 | 除非 `ENABLE_DOCS_IN_PRODUCTION=true`，否则关闭 |

日志行为由 `src/infrastructure/logging/config.py` 控制，安全校验器位于 `src/infrastructure/security/`。

<a id="development"></a>

## 开发环境

适用于日常本地开发。

```env
ENVIRONMENT=development
DEBUG=true

# App metadata
APP_NAME=MyApp (Development)
VERSION=0.1.0-dev

# Database (local Postgres or Docker Compose)
POSTGRES_USER=postgres
POSTGRES_PASSWORD=postgres
POSTGRES_DB=myapp_dev
POSTGRES_SERVER=localhost     # or "postgres" with Docker Compose
POSTGRES_PORT=5432

# Security — keep a placeholder, never reuse for staging/prod
SECRET_KEY=insecure-dev-key-replace-me

# Sessions — relax cookie security for plain HTTP
SESSION_SECURE_COOKIES=false
CSRF_ENABLED=false             # often easier when testing with curl

# Cache / Rate limiter / Taskiq — point at localhost
CACHE_REDIS_HOST=localhost
RATE_LIMITER_REDIS_HOST=localhost
TASKIQ_REDIS_HOST=localhost

# Looser limits while iterating
DEFAULT_RATE_LIMIT_LIMIT=1000
DEFAULT_RATE_LIMIT_PERIOD=60

# Admin user (used by setup_initial_data on first run)
ADMIN_NAME=Dev Admin
ADMIN_EMAIL=admin@localhost
ADMIN_USERNAME=admin
ADMIN_PASSWORD=admin123

# Logging
LOG_DEVELOPMENT_VERBOSE=true   # gives you DEBUG-level console output
```

!!! tip "为什么开发时关闭 CSRF？"
    会话 Cookie 是 HttpOnly，响应还会返回 CSRF 令牌。浏览器客户端应同时携带两者；使用 curl / Postman 测试时，`CSRF_ENABLED=false` 可简化流程。测试真实前端流程时应重新启用。

<a id="staging"></a>

## 预发布环境

生产前的演练环境：使用与生产相同的代码路径和独立数据，用于发现特定环境问题。

```env
ENVIRONMENT=staging
DEBUG=false

APP_NAME=MyApp (Staging)
VERSION=0.1.0-staging

POSTGRES_USER=staging_user
POSTGRES_PASSWORD=use-a-strong-password
POSTGRES_SERVER=staging-db.example.com
POSTGRES_PORT=5432
POSTGRES_DB=myapp_staging

# Real key, distinct from prod
SECRET_KEY=<openssl rand -hex 32>

# Lock cookies and CSRF down
SESSION_SECURE_COOKIES=true
CSRF_ENABLED=true

# Restrict CORS to staging domains
CORS_ORIGINS=https://staging.example.com
CORS_ALLOW_METHODS=GET,POST,PUT,DELETE,PATCH
CORS_ALLOW_HEADERS=Authorization,Content-Type

# Real Redis hostnames — use separate DBs for cache/rate limit/taskiq
CACHE_REDIS_HOST=staging-redis.example.com
CACHE_REDIS_PASSWORD=<from secrets manager>
RATE_LIMITER_REDIS_HOST=staging-redis.example.com
RATE_LIMITER_REDIS_PASSWORD=<from secrets manager>
TASKIQ_REDIS_HOST=staging-redis.example.com
TASKIQ_REDIS_PASSWORD=<from secrets manager>

DEFAULT_RATE_LIMIT_LIMIT=100
DEFAULT_RATE_LIMIT_PERIOD=60

ADMIN_NAME=Staging Admin
ADMIN_EMAIL=admin@staging.example.com
ADMIN_USERNAME=staging_admin
ADMIN_PASSWORD=<from secrets manager>

# Logging — staging picks INFO automatically; can opt into file output
LOG_FILE_ENABLED=true
LOG_FILE_PATH=logs/app.log
```

<a id="production"></a>

## 生产环境

处理真实流量，每项设置都应从安全角度审视。

```env
ENVIRONMENT=production
DEBUG=false

APP_NAME=MyApp
VERSION=1.0.0
API_CONTACT_NAME=Support Team
API_CONTACT_EMAIL=support@example.com

POSTGRES_USER=prod_user
POSTGRES_PASSWORD=<from secrets manager>
POSTGRES_SERVER=prod-db.example.com
POSTGRES_PORT=5432
POSTGRES_DB=myapp_production

# Generated with: python -c "import secrets; print(secrets.token_urlsafe(64))"
SECRET_KEY=<from secrets manager>

# Production security validator is on by default
PRODUCTION_SECURITY_VALIDATION_ENABLED=true

# Sessions
SESSION_SECURE_COOKIES=true
SESSION_TIMEOUT_MINUTES=30
SESSION_BACKEND=redis
CSRF_ENABLED=true
# Trusted reverse proxies in front of the app, so crudauth keys login lockout on
# the real client IP. Set 1 behind a single nginx/Caddy, 2 if Cloudflare is also in front.
TRUSTED_PROXY_HOPS=1

# Strict CORS
CORS_ORIGINS=https://example.com,https://www.example.com
CORS_ALLOW_METHODS=GET,POST,PUT,DELETE,PATCH
CORS_ALLOW_HEADERS=Authorization,Content-Type

# Docs gated off by default in production
ENABLE_DOCS_IN_PRODUCTION=false

# Real Redis with passwords; separate DBs per concern
CACHE_REDIS_HOST=prod-redis.example.com
CACHE_REDIS_PASSWORD=<from secrets manager>
CACHE_REDIS_DB=0
RATE_LIMITER_REDIS_HOST=prod-redis.example.com
RATE_LIMITER_REDIS_PASSWORD=<from secrets manager>
RATE_LIMITER_REDIS_DB=1
SESSION_REDIS_DB=2
TASKIQ_REDIS_HOST=prod-redis.example.com
TASKIQ_REDIS_PASSWORD=<from secrets manager>
TASKIQ_REDIS_DB=3

DEFAULT_RATE_LIMIT_LIMIT=100
DEFAULT_RATE_LIMIT_PERIOD=60

# Admin (used by setup_initial_data only on first deploy)
ADMIN_NAME=System Administrator
ADMIN_EMAIL=admin@example.com
ADMIN_USERNAME=sysadmin
ADMIN_PASSWORD=<from secrets manager>

# Logging — production picks JSON automatically; LOG_PRODUCTION_OPTIMIZE quiets noisy libs
LOG_PRODUCTION_OPTIMIZE=true
LOG_FILE_ENABLED=true
LOG_FILE_PATH=/var/log/app/app.log
```

!!! danger "生产安全校验器"
    当 `ENVIRONMENT=production` 且 `PRODUCTION_SECURITY_VALIDATION_ENABLED=true` 时，应用发现不安全配置会拒绝启动，例如占位 `SECRET_KEY`、默认数据库凭据、后台缺少凭据或 `CORS_ORIGINS=*`。原文在此将两项均称为默认值。

<a id="detecting-the-environment-in-code"></a>

## 在代码中检测环境

`src/infrastructure/config/enums.py` 定义了 `EnvironmentOption`，避免直接比较魔法字符串：

```python
from src.infrastructure.config.settings import EnvironmentOption, get_settings

settings = get_settings()

if settings.ENVIRONMENT == EnvironmentOption.PRODUCTION:
    # production-only code
    ...
```

如需辅助方法，可在配置类中添加小型属性，但大多数情况上述方式已足够。避免在业务逻辑中分散环境分支，尽量集中在启动或中间件中。

<a id="managing-multiple-environments"></a>

## 管理多个环境

<a id="one-env-per-environment"></a>

### 每个环境一个 `.env`

最简单的方式：把 `.env.development`、`.env.staging`、`.env.production` 保存在版本控制之外，如密钥管理器中，并为当前环境创建符号链接：

```bash
# Switch environments locally
ln -sf .env.staging backend/.env
```

预发布和生产环境更常见的做法是：

- 部署时从 AWS Secrets Manager、Vault、Doppler 或 1Password 获取密钥。
- 在 CI 中生成 `.env`，或直接在 Kubernetes、ECS、systemd 等运行环境设置变量。

<a id="docker-compose-overrides"></a>

### Docker Compose 覆盖配置

Compose 使用叠加文件：

```bash
# Development: docker-compose.yml + docker-compose.override.yml (auto-loaded)
docker compose up

# Staging
docker compose -f docker-compose.yml -f docker-compose.staging.yml up

# Production
docker compose -f docker-compose.yml -f docker-compose.production.yml up
```

覆盖文件只描述变化，如目标构建阶段、移除开发卷、扩容和密钥，无需复制完整服务定义。

<a id="validating-configuration"></a>

## 验证配置

快速确认应用读取了预期配置：

```bash
cd backend
uv run --no-sync python -c "
from src.infrastructure.config.settings import get_settings
s = get_settings()
print(f'env       : {s.ENVIRONMENT}')
print(f'debug     : {s.DEBUG}')
print(f'app       : {s.APP_NAME} v{s.VERSION}')
print(f'db host   : {s.POSTGRES_SERVER}:{s.POSTGRES_PORT}/{s.POSTGRES_DB}')
print(f'cache     : {s.CACHE_BACKEND} -> {s.CACHE_REDIS_HOST}:{s.CACHE_REDIS_PORT}')
print(f'cors      : {s.CORS_ORIGINS}')
print(f'sessions  : secure={s.SESSION_SECURE_COOKIES} csrf={s.CSRF_ENABLED}')
"
```

生产安全校验器在启动时运行，失败则在绑定端口前退出，这是最直接的配置信号。

<a id="best-practices"></a>

## 最佳实践

<a id="security"></a>

### 安全

- 每个环境生成独立 `SECRET_KEY`，不重复使用。
- 从密钥管理器获取密钥，不从提交到 Git 的文件读取。
- 开发环境之外始终设置 `SESSION_SECURE_COOKIES=true`。
- 预发布和生产环境的 `CORS_ORIGINS` 限定为真实域名。
- 为预发布和生产 Redis 设置密码。
- 生产环境保持 `PRODUCTION_SECURITY_VALIDATION_ENABLED=true`。

<a id="performance"></a>

### 性能

- 需要持久化或多 DB 隔离时使用 Redis，而非 Memcached。
- 缓存、限流和 Taskiq 使用不同 Redis DB，默认为 0 / 1 / 3。
- 按负载调整 `POSTGRES_POOL_SIZE`，默认 20。
- I/O 密集任务可提高 Taskiq 的 `--max-async-tasks`。

<a id="operations"></a>

### 运维

- 环境专属值保存在密钥管理器中，而非环境文件。
- 在 `.env.example` 记录新增自定义变量。
- 生产前先在预发布环境测试部署。
- 关注启动日志，安全校验器会指出问题。

<a id="see-also"></a>

## 另请参阅

- **[环境变量](environment-variables.md)**：全部变量参考。
- **[配置类](settings-classes.md)**：变量如何转为 Python 配置。
- **[Docker 配置](docker-setup.md)**：各环境的 Compose 配置。
- **[生产环境](../production.md)**：生产部署指南。
