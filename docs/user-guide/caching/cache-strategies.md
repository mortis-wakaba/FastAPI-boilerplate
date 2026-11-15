<a id="cache-strategies"></a>

# 缓存策略

本页汇总键名、失效、TTL 与写模式配合，以及无需独立调度系统的预热方式。示例均使用真实 API：[路由装饰器](redis-cache.md)及 `src.infrastructure.cache` 导出的 `get`、`set`、`delete`、`delete_pattern`、`exists`、`clear`。

<a id="picking-a-key-naming-scheme"></a>

## 键命名

格式为 `{formatted_key_prefix}[:{resource_id}][:q={query}][:u={caller}]`。包含声明的资源 ID、按路由声明名称读取后排序和百分号编码的查询参数，以及默认的调用者。参数别名、依赖读取参数和查询模型字段均计入。参数顺序不同共享条目，不同页不共享，分隔符不能混淆。`per_caller=False` 时省略调用者。

前缀和 `{kwarg}` 应避免无关碰撞、匹配失效范围，并便于在 `redis-cli KEYS '*'` 调试时阅读。

```python
# Simple resource by id
@cache(key_prefix="widget", resource_id_name="widget_id")
# → widget:42

# Per-user list, paginated
@cache(key_prefix="user_{user_id}_widgets:page_{page}:size_{items_per_page}",
       resource_id_name="user_id")
# → user_5_widgets:page_1:size_10:5

# String IDs (usernames, slugs, hashed query strings)
@cache(key_prefix="user", resource_id_name="username", per_caller=False)
# → user:johndoe

# Time-windowed analytics — bake the window into the prefix
@cache(key_prefix="analytics_{user_id}_30d", resource_id_name="report_id")
# → analytics_5_30d:summary
```

避免：以原始 ID 起头如 `{post_id}_comments`；直接拼接无限长用户输入如 `search:{raw_query}`，应先哈希；无意义的混合前缀如 `data:42`。

<a id="invalidation-strategies"></a>

## 失效策略

<a id="1-ttl-only-eventually-correct"></a>

### 1. 仅 TTL：最终更新

仅过期，不显式失效：

```python
@cache(key_prefix="popular_widgets", expiration=300)  # 5 minutes
async def get_popular(request: Request, ...):
    ...
```

适合可接受 1–5 分钟陈旧的只读或近只读数据，如国家、等级、Top-N 聚合和输入很少变化的计算。不适合用户刚编辑就期望看见的新值。

<a id="2-write-through-invalidation-strict-consistency"></a>

### 2. 写后失效

相同 `(key_prefix, resource_id_name)` 的写操作自动清对应键，关联项使用 `to_invalidate_extra`：

```python
@router.patch("/{widget_id}")
@cache(
    key_prefix="widget",
    resource_id_name="widget_id",
    to_invalidate_extra={
        "user_widgets": "owner_id",      # invalidate the owner's list
        "widget_count": "global",        # invalidate the global counter
    },
)
async def update_widget(
    request: Request,
    widget_id: int,
    owner_id: int,
    ...,
) -> dict[str, Any]:
    return await widget_service.update(widget_id, values, db)
```

处理器成功返回**之后**清理，失败不动缓存。适合直接缓存的资源和少量固定关联缓存，不适合广泛的逐用户列表清理。

> 译注：原文将本节称为 “Write-Through Invalidation（strict consistency）”。这里按其描述译为写后失效，该名称本身不构成严格一致性的保证。

<a id="3-pattern-based-invalidation-blast-radius"></a>

### 3. 模式失效

一次清多页或搜索结果：

```python
@router.delete("/{widget_id}")
@cache(
    key_prefix="widget",
    resource_id_name="widget_id",
    pattern_to_invalidate_extra=[
        "user_{owner_id}_widgets:*",     # all paginated lists for this user
        "widget_search:*",                # all search-result caches
    ],
)
async def delete_widget(request: Request, widget_id: int, owner_id: int, ...) -> None:
    await widget_service.delete(widget_id, db)
```

!!! warning "Memcached 不支持模式"
    会抛 `PatternMatchingNotSupportedError`，普通删除仍执行，需要模式时使用 Redis。

适合数量未知的分页、过滤、搜索缓存及稳定命名空间，不适合窄范围清理，因为每次扫描比单个 `DEL` 更贵。

<a id="combining-the-three"></a>

## 组合三种方式

实际服务常混合使用：

```python
@router.put("/{widget_id}")
@cache(
    key_prefix="widget",
    resource_id_name="widget_id",
    expiration=900,                                       # TTL fallback (15 min)
    to_invalidate_extra={                                 # narrow related-key wipes
        "widget_count": "global",
    },
    pattern_to_invalidate_extra=[                         # broad list wipes
        "user_{owner_id}_widgets:*",
        "widget_search:*",
    ],
)
async def update_widget(request: Request, widget_id: int, owner_id: int, ...) -> dict[str, Any]:
    return await widget_service.update(widget_id, values, db)
```

TTL 是兜底，即使漏失效，示例缓存也在 15 分钟内过期恢复。

<a id="cache-aside-service-layer-caching"></a>

## 旁路缓存：服务层

路由之外的服务和任务直接使用 provider：

```python
from src.infrastructure.cache.provider import get, set, delete

KEY_TTL = 1800  # 30 minutes


async def get_user_score(user_id: int, db: AsyncSession) -> int:
    cache_key = f"user_score:{user_id}"

    # Try the cache first
    cached = await get(key=cache_key)
    if cached is not None:
        return int(cached)

    # Miss — compute and store
    score = await _compute_user_score(user_id, db)
    await set(key=cache_key, value=score, expiration=KEY_TTL)
    return score


async def invalidate_user_score(user_id: int) -> None:
    await delete(key=f"user_score:{user_id}")
```

约定：读取与失效使用完全相同键；先成功计算再 set，避免缓存错误；读取和刷新使用一致 TTL。

<a id="cache-stampede-mitigation"></a>

## 缓存击穿与集中失效

热点过期时，并发请求可能在首个回填前一起打到数据库。可用：

<a id="slightly-randomized-ttls"></a>

### TTL 加随机量

让同一时间建立的大量键分散过期：

```python
import random

ttl = 1800 + random.randint(-60, 60)  # 30 min ± 1 min
await set(key=cache_key, value=payload, expiration=ttl)
```

原文认为足以覆盖多数负载。

<a id="refresh-ahead-of-expiration"></a>

### 提前刷新

在服务中按软 TTL 判断是否重新计算：

```python
SOFT_TTL = 1500  # 25 min — recompute eagerly past this
HARD_TTL = 1800  # 30 min — fail-open beyond this

async def get_payload(user_id: int) -> dict:
    cache_key = f"user_payload:{user_id}"
    payload = await get(key=cache_key)

    if payload is not None and payload.get("computed_at", 0) > time.time() - SOFT_TTL:
        return payload  # fresh enough

    fresh = await _compute(user_id)
    fresh["computed_at"] = time.time()
    await set(key=cache_key, value=fresh, expiration=HARD_TTL)
    return fresh
```

软 TTL 到期后，下个请求在硬过期前触发刷新。原文称这对中等热点有效。极热键如每秒万次榜单可在重计算路径使用 Redis `SET key value NX EX 30` 分布式锁，模板未内置。

<a id="cache-warming"></a>

## 预热

提前填充缓存，避免部署后首请求冷启动。

<a id="at-application-startup-in-the-lifespan"></a>

### 应用启动时

缓存初始化在 `infrastructure/app_factory.py` 的 `lifespan_factory`，可在其后预热，但只用于真正**小型**数据，如参考表、等级、Top-N，不要每次启动加载百万行。

自定义 `interfaces/main.py` 示例：

```python
from contextlib import asynccontextmanager
from fastapi import FastAPI

from src.infrastructure.app_factory import lifespan_factory
from src.infrastructure.cache.provider import set
from src.infrastructure.config.settings import get_settings
from src.wiring.app import LIFECYCLES


@asynccontextmanager
async def lifespan(app: FastAPI):
    settings = get_settings()
    # Run the boilerplate's default lifespan first
    base_lifespan = lifespan_factory(settings, lifecycles=LIFECYCLES)
    async with base_lifespan(app):
        await _warm_reference_data()
        yield


async def _warm_reference_data():
    # Small, slow-changing data — safe to warm at boot.
    tiers = await _load_all_tiers()
    for tier in tiers:
        await set(key=f"tier:{tier['id']}", value=tier, expiration=86400)
```

通过 `create_application(lifespan=lifespan)` 接入。

<a id="as-a-taskiq-task"></a>

### Taskiq 定时任务

较大或周期性预热用 Taskiq，见[后台任务](../background-tasks/index.md)，逻辑仍是读数据并 set：

```python
# backend/src/modules/cache/tasks.py
from ...infrastructure.cache.provider import set
from ...infrastructure.taskiq.brokers import default_broker


@default_broker.task(task_name="warm_top_widgets")
async def warm_top_widgets() -> None:
    top = await _query_top_widgets(limit=100)
    await set(key="top_widgets", value=top, expiration=600)
```

每五分钟或小于 TTL 的间隔运行，维持缓存热度。

<a id="negative-caching"></a>

## 空结果缓存

数据库也未命中时短暂缓存缺失，避免重复查询：

```python
from src.infrastructure.cache.provider import get, set

NEGATIVE_TTL = 60  # 1 minute — keep negative caches very short
SENTINEL = "__NOT_FOUND__"


async def get_widget(widget_id: int, db: AsyncSession) -> dict | None:
    cache_key = f"widget:{widget_id}"
    cached = await get(key=cache_key)

    if cached == SENTINEL:
        return None
    if cached is not None:
        return cached

    result = await widget_service.get_by_id(widget_id, db)
    if result is None:
        await set(key=cache_key, value=SENTINEL, expiration=NEGATIVE_TTL)
        return None

    await set(key=cache_key, value=result, expiration=600)
    return result
```

缺失 TTL 应远短于正常值，否则创建后还会持续返回旧 404。

<a id="per-caller-vs-shared-caches"></a>

## 逐调用者与共享缓存

默认每调用者独立，防止个性化结果串用。`caller_from` 指向包含 `Principal` 的参数：

```python
@router.get("/me/dashboard")
@cache(key_prefix="dashboard", caller_from="principal")
async def my_dashboard(request: Request, principal: CurrentPrincipalDep, ...):
    ...
# → dashboard:u=5, and dashboard:u=9 for somebody else
```

带 scope 的凭据与账户会话不同，如 `user.read` 密钥使用 `dashboard:u=5/9f0c…`，同账户不同 scope 也分开。`current_user` 字典只描述账户，无法区分凭据，指定它会跳过缓存，而非把会话响应给密钥。

**写入只清写入凭据自己的条目**。会话 PATCH 清 `dashboard:u=5`，却保留密钥条目到 TTL，其他账户也不会被清。需要同步失效时，在写路由设置 `pattern_to_invalidate_extra=['dashboard:*']`，后端需支持模式，Redis 支持、Memcached 不支持。

所有人响应相同时可共享：

```python
@cache(key_prefix="rate_limits", per_caller=False)
```

`per_caller=False` 仅在不含个人数据，且不按权限、等级或归属过滤时安全。不确定就保留默认，或不缓存，因为每用户热响应的缓存往返可能反而更贵。

默认模式下指定参数却解析为空时不缓存，警告后执行。完全未命名调用者参数则在声明时抛 `ValueError`。

<a id="picking-ttls"></a>

## 选择 TTL

默认 3600 秒，按容忍陈旧程度覆盖：

| 数据 | 建议 TTL |
|------|----------|
| 等级、国家等静态数据 | 24 小时，86400 秒 |
| 用户资料 / 公开对象 | 5–30 分钟，300–1800 秒 |
| 分页列表 | 1–5 分钟，60–300 秒 |
| 搜索结果 | 5–15 分钟，300–900 秒 |
| 高频仪表盘 | 30–60 秒 |
| 404 空结果 | 30–120 秒 |

不确定时从短 TTL 开始，验证失效路径后再增加，避免排查一天长的陈旧缓存。

<a id="operational-notes"></a>

## 运维

<a id="read-your-keys-in-production"></a>

### 在生产检查键

```bash
redis-cli -h $CACHE_REDIS_HOST KEYS 'widget:*'
redis-cli -h $CACHE_REDIS_HOST TTL widget:42
redis-cli -h $CACHE_REDIS_HOST GET widget:42
```

只看键无法判断数据及失效方式，说明前缀过于简略。

<a id="watch-for-fail-open-behavior"></a>

### 监控失败放行

装饰器捕获 Redis 错误后回源，提升可用性，但缓存故障可能在仪表盘表现为数据库变慢。关注日志：

```
Cache backend not available: <error>
```

按这些日志的发生率告警，而非只检查 Redis 可达性。

<a id="dont-cache-personal-data-without-thinking"></a>

### 缓存个人数据需完整键维度

认证、请求头、查询参数影响结果时必须反映在键中。本节原文称装饰器只看前缀占位符和资源 ID，与上文自动包含查询及 principal 的新描述不一致。

<a id="anti-patterns-to-avoid"></a>

## 避免的模式

- **缓存写响应**：装饰器只缓存 GET，一般应缓存更新后将读取的 GET。
- **缓存保存不可重建状态**：丢缓存会丢业务数据时，应使用数据库行。
- **分页 TTL 混用**：同前缀各页不同过期会产生不一致。
- **每次写都扫描模式**：规模大时昂贵，只在确需清多键时使用。

<a id="key-files"></a>

## 关键文件

| 组件 | 位置 |
|------|------|
| 装饰器 | `backend/src/infrastructure/cache/decorator.py` |
| Provider | `backend/src/infrastructure/cache/provider.py` |
| 后端 | `backend/src/infrastructure/cache/backends/` |
| 生命周期 | `backend/src/infrastructure/app_factory.py` |

<a id="next-steps"></a>

## 后续步骤

- **[Redis 缓存](redis-cache.md)**：参数和 provider 参考。
- **[客户端缓存](client-cache.md)**：浏览器缓存头。
- **[后台任务](../background-tasks/index.md)**：Taskiq 定时预热。
