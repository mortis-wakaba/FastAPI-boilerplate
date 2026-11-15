<a id="crud-operations"></a>

# CRUD 操作

本指南介绍模板通过 [FastCRUD](https://benavlabs.github.io/fastcrud/) 提供的创建、读取、更新和删除操作。

<a id="overview"></a>

## 概览

模板使用 **FastCRUD** 访问数据库，提供：

- 各模型一致的异步 API。
- 自动分页辅助函数。
- 模型具有 `SoftDeleteMixin` 时的内置软删除支持。
- 通过 `schema_to_select` 选择加载字段。
- 获取关联数据的连接查询。

<a id="where-crud-lives"></a>

## CRUD 位置

每个模块拥有自己的 FastCRUD 实例，保持轻量且行为明确：

```python
# backend/src/modules/user/crud.py
from fastcrud import FastCRUD

from .models import User

crud_users: FastCRUD = FastCRUD(User)
```

```python
# backend/src/modules/tier/crud.py
from fastcrud import FastCRUD

from .models import Tier

crud_tiers: FastCRUD = FastCRUD(Tier)
```

模块的 `service.py` 导入 CRUD 实例，在其上添加输入校验、权限检查、密码哈希和多步骤编排等业务逻辑。

```python
# Typical service method (modules/user/service.py)
from .crud import crud_users
from .schemas import UserCreate, UserCreateInternal, UserRead


async def create(self, user: UserCreate, db: AsyncSession) -> dict[str, Any]:
    if await crud_users.exists(db=db, email=user.email):
        raise UserExistsError("Email already registered")
    if await crud_users.exists(db=db, username=user.username):
        raise UserExistsError("Username already taken")

    payload = user.model_dump()
    payload["hashed_password"] = get_password_hash(payload.pop("password"))
    user_internal = UserCreateInternal(**payload)

    return await crud_users.create(db=db, object=user_internal, schema_to_select=UserRead)
```

<a id="read-operations"></a>

## 读取操作

<a id="get-a-single-record"></a>

### 获取单条记录

```python
# By id
user = await crud_users.get(db=db, id=user_id)

# By any indexed field
user = await crud_users.get(db=db, username="userson")
user = await crud_users.get(db=db, email="user.userson@example.com")

# Restrict the returned shape with a Pydantic schema
user = await crud_users.get(
    db=db,
    schema_to_select=UserRead,
    username=username,
    is_deleted=False,
)
```

<a id="get-multiple-records"></a>

### 获取多条记录

```python
# All non-deleted users, first 10
result = await crud_users.get_multi(
    db=db,
    is_deleted=False,
    offset=0,
    limit=10,
)
```

`get_multi` 返回以下结构的字典：

```python
{
    "data": [...],
    "total_count": 25,
}
```

需要包含 `page`、`has_more`、`items_per_page` 的完整分页响应时，用 `paginated_response()` 包装，见[分页](#pagination)。

<a id="filter-operators"></a>

### 过滤操作符

FastCRUD 支持字段名后的 `__` 操作符：

```python
# Substring match
await crud_users.get_multi(db=db, username__icontains="john")

# Range
await crud_users.get_multi(db=db, created_at__gt=cutoff_datetime)

# Set membership
await crud_users.get_multi(db=db, tier_id__in=[1, 2, 3])
```

包括 `__contains`、`__icontains`、`__startswith`、`__endswith`、`__gt`、`__ge`、`__lt`、`__le`、`__in`、`__not_in` 等。完整列表见 [FastCRUD 文档](https://benavlabs.github.io/fastcrud/)。

<a id="check-existence"></a>

### 检查是否存在

```python
if await crud_users.exists(db=db, email="user@example.com"):
    raise UserExistsError("Email already registered")
```

只需布尔结果时，`exists()` 比 `get()` 更快，因为不传输记录内容。

<a id="count-records"></a>

### 统计记录数

```python
total = await crud_users.count(db=db)
admins = await crud_users.count(db=db, is_superuser=True)
active = await crud_users.count(db=db, is_deleted=False)
```

<a id="create-operations"></a>

## 创建操作

```python
user_internal = UserCreateInternal(
    name="User Userson",
    username="userson",
    email="user.userson@example.com",
    hashed_password=get_password_hash("Str1ngst!"),
)

created = await crud_users.create(db=db, object=user_internal)
```

服务层的处理模式：

1. 校验输入的**外部** Schema（`UserCreate`）。
2. 应用唯一性检查、密码哈希等业务规则。
3. 用实际需要持久化的值构造**内部** Schema（`UserCreateInternal`）。
4. 调用 `crud.create(db=db, object=internal_schema)`。

需要将返回字典限制为公开结构时，传入 `schema_to_select=UserRead`：

```python
created = await crud_users.create(db=db, object=user_internal, schema_to_select=UserRead)
```

<a id="creating-records-with-foreign-keys"></a>

### 创建带外键的记录

引用其他行的模型，只需在创建 Schema 中包含外键列：

```python
new_rate_limit = RateLimitCreate(
    tier_id=tier.id,
    name="users_list",
    path="/api/v1/users/",
    limit=100,
    period=60,
)
await crud_rate_limits.create(db=db, object=new_rate_limit)
```

<a id="update-operations"></a>

## 更新操作

```python
# Update by id
await crud_users.update(
    db=db,
    object=UserUpdate(email="newemail@example.com"),
    id=user_id,
)

# Update by any field
await crud_users.update(db=db, object=UserUpdate(name="New Name"), username=username)
```

只写入更新 Schema 中显式设置的字段。`*Update` 的全部字段均为 `Optional[T] = None`，未提供的字段会跳过。

<a id="common-pattern-validate-before-update"></a>

### 常见模式：更新前校验

```python
# Service method
if values.username and values.username != db_user["username"]:
    if await crud_users.exists(db=db, username=values.username):
        raise UserExistsError("Username not available")

await crud_users.update(db=db, object=values, id=db_user["id"])
```

<a id="bulk-update"></a>

### 批量更新

`update()` 接受与 `get_multi()` 相同的查找参数，传入非 ID 条件可更新多行：

```python
# Reset profile_image_url for everyone in a deprecated tier
await crud_users.update(
    db=db,
    object=UserUpdate(profile_image_url="https://www.profileimageurl.com"),
    tier_id=deprecated_tier_id,
)
```

<a id="delete-operations"></a>

## 删除操作

<a id="soft-delete-default-for-models-with-softdeletemixin"></a>

### 软删除（具有 `SoftDeleteMixin` 的模型的默认行为）

```python
# Sets is_deleted=True and deleted_at=now()
await crud_users.delete(db=db, id=user_id)

# The row stays — query it explicitly
soft_deleted = await crud_users.get(db=db, id=user_id, is_deleted=True)
```

<a id="hard-delete"></a>

### 硬删除

```python
# DELETE FROM user WHERE id = ?
await crud_users.db_delete(db=db, id=user_id)
```

<a id="filtering-out-soft-deleted-records"></a>

### 排除软删除记录

查询时添加 `is_deleted=False`：

```python
active_users = await crud_users.get_multi(db=db, is_deleted=False, limit=10)
```

<a id="joined-queries"></a>

## 连接查询

已有关联的模型，如 `User.tier`，通过 `lazy="selectin"` 自动加载关联。

未配置关联的临时连接查询，使用 `get_joined` / `get_multi_joined`：

```python
posts_with_authors = await crud_posts.get_multi_joined(
    db=db,
    join_model=User,
    join_on=Post.created_by_user_id == User.id,
    schema_to_select=PostRead,
    join_schema_to_select=UserRead,
    join_prefix="author_",
    offset=0,
    limit=10,
)
# Each row: {..., "author_username": ..., "author_email": ...}
```

模板也使用 **`JoinConfig`** 构建更复杂的多表连接，包含两次连接的实际示例见 `UserService.get_rate_limits`。

<a id="pagination"></a>

## 分页

使用 FastCRUD 的 `paginated_response()` 将 `get_multi` 结果转换为公开分页响应：

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

响应结构：

```json
{
  "data": [{ "id": 1, "name": "...", "username": "..." }],
  "total_count": 150,
  "has_more": true,
  "page": 1,
  "items_per_page": 10
}
```

<a id="selective-field-loading"></a>

## 选择性字段加载

`schema_to_select` 使数据库只返回调用者需要的列，结果是与 Schema 字段对应的普通字典：

```python
# Returns just id, name, username, email, profile_image_url, ...
result = await crud_users.get_multi(
    db=db,
    schema_to_select=UserRead,
    is_deleted=False,
    limit=100,
)
```

可用来避免读取 `hashed_password` 或其他不会使用的大字段。

<a id="complete-workflow-example"></a>

## 完整工作流示例

```python
from sqlalchemy.ext.asyncio import AsyncSession

from src.modules.user.crud import crud_users
from src.modules.user.schemas import (
    UserCreateInternal,
    UserRead,
    UserUpdate,
)
from crudauth import get_password_hash


async def user_lifecycle(db: AsyncSession) -> None:
    # 1. CREATE
    new_user = await crud_users.create(
        db=db,
        object=UserCreateInternal(
            name="Demo User",
            username="demo_user",
            email="demo@example.com",
            hashed_password=get_password_hash("Str1ngst!"),
        ),
        schema_to_select=UserRead,
    )

    # 2. READ
    fetched = await crud_users.get(
        db=db,
        id=new_user["id"],
        schema_to_select=UserRead,
    )

    # 3. UPDATE
    await crud_users.update(
        db=db,
        object=UserUpdate(name="Demo Userson"),
        id=fetched["id"],
    )

    # 4. SOFT DELETE
    await crud_users.delete(db=db, id=fetched["id"])

    # 5. FETCH SOFT-DELETED
    soft_deleted = await crud_users.get(db=db, id=fetched["id"], is_deleted=True)
    assert soft_deleted["deleted_at"] is not None
```

<a id="error-handling"></a>

## 错误处理

共享异常位于 `modules/common/exceptions.py`：`ResourceNotFoundError`、`ResourceExistsError`、`ValidationError`、`PermissionDeniedError`、`PersistenceError`。各功能继承它们定义 `UserExistsError`、`TierNotFoundError` 等异常。服务抛出：

```python
async def create(self, user: UserCreate, db: AsyncSession) -> dict[str, Any]:
    if await crud_users.exists(db=db, email=canonical_email(user.email)):
        raise UserExistsError("Email already registered")
    # ... create user ...
```

路由不需要错误转换代码。应用的 `DomainError` 处理器将异常映射为状态码，并返回异常选定的消息：

```python
return await user_service.create(user, db)
```

<a id="performance-tips"></a>

## 性能建议

<a id="use-schema_to_select"></a>

### 使用 `schema_to_select`

避免加载不会使用的列：

```python
# Good — only the public fields
user = await crud_users.get(db=db, id=user_id, schema_to_select=UserRead)

# Avoid — pulls the password hash too
user = await crud_users.get(db=db, id=user_id)
```

<a id="use-exists-for-existence-checks"></a>

### 使用 `exists()` 检查存在性

```python
# Good — boolean, no row transfer
if await crud_users.exists(db=db, email=email):
    raise UserExistsError("Email taken")

# Avoid — fetches the entire row to check None
user = await crud_users.get(db=db, email=email)
if user:
    raise UserExistsError("Email taken")
```

<a id="use-count-for-counts"></a>

### 使用 `count()` 统计数量

```python
# Good
total = await crud_users.count(db=db, is_deleted=False)

# Avoid
result = await crud_users.get_multi(db=db, is_deleted=False, limit=10000)
total = result["total_count"]  # works, but transfers data
```

<a id="pre-fetch-related-data-in-services"></a>

### 在服务中预取关联数据

路由如果分别调用 `crud_users.get` 和 `crud_tiers.get(tier_id)`，应优先使用现有的 `User.tier` 关联（通过 selectin 自动加载）或 `get_joined`。原文将其描述为减少数据库往返的方式。

<a id="next-steps"></a>

## 后续步骤

- **[迁移](migrations.md)**：通过 Alembic 管理结构变更。
- **[API 接口](../api/endpoints.md)**：将 CRUD 接入 FastAPI 路由。
- **[缓存](../caching/index.md)**：缓存 CRUD 结果。
