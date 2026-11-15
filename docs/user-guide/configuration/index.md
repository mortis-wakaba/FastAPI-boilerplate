<a id="configuration"></a>

# 配置

本章介绍如何为不同环境配置 FastAPI Boilerplate。配置由环境变量驱动，并通过 Python 配置类校验。

<a id="what-youll-learn"></a>

## 内容概览

- **[环境变量](environment-variables.md)**：通过 `.env` 文件配置。
- **[配置类](settings-classes.md)**：基于 Python 的配置管理。
- **[Docker 配置](docker-setup.md)**：容器和服务配置。
- **[分环境配置](environment-specific.md)**：开发、预发布和生产环境。

<a id="quick-start"></a>

## 快速开始

```bash
cd backend
cp .env.example .env
$EDITOR .env
```

必要变量：

```env
# Application
APP_NAME=My FastAPI App
SECRET_KEY=your-super-secret-key-here

# Database
POSTGRES_USER=your_user
POSTGRES_PASSWORD=your_password
POSTGRES_DB=your_database

# Admin Account (used by setup_initial_data)
ADMIN_NAME=Admin User
ADMIN_EMAIL=admin@example.com
ADMIN_USERNAME=admin
ADMIN_PASSWORD=secure_password
```

<a id="configuration-architecture"></a>

## 配置架构

```
Environment Variables (.env file)
         ↓
Settings Classes (Pydantic BaseSettings)
         ↓
Application Code (via get_settings())
```

<a id="layer-1-environment-variables"></a>

### 第一层：环境变量

主要通过 `backend/.env` 配置：

```env
POSTGRES_USER=myuser
POSTGRES_PASSWORD=mypassword
CACHE_REDIS_HOST=localhost
SECRET_KEY=your-secret-key
```

<a id="layer-2-settings-classes"></a>

### 第二层：配置类

`src/infrastructure/config/settings.py` 中的 Pydantic `BaseSettings` 类校验配置并提供结构化访问：

```python
class DatabaseSettings(BaseSettings):
    POSTGRES_USER: str = config("POSTGRES_USER", default="postgres")
    POSTGRES_PASSWORD: str = config("POSTGRES_PASSWORD", default="postgres")
    POSTGRES_SERVER: str = config("POSTGRES_SERVER", default="localhost")
    POSTGRES_PORT: int = config("POSTGRES_PORT", default=5432)
    POSTGRES_DB: str = config("POSTGRES_DB", default="postgres")
```

最终由一个组合的 `Settings` 类整合。

<a id="layer-3-application-use"></a>

### 第三层：应用使用

应用任意位置都可以通过 `get_settings()` 获取配置：

```python
from src.infrastructure.config.settings import get_settings

settings = get_settings()
print(settings.DATABASE_URL)
```

<a id="key-configuration-areas"></a>

## 主要配置领域

<a id="application-settings"></a>

### 应用设置

```env
APP_NAME=Your App Name
VERSION=1.0.0
ENVIRONMENT=development  # development | staging | production | local
DEBUG=false
```

<a id="database"></a>

### 数据库

```env
POSTGRES_USER=your_user
POSTGRES_PASSWORD=your_password
POSTGRES_SERVER=localhost   # use "postgres" with Docker Compose
POSTGRES_PORT=5432
POSTGRES_DB=your_database
CREATE_TABLES_ON_STARTUP=true   # the default in local and development
```

<a id="security-sessions"></a>
<a id="security--sessions"></a>

### 安全与会话

```env
SECRET_KEY=your-super-secret-key-here

SESSION_TIMEOUT_MINUTES=30
SESSION_SECURE_COOKIES=true
SESSION_BACKEND=redis           # redis | database | memory
CSRF_ENABLED=true
TRUSTED_PROXY_HOPS=0            # trusted reverse proxies in front of the app
```

登录锁定由 `crudauth` 内部实现，按 IP 和登录标识逐级加重，返回带 `Retry-After` 的 `429`。本节原文称其不通过环境变量配置。

> 译注：此处与入门配置指南列出的 `LOGIN_*` 变量不一致，使用时请核对当前代码。

<a id="cache-redis-or-memcached"></a>

### 缓存（Redis 或 Memcached）

```env
CACHE_ENABLED=true
CACHE_BACKEND=redis           # or "memcached"
CACHE_REDIS_HOST=localhost    # use "redis" with Docker Compose
CACHE_REDIS_PORT=6379
CACHE_REDIS_DB=0
```

<a id="background-tasks-taskiq"></a>

### 后台任务（Taskiq）

```env
TASKIQ_BROKER_TYPE=redis      # or "rabbitmq"
TASKIQ_REDIS_HOST=localhost   # use "redis" with Docker Compose
TASKIQ_REDIS_PORT=6379
TASKIQ_REDIS_DB=3
```

<a id="rate-limiting"></a>

### 限流

```env
RATE_LIMITER_ENABLED=true
RATE_LIMITER_REDIS_HOST=localhost
RATE_LIMITER_REDIS_DB=1
DEFAULT_RATE_LIMIT_LIMIT=100
DEFAULT_RATE_LIMIT_PERIOD=60
```

<a id="admin-user-initial-setup"></a>

### 管理员用户（初始化）

首次运行 `python -m scripts.setup_initial_data` 时读取：

```env
ADMIN_NAME=Admin User
ADMIN_EMAIL=admin@example.com
ADMIN_USERNAME=admin
ADMIN_PASSWORD=secure_password
```

<a id="environment-specific-configurations"></a>

## 分环境配置

<a id="development"></a>

### 开发环境

```env
ENVIRONMENT=development
DEBUG=true
POSTGRES_SERVER=localhost
CACHE_REDIS_HOST=localhost
TASKIQ_REDIS_HOST=localhost
RATE_LIMITER_REDIS_HOST=localhost
```

<a id="staging"></a>

### 预发布环境

```env
ENVIRONMENT=staging
DEBUG=false
POSTGRES_SERVER=staging-db.example.com
CACHE_REDIS_HOST=staging-redis.example.com
SESSION_SECURE_COOKIES=true
```

<a id="production"></a>

### 生产环境

```env
ENVIRONMENT=production
DEBUG=false
POSTGRES_SERVER=prod-db.example.com
CACHE_REDIS_HOST=prod-redis.example.com
PRODUCTION_SECURITY_VALIDATION_ENABLED=true
ENABLE_DOCS_IN_PRODUCTION=false
```

<a id="docker-configuration"></a>

## Docker 配置

Docker Compose 自动加载 `.env` 中的变量。Compose 服务之间通过服务名访问：

```env
POSTGRES_SERVER=postgres
CACHE_REDIS_HOST=redis
RATE_LIMITER_REDIS_HOST=redis
TASKIQ_REDIS_HOST=redis
```

<a id="service-overview"></a>

### 服务概览

```yaml
services:
  web:    # FastAPI application
  db:     # PostgreSQL
  redis:  # Cache, rate limiting, sessions, taskiq broker
```

运行 Taskiq Worker 时，在 Compose 文件中添加 Worker 服务，使用命令 `taskiq worker src.infrastructure.taskiq.worker:default_broker`。

<a id="common-configuration-patterns"></a>

## 常见配置模式

<a id="feature-toggles"></a>

### 功能开关

模板已有 `CACHE_ENABLED`、`RATE_LIMITER_ENABLED`、`ADMIN_ENABLED` 和 `CSRF_ENABLED` 等开关。也可以在配置类中添加自己的开关：

```python
class FeatureSettings(BaseSettings):
    ENABLE_ANALYTICS: bool = config("ENABLE_ANALYTICS", default=False, cast=bool)
```

然后使用：

```python
from src.infrastructure.config.settings import get_settings

if get_settings().ENABLE_ANALYTICS:
    track_event(...)
```

<a id="environment-detection"></a>

### 环境检测

```python
from src.infrastructure.config.settings import EnvironmentOption, get_settings

if get_settings().ENVIRONMENT == EnvironmentOption.PRODUCTION:
    ...
```

<a id="generate-secret-key"></a>

### 生成密钥

```bash
python -c "import secrets; print(secrets.token_urlsafe(64))"
```

<a id="best-practices"></a>

## 最佳实践

<a id="security"></a>

### 安全

- 不要把 `.env` 提交到版本控制。
- 每个环境使用独立、足够强的 `SECRET_KEY`。
- 生产环境禁用调试模式。
- 设置 `ENVIRONMENT=production`，启用安全校验器。
- 将 `CORS_ORIGINS` 限定为具体域名。

<a id="performance"></a>

### 性能

- 根据负载调整 `POSTGRES_POOL_SIZE` 和 `POSTGRES_MAX_OVERFLOW`。
- 缓存、限流和 Taskiq 分别使用独立 Redis DB，默认是 `CACHE_REDIS_DB=0`、`RATE_LIMITER_REDIS_DB=1`、`TASKIQ_REDIS_DB=3`。
- 合理设置 `SESSION_TIMEOUT_MINUTES` 和 `MAX_SESSIONS_PER_USER`。

<a id="maintenance"></a>

### 维护

- 在 `.env.example` 记录自定义变量。
- 在配置类中添加校验，Pydantic 类型可以捕获多数问题。
- 生产部署前先在预发布环境验证配置。

<a id="getting-started"></a>

## 阅读顺序

1. **[环境变量](environment-variables.md)**：全部变量参考。
2. **[配置类](settings-classes.md)**：Python 配置组织方式。
3. **[Docker 配置](docker-setup.md)**：Compose 文件和覆盖配置。
4. **[分环境配置](environment-specific.md)**：各环境的最佳实践。

模板提供合理的默认值，只覆盖需要调整的选项即可。
