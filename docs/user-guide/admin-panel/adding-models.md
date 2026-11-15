<a id="adding-models-to-the-admin"></a>

# 向后台添加模型

注册模型较简单，但模板使用 SQLAlchemy 的 `MappedAsDataclass`，需特殊 mixin 适配 SQLAdmin。

完整选项见 [SQLAdmin 文档](https://github.com/smithyhq/sqladmin)。

<a id="the-dataclassmodelmixin"></a>

## DataclassModelMixin

SQLAdmin 默认先创建空实例，再逐个设置属性；对于有无默认值必填字段的 dataclass，这会失败。

`backend/src/interfaces/admin/mixins.py` 的 `DataclassModelMixin` 改为一次性传入全部表单数据构造：

```python
from ...interfaces.admin.mixins import DataclassModelMixin

class MyModelAdmin(DataclassModelMixin, ModelView, model=MyModel):
    ...
```

**代码库的所有后台视图都使用它**。遗漏会造成创建时的 `AttributeError`，甚至静默写入 NULL。

<a id="the-textcsvexportmixin"></a>

## TextCsvExportMixin

SQLAdmin CSV 导出将每列转为 `str(value)`，用户注册输入原样进入文件。表格软件会把以 `=`、`+`、`-`、`@`、制表符或回车开头的单元格当公式。相同模块的 `TextCsvExportMixin` 为这些文本加单引号前缀。所有 `can_export = True` 的视图应添加：

```python
from ...interfaces.admin.mixins import DataclassModelMixin, TextCsvExportMixin

class MyModelAdmin(DataclassModelMixin, TextCsvExportMixin, ModelView, model=MyModel):
    can_export = True
```

同时覆盖普通 CSV 和 `use_pretty_export = True` 路径。数字、日期和 None 保持 SQLAdmin 原样，负数仍是数字；JSON 导出不变。

视图自行实现 `custom_export_cell` 时会覆盖 mixin，应对返回值调用同模块 `as_spreadsheet_text`。

SQLAdmin 的 `can_export` 默认 True，未声明仍提供 `/admin/{identity}/export/csv`。Mixin 必须位于 `ModelView` **之前**，否则继承顺序使原导出生效。`tests/unit/interfaces/admin/test_csv_export.py` 遍历 `ADMIN_VIEWS`，确保所有可导出视图都由安全 mixin 接管。

<a id="adding-a-new-model-view"></a>

## 添加模型视图

<a id="1-create-the-view-file"></a>

### 1. 创建视图文件

视图与模型所属功能放在一起，移除功能时一并移除。

```python
# backend/src/modules/widgets/admin.py
from sqladmin import ModelView

from ...interfaces.admin.mixins import DataclassModelMixin, TextCsvExportMixin
from .models import Widget
from .schemas import WidgetCreate, WidgetUpdate


class WidgetAdmin(DataclassModelMixin, TextCsvExportMixin, ModelView, model=Widget):
    name = "Widget"
    name_plural = "Widgets"
    icon = "fa-solid fa-cube"
    category = "Inventory"

    # List view
    column_list = [Widget.id, Widget.name, Widget.owner_id, Widget.created_at]
    column_searchable_list = [Widget.name]
    column_sortable_list = [Widget.id, Widget.name, Widget.created_at]
    column_default_sort = [(Widget.id, True)]   # True = descending

    # Detail view
    column_details_list = "__all__"

    # Forms — derived from your Pydantic schemas
    form_create_rules = list(WidgetCreate.model_fields.keys())
    form_edit_rules = list(WidgetUpdate.model_fields.keys())

    # Permissions
    can_create = True
    can_edit = True
    can_delete = True
    can_view_details = True
    can_export = True
```

<a id="2-register-it"></a>

### 2. 注册

加入唯一集中列出视图的 wiring：

```python
# backend/src/wiring/admin.py
from ..modules.tier.admin import TierAdmin
from ..modules.user.admin import UserAdmin
from ..modules.widgets.admin import WidgetAdmin   # new

ADMIN_VIEWS: tuple[type, ...] = (
    UserAdmin,
    TierAdmin,
    WidgetAdmin,                                  # new
)
```

重启后，Widgets 显示在侧栏 **Inventory** 分类。

<a id="configuration-options"></a>

## 配置选项

<a id="column-display"></a>

### 列展示

```python
column_list = [MyModel.id, MyModel.name, MyModel.status]
column_labels = {
    "hashed_password": "Password",  # rename a column header
}
```

`UserAdmin` 用 `column_labels` 将 `hashed_password` 显示为 Password，实际哈希在 `on_model_change` 完成。

<a id="search-and-sort"></a>

### 搜索与排序

```python
column_searchable_list = [MyModel.name, MyModel.email]
column_sortable_list = [MyModel.id, MyModel.created_at]
column_default_sort = [(MyModel.created_at, True)]   # True = descending
```

<a id="form-rules"></a>

### 表单规则

通过 Pydantic Schema 驱动字段，保持与 API 校验一致：

```python
form_create_rules = list(MyModelCreate.model_fields.keys())
form_edit_rules = list(MyModelUpdate.model_fields.keys())
```

需要不同顺序或包含外键列时，也可显式编写列表。

<a id="foreign-keys-and-relationships"></a>

## 外键与关联关系

模型采用双重模式：数据库操作使用外键列，SQLAdmin 展示使用关联关系。

<a id="the-model-pattern"></a>

### 模型模式

有外键时定义对应关系：

```python
# modules/widgets/models.py

from typing import TYPE_CHECKING
from sqlalchemy import ForeignKey, Integer
from sqlalchemy.orm import Mapped, mapped_column, relationship

from ...infrastructure.database.session import Base

if TYPE_CHECKING:
    from ..user.models import User


class Widget(Base, ...):
    __tablename__ = "widgets"
    ...

    # Foreign-key column — used by FastCRUD and DB constraints
    owner_id: Mapped[int] = mapped_column(
        Integer, ForeignKey("user.id"), index=True,
    )

    # Relationship — used by SQLAdmin for display and form dropdowns.
    # Required: lazy="selectin" (async) and init=False (excluded from dataclass __init__)
    owner: Mapped["User"] = relationship(
        "User", lazy="selectin", init=False,
    )
```

<a id="why-both"></a>

### 为何同时存在

- **FastCRUD** 直接用外键列并返回字典，如 `widget['owner_id']`。
- **SQLAdmin** 用关联展示友好下拉框，显示对象 `__repr__` 而非整数。

<a id="column_list-uses-the-relationship"></a>

### `column_list` 使用关系

```python
class WidgetAdmin(DataclassModelMixin, TextCsvExportMixin, ModelView, model=Widget):
    # Use Widget.owner (relationship), not Widget.owner_id (FK column).
    # This shows "user@example.com" instead of just an integer.
    column_list = [Widget.id, Widget.name, Widget.owner, Widget.created_at]
```

`UserAdmin` 等级列示例：

```python
column_list = [User.id, User.name, User.username, User.email, User.is_superuser, User.tier]
```

使用 `User.tier`，而非 `User.tier_id`。

<a id="form-rules-use-fk-column-names"></a>

### 表单规则使用外键列名

规则中使用带 `_id` 的**外键列名**，SQLAdmin 自动生成可搜索下拉框：

```python
form_create_rules = [*WidgetCreate.model_fields.keys(), "owner_id"]
```

<a id="lazyselectin-is-required"></a>

### `lazy="selectin"`

SQLAdmin 在异步上下文运行，常规关系使用 `lazy="selectin"` 避免延迟加载异常，遗漏表现为 `MissingGreenlet` 或 `greenlet_spawn has not been called`。模板的 `User.tier` 已这样配置。

大的一对多集合是例外，例如 `Tier.users` 使用 `lazy="select"`，避免加载等级时带出所有用户。SQLAdmin 会对列表与表单中的关系执行 `selectinload`，详情页复用编辑查询；关系表单还会列出关联表全部行。因此大集合应从列表、表单、详情排除，如 `TierAdmin`：

```python
form_excluded_columns = [Tier.users]
column_details_exclude_list = [Tier.users]
```

`column_details_exclude_list` 替代 `column_details_list = "__all__"`，两者不能同时设置，其余列默认展示。

<a id="dont-set-defaultnone-on-relationships"></a>

### 关系不要设置 `default=None`

可空外键也不要在关系本身设置 `default=None`：

```python
# WRONG — SQLAlchemy clears the FK during commit
tier: Mapped["Tier | None"] = relationship("Tier", default=None, init=False)

# CORRECT — relationship returns None naturally when FK is null
tier: Mapped["Tier | None"] = relationship("Tier", init=False)
```

用户模型展示了正确方式。`DataclassModelMixin` 构造前会过滤关系对象，因此 `owner_id=42` 有效，混入 `owner=<User instance>` 会忽略。

<a id="data-transformation-hooks"></a>

## 数据转换钩子

<a id="on_model_change-transform-before-save"></a>
<a id="on_model_change--transform-before-save"></a>

### `on_model_change`：保存前转换

插入和更新前运行，可哈希密码、规范化字段：

```python
from typing import Any
from starlette.requests import Request


class UserAdmin(DataclassModelMixin, ModelView, model=User):
    async def on_model_change(
        self,
        data: dict[str, Any],
        model: Any,
        is_created: bool,
        request: Request,
    ) -> None:
        if is_created and data.get("hashed_password"):
            # Form's "Password" field maps to hashed_password column;
            # hash the plaintext before the row is created
            data["hashed_password"] = get_password_hash(data["hashed_password"])
```

`is_created` 区分创建与更新，创建时 `model` 为 None。

<a id="after_model_change-side-effects-after-save"></a>
<a id="after_model_change--side-effects-after-save"></a>

### `after_model_change`：保存后副作用

记录提交后运行，适合欢迎邮件、webhook 等：

```python
async def after_model_change(
    self,
    data: dict[str, Any],
    model: Any,
    is_created: bool,
    request: Request,
) -> None:
    if is_created:
        await notify_new_user(model)
```

<a id="delete_model-custom-delete-behavior"></a>
<a id="delete_model--custom-delete-behavior"></a>

### `delete_model`：自定义删除

删除不只是 `DELETE FROM` 时覆盖。`TierAdmin` 调用 `permanent_delete` 检查用户和限流引用：

```python
async def delete_model(self, request: Request, pk: str) -> None:
    from ....modules.tier.crud import crud_tiers

    async with local_session() as db:
        tier_service = TierService()

        tier = await crud_tiers.get(db=db, id=int(pk))
        if not tier:
            raise ValueError(f"Tier with ID {pk} not found")

        await tier_service.permanent_delete(tier["name"], db)
```

<a id="bulk-actions"></a>

## 批量操作

管理员可多选记录，通过 `@action` 执行：

```python
from sqladmin import action
from starlette.requests import Request
from starlette.responses import RedirectResponse


class WidgetAdmin(DataclassModelMixin, TextCsvExportMixin, ModelView, model=Widget):
    @action(
        name="deactivate",
        label="Deactivate Selected",
        confirmation_message="Deactivate these widgets?",
        add_in_list=True,
    )
    async def action_deactivate(self, request: Request) -> RedirectResponse:
        pks = request.query_params.get("pks", "").split(",")
        if pks and pks[0]:
            ids = [int(pk) for pk in pks]
            async with local_session() as db:
                await crud_widgets.update(
                    db=db,
                    object={"is_active": False},
                    allow_multiple=True,
                    id__in=ids,
                )
                await db.commit()

        referer = request.headers.get("Referer")
        return RedirectResponse(referer or request.url_for("admin:list", identity=self.identity))
```

说明：

- ID 从 `request.query_params['pks']` 获取，为逗号分隔字符串。
- 从 `infrastructure/database/session.py` 导入 `local_session()`，在共享引擎上打开会话。
- 重定向前提交，否则请求结束时回滚。
- SQLAdmin action 使用 GET，其他站点可诱导浏览器访问。后台[来源保护](configuration.md#requests-from-another-site)对跨源 action 返回 403，写操作只能从后台自身页面发起，邮件或外站链接会被拒绝。

<a id="icons"></a>

## 图标

使用 [Font Awesome](https://fontawesome.com/icons)，通过 `icon` 配置：

```python
icon = "fa-solid fa-user"          # users
icon = "fa-solid fa-layer-group"   # tiers / categories
icon = "fa-solid fa-key"           # api keys
icon = "fa-solid fa-gauge-high"    # rate limits
icon = "fa-solid fa-cube"          # generic
```

<a id="categories"></a>

## 分类

用 `category` 归组：

```python
class WidgetAdmin(...):
    category = "Inventory"
```

同类视图出现在相同侧栏标题下，已有视图使用 `Users & Access`。

<a id="soft-delete-vs-hard-delete"></a>

## 软删除与硬删除

`SoftDeleteMixin` 提供 `is_deleted`、`deleted_at`，但 SQLAdmin 默认硬删除。需要软删除时覆盖：

```python
async def delete_model(self, request: Request, pk: str) -> None:
    async with local_session() as db:
        await crud_widgets.delete(db=db, id=int(pk))   # FastCRUD soft-deletes via the mixin
        await db.commit()
```

原文认为后台常需要直接删除规范记录，但应明确选择行为。

<a id="real-examples-in-the-codebase"></a>

## 实际参考

本节介绍两个参考视图：

| 文件 | 展示内容 |
|------|----------|
| `backend/src/modules/user/admin.py` | 密码哈希钩子、OAuth 选择框、关联列、自定义标签 |
| `backend/src/modules/tier/admin.py` | 调用服务的删除覆盖、Schema 驱动表单 |

<a id="key-files"></a>

## 关键文件

| 组件 | 位置 |
|------|------|
| dataclass mixin | `backend/src/interfaces/admin/mixins.py` |
| 注册表 | `backend/src/wiring/admin.py` |
| 示例视图 | `backend/src/modules/user/admin.py`、`backend/src/modules/tier/admin.py` |
| 认证后端 | `backend/src/interfaces/admin/auth.py` |

<a id="next-steps"></a>

## 后续步骤

- **[用户管理](user-management.md)**：强化后台认证。
- **[模型](../database/models.md)**：定义视图展示的模型。
- **[Schema](../database/schemas.md)**：表单规则。
