<a id="api-endpoints"></a>

# API 接口

本指南介绍模板的接口模式，使新增接口与现有模块保持一致。

<a id="dependency-injection"></a>

## 依赖注入

模板支持两种等价的 FastAPI 依赖注入方式，代码库中两种都存在，均可使用。

<a id="traditional-style-explicit-depends"></a>

### 传统方式：显式 `Depends()`

```python
from fastapi import Depends
from sqlalchemy.ext.asyncio import AsyncSession

from ...infrastructure.database.session import async_session


@router.get("/items")
async def list_items(
    db: AsyncSession = Depends(async_session),
) -> list[dict[str, Any]]:
    ...
```

<a id="modern-style-annotated-type-aliases"></a>

### 现代方式：Annotated 类型别名

```python
from ...infrastructure.dependencies import AsyncSessionDep


@router.get("/items")
async def list_items(
    db: AsyncSessionDep,
) -> list[dict[str, Any]]:
    ...
```

`infrastructure/dependencies.py` 提供所有项目都需要的别名，与所选功能无关：

| 别名 | 解析为 |
|------|--------|
| `AsyncSessionDep` | `Annotated[AsyncSession, Depends(async_session)]` |

解析调用者的别名属于 accounts，位于 `infrastructure/auth/deps.py`：

| 别名 | 解析为 |
|------|--------|
| `CurrentUserDep` | `Annotated[dict[str, Any], Depends(get_current_user)]` |
| `CurrentSuperUserDep` | `Annotated[dict[str, Any], Depends(get_current_superuser)]` |
| `OptionalUserDep` | `Annotated[dict[str, Any] \| None, Depends(get_optional_user)]` |
| `CurrentPrincipalDep` | `Annotated[Principal, Depends(get_current_principal)]` |
| `CurrentPermissionsDep` | `Annotated[frozenset[str], Depends(get_current_permissions)]` |
| `OAuth2FormDep` | `Annotated[OAuth2PasswordRequestForm, Depends()]` |

模块服务别名位于 `modules/<name>/dependencies.py`：

| 文件 | 别名 |
|------|------|
| `modules/user/dependencies.py` | `UserServiceDep` |
| `modules/tier/dependencies.py` | `TierServiceDep` |
| `modules/rate_limit/dependencies.py` | `RateLimitServiceDep` |
| `modules/api_keys/dependencies.py` | `APIKeyServiceDep` |

两种方式运行行为一致，别名可减少重复，使路由签名更易阅读。

<a id="quick-start"></a>

## 快速开始

典型接口位于 `modules/<feature>/routes.py`，将工作交给服务：

```python
# backend/src/modules/widgets/routes.py
from typing import Annotated, Any

from fastapi import APIRouter, Depends
from sqlalchemy.ext.asyncio import AsyncSession

from ...infrastructure.auth.dependencies import get_current_user
from ...infrastructure.database.session import async_session
from ..common.exceptions import ResourceNotFoundError
from .schemas import WidgetCreate, WidgetRead
from .service import WidgetService

router = APIRouter(tags=["Widgets"])


def get_widget_service() -> WidgetService:
    """Per-module service factory used by Depends()."""
    return WidgetService()


@router.get("/{widget_id}", response_model=WidgetRead)
async def get_widget(
    widget_id: int,
    db: Annotated[AsyncSession, Depends(async_session)],
    widget_service: Annotated[WidgetService, Depends(get_widget_service)],
) -> dict[str, Any]:
    """Get a widget by id."""
    widget = await widget_service.get_by_id(widget_id, db)
    if widget is None:
        raise ResourceNotFoundError(f"widget {widget_id}")

    return widget
```

在 `interfaces/api/v1/__init__.py` 注册路由：

```python
from ....modules.widgets.routes import router as widgets_router

router.include_router(widgets_router, prefix="/widgets")
```

接口地址为 `GET /api/v1/widgets/{widget_id}`。

<a id="common-patterns"></a>

## 常见模式

所有模块遵循相同模式：

1. **路由**定义 HTTP 结构，并调用服务。
2. **服务**包含权限检查、多步骤编排等业务逻辑。
3. **CRUD** 执行数据库 I/O。

以下示例对应 `modules/user/routes.py`、`modules/tier/routes.py` 等现有实现。

<a id="get-a-single-item"></a>

### 获取单条记录

```python
@router.get("/{widget_id}", response_model=WidgetRead)
async def get_widget(
    widget_id: int,
    db: Annotated[AsyncSession, Depends(async_session)],
    widget_service: Annotated[WidgetService, Depends(get_widget_service)],
) -> dict[str, Any]:
    widget = await widget_service.get_by_id(widget_id, db)
    if widget is None:
        raise ResourceNotFoundError(f"widget {widget_id}")

    return widget
```

<a id="get-multiple-items-paginated"></a>

### 获取多条记录（分页）

```python
from fastcrud import PaginatedListResponse, compute_offset, paginated_response


@router.get("/", response_model=PaginatedListResponse[WidgetRead])
async def list_widgets(
    db: Annotated[AsyncSession, Depends(async_session)],
    widget_service: Annotated[WidgetService, Depends(get_widget_service)],
    page: int = 1,
    items_per_page: int = 10,
) -> dict[str, Any]:
    result = await widget_service.get_paginated(
        skip=compute_offset(page, items_per_page),
        limit=items_per_page,
        db=db,
    )
    return paginated_response(crud_data=result, page=page, items_per_page=items_per_page)
```

完整模式见[分页](pagination.md)。

<a id="create"></a>

### 创建

```python
@router.post("/", response_model=WidgetRead, status_code=201)
async def create_widget(
    widget: WidgetCreate,
    db: Annotated[AsyncSession, Depends(async_session)],
    widget_service: Annotated[WidgetService, Depends(get_widget_service)],
) -> dict[str, Any]:
    return await widget_service.create(widget, db)
```

服务负责重复检查和业务校验：

```python
# modules/widgets/service.py
async def create(self, widget: WidgetCreate, db: AsyncSession) -> dict[str, Any]:
    if await crud_widgets.exists(db=db, name=widget.name):
        raise ResourceExistsError("Widget with this name already exists")
    return await crud_widgets.create(db=db, object=widget, schema_to_select=WidgetRead)
```

<a id="update"></a>

### 更新

```python
@router.patch("/{widget_id}", response_model=WidgetRead)
async def update_widget(
    widget_id: int,
    values: WidgetUpdate,
    db: Annotated[AsyncSession, Depends(async_session)],
    widget_service: Annotated[WidgetService, Depends(get_widget_service)],
) -> dict[str, Any]:
    return await widget_service.update(widget_id, values, db)
```

<a id="delete-soft-delete"></a>

### 删除（软删除）

```python
@router.delete("/{widget_id}", status_code=204)
async def delete_widget(
    widget_id: int,
    db: Annotated[AsyncSession, Depends(async_session)],
    widget_service: Annotated[WidgetService, Depends(get_widget_service)],
) -> None:
    await widget_service.delete(widget_id, db)
```

模型使用 `SoftDeleteMixin` 时，`crud_widgets.delete()` 设置 `is_deleted=True`。真正移除行时使用 `db_delete()`。

<a id="authentication"></a>

## 身份认证

会话认证依赖均位于 `infrastructure/auth/dependencies`。

<a id="require-login"></a>

### 要求登录

```python
from ...infrastructure.auth.dependencies import get_current_user


@router.get("/me", response_model=WidgetRead)
async def my_widget(
    current_user: Annotated[dict[str, Any], Depends(get_current_user)],
    db: Annotated[AsyncSession, Depends(async_session)],
    widget_service: Annotated[WidgetService, Depends(get_widget_service)],
) -> dict[str, Any]:
    return await widget_service.get_by_owner(current_user["id"], db)
```

<a id="optional-auth"></a>

### 可选认证

```python
from ...infrastructure.auth.dependencies import get_optional_user


@router.get("/", response_model=list[WidgetRead])
async def list_widgets(
    user: Annotated[dict[str, Any] | None, Depends(get_optional_user)],
    ...
):
    # Show extra fields when logged in
    ...
```

<a id="superuser-only"></a>

### 仅超级用户

```python
from ...infrastructure.auth.dependencies import get_current_superuser


@router.delete("/{widget_id}/permanent")
async def hard_delete_widget(
    widget_id: int,
    db: Annotated[AsyncSession, Depends(async_session)],
    widget_service: Annotated[WidgetService, Depends(get_widget_service)],
    _: Annotated[dict[str, Any], Depends(get_current_superuser)],
) -> dict[str, str]:
    await widget_service.permanent_delete(widget_id, db)
    return {"message": "Widget permanently deleted"}
```

仅用于触发依赖的参数以前导下划线命名，这是模板的通用约定。

<a id="permission-required"></a>

### 要求权限

`require_permissions` 返回依赖，调用者缺少列出的任意权限则返回 403，超级用户始终通过。它不向处理器注入值，应放在装饰器的 `dependencies` 列表：

```python
from ...infrastructure.auth.authorization import require_permissions


@router.get(
    "/",
    response_model=list[WidgetRead],
    dependencies=[require_permissions("widget.read")],
)
async def list_widgets(
    db: AsyncSessionDep,
    widget_service: WidgetServiceDep,
) -> list[dict[str, Any]]:
    return await widget_service.get_all(db)
```

处理器需要权限集合来分支处理或与其他用户比较时，注入 `CurrentPermissionsDep`：

```python
from ...infrastructure.auth.deps import CurrentPermissionsDep


@router.patch("/{widget_id}")
async def update_widget(
    widget_id: int,
    values: WidgetUpdate,
    current_user: CurrentUserDep,
    permissions: CurrentPermissionsDep,
    db: AsyncSessionDep,
    widget_service: WidgetServiceDep,
) -> dict[str, str]:
    await widget_service.verify_update_permission(current_user, widget_id, permissions, db)
    ...
```

权限名必须注册到 `role` 模块的注册表，否则 `require_permissions` 在导入时抛错。声明方式见[权限](../authentication/permissions.md#role-based-permissions)。

<a id="api-key-authentication"></a>

### API 密钥认证

机器间调用见[身份认证](../authentication/index.md)。API 密钥通过 `modules/api_keys/routes.py` 的 `/api/v1/api-keys/*` 接口管理。

<a id="path-query-parameters"></a>
<a id="path--query-parameters"></a>

## 路径和查询参数

<a id="path-parameters"></a>

### 路径参数

```python
@router.get("/{widget_id}")
async def get_widget(widget_id: int, ...):
    ...
```

FastAPI 自动校验 `widget_id` 为整数，非法输入返回 `422`。

<a id="simple-query-parameters"></a>

### 简单查询参数

```python
@router.get("/search")
async def search_widgets(
    db: Annotated[AsyncSession, Depends(async_session)],
    widget_service: Annotated[WidgetService, Depends(get_widget_service)],
    name: str | None = None,
    is_active: bool = True,
) -> list[dict[str, Any]]:
    return await widget_service.search(db=db, name=name, is_active=is_active)
```

<a id="query-validation"></a>

### 查询校验

```python
from fastapi import Query


@router.get("/")
async def list_widgets(
    db: Annotated[AsyncSession, Depends(async_session)],
    page: Annotated[int, Query(ge=1)] = 1,
    items_per_page: Annotated[int, Query(ge=1, le=100)] = 10,
    search: Annotated[str | None, Query(max_length=50)] = None,
):
    ...
```

<a id="error-handling"></a>

## 错误处理

模板分为两层异常。

<a id="domain-exceptions-services"></a>

### 领域异常（服务层）

原文列出 `modules/common/exceptions.py` 中的类型：

- `ResourceNotFoundError`
- `ResourceExistsError`
- `PermissionDeniedError`
- `UserNotFoundError`、`UserExistsError`
- `TierNotFoundError`
- `ValidationError`

服务抛出这些异常，不感知 HTTP。

<a id="http-exceptions-routes"></a>

### HTTP 异常（路由层）

由 `infrastructure/http_exceptions.py` 从 FastCRUD 重新导出：

- `HTTPException`：FastAPI 基类。
- `BadRequestException`：400。
- `UnauthorizedException`：401。
- `ForbiddenException`：403。
- `NotFoundException`：404。
- `UnprocessableEntityException`：422。
- `DuplicateValueException`：本节原文写为 409，异常处理专章写为 422。
- `RateLimitException`：429。

<a id="domain-errors-need-no-route-code"></a>

### 领域错误无需路由处理

`register_exception_handlers(app)` 安装 `DomainError` 处理器。路由让异常继续传播，客户端即可获得正确状态和 `support_id`：

```python
@router.patch("/{widget_id}", response_model=WidgetRead)
async def update_widget(...) -> dict[str, Any]:
    return await widget_service.update(widget_id, values, db)
```

只有需要不同响应时才捕获领域错误，见[异常处理](exceptions.md)。

<a id="direct-http-exceptions"></a>

### 直接抛出 HTTP 异常

无需服务参与的即时 HTTP 错误，可以直接抛出：

```python
from ...infrastructure.http_exceptions import BadRequestException


@router.post("/{widget_id}/export")
async def export_widget(widget_id: int, format: str) -> dict[str, str]:
    if format not in {"csv", "json"}:
        raise BadRequestException("Format must be csv or json")

    ...
```

服务失败无需路由代码处理，服务抛出领域异常后由全局处理器响应，完整说明见[异常处理](exceptions.md)。

<a id="file-uploads"></a>

## 文件上传

```python
from fastapi import File, UploadFile


@router.post("/{user_id}/avatar")
async def upload_avatar(
    user_id: int,
    current_user: Annotated[dict[str, Any], Depends(get_current_user)],
    db: Annotated[AsyncSession, Depends(async_session)],
    file: UploadFile = File(...),
) -> dict[str, str]:
    if not file.content_type or not file.content_type.startswith("image/"):
        raise HTTPException(status_code=400, detail="File must be an image")

    # ...persist the file via your storage backend, then update the user...
    return {"message": "Avatar uploaded successfully"}
```

模板不提供默认存储后端。需要时自行选择本地磁盘、S3 或 GCS，并添加配置分组。

<a id="adding-a-new-endpoint-module"></a>

## 添加新的接口模块

添加 `widgets` 的完整流程：

<a id="1-create-the-module"></a>

### 1. 创建模块

```bash
mkdir -p backend/src/modules/widgets
touch backend/src/modules/widgets/__init__.py
```

<a id="2-add-the-stack"></a>

### 2. 添加各层实现

| 文件 | 内容 |
|------|------|
| `models.py` | SQLAlchemy `Widget` 模型，见[模型](../database/models.md) |
| `schemas.py` | `WidgetCreate`、`WidgetRead`、`WidgetUpdate`，见 [Schema](../database/schemas.md) |
| `crud.py` | `crud_widgets: FastCRUD = FastCRUD(Widget)` |
| `service.py` | `WidgetService`，提供 `create`、`get_by_id`、`update`、`delete` |
| `routes.py` | 包含接口的 `APIRouter` |
| `permissions.py` | 接口有权限要求时，定义权限 `StrEnum`，见[权限](../authentication/permissions.md#role-based-permissions) |

<a id="3-register-the-model"></a>

### 3. 注册模型

在 `backend/src/modules/__init__.py`：

```python
from .widgets.models import Widget

__all__ = [..., "Widget"]
```

<a id="4-mount-the-router"></a>

### 4. 挂载路由

在 `backend/src/interfaces/api/v1/__init__.py`：

```python
from ....modules.widgets.routes import router as widgets_router

router.include_router(widgets_router, prefix="/widgets")
```

<a id="5-generate-a-migration"></a>

### 5. 生成迁移

```bash
cd backend
uv run --no-sync alembic revision --autogenerate -m "Add widgets table"
uv run --no-sync alembic upgrade head
```

<a id="6-test"></a>

### 6. 测试

```bash
curl http://localhost:8000/api/v1/widgets/
```

路由现在可在 `/docs` 中查看。

<a id="best-practices"></a>

## 最佳实践

1. **委派给服务**：保持 `routes.py` 轻量，路由负责 HTTP，服务负责规则。
2. **让领域异常传播**：全局处理器统一映射，保持错误响应一致。
3. **优先使用 `schema_to_select=`**：只返回响应模型需要的列。
4. **更新 Schema 全部字段可选**：按部分更新约定实现。
5. **状态码匹配操作**：创建 201，无响应体删除 204，默认 200。
6. **路由签名保持一致**：通过 `Annotated[..., Depends(...)]` 注入 `db` 和 `<feature>_service`，仅触发认证的依赖参数用 `_`。
7. **不要跨模块导入模型**：外键关联除外，也尽量通过 `TYPE_CHECKING`。

<a id="whats-next"></a>

## 后续步骤

- **[分页](pagination.md)**：使用 `PaginatedListResponse`。
- **[异常处理](exceptions.md)**：完整异常模型。
- **[API 版本管理](versioning.md)**：`/api/v1/` 的组装与新增 `/api/v2/`。
- **[CRUD 操作](../database/crud.md)**：服务下方的数据层。
