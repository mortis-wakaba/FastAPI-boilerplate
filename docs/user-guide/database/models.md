<a id="database-models"></a>

# 数据库模型

本页介绍模板如何组织 SQLAlchemy 2.0 模型，关联关系和时间戳的使用模式，以及添加新模型的方式。

<a id="where-models-live"></a>

## 模型位置

模型位于 **`backend/src/modules/<feature>/models.py`**，与对应功能的 Schema、CRUD、服务和路由放在一起：

```text
backend/src/modules/
├── user/models.py          # User
├── role/models.py          # Role, RolePermission, UserRole
├── tier/models.py          # Tier
├── rate_limit/models.py    # RateLimit
└── api_keys/models.py      # APIKey, KeyUsage
```

共享基类和可复用 mixin 位于 `backend/src/infrastructure/database/`：

```text
backend/src/infrastructure/database/
├── session.py              # Base, async_session, create_tables
└── models.py               # TimestampMixin, SoftDeleteMixin, UUIDMixin
```

<a id="the-base-class"></a>

## 基类

所有模型继承 `infrastructure/database/session.py` 定义的 `Base`：

```python
from sqlalchemy.orm import DeclarativeBase, MappedAsDataclass


class Base(DeclarativeBase, MappedAsDataclass):
    """SQLAlchemy 2.0 base — also a dataclass for ergonomic instantiation."""
    pass
```

`DeclarativeBase` 与 `MappedAsDataclass` 组合后，模型像 Python dataclass 一样工作，可以通过 `User(name=..., username=..., ...)` 实例化。只有显式标记 `init=False` 的列不出现在构造函数中。

<a id="reusable-mixins"></a>

## 可复用 mixin

`infrastructure/database/models.py` 提供三个 mixin，可组合到模型上：

```python
from ...infrastructure.database.models import (
    TimestampMixin,    # adds created_at, updated_at
    SoftDeleteMixin,   # adds is_deleted, deleted_at
    UUIDMixin,         # adds a uuid primary key (alternative to id)
)

class MyModel(Base, TimestampMixin, SoftDeleteMixin):
    ...
```

| Mixin | 添加的内容 | 说明 |
|-------|------------|------|
| `TimestampMixin` | 带 UTC 时区的 `created_at`、`updated_at` | 均为 `init=False`，通过 `datetime.now(UTC)` 提供默认值 |
| `SoftDeleteMixin` | `is_deleted` 布尔值、可空 `deleted_at` 时间 | FastCRUD 的 `.delete()` 修改字段，不执行 `DELETE FROM` |
| `UUIDMixin` | `uuid` 主键，以服务端 `gen_random_uuid()` 兜底 | 适用于对外标识 |

<a id="auto-discovery-for-alembic"></a>

## Alembic 自动发现

无需手动维护模型列表。`src/modules/__init__.py` 不导入任何内容，两个遍历器从磁盘发现模块：

```python
# backend/src/infrastructure/database/registry.py
def import_models(package_name: str = MODELS_PACKAGE) -> None:
    package = importlib.import_module(package_name)
    for _, module_name, _ in pkgutil.walk_packages(package.__path__, package.__name__ + "."):
        importlib.import_module(module_name)
```

`import_models()` 将模型注册到 `Base.metadata`，供 Alembic 的 `--autogenerate` 和 `scripts/create_tables.py` 使用。`infrastructure/permissions.py` 中的 `discover_permissions()` 遍历相同包中的 `permissions.py`，见[权限](../authentication/permissions.md#role-based-permissions)。

新增模块文件存在后，两者即可发现，无需修改注册表。

<a id="relationships"></a>

## 关联关系

模板按需使用 SQLAlchemy `relationship()`。经常读取的关联（如 `User.tier`）使用 `lazy="selectin"`，通过一次后续查询获取关联行，避免 N+1。很少读取的大集合（如 `Tier.users`）使用 `lazy="select"`，避免加载等级时带出所有用户。

例如 `User.tier` 和 `Tier.users` 的双向配置：

```python
# modules/user/models.py
class User(Base, TimestampMixin, SoftDeleteMixin):
    __tablename__ = "user"
    ...
    tier_id: Mapped[int | None] = mapped_column(
        Integer, ForeignKey("tiers.id"), index=True, default=None,
    )
    tier: Mapped["Tier | None"] = relationship(
        "Tier", back_populates="users", lazy="selectin", init=False,
    )

# modules/tier/models.py
class Tier(Base, TimestampMixin, SoftDeleteMixin):
    __tablename__ = "tiers"
    ...
    users: Mapped[list["User"]] = relationship(
        "User", back_populates="tier", lazy="select",
        default_factory=list, init=False,
    )
```

<a id="avoiding-circular-imports"></a>

### 避免循环导入

关联两端需要对方模型类型。用 `TYPE_CHECKING` 包裹导入，并在 `relationship(...)` 中使用字符串类名：

```python
from typing import TYPE_CHECKING

if TYPE_CHECKING:
    from ..tier.models import Tier


class User(Base, ...):
    tier: Mapped["Tier | None"] = relationship("Tier", back_populates="users", ...)
```

<a id="when-to-skip-relationships"></a>

### 何时省略关联关系

外键只向外引用且无需反向遍历时，可只保留外键列：

```python
class APIKey(Base, ...):
    user_id: Mapped[int] = mapped_column(ForeignKey("user.id"), index=True)
    # No User relationship — the API key knows its user via user_id;
    # users don't need a list of all their keys at the ORM level.
```

需要关联数据时，仍可通过 FastCRUD 连接查询。

<a id="the-user-model"></a>

## 用户模型

`modules/user/models.py` 是最完整的示例，以下为节选：

```python
from datetime import datetime
from typing import TYPE_CHECKING
from sqlalchemy import DateTime, ForeignKey, Integer, String
from sqlalchemy.orm import Mapped, mapped_column, relationship

from ...infrastructure.database.models import SoftDeleteMixin, TimestampMixin
from ...infrastructure.database.session import Base

if TYPE_CHECKING:
    from ..tier.models import Tier


class User(Base, TimestampMixin, SoftDeleteMixin):
    __tablename__ = "user"

    id: Mapped[int] = mapped_column(
        "id", autoincrement=True, nullable=False,
        primary_key=True, init=False,
    )

    # Profile
    name: Mapped[str] = mapped_column(String(30))
    username: Mapped[str] = mapped_column(String(32), unique=True, index=True)
    email: Mapped[str] = mapped_column(String(50), unique=True, index=True)
    hashed_password: Mapped[str] = mapped_column(String(100))
    profile_image_url: Mapped[str] = mapped_column(
        String(2048), default="https://profileimageurl.com",
    )

    # Tier (foreign key + relationship)
    tier_id: Mapped[int | None] = mapped_column(
        Integer, ForeignKey("tiers.id"), index=True, default=None,
    )
    tier: Mapped["Tier | None"] = relationship(
        "Tier", back_populates="users", lazy="selectin", init=False,
    )

    is_superuser: Mapped[bool] = mapped_column(default=False)

    # OAuth fields (filled when user signs in via Google/GitHub)
    google_id: Mapped[str | None] = mapped_column(String(50), unique=True, index=True, default=None)
    github_id: Mapped[str | None] = mapped_column(String(50), unique=True, index=True, default=None)
    oauth_provider: Mapped[str | None] = mapped_column(String(20), default=None)
    email_verified: Mapped[bool] = mapped_column(default=False)

    @property
    def is_active(self) -> bool:
        return not self.is_deleted
```

要点：

- `init=False` 从 dataclass 的 `__init__` 排除字段，适用于不希望调用者指定的主键和时间戳。
- `index=True` 为频繁查询的列添加数据库索引，如 `username`、`email`、`tier_id` 和 OAuth ID。
- `unique=True` 在数据库层保证唯一性。
- `is_active` 是**派生属性**而非列，返回 `not is_deleted`。`crudauth` 登录时读取它，使软删除用户无法认证，同时由 `is_deleted` 作为唯一状态来源。

<a id="the-ratelimit-model"></a>

## RateLimit 模型

`modules/rate_limit/models.py` 展示只有外键、没有关联关系的模型：

```python
from sqlalchemy import ForeignKey, Index, Integer, String, text
from sqlalchemy.orm import Mapped, mapped_column

from ...infrastructure.database import Base
from ...infrastructure.database.models import SoftDeleteMixin, TimestampMixin


class RateLimit(Base, TimestampMixin, SoftDeleteMixin):
    __tablename__ = "rate_limits"

    id: Mapped[int] = mapped_column(
        "id", autoincrement=True, nullable=False,
        primary_key=True, init=False,
    )
    tier_id: Mapped[int] = mapped_column(ForeignKey("tiers.id"), index=True)
    name: Mapped[str] = mapped_column(String, nullable=False, unique=True)
    path: Mapped[str] = mapped_column(String, nullable=False)
    limit: Mapped[int] = mapped_column(Integer, nullable=False)
    period: Mapped[int] = mapped_column(Integer, nullable=False)

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

每行表示：等级 X 对 `path` 的请求，每 `period` 秒最多 `limit` 次。索引是部分索引，仅约束未软删除的规则。

<a id="soft-deletion"></a>

## 软删除

`SoftDeleteMixin` 添加 `is_deleted` 和 `deleted_at`。FastCRUD 的 `.delete()` 修改它们，而非删除记录：

```python
# Soft delete (default for models with the mixin)
await crud_users.delete(db=db, id=user_id)

# Actual DELETE FROM
await crud_users.db_delete(db=db, id=user_id)

# Filter out soft-deleted rows
await crud_users.get_multi(db=db, is_deleted=False)
```

<a id="adding-a-new-model"></a>

## 添加新模型

<a id="step-by-step"></a>

### 步骤

1. **创建模块目录**（尚不存在时）：`mkdir -p backend/src/modules/widgets`。
2. **定义模型**：在 `modules/widgets/models.py` 中编写。
3. **注册模型**：原文此处要求在 `modules/__init__.py` 注册，让 Alembic 识别。
4. **生成迁移**：`cd backend && uv run --no-sync alembic revision --autogenerate -m "add widgets"`。
5. **审查迁移**：查看 `migrations/versions/...`，自动生成并非总是正确。
6. **应用迁移**：`uv run --no-sync alembic upgrade head`。

> 译注：步骤 3 与上文自动发现的说明不一致，使用时请核对当前代码。

<a id="example-a-widget-model"></a>

### 示例：`Widget` 模型

```python
# backend/src/modules/widgets/models.py
from sqlalchemy import ForeignKey, String, Text
from sqlalchemy.orm import Mapped, mapped_column

from ...infrastructure.database.models import SoftDeleteMixin, TimestampMixin
from ...infrastructure.database.session import Base


class Widget(Base, TimestampMixin, SoftDeleteMixin):
    __tablename__ = "widgets"

    id: Mapped[int] = mapped_column(
        "id", autoincrement=True, nullable=False,
        primary_key=True, init=False,
    )
    name: Mapped[str] = mapped_column(String(100), unique=True, index=True)
    description: Mapped[str | None] = mapped_column(Text, default=None)
    owner_id: Mapped[int] = mapped_column(ForeignKey("user.id"), index=True)
```

然后添加到 `backend/src/modules/__init__.py`：

```python
from .widgets.models import Widget

__all__ = [
    # ...existing exports...
    "Widget",
]
```

<a id="common-patterns"></a>

## 常见模式

<a id="database-level-constraints"></a>

### 数据库级约束

```python
from sqlalchemy import CheckConstraint, Index, UniqueConstraint


class Product(Base, TimestampMixin):
    __tablename__ = "products"

    id: Mapped[int] = mapped_column(primary_key=True, init=False)
    price_cents: Mapped[int] = mapped_column(nullable=False)
    quantity: Mapped[int] = mapped_column(nullable=False)
    sku: Mapped[str] = mapped_column(String(50))
    org_id: Mapped[int] = mapped_column(ForeignKey("orgs.id"))

    __table_args__ = (
        CheckConstraint("price_cents > 0", name="positive_price"),
        CheckConstraint("quantity >= 0", name="non_negative_quantity"),
        UniqueConstraint("org_id", "sku", name="uq_org_sku"),
        Index("ix_product_price", "price_cents"),
    )
```

<a id="enum-fields"></a>

### 枚举字段

模板偏好 `StrEnum`，如 `OAuthProvider`、`EnvironmentOption`：

```python
from enum import StrEnum
from sqlalchemy import String


class WidgetStatus(StrEnum):
    ACTIVE = "active"
    INACTIVE = "inactive"
    ARCHIVED = "archived"


class Widget(Base, TimestampMixin):
    __tablename__ = "widgets"

    id: Mapped[int] = mapped_column(primary_key=True, init=False)
    status: Mapped[str] = mapped_column(String(20), default=WidgetStatus.ACTIVE.value)
```

以 `String` 存储可简化迁移。也可以用 SQLAlchemy 的 `SQLEnum` 和真实 Postgres 枚举类型，但新增枚举值需要迁移。

<a id="json-fields"></a>

### JSON 字段

```python
from sqlalchemy.dialects.postgresql import JSONB


class UserProfile(Base, TimestampMixin):
    __tablename__ = "user_profiles"

    id: Mapped[int] = mapped_column(primary_key=True, init=False)
    user_id: Mapped[int] = mapped_column(ForeignKey("user.id"), unique=True)
    preferences: Mapped[dict] = mapped_column(JSONB, default_factory=dict, init=False)
```

<a id="migration-considerations"></a>

## 迁移注意事项

<a id="backwards-compatible-changes-safe"></a>

### 向后兼容的变更

- 添加可空列。
- 添加新表。
- 添加索引。
- 增加字符串列长度。

<a id="breaking-changes-need-care"></a>

### 需要谨慎处理的破坏性变更

- 改为非空列，需要默认值或回填方案。
- 删除列，应在部署稳定后进行。
- 改变列类型，通常分为新增列、迁移数据、删除旧列。
- 删除表。

完整流程见[迁移](migrations.md)。

<a id="next-steps"></a>

## 后续步骤

1. **[Schema](schemas.md)**：Pydantic 请求与响应结构。
2. **[CRUD 操作](crud.md)**：FastCRUD 使用模式。
3. **[迁移](migrations.md)**：Alembic 工作流。
