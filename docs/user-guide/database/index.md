<a id="database-layer"></a>

# 数据库层

本章介绍 FastAPI Boilerplate 的数据库层，以及高效存储和读取数据的方法。

<a id="what-youll-learn"></a>

## 内容概览

- **[模型](models.md)**：通过 SQLAlchemy 2.0 定义数据库表。
- **[Schema](schemas.md)**：通过 Pydantic 校验和序列化数据。
- **[CRUD 操作](crud.md)**：通过 FastCRUD 访问数据库。
- **[迁移](migrations.md)**：通过 Alembic 管理结构变更。
- **[Neon](neon.md)**：使用无服务器 Postgres 替代本地容器。

<a id="quick-overview"></a>

## 快速概览

数据层按功能拆分，各模块拥有完整的数据处理栈：

```python
# modules/user/routes.py — request comes in, validated by UserCreate
@router.post("/", response_model=UserRead)
async def create_user(
    user: UserCreate,
    db: Annotated[AsyncSession, Depends(async_session)],
    user_service: Annotated[UserService, Depends(get_user_service)],
):
    return await user_service.create(user, db)

# modules/user/service.py — business logic, calls into FastCRUD
# modules/user/crud.py    — crud_users = FastCRUD(User)
# modules/user/models.py  — User SQLAlchemy model
```

<a id="architecture"></a>

## 架构

```text
HTTP Request
    ↓
Pydantic Schema      (modules/<feature>/schemas.py)
    ↓
APIRouter            (modules/<feature>/routes.py)
    ↓
Service              (modules/<feature>/service.py)
    ↓
FastCRUD             (modules/<feature>/crud.py)
    ↓
SQLAlchemy Model     (modules/<feature>/models.py)
    ↓
PostgreSQL
```

服务层包含权限检查、多步骤编排等业务规则；FastCRUD 处理通用查询；模型定义表结构。

<a id="key-components"></a>

## 关键组件

<a id="sqlalchemy-20-models"></a>

### SQLAlchemy 2.0 模型

模型继承 `Base`（由 `DeclarativeBase` 和 `MappedAsDataclass` 组合），并按需添加 mixin：

```python
# modules/user/models.py
from sqlalchemy import String
from sqlalchemy.orm import Mapped, mapped_column

from ...infrastructure.database.session import Base
from ...infrastructure.database.models import SoftDeleteMixin, TimestampMixin


class User(Base, TimestampMixin, SoftDeleteMixin):
    __tablename__ = "user"

    id: Mapped[int] = mapped_column(primary_key=True, init=False)
    username: Mapped[str] = mapped_column(String(32), unique=True, index=True)
    email: Mapped[str] = mapped_column(String(50), unique=True, index=True)
    hashed_password: Mapped[str] = mapped_column(String(100))
```

`infrastructure/database/models.py` 提供：

- `TimestampMixin`：添加 `created_at` 和 `updated_at`。
- `SoftDeleteMixin`：添加 `is_deleted` 和 `deleted_at`。
- `UUIDMixin`：提供 UUID 主键，替代整数 ID。

<a id="pydantic-schemas"></a>

### Pydantic Schema

Schema 与模型放在一起，区分请求和响应结构：

```python
# modules/user/schemas.py
from pydantic import BaseModel, EmailStr, Field


class UserCreate(BaseModel):
    name: str = Field(min_length=2, max_length=30)
    username: str = Field(min_length=2, max_length=32)
    email: EmailStr
    password: str = Field(min_length=8)


class UserRead(BaseModel):
    id: int
    name: str
    username: str
    email: EmailStr
    # No hashed_password — schemas exclude sensitive fields
```

<a id="fastcrud-operations"></a>

### FastCRUD 操作

每个模块提供轻量的 FastCRUD 封装：

```python
# modules/user/crud.py
from fastcrud import FastCRUD
from .models import User

crud_users: FastCRUD = FastCRUD(User)
```

随后在服务中使用：

```python
# modules/user/service.py
from .crud import crud_users

# Create
user = await crud_users.create(db=db, object=user_create)

# Read one
user = await crud_users.get(db=db, id=user_id)

# Read many
result = await crud_users.get_multi(db=db, offset=0, limit=10)

# Update
await crud_users.update(db=db, object=user_update, id=user_id)

# Soft delete (sets is_deleted=True via the mixin)
await crud_users.delete(db=db, id=user_id)

# Hard delete
await crud_users.db_delete(db=db, id=user_id)
```

<a id="database-migrations"></a>

### 数据库迁移

在 `backend/` 中执行：

```bash
# Generate a migration from model changes
uv run --no-sync alembic revision --autogenerate -m "Add user table"

# Apply migrations
uv run --no-sync alembic upgrade head

# Roll back the most recent migration
uv run --no-sync alembic downgrade -1
```

<a id="database-setup"></a>

## 数据库设置

模板通过 `asyncpg` 异步访问 PostgreSQL。

<a id="environment-configuration"></a>

### 环境配置

```env
# backend/.env
POSTGRES_USER=postgres
POSTGRES_PASSWORD=postgres
POSTGRES_SERVER=localhost     # or "postgres" for Docker Compose
POSTGRES_PORT=5432
POSTGRES_DB=postgres
POSTGRES_ASYNC_PREFIX=postgresql+asyncpg://
POSTGRES_POOL_SIZE=20
POSTGRES_MAX_OVERFLOW=0
CREATE_TABLES_ON_STARTUP=true   # the default in local and development
```

`DatabaseSettings` 的 `DATABASE_URL` 属性由这些值计算得到。直接设置环境变量 `DATABASE_URL` 会覆盖其他配置。

<a id="connection-management"></a>

### 连接管理

会话依赖位于 `infrastructure/database/session.py`：

```python
from collections.abc import AsyncGenerator
from sqlalchemy.ext.asyncio import AsyncSession


async def async_session() -> AsyncGenerator[AsyncSession, None]:
    async with local_session() as db:
        yield db
```

引擎由 `get_engine()` 首次使用时创建，而非导入时创建。`local_session()` 在该引擎上打开会话；`infrastructure/database/initialize.py` 中的 `close_database()` 释放引擎。应用生命周期在关闭时调用它，独立脚本使用会话后也应在退出前调用。

路由通过 FastAPI 的 `Depends` 使用：

```python
from typing import Annotated
from fastapi import Depends
from sqlalchemy.ext.asyncio import AsyncSession

from ...infrastructure.database.session import async_session


@router.get("/")
async def list_users(
    db: Annotated[AsyncSession, Depends(async_session)],
):
    ...
```

<a id="included-models"></a>

## 随附模型

模板包含以下模型，分别属于对应功能模块：

<a id="user-modulesusermodelspy"></a>
<a id="user--modulesusermodelspy"></a>

### `User`：`modules/user/models.py`

- 用户名、邮箱、密码哈希、姓名和头像。
- OAuth 字段：`oauth_provider`、`google_id`、`github_id`。
- 指向 `tier` 的外键。
- 使用 `TimestampMixin`、`SoftDeleteMixin`。
- 表名为单数 **`user`**。

<a id="tier-modulestiermodelspy"></a>
<a id="tier--modulestiermodelspy"></a>

### `Tier`：`modules/tier/models.py`

- 仅包含 `name` 和 `description`，不包含定价或业务逻辑。
- 与用户是一对多关系。
- 使用 `TimestampMixin`、`SoftDeleteMixin`。
- 表名：**`tiers`**。

<a id="ratelimit-modulesrate_limitmodelspy"></a>
<a id="ratelimit--modulesrate_limitmodelspy"></a>

### `RateLimit`：`modules/rate_limit/models.py`

- 按 API 路径配置各等级的限流。
- 字段：`tier_id`、`name`、`path`、`limit`、`period`。
- 每个等级和路径只允许一条未删除规则，由 `uq_rate_limits_tier_path` 保证。
- 使用 `TimestampMixin`、`SoftDeleteMixin`。
- 表名：**`rate_limits`**。

<a id="apikey-keyusage-modulesapi_keysmodelspy"></a>
<a id="apikey-keyusage--modulesapi_keysmodelspy"></a>

### `APIKey`、`KeyUsage`：`modules/api_keys/models.py`

- 签发 API 密钥，每个密钥通过注册权限名列表限定范围，并跟踪使用情况。
- 表名：`api_keys`、`key_usage`。

<a id="directory-structure"></a>

## 目录结构

每个功能拥有自己的数据处理栈：

```text
backend/src/
├── infrastructure/
│   └── database/
│       ├── session.py        # get_engine, local_session, async_session dep, Base class, create_tables
│       └── models.py         # TimestampMixin, SoftDeleteMixin, UUIDMixin
└── modules/
    ├── user/
    │   ├── models.py         # SQLAlchemy User
    │   ├── schemas.py        # Pydantic UserCreate/UserRead/UserUpdate
    │   ├── crud.py           # crud_users = FastCRUD(User)
    │   ├── service.py        # UserService (business rules)
    │   └── routes.py         # /api/v1/users endpoints
    ├── tier/
    ├── rate_limit/
    └── api_keys/
```

共享 `Base` 和 mixin 位于 `infrastructure/database/`，功能专属内容都集中在模块目录。

<a id="common-patterns"></a>

## 常见模式

<a id="create-with-validation"></a>

### 校验后创建

```python
@router.post("/", response_model=UserRead, status_code=201)
async def create_user(
    user: UserCreate,
    db: Annotated[AsyncSession, Depends(async_session)],
    user_service: Annotated[UserService, Depends(get_user_service)],
):
    # service.create checks for duplicate username/email and hashes the password
    return await user_service.create(user, db)
```

<a id="query-with-filters"></a>

### 带条件查询

```python
# Active users only (excludes soft-deleted)
result = await crud_users.get_multi(
    db=db,
    is_deleted=False,
    offset=0,
    limit=10,
)

# Substring search
result = await crud_users.get_multi(
    db=db,
    username__icontains="john",
    schema_to_select=UserRead,
)
```

FastCRUD 支持字段名后的 `__` 操作符，如 `__contains`、`__icontains`、`__gt`、`__lt`、`__in`。

<a id="soft-delete-pattern"></a>

### 软删除

`SoftDeleteMixin` 添加 `is_deleted` 和 `deleted_at`。FastCRUD 的 `.delete()` 修改标记，而非移除记录：

```python
# Soft delete (default for models with the mixin)
await crud_users.delete(db=db, id=user_id)

# Hard delete (actually DELETE FROM)
await crud_users.db_delete(db=db, id=user_id)

# Filter to exclude soft-deleted records
await crud_users.get_multi(db=db, is_deleted=False)
```

<a id="whats-next"></a>

## 后续步骤

1. **[模型](models.md)**：定义表和关联关系。
2. **[Schema](schemas.md)**：添加 Pydantic 校验和序列化。
3. **[CRUD 操作](crud.md)**：通过 FastCRUD 读写数据。
4. **[迁移](migrations.md)**：跟踪和部署结构变更。
