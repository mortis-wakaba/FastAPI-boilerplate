<a id="api-development"></a>

# API 开发

本章介绍使用 FastAPI Boilerplate 构建可靠、适用于生产的 REST API。

<a id="what-youll-learn"></a>

## 内容概览

- **[接口](endpoints.md)**：创建带认证和校验的接口。
- **[分页](pagination.md)**：为列表接口添加分页。
- **[异常处理](exceptions.md)**：使用模板异常类型处理错误。
- **[API 版本管理](versioning.md)**：维护版本及向后兼容性。

<a id="quick-overview"></a>

## 快速概览

路由定义在各模块的 `routes.py`，由 `interfaces/api/v1/__init__.py` 汇总并挂载在 `/api/v1` 下。

```python
# backend/src/modules/user/routes.py
from typing import Annotated, Any
from fastapi import APIRouter, Depends
from sqlalchemy.ext.asyncio import AsyncSession

from ...infrastructure.database.session import async_session
from .schemas import UserCreate, UserRead
from .service import UserService

router = APIRouter(tags=["Users"])


def get_user_service() -> UserService:
    return UserService()


@router.post("/", response_model=UserRead, status_code=201)
async def create_user(
    user: UserCreate,
    db: Annotated[AsyncSession, Depends(async_session)],
    user_service: Annotated[UserService, Depends(get_user_service)],
) -> dict[str, Any]:
    return await user_service.create(user, db)
```

汇总路由将其接入：

```python
# backend/src/interfaces/api/v1/__init__.py
from fastapi import APIRouter

from ....modules.user.routes import router as users_router

router = APIRouter(prefix="/v1")
router.include_router(users_router, prefix="/users")
```

最终 URL：`POST /api/v1/users/`。

<a id="key-features"></a>

## 主要功能

<a id="built-in-authentication"></a>

### 内置认证

使用 HttpOnly Cookie 的会话认证，通过 `infrastructure/auth/dependencies` 获取当前用户：

```python
from ...infrastructure.auth.dependencies import get_current_user

@router.get("/me", response_model=UserRead)
async def get_profile(
    current_user: Annotated[dict[str, Any], Depends(get_current_user)],
) -> dict[str, Any]:
    return current_user
```

仅超级用户接口改用 `get_current_superuser`。由角色权限控制的接口，在路由 `dependencies` 中添加 `require_permissions(...)`：

```python
from ...infrastructure.auth.authorization import require_permissions

@router.get("/", dependencies=[require_permissions("user.read")])
async def list_users(...): ...
```

`infrastructure/auth/deps.py` 还提供 `CurrentPrincipalDep`（crudauth 的 `Principal`）和 `CurrentPermissionsDep`（调用者有效权限）。完整说明见[身份认证](../authentication/index.md)。

<a id="easy-pagination"></a>

### 简便分页

使用 FastCRUD 的 `PaginatedListResponse` 和 `paginated_response()`：

```python
from fastcrud import PaginatedListResponse, compute_offset, paginated_response

@router.get("/", response_model=PaginatedListResponse[UserRead])
async def list_users(
    db: Annotated[AsyncSession, Depends(async_session)],
    user_service: Annotated[UserService, Depends(get_user_service)],
    page: int = 1,
    items_per_page: int = 10,
) -> dict[str, Any]:
    result = await user_service.get_paginated(
        skip=compute_offset(page, items_per_page),
        limit=items_per_page,
        db=db,
    )
    return paginated_response(crud_data=result, page=page, items_per_page=items_per_page)
```

<a id="automatic-validation"></a>

### 自动校验

请求体、查询参数和响应模型均由 Pydantic 校验：

```python
@router.post("/", response_model=UserRead)
async def create_user(user: UserCreate):  # ← validates input
    return await user_service.create(user, db)  # ← validates output via response_model
```

<a id="error-handling"></a>

### 错误处理

领域错误位于 `modules/common/exceptions.py`，各功能定义其子类。应用统一处理，路由可直接抛出或让服务抛出异常，客户端得到映射后的状态及 `support_id`：

```python
from ..common.exceptions import ResourceNotFoundError


@router.get("/{username}", response_model=UserRead)
async def get_user_by_username(
    username: str,
    db: Annotated[AsyncSession, Depends(async_session)],
    user_service: Annotated[UserService, Depends(get_user_service)],
) -> dict[str, Any]:
    user = await user_service.get_by_username(username, db)
    if user is None:
        raise ResourceNotFoundError(f"user {username}")

    return user
```

完整类型见[异常处理](exceptions.md)。

<a id="architecture"></a>

## 架构

```text
HTTP Request
    ↓
APIRouter         (modules/<feature>/routes.py)
    ↓
Service           (modules/<feature>/service.py)  — business rules, permission checks
    ↓
FastCRUD          (modules/<feature>/crud.py)
    ↓
SQLAlchemy Model  (modules/<feature>/models.py)
    ↓
PostgreSQL
```

职责分工：

- `routes.py` 处理状态码、Schema、依赖等 HTTP 事项。
- `service.py` 处理校验和编排等业务逻辑。
- `crud.py` 处理数据库 I/O。

测试时可以模拟任意层，也可独立修改某层。

<a id="directory-structure"></a>

## 目录结构

```text
backend/src/
├── interfaces/
│   └── api/
│       ├── __init__.py            # mounts /api
│       └── v1/
│           └── __init__.py        # mounts /v1 + every module's router
├── infrastructure/
│   └── auth/
│       └── routes.py              # /api/v1/auth/* (login, OAuth, check-auth)
└── modules/
    ├── user/routes.py             # /api/v1/users/*
    ├── tier/routes.py             # /api/v1/tiers/*
    ├── rate_limit/routes.py       # /api/v1/rate-limits/*
    └── api_keys/routes.py         # /api/v1/api-keys/*
```

认证位于 `infrastructure/auth/routes.py` 而非业务模块，因为它是其他功能共同依赖的基础能力。

<a id="mounted-endpoints"></a>

## 已挂载接口

模板提供：

| 前缀 | 来源 | 说明 |
|------|------|------|
| `POST/GET/PATCH/DELETE /api/v1/users/*` | `modules/user/routes.py` | 开放创建；读取、更新需会话；按用户名查询不返回邮箱；列出全部用户需 `user.read` 权限 |
| `GET /api/v1/tiers/*` | `modules/tier/routes.py` | 认证后列出、按名称查询 |
| `GET/PATCH/DELETE /api/v1/rate-limits/*` | `modules/rate_limit/routes.py` | 仅超级用户 |
| `POST /api/v1/auth/login`、`logout`、`check-auth` | `infrastructure/auth/routes.py` | 会话认证 |
| `GET /api/v1/auth/sessions`、`DELETE /api/v1/auth/sessions/{handle}`、`POST /api/v1/auth/logout-all`、`POST /api/v1/auth/csrf/refresh` | crudauth，由 `infrastructure/auth/routes.py` 挂载 | 设备管理 |
| `POST /api/v1/auth/change-password`、`GET /api/v1/auth/me` | crudauth 账户路由，由 `infrastructure/auth/routes.py` 挂载 | 会话认证，不挂载 `set-password` |
| `GET /api/v1/auth/oauth/{provider}`、`oauth/callback/{provider}` | crudauth，由 `infrastructure/auth/routes.py` 挂载 | Google OAuth，在 `infrastructure/auth/setup.py` 配置 |
| `POST/GET/PATCH/DELETE /api/v1/api-keys/*` | `modules/api_keys/routes.py` | 认证后的密钥管理 |
| `GET /admin/*` | `interfaces/admin/initialize.py` | SQLAdmin 界面 |
| `GET /docs`、`/redoc`、`/openapi.json` | 应用工厂，按环境保护 | 生产默认关闭；生产启用或 staging 环境需超级用户认证 |
| `GET /health` | `interfaces/main.py` | 存活检查，确认进程提供服务 |
| `GET /health/ready` | `interfaces/main.py` | 就绪检查，关键依赖不可达时返回 `503` |

<a id="whats-next"></a>

## 后续步骤

从基础开始：

1. **[接口](endpoints.md)**：新路由常见模式。
2. **[分页](pagination.md)**：分页列表响应。
3. **[异常处理](exceptions.md)**：模板异常模型。
4. **[API 版本管理](versioning.md)**：版本策略。

深入阅读：

5. **[数据 Schema](../database/schemas.md)**：路由使用的 Pydantic 结构。
6. **[CRUD 操作](../database/crud.md)**：服务下方的数据层。
