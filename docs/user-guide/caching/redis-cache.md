<a id="server-side-cache-redis-or-memcached"></a>

# 服务端缓存（Redis 或 Memcached）

服务端按请求结构缓存响应，让相同请求跳过数据库。本页介绍 `@cache` 和 provider API。通过 `CACHE_BACKEND` 选后端；“Redis”是历史名称，除特别说明外两种后端都适用。

<a id="the-cache-decorator"></a>

## `@cache` 装饰器

缓存 GET，写操作后失效。

<a id="basic-usage"></a>

### 基本用法

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

生成键 `widget:{widget_id}`，命中时直接返回、不执行处理器。

!!! warning "必须有 `request: Request`"
    装饰器读取 `request.method` 决定缓存或失效，被装饰路由的首个参数必须为 `request: Request`，否则报错。

<a id="how-it-works"></a>

### 工作方式

1. **GET**：先查缓存；命中直接返回，未命中执行并缓存。`[]`、`{}`、`""` 等空结果也缓存，避免空列表不断查询。
2. **PUT / PATCH / POST / DELETE**：成功执行后删除相同 `(key_prefix, resource_id)` 的缓存键。写请求的查询字符串不参与该键，确保清除资源条目而非从未缓存的键。分页列表每页独立，全部清理需 `pattern_to_invalidate_extra`。
3. **失败放行**：缓存错误记录警告后执行处理器，接口继续可用。

<a id="decorator-parameters"></a>

### 参数

```python
@cache(
    key_prefix: str,                                  # required — cache namespace
    resource_id_name: Any = None,                     # name of the parameter holding the resource ID
    expiration: int = 3600,                           # TTL in seconds (default: 1 hour)
    to_invalidate_extra: dict[str, Any] | None = None,
    pattern_to_invalidate_extra: list[str] | None = None,
    backend_name: str | None = None,                  # if you've registered multiple backends
    per_caller: bool = True,                          # one entry per caller, unless the response is shared
    caller_from: str | None = None,                   # the argument the caller is read from
)
```

键格式为 `prefix[:resource_id][:q=query][:u=caller]`。查询部分读取路由声明的参数，包括别名、依赖读取参数和查询模型字段，排序并百分号转义，因此参数顺序不同共享条目，值中的 `&`、`=` 不会被误解。无法读取路由时使用请求全部参数，避免完全忽略。不同页、不同调用者及同账户不同 scope 不共享条目。

<a id="key_prefix-cache-namespace"></a>
<a id="key_prefix--cache-namespace"></a>

### `key_prefix`：命名空间

可用 `{kwarg_name}` 插值路由参数：

```python
@cache(key_prefix="widget", ...)
# → "widget:42"

@cache(key_prefix="user_{username}_widgets", ...)
# → "user_johndoe_widgets:42"

@cache(key_prefix="user_{user_id}_widgets:page_{page}:size_{items_per_page}", ...)
# → "user_5_widgets:page_1:size_10:42"
```

占位符来自处理器关键字参数，包括路径、查询和依赖。

<a id="resource_id_name-which-argument-is-the-id"></a>
<a id="resource_id_name--which-argument-is-the-id"></a>

### `resource_id_name`：资源 ID 参数

在前缀后附加 `:{resource_id}`，解析方式：

```python
# Explicit — recommended
@cache(key_prefix="widget", resource_id_name="widget_id", expiration=600)
async def get_widget(request: Request, widget_id: int, ...): ...

# A string id reads the same as any other
@cache(key_prefix="user", resource_id_name="username", per_caller=False)
async def get_user(request: Request, username: str, ...): ...
```

未设置则没有 ID 部分，适合以查询字符串标识页码的列表。

<a id="per_caller-who-the-entry-belongs-to"></a>
<a id="per_caller--who-the-entry-belongs-to"></a>

### `per_caller`：条目归属

默认每个调用者独立，`caller_from` 指定包含 `Principal`（`CurrentPrincipalDep`）的参数。启用逐调用者缓存却未指定参数名时，声明处抛 `ValueError`。带 scope 的密钥与所有者会话、其他范围分别存储，因为能力不同。

必须是 principal；`current_user` 字典仅描述账户，无法区分凭据，可能让密钥读取会话结果。参数不是 principal 或为空时，**完全不缓存**，记录警告后执行，避免串用他人数据。

所有人响应一致时用 `per_caller=False`，见[逐调用者与共享缓存](cache-strategies.md#per-caller-vs-shared-caches)。

<a id="what-ships-cached"></a>

### 模板中已有的缓存

`GET /api/v1/rate-limits/` 是随附的缓存路由：超级用户查看等级配置，对所有人相同，`per_caller=False`，TTL 为 `RATE_LIMITS_CACHE_SECONDS`。PATCH 和 DELETE 设置 `pattern_to_invalidate_extra=['rate_limits:*']`，清全部页。Redis 支持，Memcached 不支持并记录日志。

路由通过 `wiring/cache.py` 获取装饰器：

```python
from ...wiring.cache import cached

@cached(key_prefix="rate_limits", per_caller=False)
```

缓存可移除，wiring 因此提供真实装饰器或原样返回路由的 `infrastructure/uncached.py`，路由无需依赖项目是否保留缓存。

<a id="invalidation"></a>

## 失效

<a id="automatic-on-mutations"></a>

### 写操作自动失效

相同 `(key_prefix, resource_id_name)` 的非 GET 路由自动清理对应键：

```python
@router.patch("/{widget_id}")
@cache(key_prefix="widget", resource_id_name="widget_id")
async def update_widget(request: Request, widget_id: int, ...) -> dict[str, Any]:
    # PATCH automatically deletes "widget:{widget_id}" after the handler runs
    return await widget_service.update(widget_id, values, db)
```

在处理器成功返回**之后**删除，抛错不动缓存。

<a id="invalidating-related-keys-to_invalidate_extra"></a>

### 关联键失效：`to_invalidate_extra`

写资源时清理引用它的其他缓存，使用 `{prefix: id_kwarg}` 字典：

```python
@router.post("/")
@cache(
    key_prefix="widget",
    resource_id_name="widget_id",
    to_invalidate_extra={
        "user_widgets": "owner_id",      # also invalidate "user_widgets:{owner_id}"
        "widget_count": "global",        # invalidate "widget_count:global"
    },
)
async def create_widget(
    request: Request,
    widget: WidgetCreate,
    owner_id: int,
    ...,
) -> dict[str, Any]:
    return await widget_service.create(widget, owner_id, db)
```

`id_kwarg` 可以是字面值如 `global`，或从路由关键字解析的 `owner_id`。

!!! note "仅写操作"
    GET 不允许 `to_invalidate_extra` 和 `pattern_to_invalidate_extra`，否则抛 `InvalidRequestError`。仅 PUT / PATCH / POST / DELETE 执行失效。

<a id="pattern-based-invalidation-pattern_to_invalidate_extra"></a>

### 模式失效：`pattern_to_invalidate_extra`

批量清理使用 Redis 模式：

```python
@router.delete("/{widget_id}")
@cache(
    key_prefix="widget",
    resource_id_name="widget_id",
    pattern_to_invalidate_extra=[
        "user_{owner_id}_widgets:*",       # all paginated lists for this user
        "widget_search:*",                  # all search result caches
    ],
)
async def delete_widget(
    request: Request,
    widget_id: int,
    owner_id: int,
    ...,
) -> None:
    await widget_service.delete(widget_id, db)
```

**Memcached 不支持模式**，会抛 `PatternMatchingNotSupportedError` 并记录错误，普通键删除仍执行。

<a id="the-provider-api"></a>

## Provider API

后台任务、服务和脚本等非路由代码直接操作：

```python
from src.infrastructure.cache.provider import (
    cache_provider,
    clear,
    delete,
    delete_pattern,
    exists,
    get,
    set,
)


# Store a value (any JSON-serializable type)
await set(key="config:site_name", value="My App", expiration=3600)

# Retrieve it
value = await get(key="config:site_name")

# Existence check
if await exists(key="config:site_name"):
    ...

# Delete a key
await delete(key="config:site_name")

# Delete by pattern (Redis only)
await delete_pattern(pattern="user:42:*")

# Clear everything
await clear()
```

值自动通过 `fastapi.encoders.jsonable_encoder` 序列化，支持字典、列表等 JSON 兼容结构。

<a id="cache-key-conventions"></a>

## 缓存键约定

生成格式：

```
{formatted_key_prefix}:{resource_id}
```

| 模式 | 用途 |
|------|------|
| `widget:42` | 按 ID 的单资源 |
| `user_widgets:5` | 用户资源列表 |
| `user_{username}_widgets:page_{page}` | 用户范围分页 |
| `search:{query_hash}` | 哈希后的搜索 |
| `analytics_{user_id}_30d:report` | 时间窗口分析 |

前缀应唯一，避免无关缓存碰撞；匹配失效方式，例如按用户清理就包含用户 ID；可读可预测，便于调试。

<a id="configuration"></a>

## 配置

```env
CACHE_ENABLED=true
CACHE_BACKEND=redis              # or "memcached"

# Redis backend
CACHE_REDIS_HOST=redis           # use "localhost" without Docker
CACHE_REDIS_PORT=6379
CACHE_REDIS_DB=0
CACHE_REDIS_PASSWORD=
CACHE_REDIS_CONNECT_TIMEOUT=5
CACHE_REDIS_POOL_SIZE=10
```

关闭缓存后每次执行处理器，适合测试或定位性能问题。

<a id="picking-expiration-times"></a>

## 过期时间

| 数据 | 建议 TTL |
|------|----------|
| 国家、等级等静态参考 | 24 小时，86400 秒 |
| 用户资料 / 公开对象 | 5–30 分钟，300–1800 秒 |
| 分页列表 | 1–5 分钟，60–300 秒 |
| 搜索结果 | 5–15 分钟，300–900 秒 |
| 高频变化仪表盘 | 30–60 秒 |

默认一小时 3600 秒，按可接受陈旧程度覆盖。

<a id="real-examples"></a>

## 实际示例

本节原文称内置路由尚未使用 `@cache`，建议在读多写少数据上自行添加。

> 译注：与上文已缓存 `GET /api/v1/rate-limits/` 的说明不一致。

<a id="performance-tips"></a>

## 性能建议

<a id="use-schema_to_select-together-with-caching"></a>

### 配合 `schema_to_select`

底层使用 `schema_to_select=WidgetRead`，可缓存精简字典，减少体积和序列化成本。

<a id="dont-cache-personalized-responses-globally"></a>

### 不要全局缓存个性化响应

响应按用户不同但键只有资源 ID，会导致用户看到彼此数据。原文建议前缀包含 `user_id`，如 `widget_for_user_{user_id}`，或不要缓存；上文的新模式则使用 principal 隔离。

<a id="cache-the-response-not-the-computation"></a>

### 缓存响应或子计算

装饰器缓存处理器返回值，只需缓存昂贵子计算时，在服务中直接用 provider。

<a id="watch-pool-saturation"></a>

### 关注连接池

`CACHE_REDIS_POOL_SIZE=10` 适合一般负载，高并发可提高，日志 `redis.exceptions.ConnectionError` 可能表示连接池耗尽。

<a id="graceful-degradation"></a>

## 降级

缓存不可达时捕获错误并执行处理器，以未缓存速度运行。数据可重建，因此采用失败放行，日志会出现：

```
Cache backend not available: <error>
```

据此排查缓存基础设施。

<a id="troubleshooting"></a>

## 故障排查

<a id="decorator-never-reads-from-cache"></a>

### 始终不读缓存

检查首个参数是否 `request: Request`，否则无法识别方法。

<a id="pattern-invalidation-fails-on-memcached"></a>

### Memcached 模式失效失败

属于预期，改用 Redis 或 `to_invalidate_extra` 显式删除。

<a id="cached-data-is-stale-after-a-mutation"></a>

### 写入后数据仍旧

写和读必须使用相同 `(key_prefix, resource_id_name)`，例如 `widget` 与 `widget_cache` 不会相互失效。

<a id="cache-returns-the-wrong-users-data"></a>

### 返回其他用户数据

键只有资源 ID，没有用户维度，见上文个性化响应说明。

<a id="key-files"></a>

## 关键文件

| 组件 | 位置 |
|------|------|
| 装饰器 | `backend/src/infrastructure/cache/decorator.py` |
| Provider | `backend/src/infrastructure/cache/provider.py` |
| Redis | `backend/src/infrastructure/cache/backends/redis.py` |
| Memcached | `backend/src/infrastructure/cache/backends/memcached.py` |
| 配置 | `backend/src/infrastructure/config/settings.py` 的 `CacheSettings` |

<a id="next-steps"></a>

## 后续步骤

- **[客户端缓存](client-cache.md)**：浏览器缓存头。
- **[缓存策略](cache-strategies.md)**：键、关联失效、旁路缓存。
- **[环境变量](../configuration/environment-variables.md#cache)**：完整配置。
