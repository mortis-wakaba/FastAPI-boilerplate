<a id="rate-limiting"></a>

# 限流

模板使用 crudauth 限流器，以 Redis 支持按等级、按路径的限制。API 默认受保护，认证请求按用户 ID，匿名请求按可信代理边界解析的客户端 IP 计数。

`ratelimit` 提供节流，`tier_limits` 添加逐等级规则和配置接口。没有这两项则不执行此处 API 节流；仅 ratelimit 时对所有调用者应用默认限制，见[可组合功能](../composable-features.md)。原文称两项都没有时不返回 429，该说法应结合认证自身限流理解。

!!! tip "构建完整 SaaS？"
    限流是免费基础功能，[FastroAI](https://fastro.ai) 将它与支付、权益、邮件、前端和 AI Agent 整合。[更快交付 SaaS →](https://fastro.ai)

<a id="whats-built-in"></a>

## 内置内容

```text
backend/src/infrastructure/auth/setup.py
└── resolve_api_rate_limit()  crudauth per-request resolver

backend/src/modules/rate_limit/
├── models.py          RateLimit (tier_id, path, limit, period)
├── routes.py          GET / GET-by-name / PATCH / DELETE on /api/v1/rate-limits/
├── crud.py / service.py
└── schemas.py
```

crudauth 后端与认证单例在应用生命周期中初始化。

<a id="how-a-request-flows-through-it"></a>

## 请求流程

1. 每次 API 请求运行路由器级 crudauth 依赖。
2. `resolve_api_rate_limit` 查用户等级及匹配路由模板的规则。
3. `api_rate_limit_key` 以调用者和路由模板组合预算，认证用用户 ID，匿名用 IP，各路由独立。
4. 限流器原子增加当前固定窗口计数，返回 `(count, is_limited)`。窗口长 `period` 秒，按时钟对齐，键自行过期。
5. 超限抛带 `Retry-After` 的 429，否则附加 `X-RateLimit-Limit`、`X-RateLimit-Remaining`。之后再被 401、404、422 拒绝仍计数，响应也携带头。

无法解析的凭据，如垃圾 `X-API-Key` 或过期 bearer，消耗匿名地址预算。有效凭据有独立调用者预算，不被同地址垃圾请求消耗。

Redis 键末尾为窗口开始时间：

```text
crudauth:rl:ratelimit:api:user:{user_id}:{route_template}:{window_start}
crudauth:rl:ratelimit:api:ip:{client_ip}:{route_template}:{window_start}
```

<a id="custom-enforcement"></a>

## 自定义执行

现有 API 已在路由器级设置依赖，其他路由可复用：

```python
from fastapi import APIRouter, Depends
from src.infrastructure.auth.setup import auth
from crudauth.ratelimit import KeyBy, RateLimit

router = APIRouter()


@router.post("/widgets/", dependencies=[Depends(auth.rate_limit("widgets", RateLimit(10, 60), key=KeyBy.USER_OR_IP))])
async def create_widget(...): ...
```

或保护整个路由器：

```python
router = APIRouter(dependencies=[Depends(auth.rate_limit("widgets", RateLimit(10, 60), key=KeyBy.USER_OR_IP))])
```

设置 `RATE_LIMITER_ENABLED=true` 即可启用。

<a id="configuration"></a>

## 配置

```env
# Master toggle for the API limits (the login lockout runs either way)
RATE_LIMITER_ENABLED=true

# Where counters live: redis (default), database or memory. Memory is per process, so it
# only holds for a single worker. Database keeps them in this project's own tables, shared
# by every worker. The memcached limiter was removed; that value fails at startup.
RATE_LIMITER_BACKEND=redis

# Defaults applied when the user has no tier or no matching rate-limit row
DEFAULT_RATE_LIMIT_LIMIT=100
DEFAULT_RATE_LIMIT_PERIOD=60          # seconds — 100/60s by default

# Redis backend
RATE_LIMITER_REDIS_HOST=redis         # use "localhost" without Docker
RATE_LIMITER_REDIS_PORT=6379
RATE_LIMITER_REDIS_DB=1               # rate-limiter DB (cache DB 0, sessions DB 2, taskiq DB 3)
RATE_LIMITER_REDIS_PASSWORD=
RATE_LIMITER_REDIS_CONNECT_TIMEOUT=5
RATE_LIMITER_REDIS_POOL_SIZE=10
```

关闭后 API 依赖不计数，适合测试和性能定位。但登录锁定仍使用 `RATE_LIMITER_BACKEND`，并失败拒绝：后端不可达时不允许无限制登录。

`RATE_LIMITER_BACKEND=database` 使用项目数据库的 `crudauth_counters` 共享计数，无需 Redis。尤其适用于登录锁定：memory 的 N 个 Worker 分别计数，使攻击者得到 N 倍机会。数据库后端写入时删除过期计数，清理量大致对应创建量，避免累积。选择该后端会增加 `crudauth_counters`、`crudauth_store`，需迁移或开发自动建表。

所有受限 API 请求都会写数据库，不仅登录。高流量优先 Redis，database 适合明确希望不部署 Redis 的环境。

<a id="user-tier-vs-ip-based-limits"></a>

## 用户等级与 IP 限制

有认证时以 principal 为键，否则用 `TRUSTED_PROXY_HOPS` 解析的 IP，再加入路由模板。解析器查询用户等级对应模板，未找到使用默认值。

<a id="path-matching"></a>

## 路径匹配

规则匹配路由声明的**模板**，包含 API 前缀。默认 `/api`，设置 `API_PREFIX=/service` 则存 `/service/v1/...`：

```text
/api/v1/users/               # the listing route, trailing slash included
/api/v1/users/{username}     # every user lookup, sharing one counter per caller
/api/v1/items/{id:int}       # a converter, as the route declares it
/api/v1/files/{path:path}    # a wildcard segment
```

尾斜杠也是模板一部分，`/api/v1/users` 不匹配 `/api/v1/users/`。应从应用读取：

```python
from fastapi.routing import APIRoute
from src.interfaces.main import app

print(sorted(route.path for route in app.routes if isinstance(route, APIRoute)))
```

具体路径如 `/api/v1/users/42` 不匹配，因为解析模板是 `/api/v1/users/{username}`。各调用者每路由独立，一个热点资源不会消耗另一条路由预算。

`path` 要求前导斜杠。每段可为字面文本、一个 `{name}`（可带 str / path / int / float / uuid 转换器），或围绕一个占位符的文本，如 `/items/{id}.json`、`/v{version}/items`。括号不平衡、未知转换器、同段两个占位符、尾空白或换行均拒绝。

<a id="managing-rate-limit-rules"></a>

## 管理规则

`RateLimit` 模型：

```python
class RateLimit(Base, TimestampMixin, SoftDeleteMixin):
    __tablename__ = "rate_limits"

    id: int
    tier_id: int        # FK to tiers.id
    name: str           # unique — used as the URL path on /rate-limits/{name}
    path: str           # route template the rule applies to
    limit: int          # max requests per period
    period: int         # seconds

    __table_args__ = (
        Index(
            "uq_rate_limits_tier_path",
            "tier_id",
            "path",
            unique=True,
            postgresql_where=text("is_deleted = false"),
        ),
    )
```

`limit`、`period` 是 Integer，范围 `1..2147483647`，创建和 PATCH 越界返回 `422`。直接 SQL / CRUD 写入同样受 Postgres 范围限制。

每等级每路径只有一条未软删除规则，唯一索引拒绝第二条，避免 10 和 1000 两条规则产生随机生效。PATCH 移到已有限制路径、或 `RateLimitService.create` 重复创建，均 409。软删除行不参与索引，可重新配置。

写入前先读取等级；期间被删除则外键拒绝，返回等级 404，而非错误报告规则冲突。拒绝约束从驱动错误读取，驱动未提供时才从错误消息中引用的名称提取，见 `modules/common/collisions.py`。

<a id="what-the-api-exposes"></a>

### API 提供的操作

| 方法 | 路径 | 认证 | 说明 |
|------|------|------|------|
| GET | `/api/v1/rate-limits/` | 超级用户 | 分页规则列表 |
| GET | `/api/v1/rate-limits/{name}` | 超级用户 | 按名称查询 |
| PATCH | `/api/v1/rate-limits/{name}` | 超级用户 | 更新 |
| DELETE | `/api/v1/rate-limits/{name}` | 超级用户 | 删除 |

**没有 POST 创建接口**。初始化可选以下三种方式，路径必须是含 API 前缀的完整模板，见[路径匹配](#path-matching)。示例默认 `/api`，修改前缀后应存 `/service/v1/widgets/`。

<a id="option-1-sql-migration"></a>
<a id="option-1-sql--migration"></a>

### 方式一：SQL / 迁移

通过 Alembic 插入：

```python
# alembic/versions/xxxx_seed_rate_limits.py
def upgrade():
    op.execute("""
        INSERT INTO rate_limits (tier_id, name, path, "limit", period, created_at)
        VALUES
            (1, 'free_widgets_create', '/api/v1/widgets/', 10, 60, NOW()),
            (2, 'pro_widgets_create',  '/api/v1/widgets/', 100, 60, NOW())
    """)
```

<a id="option-2-custom-seed-script"></a>

### 方式二：初始化脚本

在 `backend/scripts/` 添加：

```python
# backend/scripts/setup_rate_limits.py
import asyncio

from src.infrastructure.database.session import local_session
from src.modules.rate_limit.crud import crud_rate_limits


async def main():
    async with local_session() as db:
        await crud_rate_limits.create(db=db, object={
            "tier_id": 1, "name": "free_widgets_create",
            "path": "/api/v1/widgets/", "limit": 10, "period": 60,
        })
        await db.commit()


if __name__ == "__main__":
    asyncio.run(main())
```

从 `backend/` 执行 `uv run --no-sync python -m scripts.setup_rate_limits`。

缺前缀或具体路径会被存储但不匹配。应读取应用模板而非猜测。三种方式的重复规则均由 `uq_rate_limits_tier_path` 拒绝。

<a id="option-3-add-a-sqladmin-view"></a>

### 方式三：SQLAdmin 视图

参考 UserAdmin / TierAdmin 添加 `RateLimitAdmin`，见[添加模型](../admin-panel/adding-models.md)，即可在 UI 增删改。

<a id="response-headers"></a>

## 响应头

计数请求的所有响应，包括错误：

| 头 | 含义 |
|----|------|
| `X-RateLimit-Limit` | 调用者与路径的配置限制 |
| `X-RateLimit-Remaining` | 当前窗口剩余请求数 |

429 还携带窗口重置前秒数 `Retry-After`。应用工厂安装 crudauth `RateLimitHeadersMiddleware`，使认证等后续 401 / 404 / 422 也报告已消耗预算。

操作专属预算按路由报告。例如每账户每小时五次改密检查，`POST /api/v1/auth/change-password` 由 crudauth 自身保护，密码错误也返回 `X-RateLimit-Limit: 5` 及剩余。原文另以 `PATCH /api/v1/users/{username}` 描述邮箱变更，称它同时计路径预算，200 / 错误密码 403 报路径额度，仅 429 报 5、0 和 Retry-After。

> 译注：邮箱确认流程在用户管理章节使用 `/api/v1/auth/email/change-request`，本节相关描述可能来自旧版。

这些头沿用类似 GitHub / Stripe 的惯例，原文说明并非 RFC 6585 定义格式。前端可据此提示额度即将用尽。

<a id="production-considerations"></a>

## 生产考虑

<a id="pool-sizing"></a>

### 连接池

`RATE_LIMITER_REDIS_POOL_SIZE=10` 通常足够。高负载 `redis.exceptions.ConnectionError` 常是池耗尽，先检查连接泄漏或提高大小。

<a id="backend-errors"></a>

### 后端故障

窗口限流在 Redis 故障时失败放行，避免临时故障变成所有人 429。登录锁定例外，失败拒绝，已锁账户不能趁故障通过。

<a id="window-behavior"></a>

### 固定窗口

时钟对齐的固定窗口边界附近，短时间可能达到 `2 × limit`。通常可接受，需要严格滑动窗口时自行扩展。

<a id="anonymous-user-limits"></a>

### 匿名限制

IP 限流可通过代理和 IPv6 轮换绕过，NAT 也会影响公平性。它只增加滥用成本，不能替代认证、验证码或上游防火墙。

<a id="troubleshooting"></a>

## 故障排查

<a id="my-limit-is-not-matching"></a>

### 规则不匹配

检查开关、认证单例初始化、路径是否包含 `/api/v1` 的路由模板而非具体地址。

<a id="all-requests-look-anonymous-even-though-users-are-logged-in"></a>

### 登录后仍像匿名

检查 Cookie 是否到达应用、`SESSION_BACKEND`、`SESSION_REDIS_DB`，以及代理是否删除 Cookie。

<a id="the-limiter-cant-reach-redis"></a>

### 无法连接 Redis

窗口检查放行，登录拒绝。修复连接，或用 `RATE_LIMITER_ENABLED=false` 去除 API 限流；这不改变登录锁定的后端依赖。

<a id="key-files"></a>

## 关键文件

| 组件 | 位置 |
|------|------|
| 解析器和依赖 | `backend/src/infrastructure/auth/setup.py` |
| 模型 | `backend/src/modules/rate_limit/models.py` |
| 路由 | `backend/src/modules/rate_limit/routes.py` |
| 配置 | `backend/src/infrastructure/config/settings.py` 的 `RateLimiterSettings` |

<a id="next-steps"></a>

## 后续步骤

- **[等级](../authentication/permissions.md#tier-based-authorization)**：设置用户等级。
- **[添加后台模型](../admin-panel/adding-models.md)**：规则管理 UI。
- **[缓存策略](../caching/cache-strategies.md)**：Redis 状态模式。
