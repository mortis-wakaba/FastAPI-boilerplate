<a id="permissions-and-authorization"></a>

# 权限与授权

认证确定身份，授权确定可执行操作。本页介绍角色权限、超级用户标记、资源归属、等级限制和 API 密钥权限。

角色属于 `rbac` 功能，权限注册表属于核心。移除 rbac 后，注册表和 `require_permissions` 检查保留，但只有超级用户通过，见[可组合功能](../composable-features.md)。

<p align="center">
  <img src="../../assets/credential-narrowing-light.png#only-light" alt="凭据的有效权限等于账户权限与凭据范围的交集" width="100%">
  <img src="../../assets/credential-narrowing-dark.png#only-dark" alt="凭据的有效权限等于账户权限与凭据范围的交集" width="100%">
</p>

<a id="authorization-patterns"></a>

## 授权模式

五种机制可组合使用：

| 模式 | 位置 | 场景 |
|------|------|------|
| **角色权限** | `Role`、`RolePermission`、`UserRole`、`require_permissions` | 向用户组授予命名能力 |
| **超级用户** | `User.is_superuser` | 仅管理员操作 |
| **资源归属** | 服务层检查 | 用户只能修改自己的资源 |
| **等级限制** | `Tier`、`RateLimit` | 套餐功能控制、限流 |
| **API 密钥 scope** | `APIKey.permissions` 注册权限名列表 | 密钥认证后允许执行的操作 |

典型请求依次经过：认证身份、角色或超级用户粗粒度检查、服务层归属或等级细粒度检查，以及独立的按等级逐路由限流。

<a id="role-based-permissions"></a>

## 基于角色的权限

权限是扁平 `resource.action` 字符串，如 `user.read`、`tier.update`。不直接授予用户，而是放入 `Role`，再给用户分配角色。

```text
User ──< UserRole >── Role ──< RolePermission
```

`modules/role/models.py` 定义三表：

| 表 | 列 | 说明 |
|----|----|------|
| `roles` | `id`、唯一 `name`、`description` | 不软删除 |
| `role_permissions` | `role_id`、`permission_name` 复合主键 | 从 roles `ON DELETE CASCADE` |
| `user_roles` | `user_id`、`role_id` 复合主键 | 两端均 `ON DELETE CASCADE` |

<a id="declaring-a-modules-permissions"></a>

### 声明模块权限

模块在自己的 `permissions.py` 用 `StrEnum` 声明，并标记 `@register_permissions("<resource>")`：

```python
# modules/user/permissions.py
from enum import StrEnum

from ...infrastructure.permissions import register_permissions


@register_permissions("user")
class UserPermission(StrEnum):
    READ = "user.read"
    CREATE = "user.create"
    UPDATE = "user.update"
    DELETE = "user.delete"
```

注册时校验：资源名匹配 `^[a-z][a-z0-9_]*$`，成员格式为 `<resource>.<action>`，action 使用相同模式，完整名称不超过 `permission_name` 的 100 字符。资源只能注册一次。

`discover_permissions()` 遍历导入 `src.modules.*.permissions`，`modules/__init__.py` 导入时调用它，新增文件无需其他注册。

应用读取 `infrastructure/permissions.py` 的注册表：

| 函数 | 返回 |
|------|------|
| `all_permissions()` | 全部名称的 `frozenset[str]` |
| `permission_groups()` | `{resource: (names, ...)}`，供 UI 按资源分组 |
| `is_known_permission(name)` | 是否已注册 |

`RolePermission` 拒绝存储未注册名称。已存但后来从代码移除的名称，在加载时忽略，删除枚举成员不会意外授予权限。

<a id="protecting-a-route"></a>

### 保护路由

`require_permissions(*names)` 要求调用者持有所有名称，否则 403。不注入值，应放 `dependencies`：

```python
# modules/user/routes.py
from ...infrastructure.auth.authorization import require_permissions


@router.get(
    "/",
    response_model=PaginatedListResponse[UserRead],
    dependencies=[require_permissions("user.read")],
)
async def get_users(
    db: AsyncSessionDep,
    user_service: UserServiceDep,
    page: int = 1,
    items_per_page: int = 10,
) -> dict[str, Any]:
    ...
```

未知名称是编程错误，声明路由时即抛错，拼写错误在导入时暴露。

**超级用户绕过全部权限检查**。`require_permissions` 无需查表即可通过，`get_current_permissions` 返回所有注册权限，无需角色。

<a id="reading-the-callers-permissions"></a>

### 读取调用者权限

处理器需要自行判断时，使用 `infrastructure/auth/deps.py` 的 `CurrentPermissionsDep`，即 `get_current_permissions` 的 Annotated 别名。请求内只解析一次，多处保护共用查询：

```python
from ...infrastructure.auth.deps import CurrentPermissionsDep


@router.patch("/{username}")
async def update_user_profile(
    username: str,
    values: UserUpdate,
    current_user: CurrentUserDep,
    permissions: CurrentPermissionsDep,
    db: AsyncSessionDep,
    user_service: UserServiceDep,
) -> dict[str, str]:
    await user_service.verify_update_permission(current_user, username, permissions)
    ...
```

`load_permissions(db, user_id, is_superuser=...)` 是相同逻辑的普通函数，可在请求外或查询其他用户权限进行比较时使用。

<a id="granting-permissions"></a>

### 授予权限

角色对应一行 `roles` 和每权限一行 `role_permissions`，分配对应一行 `user_roles`。可从脚本、迁移或自建管理工具使用 ORM 写入：

```python
from src.modules.role.models import Role, RolePermission, UserRole

role = Role(name="support", description="Read-only access to user records")
db.add(role)
await db.flush()

db.add(RolePermission(role_id=role.id, permission_name="user.read"))
db.add(UserRole(user_id=user_id, role_id=role.id))
await db.commit()
```

<a id="managing-roles"></a>

## 管理角色

rbac 在 `/api/v1/roles` 挂载 API，每条路由要求自身 `role.*` 权限及委派检查。调用者不能授予自己没有的权限，也不能分配包含这些权限的角色，超级用户两项均通过。

| 路由 | 权限 | 额外检查 |
|------|------|----------|
| `GET /api/v1/roles/` | `role.read` | 无 |
| `GET /api/v1/roles/{role_id}` | `role.read` | 无 |
| `POST /api/v1/roles/` | `role.create` | 拥有新角色全部权限 |
| `PATCH /api/v1/roles/{role_id}` | `role.update` | 拥有角色全部权限 |
| `PUT /api/v1/roles/{role_id}/permissions` | `role.update` | 拥有新增和删除的全部权限；所有活跃持有者均非超级用户，且不持有调用者没有的权限 |
| `DELETE /api/v1/roles/{role_id}` | `role.delete` | 拥有角色全部权限，且所有活跃持有者不强于调用者、不是超级用户 |
| `POST /api/v1/roles/{role_id}/users/{user_id}` | `role.assign` | 拥有角色全部权限，目标账户非超级用户且不持有调用者缺少的权限 |
| `DELETE /api/v1/roles/{role_id}/users/{user_id}` | `role.assign` | 同上 |
| `GET /api/v1/users/{user_id}/roles` | `role.read` | 无 |

```bash
# Create a role carrying permissions you hold
curl -X POST http://localhost:8000/api/v1/roles/ -b cookies.txt \
  -H "Content-Type: application/json" -H "X-CSRF-Token: <token>" \
  -d '{"name": "editor", "description": "Edits profiles", "permissions": ["user.read", "user.update"]}'

# Hand it to somebody
curl -X POST http://localhost:8000/api/v1/roles/3/users/42 -b cookies.txt -H "X-CSRF-Token: <token>"
```

只接受注册权限，否则返回不回显输入的 `422`。拒绝委派返回 `403` 且不写入，包括反向操作：清空或改名一个调用者无权持有的角色，或改变更强账户的角色。修改角色也是接管账户的方式，与 `user.update` 的规则一致；清空或删除角色会同时移除全部持有者的权限，因此检查所有账户。软删除账户不参与，因为其权限不会授予任何有效用户，分配行仍保留。

注册表不再认识的旧名称既不授予权限，也不阻止调用者管理角色。拥有角色其余有效权限的人仍可改名、清空或删除，设置权限时会清掉旧名称，否则移除功能会使携带其权限的角色无法管理。

同一角色的写入串行而非竞态。每条写路径先 `SELECT … FOR UPDATE` 锁定角色：并发分配同角色到同账户均返回 `201`；并发替换最终为其中一套而非并集；角色删除后到达的请求返回 `404`。软删除账户返回 `404`，后台并发删除账户会等待分配写入。

目标账户也被锁定，因为强弱检查读取其权限。分配使用 `FOR NO KEY UPDATE`，确保超级用户授予更强角色不会插在检查与写入之间。撤销角色或替换权限影响所有持有者，因此对这些账户使用 `FOR SHARE`，两次此类操作可并发，但角色分配等待。始终先锁角色再锁账户，保持顺序一致。

角色写入持有账户行期间，资料更新和登录记账等账户写入会等待。撤销角色持有全部相关账户直到删除提交。锁时长为少量查询并提交的一次角色写入。仅通过外键引用账户的 API 密钥、用量行不受这些锁等待影响。

`role_id`、`user_id` 超出 `1..2147483647` 返回 `422`，避免 Integer 列范围外的值进入数据库。

角色名称唯一，数据库约束最终裁决并发，冲突方返回 `409` 而非 500。

<a id="reading-the-permissions"></a>

## 读取权限

rbac 提供两条要求会话的读取路由：

```bash
# Every permission the running project has, grouped by the resource that declared it —
# what a UI offers when it builds a role, since a role may carry nothing else.
curl http://localhost:8000/api/v1/permissions -b cookies.txt
# → {"permissions": {"api_key": ["api_key.read", ...], "role": ["role.read", ...], ...}}

# What the caller holds, from every source the project wired.
curl http://localhost:8000/api/v1/permissions/me -b cookies.txt
# → {"permissions": ["user.update"]}
```

超级用户自身列表包含所有注册权限。已失效名称不出现在任何列表，注册表赋予名称实际含义。

此处 RBAC 是**全局角色**，没有租户或对象范围。权限判断账户能否执行某类操作，具体能否操作某行仍由服务检查归属。

启用 admin 后，后台包含同三张表，见[管理后台](../admin-panel/index.md#whats-included)。后台使用 `ADMIN_*` 独立凭据，不对应普通账户，已可修改任何人的 `is_superuser`，因此没有需要委派规则约束的较弱操作员，但仍要求权限名已注册。

<a id="superuser-authorization"></a>

## 超级用户授权

用户模型的 `is_superuser: bool` 用于管理员接口，通过 `get_current_superuser` 检查：

```python
from typing import Annotated, Any

from fastapi import APIRouter, Depends

from ...infrastructure.auth.dependencies import get_current_superuser

router = APIRouter()


@router.delete("/admin/users/{username}")
async def gdpr_anonymize(
    username: str,
    _: Annotated[dict[str, Any], Depends(get_current_superuser)],
) -> dict[str, str]:
    # Only superusers reach this code
    ...
```

未使用的依赖参数以 `_:` 命名。未认证返回 401，已认证非超级用户返回 403，见[会话](sessions.md)。

<a id="when-to-use-the-superuser-flag"></a>

### 适用场景

- 创建或删除其他用户。
- `PATCH /api/v1/users/{username}/tier` 分配等级。
- `PATCH /api/v1/rate-limits/{name}` 配置限流。
- GDPR 匿名化。
- 系统配置变更。

只允许部分员工执行的操作，更适合角色权限，而非再增加超级用户。

<a id="bootstrapping-the-first-superuser"></a>

### 初始化首个超级用户

`scripts/setup_initial_data.py` 首次运行时读取 `ADMIN_*`：

```bash
cd backend
uv run --no-sync python -m scripts.setup_initial_data
```

已有用户可通过 `/admin` 或一次性 SQL 修改该列。

<a id="resource-ownership"></a>

## 资源归属

用户只能修改自身数据等规则放在**服务层**。服务抛 `PermissionDeniedError`，全局映射为 HTTP 403。

`modules/user/service.py` 示例：

```python
from ..common.exceptions import PermissionDeniedError


async def verify_user_permission(
    self,
    current_user: dict[str, Any],
    target_username: str,
    action: str,
) -> None:
    """Raise PermissionDeniedError if current_user can't act on target_username."""
    if current_user["username"] != target_username and not current_user["is_superuser"]:
        raise PermissionDeniedError(f"Cannot {action} for another user")
```

路由在执行操作前调用检查。归属和角色权限同时适用时，也传入权限集合。模板的 `PATCH /api/v1/users/{username}`：

```python
# modules/user/routes.py
@router.patch("/{username}")
async def update_user_profile(
    username: str,
    values: UserUpdate,
    current_user: CurrentUserDep,
    principal: CurrentPrincipalDep,
    permissions: CurrentPermissionsDep,
    db: AsyncSessionDep,
    user_service: UserServiceDep,
) -> dict[str, str]:
    await user_service.verify_update_permission(
        current_user, username, permissions, is_superuser=principal.is_superuser, scoped=carries_a_scope(principal)
    )
    user = await user_service.get_by_username(username, db)

    if not user_service.is_self_or_superuser(current_user, user["username"], is_superuser=principal.is_superuser):
        target_permissions = await load_permissions(db, user["id"])
        user_service.verify_no_privilege_escalation(user, values, permissions, target_permissions)

    await user_service.update(user["id"], values, db)
    return {"message": "User updated successfully"}
```

规则如下：

- 用户可改自己的资料；带 scope 的凭据也需 `user.update`。
- 超级用户可改任何人。
- 非超级用户的 `user.update` 持有者只能修改非超级用户、且权限不超过自己的其他账户，因为编辑可用于接管账户。
- 只有超级用户可修改他人邮箱，已验证的提供方邮箱用于 OAuth 匹配已有账户。

本节原文称公开 `UserUpdate` 接受 `name`、`username`、`email`、`profile_image_url`；OAuth 标识、验证标记和更新时间移至后台 `UserAdminUpdate`，发到 API 返回 422。

> 译注：用户管理专章说明邮箱也已从公开更新 Schema 移除，应通过确认流程变更。

异常向上传给 `infrastructure/app_factory.py` 注册的处理器，通过 `EXCEPTION_MAPPING` 将 `PermissionDeniedError` 映射到 403，见[异常处理](../api/exceptions.md)。

<a id="generic-ownership-pattern"></a>

### 通用归属模式

自定义模块示例：

```python
# modules/widgets/service.py
from ..common.exceptions import PermissionDeniedError, ResourceNotFoundError


class WidgetService:
    async def delete(
        self, widget_id: int, current_user: dict[str, Any], db: AsyncSession,
    ) -> None:
        widget = await crud_widgets.get(db=db, id=widget_id)
        if widget is None:
            raise ResourceNotFoundError("Widget not found")

        if widget["owner_id"] != current_user["id"] and not current_user["is_superuser"]:
            raise PermissionDeniedError("Cannot delete another user's widget")

        await crud_widgets.delete(db=db, id=widget_id)
```

遵循三项规则：

1. **服务抛领域异常而非 HTTP 异常**，便于脚本、测试、Taskiq 复用。
2. **显式超级用户绕过**，用 `not current_user["is_superuser"]` 清晰表达。
3. **先检查存在，再检查归属**。原文建议对用户不应知道的资源返回 404 而非 403，见[隐藏资源存在性](../api/exceptions.md#hide-resource-existence)。

<a id="tier-based-authorization"></a>

## 基于等级的授权

每个用户以 `tier_id` 引用 `Tier`。模板只提供**基础等级**，包含 `name` 和 `description`，不内置功能映射或定价，由项目自行定义含义。

<a id="reading-the-users-tier"></a>

### 读取等级

`User.tier` 通过 `lazy="selectin"` 自动加载：

```python
@router.get("/me", response_model=UserRead)
async def me(
    current_user: Annotated[dict[str, Any], Depends(get_current_user)],
) -> dict[str, Any]:
    # current_user["tier"] is the joined Tier dict (or None)
    return current_user
```

<a id="gating-a-feature-on-tier-name"></a>

### 按等级名控制功能

简单开关可在服务中检查等级名：

```python
async def export_data(self, current_user: dict[str, Any], db: AsyncSession) -> bytes:
    tier = current_user.get("tier") or {}
    if tier.get("name") not in {"pro", "enterprise"}:
        raise PermissionDeniedError("Data export requires the Pro or Enterprise tier")
    # ...generate export...
```

适合二元功能控制。每功能配额、多附加套餐等复杂需求，可另建权益系统，不在模板范围内。

<a id="tier-based-rate-limits"></a>

### 按等级限流

每条 `RateLimit` 将等级、路径、`limit` 和 `period` 关联。`infrastructure/auth/setup.py` 组装的 crudauth 限流器逐请求执行，见[限流](../rate-limiting/index.md)。

配置等级规则：

```bash
# Create a rate limit (admin only)
curl -X POST http://localhost:8000/api/v1/rate-limits/ \
  -b superuser_cookies.txt \
  -H "Content-Type: application/json" \
  -H "X-CSRF-Token: <token>" \
  -d '{
    "tier_id": 2,
    "name": "pro_users",
    "path": "/api/v1/widgets/",
    "limit": 1000,
    "period": 60
  }'
```

<a id="api-key-scope"></a>

## API 密钥 scope

scope 存在密钥行，是与角色、`require_permissions` 共用的注册权限名列表：

```python
# modules/api_keys/models.py
class APIKey(Base, TimestampMixin):
    __tablename__ = "api_keys"

    permissions: Mapped[list[str]] = mapped_column(JSON, insert_default=list)
```

创建和更新逐项验证，未注册名称返回不回显输入的 `422`，返回名称排序并去重。省略 `permissions` 的密钥**没有权限范围**，只标识所有者，不附带任何权限。

```bash
curl -X POST http://localhost:8000/api/v1/api-keys/ \
  -b cookies.txt \
  -H "Content-Type: application/json" \
  -H "X-CSRF-Token: <token>" \
  -d '{"name": "Read-only integration", "permissions": ["user.read"], "usage_limits": {"requests_per_day": 10000}}'
```

已存但失效的名称按原样读取、不授予权限。旧版本存储的对象等非名称列表结构不读取，密钥返回 `401` 并记录警告，直到 `scripts/cleanup_api_key_json.py` 修复。

<a id="what-a-key-holds"></a>

### 密钥拥有的权限

有效权限是**所有者权限与 scope 的交集**，scope 不授予新能力。所有者失去的权限在下次请求消失，空 scope 只有身份。`GET /api/v1/permissions/me` 返回收窄结果。

创建及扩大 scope 的 PATCH 只能使用创建者持有的权限，否则 `403`，密钥不会比签发凭据更强。

!!! warning "密钥永远不是超级用户"
    `is_superuser` 不只绕过权限门，还绕过服务归属检查，无法用 scope 表达。因此带 scope 的凭据始终受范围约束，无论账户标记如何。超级用户的 `['user.read']` 密钥只能通过相应路由，不能通过 `role.read` 和任何超级用户接口；超级用户自身会话不受影响。

    所有检查都使用行动凭据的权限和标记。仅有 `role.assign` 的超级用户密钥不能分配含范围外权限的角色；修改他人账户、读取其等级和限流也不能借账户本身的标记绕过。

    scope 收窄权限，不改变资源归属，但归属也不是独立许可。改自己的资料仍需 `user.update`，只读密钥不能重命名账户。密码、邮箱、关闭账户和密钥管理直接要求会话，见[密钥不能做什么](api-keys.md#what-a-key-cannot-do)。

生命周期由 `is_active`、`expires_at` 控制。撤销不可逆，到期只能提前，见[撤销、到期与轮换](api-keys.md#revoking-expiring-and-rotating)。

<a id="combining-patterns"></a>

## 组合模式

实际接口常同时使用多项检查：

```python
@router.delete("/widgets/{widget_id}", status_code=204)
async def delete_widget(
    widget_id: int,
    current_user: Annotated[dict[str, Any], Depends(get_current_user)],   # 1. authn
    db: Annotated[AsyncSession, Depends(async_session)],
    widget_service: Annotated[WidgetService, Depends(get_widget_service)],
) -> None:
    # The service handles:
    #   2. Existence check
    #   3. Ownership check (superuser bypass)
    #   4. Tier feature gate (e.g. "delete requires Pro tier")
    await widget_service.delete(widget_id, current_user, db)
```

路由保持简单，授权规则集中在服务中以便测试和复用。

<a id="testing-authorization"></a>

## 授权测试

权限规则优先测试**服务**，更易准备、执行更快：

```python
import pytest
from src.modules.user.service import UserService
from src.modules.common.exceptions import PermissionDeniedError


@pytest.mark.asyncio
async def test_normal_user_cannot_update_other_users():
    service = UserService()
    current_user = {"username": "alice", "is_superuser": False}

    with pytest.raises(PermissionDeniedError):
        await service.verify_user_permission(current_user, "bob", "update profile")


@pytest.mark.asyncio
async def test_superuser_can_update_other_users():
    service = UserService()
    current_user = {"username": "alice", "is_superuser": True}

    # Should not raise
    await service.verify_user_permission(current_user, "bob", "update profile")
```

完整流程通过 `TestClient` 集成测试验证会话 Cookie 与权限检查，见[测试](../testing.md)。

<a id="best-practices"></a>

## 最佳实践

<a id="keep-authorization-in-services"></a>

### 授权放在服务中

路由负责注入和 HTTP，服务负责规则。服务中抛出权限异常应与业务逻辑一致。

<a id="order-checks-authn-existence-ownership-quota"></a>
<a id="order-checks-authn--existence--ownership--quota"></a>

### 检查顺序：认证 → 存在 → 归属 → 配额

```python
# 1. Authenticated? — done by the dependency
# 2. Resource exists?
if widget is None:
    raise ResourceNotFoundError(...)
# 3. User owns it?
if widget["owner_id"] != current_user["id"] and not current_user["is_superuser"]:
    raise PermissionDeniedError(...)
# 4. Quota / tier OK?
if not within_tier_limits(...):
    raise PermissionDeniedError(...)
```

原文建议此顺序将低成本检查放前，并用 404 优先于 403 减少存在性泄露。

<a id="dont-reinvent-rate-limits"></a>

### 不重复实现限流

内置按等级限流在路由前执行，只有无法表达需求时才添加独立计数，见[限流](../rate-limiting/index.md)。

<a id="audit-superuser-actions"></a>

### 审计超级用户操作

服务端记录操作者和操作，日志基础设施提供 `correlation_id`、`support_id`，见[日志配置](../../user-guide/configuration/index.md)。

<a id="next-steps"></a>

## 后续步骤

- **[会话](sessions.md)**：会话认证原理。
- **[限流](../rate-limiting/index.md)**：按等级的限流。
- **[异常处理](../api/exceptions.md)**：权限异常映射。
- **[生产环境](../production.md)**：安全加固。
