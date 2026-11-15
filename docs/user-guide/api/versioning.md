<a id="api-versioning"></a>

# API 版本管理

模板在 `/api/v1/` 下提供 `v1` 命名空间。本页介绍现有组装方式，以及发生破坏性变更时如何添加 `/api/v2/`。

<a id="how-its-wired-today"></a>

## 当前组装方式

```text
backend/src/interfaces/api/
├── __init__.py            # mounts /api → v1
└── v1/
    └── __init__.py        # mounts /v1 + each module's router
```

`interfaces/api/__init__.py`：

```python
from fastapi import APIRouter

from .v1 import router as v1_router

router = APIRouter(prefix="/api")
router.include_router(v1_router)
```

`interfaces/api/v1/__init__.py`：

```python
from fastapi import APIRouter

from ....infrastructure.auth.routes import router as auth_router
from ....modules.api_keys.routes import router as api_keys_router
from ....modules.rate_limit.routes import router as rate_limits_router
from ....modules.tier.routes import router as tiers_router
from ....modules.user.routes import router as users_router

router = APIRouter(prefix="/v1")
router.include_router(users_router, prefix="/users")
router.include_router(tiers_router, prefix="/tiers")
router.include_router(rate_limits_router, prefix="/rate-limits")
router.include_router(auth_router, prefix="/auth")
router.include_router(api_keys_router, prefix="/api-keys")
```

汇总器是唯一集中感知所有模块路由的位置。各模块从 `routes.py` 导出一个 `router`，v1 将它们挂载到各自前缀。

`interfaces/main.py` 随后挂载 API 树：

```python
from ..interfaces.api import router

app.include_router(router)
```

路径依次为 `users_router → /users → /v1/users → /api/v1/users → /api/v1/users/me` 等。

<a id="endpoints-today"></a>

## 当前接口

| URL 前缀 | 来源 |
|----------|------|
| `/api/v1/users/*` | `modules/user/routes.py` |
| `/api/v1/tiers/*` | `modules/tier/routes.py` |
| `/api/v1/rate-limits/*` | `modules/rate_limit/routes.py` |
| `/api/v1/auth/*` | `infrastructure/auth/routes.py` |
| `/api/v1/api-keys/*` | `modules/api_keys/routes.py` |

<a id="adding-v2"></a>

## 添加 v2

新响应结构、移除字段、认证要求变化等破坏性变更，应新增同级版本，而非修改 v1。

<a id="step-1-create-the-v2-aggregator"></a>

### 第一步：创建 v2 汇总器

```bash
mkdir backend/src/interfaces/api/v2
touch backend/src/interfaces/api/v2/__init__.py
```

```python
# backend/src/interfaces/api/v2/__init__.py
from fastapi import APIRouter

# Import the v2-flavored route modules — see Step 2 below
from ....modules.user.routes_v2 import router as users_router

router = APIRouter(prefix="/v2")
router.include_router(users_router, prefix="/users")

# Re-export anything that didn't change in v2 from v1:
# from ....modules.tier.routes import router as tiers_router
# router.include_router(tiers_router, prefix="/tiers")
```

<a id="step-2-create-v2-routes-per-module"></a>

### 第二步：为模块创建 v2 路由

按变更选择以下模式：

**模式 A：独立 `routes_v2.py`**。v2 路由差异较大，混入 `routes.py` 会难以理解时使用。

```python
# backend/src/modules/user/routes_v2.py
from typing import Annotated, Any

from fastapi import APIRouter, Depends
from fastcrud import PaginatedListResponse, compute_offset, paginated_response
from sqlalchemy.ext.asyncio import AsyncSession

from ...infrastructure.auth.dependencies import get_current_user
from ...infrastructure.database.session import async_session
from .schemas_v2 import UserReadV2
from .service import UserService
from .routes import get_user_service  # reuse the service factory

router = APIRouter(tags=["Users (v2)"])


# v2 makes pagination mandatory and renames profile_image_url -> avatar_url
@router.get("/", response_model=PaginatedListResponse[UserReadV2])
async def list_users(
    db: Annotated[AsyncSession, Depends(async_session)],
    user_service: Annotated[UserService, Depends(get_user_service)],
    page: int = 1,
    items_per_page: int = 10,
) -> dict[str, Any]:
    result = await user_service.get_paginated_v2(
        skip=compute_offset(page, items_per_page),
        limit=items_per_page,
        db=db,
    )
    return paginated_response(crud_data=result, page=page, items_per_page=items_per_page)
```

**模式 B：复用现有路由**。行为相同、仅 URL 前缀不同时使用：

```python
# backend/src/interfaces/api/v2/__init__.py
from ....modules.tier.routes import router as tiers_router

router.include_router(tiers_router, prefix="/tiers")
```

<a id="step-3-mount-v2-alongside-v1"></a>

### 第三步：同时挂载 v1 与 v2

```python
# backend/src/interfaces/api/__init__.py
from fastapi import APIRouter

from .v1 import router as v1_router
from .v2 import router as v2_router

router = APIRouter(prefix="/api")
router.include_router(v1_router)
router.include_router(v2_router)
```

现在 `/api/v1/users/` 和 `/api/v2/users/` 同时可用。

<a id="schema-versioning"></a>

## Schema 版本管理

保留 v1 Schema 原样，在新文件中添加 v2 Schema。不要改变 v1 的传输格式，这是引入 v2 的目的。

```python
# backend/src/modules/user/schemas.py — UNCHANGED
class UserRead(BaseModel):
    id: int
    name: str
    username: str
    email: EmailStr
    profile_image_url: str
    tier_id: int | None
    is_superuser: bool = False
    email_verified: bool = False
    oauth_provider: str | None = None


# backend/src/modules/user/schemas_v2.py — NEW
class UserReadV2(BaseModel):
    id: int
    name: str
    username: str
    email: EmailStr
    avatar_url: str                          # renamed from profile_image_url
    subscription_tier: str | None            # changed from tier_id (int) to tier name
    is_superuser: bool = False
    email_verified: bool = False
    created_at: datetime                     # newly exposed
```

产生 v2 数据结构的服务方法与 v1 放在一起，例如 v1 使用 `UserService.get_paginated`，v2 使用 `UserService.get_paginated_v2`，使数据组装仍归服务负责。

<a id="sharing-code-across-versions"></a>

## 跨版本共享代码

CRUD、服务和基础设施**共享**，仅路由与 Schema 按版本区分。多数代码无需变化，因此新增版本成本较低。

```text
modules/user/
├── models.py            ← shared
├── crud.py              ← shared
├── service.py           ← shared (add v2-shaped methods if needed)
├── schemas.py           ← v1 schemas
├── schemas_v2.py        ← v2 schemas
├── routes.py            ← v1 routes
└── routes_v2.py         ← v2 routes
```

<a id="deprecating-a-version"></a>

## 弃用旧版本

v2 就绪、v1 准备下线时：

<a id="1-add-a-deprecation-header-to-v1-endpoints"></a>

### 1. 为 v1 添加弃用响应头

```python
# Inside a v1 route handler
@router.get("/", response_model=list[UserRead], deprecated=True)
async def list_users(
    response: Response,
    ...,
) -> list[dict[str, Any]]:
    response.headers["Deprecation"] = "true"
    response.headers["Sunset"] = "Wed, 31 Dec 2025 00:00:00 GMT"
    response.headers["Link"] = '</api/v2/users/>; rel="successor-version"'
    return await ...
```

原文将 `Deprecation`、`Sunset`、`Link` 头归于 IETF [API 弃用](https://datatracker.ietf.org/doc/html/rfc8594)相关草案，支持相应 HTTP 语义的客户端工具可识别它们。

`deprecated=True` 还会在 `/docs` 中标记接口。

<a id="2-track-v1-usage"></a>

### 2. 跟踪 v1 使用量

通过日志中间件或可观测性工具，按 `request.url.path.startswith("/api/v1/")` 统计请求，判断流量是否已足够低。

<a id="3-remove-v1-after-sunset"></a>

### 3. 下线后移除 v1

截止日期已过且没有流量后：

1. 删除 `interfaces/api/v1/`。
2. 删除仅用于 v1 的 `schemas.py` 内容，或将 `schemas_v2.py` 重命名为 `schemas.py`。
3. 删除仅用于 v1 的服务方法。
4. 更新 `interfaces/api/__init__.py`，只挂载 v2。

<a id="per-version-openapi-documentation"></a>

## 按版本提供 OpenAPI 文档

默认 `/docs` 展示全部路由。需要独立文档时，将每个版本作为拥有自身 `FastAPI()` 实例的子应用挂载：

```python
# backend/src/interfaces/main.py — sketch
from fastapi import FastAPI

from .api.v1 import router as v1_router
from .api.v2 import router as v2_router

main = FastAPI(title="My API")

v1 = FastAPI(title="My API v1", version="1.0.0")
v1.include_router(v1_router)
main.mount("/api/v1", v1)

v2 = FastAPI(title="My API v2", version="2.0.0")
v2.include_router(v2_router)
main.mount("/api/v2", v2)
```

这样得到独立的 `/api/v1/docs` 和 `/api/v2/docs`。模板当前是单应用，确有需要时再采用此方式。

<a id="testing-multiple-versions"></a>

## 测试多个版本

v2 存在后，对两个版本都运行测试：

```python
import pytest
from httpx import AsyncClient


@pytest.mark.asyncio
async def test_v1_users_returns_list(client: AsyncClient):
    resp = await client.get("/api/v1/users/")
    # whatever v1's contract is — list, paginated, etc.
    assert resp.status_code in {200, 401, 403}


@pytest.mark.asyncio
async def test_v2_users_paginated(client: AsyncClient):
    resp = await client.get("/api/v2/users/")
    assert resp.status_code == 200
    body = resp.json()
    assert "data" in body
    assert "total_count" in body
    assert "page" in body
```

<a id="best-practices"></a>

## 最佳实践

<a id="what-counts-as-a-breaking-change"></a>

### 什么是破坏性变更

- 删除响应字段。
- 重命名字段。
- 改变类型，例如 `tier_id: int | None` 改为 `tier_name: str`。
- 收紧校验，使以前合法的输入失败。
- 新增必填请求字段。
- 改变默认行为，例如从不分页变为分页。
- 改变认证要求。

非破坏性变更可直接在 v1 添加字段或功能。

<a id="keep-the-url-pattern-consistent"></a>

### 保持 URL 模式一致

始终使用 `/api/v{number}/resource`。原文推荐 URL 版本而非请求头版本，方便人和缓存明确识别。

<a id="dont-fork-the-service-layer-prematurely"></a>

### 不要过早拆分服务层

v2 只改变响应结构时，通过小型适配器从同一服务结果派生字典。业务逻辑实际不同后再拆分服务。

<a id="document-changes-in-a-changelog"></a>

### 在更新日志中记录变更

发布 v2 时列出破坏性变更：

```markdown
## API v2

Breaking changes vs v1:
- `GET /users/` now returns `PaginatedListResponse` instead of `list[UserRead]`
- `UserRead.profile_image_url` renamed to `avatar_url`
- `UserRead.tier_id` (int) replaced with `subscription_tier` (string)
- `POST /users/` now requires authentication
- `UserCreate` now requires `accept_terms: bool`
```

简短、直接的清单有助于调用方迁移。

<a id="whats-next"></a>

## 后续步骤

- **[数据库迁移](../database/migrations.md)**：可能触发新 API 版本的结构变化。
- **[接口](endpoints.md)**：路由模式。
- **[Schema](../database/schemas.md)**：按版本组织数据结构。
