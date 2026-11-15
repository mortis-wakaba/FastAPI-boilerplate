<a id="api-pagination"></a>

# API 分页

模板通过 FastCRUD 的 `PaginatedListResponse[T]` 和 `paginated_response()` 构建分页列表，本页说明对应模式。

<a id="quick-start"></a>

## 快速开始

```python
from typing import Any

from fastapi import APIRouter
from fastcrud import PaginatedListResponse, compute_offset, paginated_response

from ...infrastructure.dependencies import AsyncSessionDep
from .dependencies import UserServiceDep
from .schemas import UserRead


@router.get("/", response_model=PaginatedListResponse[UserRead])
async def get_users(
    db: AsyncSessionDep,
    user_service: UserServiceDep,
    page: PageDep = 1,
    items_per_page: ItemsPerPageDep = 10,
) -> dict[str, Any]:
    result = await user_service.get_paginated(
        skip=compute_offset(page, items_per_page),
        limit=items_per_page,
        db=db,
    )
    return paginated_response(crud_data=result, page=page, items_per_page=items_per_page)
```

使用文档提供的 `compute_offset(page, items_per_page)`，无需手动计算 `(page - 1) * items_per_page`。

<a id="response-shape"></a>

## 响应结构

`paginated_response()` 返回：

```json
{
  "data": [
    { "id": 1, "name": "User Userson", "username": "userson", "email": "user@example.com" }
  ],
  "total_count": 150,
  "has_more": true,
  "page": 1,
  "items_per_page": 10
}
```

当前页后仍有记录，即 `page * items_per_page < total_count` 时，`has_more` 为 `True`。模板不返回 `total_pages`，前端需要时可计算 `ceil(total_count / items_per_page)`。

<a id="where-the-service-does-the-work"></a>

## 服务层执行查询

路由保持轻量，实际 `get_multi` 调用放在服务中：

```python
# modules/user/service.py
from fastcrud.types import GetMultiResponseDict
from .crud import crud_users
from .schemas import UserRead


class UserService:
    async def get_paginated(
        self, db: AsyncSession, skip: int = 0, limit: int = 100,
    ) -> GetMultiResponseDict:
        return await crud_users.get_multi(
            db=db,
            offset=skip,
            limit=limit,
            is_deleted=False,
            schema_to_select=UserRead,
            return_total_count=True,
        )
```

`return_total_count=True` 使响应包含 `total_count`，并保证 `has_more` 准确。

<a id="filtering"></a>

## 过滤

在路由添加过滤参数并传入服务：

```python
@router.get("/", response_model=PaginatedListResponse[UserRead])
async def list_users(
    db: Annotated[AsyncSession, Depends(async_session)],
    user_service: Annotated[UserService, Depends(get_user_service)],
    page: int = 1,
    items_per_page: int = 10,
    search: str | None = None,
    tier_id: int | None = None,
) -> dict[str, Any]:
    result = await user_service.get_paginated(
        skip=compute_offset(page, items_per_page),
        limit=items_per_page,
        db=db,
        search=search,
        tier_id=tier_id,
    )
    return paginated_response(crud_data=result, page=page, items_per_page=items_per_page)
```

在服务中构造 `crud_users.get_multi` 的过滤条件：

```python
async def get_paginated(
    self,
    db: AsyncSession,
    skip: int = 0,
    limit: int = 100,
    search: str | None = None,
    tier_id: int | None = None,
) -> GetMultiResponseDict:
    filters: dict[str, Any] = {"is_deleted": False}
    if tier_id is not None:
        filters["tier_id"] = tier_id
    if search:
        filters["username__icontains"] = search

    return await crud_users.get_multi(
        db=db,
        offset=skip,
        limit=limit,
        schema_to_select=UserRead,
        return_total_count=True,
        **filters,
    )
```

FastCRUD 的 `__icontains`、`__contains`、`__gt`、`__in` 等操作符可避免编写原始 SQL，见 [CRUD 操作](../database/crud.md)。

<a id="sorting"></a>

## 排序

FastCRUD 接受 `sort_columns` 和 `sort_orders`：

```python
result = await crud_users.get_multi(
    db=db,
    offset=skip,
    limit=limit,
    sort_columns="created_at",
    sort_orders="desc",
    return_total_count=True,
)
```

多字段排序传入列表：

```python
sort_columns=["tier_id", "created_at"],
sort_orders=["asc", "desc"],
```

通过路由查询参数对外提供：

```python
from fastapi import Query


@router.get("/", response_model=PaginatedListResponse[UserRead])
async def list_users(
    db: Annotated[AsyncSession, Depends(async_session)],
    user_service: Annotated[UserService, Depends(get_user_service)],
    page: int = 1,
    items_per_page: int = 10,
    sort_by: Annotated[str, Query(pattern=r"^(created_at|username|email)$")] = "created_at",
    sort_order: Annotated[str, Query(pattern=r"^(asc|desc)$")] = "desc",
) -> dict[str, Any]:
    ...
```

`pattern` 限制客户端传入任意列名，避免暴露不应允许排序的字段。

<a id="validation"></a>

## 校验

所有列表的边界统一定义于 `modules/common/pagination.py`，避免路由遗漏：

```python
MAX_ITEMS_PER_PAGE = 100
MAX_PAGE = 2**31 - 1

PageDep = Annotated[int, Query(ge=1, le=MAX_PAGE, description="Page number")]
ItemsPerPageDep = Annotated[int, Query(ge=1, le=MAX_ITEMS_PER_PAGE, description=f"Items per page (max {MAX_ITEMS_PER_PAGE})")]
```

路由只需使用两个参数别名：

```python
@router.get("/", response_model=PaginatedListResponse[UserRead])
async def list_users(
    db: AsyncSessionDep,
    user_service: UserServiceDep,
    page: PageDep = 1,
    items_per_page: ItemsPerPageDep = 10,
) -> dict[str, Any]:
    ...
```

所有列表的 `items_per_page` 超过 100 都返回 `422`，包括以前允许 1000 的 API 密钥用量历史。`page` 超过 `2147483647` 也返回 `422`，避免数据库收到超出范围的偏移量。

<a id="real-endpoint-list-users"></a>

## 实际接口：用户列表

来自 `modules/user/routes.py`：

```python
from ...infrastructure.auth.authorization import require_permissions
from ..common.pagination import ItemsPerPageDep, PageDep


@router.get(
    "/",
    response_model=PaginatedListResponse[UserRead],
    summary="List All Users",
    responses={
        401: {"description": "Not authenticated"},
        403: {"description": "Not authorized - requires the user.read permission"},
    },
    dependencies=[require_permissions("user.read")],
)
async def get_users(
    db: AsyncSessionDep,
    user_service: UserServiceDep,
    page: PageDep = 1,
    items_per_page: ItemsPerPageDep = 10,
) -> dict[str, Any]:
    """Get paginated list of users."""
    users_data = await user_service.get_paginated(
        skip=compute_offset(page, items_per_page),
        limit=items_per_page,
        db=db,
    )
    return paginated_response(crud_data=users_data, page=page, items_per_page=items_per_page)
```

接口要求角色授予的 `user.read` 权限，而非直接检查超级用户标记。超级用户始终通过，见[权限](../authentication/permissions.md#role-based-permissions)。

<a id="real-endpoint-api-key-usage-history"></a>

## 实际接口：API 密钥用量历史

来自 `modules/api_keys/routes.py`，使用相同模式和边界：

```python
@router.get(
    "/{key_id}/usage",
    response_model=PaginatedListResponse[KeyUsageRead],
)
async def get_key_usage(
    current_user: CurrentUserDep,
    api_key_service: APIKeyServiceDep,
    db: AsyncSessionDep,
    key_id: int = Path(..., description="API key ID"),
    page: int = Query(1, ge=1, description="Page number"),
    items_per_page: ItemsPerPageDep = 100,
) -> dict[str, Any]:
    result = await api_key_service.get_key_usage(
        key_id=key_id,
        user_id=current_user["id"] if isinstance(current_user, dict) else current_user.id,
        limit=items_per_page,
        offset=compute_offset(page, items_per_page),
        db=db,
    )
    return paginated_response(crud_data=result, page=page, items_per_page=items_per_page)
```

<a id="simple-list-without-pagination"></a>

## 不分页的简单列表

确实不需要分页时，例如管理接口返回数量很小的等级集合，调用一次 `get_multi` 并直接返回 `data`：

```python
@router.get("/all", response_model=list[TierRead])
async def list_all_tiers(
    db: Annotated[AsyncSession, Depends(async_session)],
    tier_service: Annotated[TierService, Depends(get_tier_service)],
) -> list[dict[str, Any]]:
    result = await tier_service.get_all(db=db, skip=0, limit=1000)
    return result["data"]
```

即便如此，也应设置较大但有限的 `limit`，不执行无上限查询。

<a id="performance-tips"></a>

## 性能建议

<a id="cap-items_per_page"></a>

### 限制 `items_per_page`

使用 `Annotated[int, Query(ge=1, le=100)]` 设置边界。

<a id="use-schema_to_select"></a>

### 使用 `schema_to_select`

只读取响应模型需要的列，`UserRead` 可避免读取 `hashed_password`：

```python
return await crud_users.get_multi(
    db=db,
    schema_to_select=UserRead,
    return_total_count=True,
    offset=skip,
    limit=limit,
)
```

<a id="index-columns-you-sort-or-filter-on"></a>

### 为排序或过滤列建索引

新增排序或过滤参数对应的列没有索引时，通过 Alembic 迁移添加：

```python
def upgrade() -> None:
    op.create_index("ix_user_created_at", "user", ["created_at"])
```

因此用户模型已为 `username`、`email`、`tier_id`、`google_id`、`github_id` 建索引。

<a id="beware-of-large-offsets"></a>

### 留意大偏移量

`OFFSET 100000` 仍需 Postgres 扫描并丢弃十万行。超大数据集可采用键集分页，如 `created_at < cursor`，替代页码分页。FastCRUD 的 `__lt` / `__gt` 可直接支持。

<a id="whats-next"></a>

## 后续步骤

- **[CRUD 操作](../database/crud.md)**：过滤、排序、offset 和 limit 语义。
- **[Schema](../database/schemas.md)**：`*Read` 与 `schema_to_select` 的配合。
- **[身份认证](../authentication/index.md)**：为列表设置登录、权限或管理员要求。
