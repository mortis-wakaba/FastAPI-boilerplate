<a id="settings-classes"></a>

# 配置类

配置位于 `backend/src/infrastructure/config/settings.py`，使用 Pydantic `BaseSettings` 类组织。每个类组合相关变量，再由一个 `Settings` 类统一组合。默认值通过 Starlette 的 `Config()` 加载器从 `backend/.env` 获取。

<a id="settings-architecture"></a>

## 配置架构

```python
# src/infrastructure/config/settings.py
from pydantic_settings import BaseSettings
from starlette.config import Config

config = Config(env_path)  # reads backend/.env


class Settings(
    EnvironmentSettings,
    DatabaseSettings,
    CacheSettings,
    RateLimiterSettings,
    CORSSettings,
    CompressionSettings,
    APIDocSettings,
    AuthSettings,
    APISettings,
    AppSettings,
    AdminSettings,
    SQLAdminSettings,
    SecuritySettings,
    LoggingSettings,
    TaskiqSettings,
):
    """Main settings class that combines all setting categories."""

    pass


settings = Settings()


def get_settings() -> Settings:
    return settings
```

在应用任意位置使用：

```python
from src.infrastructure.config.settings import get_settings

settings = get_settings()
print(settings.APP_NAME)
```

<a id="built-in-settings-groups"></a>

## 内置配置分组

模板提供的配置类位于 `src/infrastructure/config/settings.py`：

| 类 | 内容 |
|----|------|
| `EnvironmentSettings` | `ENVIRONMENT`，可选 production / staging / development / local |
| `DatabaseSettings` | 所有 `POSTGRES_*` 变量和 `DATABASE_URL` 计算属性 |
| `CacheSettings` | `CACHE_*`，包括 Redis、Memcached 和客户端缓存 |
| `RateLimiterSettings` | `RATE_LIMITER_*`，包括 Redis、Memcached 和默认值 |
| `CORSSettings` | `CORS_*` |
| `CompressionSettings` | `GZIP_*` |
| `APIDocSettings` | `ENABLE_DOCS_IN_PRODUCTION`、`OPENAPI_PREFIX`、`DOCS_URL`、`REDOC_URL`、`OPENAPI_URL`，以及 `API_*` 文档元数据：标题、摘要、描述、版本、服务条款、联系人、许可证和标签 |
| `AuthSettings` | `SECRET_KEY`、`SESSION_*`、`CSRF_ENABLED`、`TRUSTED_PROXY_HOPS`、`OAUTH_*` |
| `APISettings` | `API_PREFIX` |
| `AppSettings` | `APP_NAME`、`APP_DESCRIPTION`、`VERSION`、`DEBUG` |
| `AdminSettings` | `ADMIN_NAME`、`ADMIN_EMAIL`、`ADMIN_USERNAME`、`ADMIN_PASSWORD`、`DEFAULT_TIER_NAME` |
| `SQLAdminSettings` | `ADMIN_ENABLED` |
| `SecuritySettings` | `SECRET_KEY`、`PRODUCTION_SECURITY_VALIDATION_ENABLED`、`SECURITY_HEADERS_ENABLED` |
| `LoggingSettings` | 所有 `LOG_*` 变量 |
| `TaskiqSettings` | `TASKIQ_*`，包括 Redis、RabbitMQ 和 Worker 调优 |
| `APIKeySettings` | `KEY_USAGE_RETENTION_DAYS` |

<a id="anatomy-of-a-settings-group"></a>

## 配置分组的结构

典型配置类：

```python
class DatabaseSettings(BaseSettings):
    """Database-related settings."""

    POSTGRES_USER: str = config("POSTGRES_USER", default="postgres")
    POSTGRES_PASSWORD: str = config("POSTGRES_PASSWORD", default="postgres")
    POSTGRES_SERVER: str = config("POSTGRES_SERVER", default="localhost")
    POSTGRES_PORT: int = config("POSTGRES_PORT", default=5432)
    POSTGRES_DB: str = config("POSTGRES_DB", default="postgres")
    POSTGRES_ASYNC_PREFIX: str = config("POSTGRES_ASYNC_PREFIX", default="postgresql+asyncpg://")
    CREATE_TABLES_ON_STARTUP: bool = config(
        "CREATE_TABLES_ON_STARTUP", default=_creates_tables_by_default(), cast=bool
    )
    POSTGRES_POOL_SIZE: int = config("POSTGRES_POOL_SIZE", default=20, cast=int)

    @property
    def DATABASE_URL(self) -> str:
        """Construct the full database URL.

        Falls back to assembling from POSTGRES_* if DATABASE_URL is not set.
        """
        direct_url = config("DATABASE_URL", default=None)
        if direct_url:
            return direct_url
        return (
            f"{self.POSTGRES_ASYNC_PREFIX}{self.POSTGRES_USER}:"
            f"{self.POSTGRES_PASSWORD}@{self.POSTGRES_SERVER}:"
            f"{self.POSTGRES_PORT}/{self.POSTGRES_DB}"
        )
```

要点：

- 字段使用 `config("VAR_NAME", default=..., cast=...)`。`Config()` 以 `backend/.env` 初始化，在导入时加载值。
- 使用 `cast=int`、`cast=bool` 等进行类型转换。
- 派生值（如 `DATABASE_URL`）使用 `@property`。调用者通过 `get_settings()` 访问，无需 `@computed_field`。

<a id="adding-custom-settings"></a>

## 添加自定义配置

<a id="basic-custom-group"></a>

### 基本分组

```python
# backend/src/infrastructure/config/settings.py

class StorageSettings(BaseSettings):
    """File-storage settings."""

    STORAGE_BACKEND: str = config("STORAGE_BACKEND", default="local")     # "local" or "s3"
    LOCAL_STORAGE_PATH: str = config("LOCAL_STORAGE_PATH", default="./uploads")

    AWS_ACCESS_KEY_ID: str = config("AWS_ACCESS_KEY_ID", default="")
    AWS_SECRET_ACCESS_KEY: str = config("AWS_SECRET_ACCESS_KEY", default="")
    AWS_BUCKET_NAME: str = config("AWS_BUCKET_NAME", default="")
    AWS_REGION: str = config("AWS_REGION", default="us-east-1")

    MAX_UPLOAD_SIZE_BYTES: int = config("MAX_UPLOAD_SIZE_BYTES", default=10_485_760, cast=int)


class Settings(
    EnvironmentSettings,
    DatabaseSettings,
    # ...existing groups...
    StorageSettings,        # add yours
):
    pass
```

在 `backend/.env.example` 中添加对应变量，便于发现和查阅。

<a id="computed-derived-values"></a>
<a id="computed--derived-values"></a>

### 计算值与派生值

```python
class StorageSettings(BaseSettings):
    STORAGE_BACKEND: str = config("STORAGE_BACKEND", default="local")
    AWS_ACCESS_KEY_ID: str = config("AWS_ACCESS_KEY_ID", default="")
    AWS_SECRET_ACCESS_KEY: str = config("AWS_SECRET_ACCESS_KEY", default="")
    AWS_BUCKET_NAME: str = config("AWS_BUCKET_NAME", default="")

    @property
    def s3_configured(self) -> bool:
        return bool(self.AWS_ACCESS_KEY_ID and self.AWS_SECRET_ACCESS_KEY and self.AWS_BUCKET_NAME)

    @property
    def storage_enabled(self) -> bool:
        if self.STORAGE_BACKEND == "local":
            return True
        if self.STORAGE_BACKEND == "s3":
            return self.s3_configured
        return False
```

<a id="validation"></a>

### 校验

需要更复杂的校验时，使用 Pydantic 校验器：

```python
from pydantic import field_validator, model_validator


class StorageSettings(BaseSettings):
    STORAGE_BACKEND: str = config("STORAGE_BACKEND", default="local")
    MAX_UPLOAD_SIZE_BYTES: int = config("MAX_UPLOAD_SIZE_BYTES", default=10_485_760, cast=int)

    @field_validator("MAX_UPLOAD_SIZE_BYTES")
    @classmethod
    def _check_upload_size(cls, v: int) -> int:
        if v < 1024:
            raise ValueError("MAX_UPLOAD_SIZE_BYTES must be at least 1KB")
        if v > 100 * 1024 * 1024:
            raise ValueError("MAX_UPLOAD_SIZE_BYTES cannot exceed 100MB")
        return v

    @model_validator(mode="after")
    def _check_backend(self) -> "StorageSettings":
        if self.STORAGE_BACKEND not in ("local", "s3"):
            raise ValueError(f"Unknown STORAGE_BACKEND: {self.STORAGE_BACKEND}")
        return self
```

校验器在启动阶段实例化 `Settings()` 时运行，因此错误配置会尽早失败。

<a id="enums-for-constrained-values"></a>

## 使用枚举约束取值

对于取值集合固定的选项，在 `src/infrastructure/config/enums.py` 定义 `StrEnum` 并用作默认值：

```python
# enums.py
from enum import StrEnum


class StorageBackend(StrEnum):
    LOCAL = "local"
    S3 = "s3"


# settings.py
from .enums import StorageBackend


class StorageSettings(BaseSettings):
    STORAGE_BACKEND: str = config("STORAGE_BACKEND", default=StorageBackend.LOCAL.value)
```

模板已对 `CacheBackend`、`LogFormat`、`LogLevel`、`SessionBackend`、`TaskiqBrokerType` 和 `EnvironmentOption` 使用此方式。

<a id="removing-built-in-groups"></a>

## 移除内置分组

不使用某功能时，从 `Settings` 的 MRO 中移除对应类：

```python
class Settings(
    EnvironmentSettings,
    DatabaseSettings,
    CORSSettings,
    AuthSettings,
    APISettings,
    AppSettings,
    LoggingSettings,
    # CacheSettings — removed
    # RateLimiterSettings — removed
    # TaskiqSettings — removed
):
    pass
```

同时需要：

- 移除依赖这些配置的代码，如缓存装饰器、Taskiq 消息代理和限流中间件。
- 从 `.env.example` 删除对应变量。
- 在 `infrastructure/app_factory.py` 禁止启动这些子系统。

<a id="testing-settings"></a>

## 测试配置

测试套件通过 fixture 覆盖配置，常见模式如下：

```python
import pytest
from src.infrastructure.config.settings import Settings


@pytest.fixture
def test_settings(monkeypatch):
    monkeypatch.setenv("ENVIRONMENT", "local")
    monkeypatch.setenv("CACHE_ENABLED", "false")
    monkeypatch.setenv("RATE_LIMITER_ENABLED", "false")
    return Settings()
```

不使用环境变量的一次性覆盖，可直接传入关键字参数实例化对应配置类：

```python
def test_storage_validation():
    with pytest.raises(ValueError, match="cannot exceed 100MB"):
        StorageSettings(MAX_UPLOAD_SIZE_BYTES=200_000_000)
```

<a id="best-practices"></a>

## 最佳实践

<a id="organization"></a>

### 组织

- 按**子系统**（缓存、认证、Taskiq）分组，而非按环境分组。
- 校验逻辑与字段放在一起。
- 每个类添加一行 docstring 说明用途。
- `.env.example` 的小节标题与配置分组对应。

<a id="security"></a>

### 安全

- `ENVIRONMENT=production` 时校验 `SECRET_KEY` 长度和强度，模板的生产安全校验器已实现。
- 凭据不要设置真实默认值，留空并由校验器报告缺失。
- 通过 `@property` 派生连接字符串，而非直接嵌入环境变量。

<a id="performance"></a>

### 性能

- `Settings` 在导入时创建一次，通过 `get_settings()` 共享，不要每个请求重新实例化。
- 校验器保持轻量，启动和任何重新实例化时都会执行。

<a id="testing"></a>

### 测试

- 使用 `monkeypatch.setenv(...)` 为每个测试调整环境变量。
- 可以直接传实例时，避免依赖全局 `settings`。

<a id="see-also"></a>

## 另请参阅

- **[环境变量](environment-variables.md)**：完整变量参考。
- **[Docker 配置](docker-setup.md)**：变量如何传入 Compose。
- **[分环境配置](environment-specific.md)**：各环境推荐值。
