<a id="caching"></a>

# 缓存

模板支持 **Redis 或 Memcached** 服务端缓存，以及可配置的客户端缓存头；后端共用相同装饰器和 provider API。

缓存属于 `cache` 功能，移除后没有可导入的 `cached`、缓存配置或就绪报告中的缓存项，见[可组合功能](../composable-features.md)。

!!! tip "构建完整 SaaS？"
    缓存属于免费基础功能。[FastroAI](https://fastro.ai) 将其与 Stripe 支付、权益、邮件、前端和 AI Agent 整合。[更快交付 SaaS →](https://fastro.ai)

<a id="overview"></a>

## 概览

三层可按需组合：

- **`@cache`**：缓存 GET，POST / PUT / PATCH / DELETE 后自动失效。
- **Provider API**：供非路由代码直接 get、set、delete。
- **客户端缓存头**：中间件添加 `Cache-Control`。

<a id="quick-example"></a>

## 快速示例

```python
from typing import Annotated, Any
from fastapi import APIRouter, Depends, Request
from sqlalchemy.ext.asyncio import AsyncSession

from ...infrastructure.cache.decorator import cache
from ...infrastructure.database.session import async_session
from .schemas import WidgetRead
from .service import WidgetService

router = APIRouter()


@router.get("/{widget_id}", response_model=WidgetRead)
@cache(key_prefix="widget", resource_id_name="widget_id", expiration=600)
async def get_widget(
    request: Request,
    widget_id: int,
    db: Annotated[AsyncSession, Depends(async_session)],
    widget_service: Annotated[WidgetService, Depends(get_widget_service)],
) -> dict[str, Any]:
    return await widget_service.get_by_id(widget_id, db)
```

结果缓存 600 秒，相同 `widget_id` 的更新和删除自动失效。首个参数必须为 `request: Request`，装饰器据此判断方法。

<a id="architecture"></a>

## 架构

```text
HTTP Request
    ↓
@cache decorator                  (decorator.py)
    ↓ (cache miss)
APIRouter handler
    ↓
service / FastCRUD                (your code)
    ↓
PostgreSQL
```

命中时直接返回缓存、不执行处理器；未命中或发生写操作时执行处理器，再按配置缓存或失效。

<a id="whats-included"></a>

## 组件

| 组件 | 用途 | 位置 |
|------|------|------|
| `@cache` | 接口缓存与自动失效 | `infrastructure/cache/decorator.py` |
| Provider API | 直接缓存操作 | `infrastructure/cache/provider.py` |
| Redis 后端 | 模式匹配、持久化、丰富类型 | `infrastructure/cache/backends/redis.py` |
| Memcached 后端 | 轻量键值存储 | `infrastructure/cache/backends/memcached.py` |
| `ClientCacheMiddleware` | 添加缓存头 | `infrastructure/middleware.py` |

<a id="configuration"></a>

## 配置

```env
CACHE_ENABLED=true
CACHE_BACKEND=redis           # or "memcached"

# Redis backend
CACHE_REDIS_HOST=redis        # use "localhost" without Docker
CACHE_REDIS_PORT=6379
CACHE_REDIS_DB=0
CACHE_REDIS_PASSWORD=
CACHE_REDIS_POOL_SIZE=10

# Memcached backend (only when CACHE_BACKEND=memcached)
CACHE_MEMCACHED_HOST=localhost
CACHE_MEMCACHED_PORT=11211
CACHE_MEMCACHED_POOL_SIZE=10

# Client-side cache headers
CLIENT_CACHE_ENABLED=true
CLIENT_CACHE_MAX_AGE=60        # seconds
```

`CACHE_ENABLED=false` 时装饰器不执行缓存，适合测试。完整参考见[环境变量](../configuration/environment-variables.md#cache)。

<a id="picking-a-backend"></a>

## 选择后端

| 能力 | Redis | Memcached |
|------|-------|-----------|
| `delete_pattern` 模式删除 | 支持 | 不支持，抛 `PatternMatchingNotSupportedError` |
| 可选 AOF / RDB 持久化 | 支持 | 不支持 |
| 列表、集合、哈希等结构 | 支持 | 仅键值 |
| 内存效率 | 良好 | 优秀 |

需要模式失效、重启保留，或已为会话、限流、Taskiq 部署 Redis 时选 **Redis**。已有 Memcached 或偏好简单键值语义时选 **Memcached**。模板默认统一使用 Redis，同时承担会话和限流后端。

<a id="graceful-degradation"></a>

## 降级

缓存不可用时，装饰器捕获错误并执行原处理器，接口仍可用但更慢。缓存可从数据库重建，因此有意采用失败放行，避免缓存故障拖垮 API。

<a id="sub-pages"></a>

## 子页面

1. **[Redis 缓存](redis-cache.md)**：装饰器、provider 和失效模式。
2. **[客户端缓存](client-cache.md)**：缓存头与中间件。
3. **[缓存策略](cache-strategies.md)**：键名、关联失效与旁路缓存。

<a id="key-files"></a>

## 关键文件

| 组件 | 位置 |
|------|------|
| 装饰器 | `backend/src/infrastructure/cache/decorator.py` |
| Provider | `backend/src/infrastructure/cache/provider.py` |
| 后端 | `backend/src/infrastructure/cache/backends/` |
| 配置 | `backend/src/infrastructure/config/settings.py` 的 `CacheSettings` |
| 客户端中间件 | `backend/src/infrastructure/middleware.py` |

<a id="next-steps"></a>

## 后续步骤

先读 [Redis 缓存](redis-cache.md)，再读[缓存策略](cache-strategies.md)中的失效方法。
