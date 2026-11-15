<a id="database-schemas"></a>

# 数据 Schema

Pydantic Schema 在本项目中承担三项职责：**输入校验**、**输出序列化**以及前后端共同依赖的 **API 契约**。Schema 与 SQLAlchemy 模型分离，以精确控制每个接口接受和返回的内容。

<a id="where-schemas-live"></a>

## Schema 位置

每个模块拥有自己的 Schema，与模型和 CRUD 放在一起：

```text
backend/src/modules/
├── user/schemas.py            # UserCreate, UserRead, UserUpdate, UserAnonymize, ...
├── tier/schemas.py            # TierCreate, TierRead, TierUpdate
├── rate_limit/schemas.py      # RateLimitCreate, RateLimitRead, RateLimitUpdate
└── api_keys/schemas.py        # APIKeyCreate, APIKeyRead, APIKeyUpdate, KeyUsageRead
```

跨模块共享的 Schema，如时间戳、软删除 mixin 和通用错误结构，位于 `backend/src/modules/common/schemas.py`。

<a id="common-mixin-schemas"></a>

## 通用 mixin Schema

`modules/common/schemas.py` 提供两个可复用的 Pydantic mixin，与 SQLAlchemy mixin 对应：

```python
# modules/common/schemas.py
class TimestampSchema(BaseModel):
    created_at: datetime = Field(default_factory=lambda: datetime.now(UTC).replace(tzinfo=None))
    updated_at: datetime | None = Field(default=None)
    # serializers cast both to ISO strings


class PersistentDeletion(BaseModel):
    deleted_at: datetime | None = Field(default=None)
    is_deleted: bool = False
```

按需将其组合到完整记录 Schema 中。

<a id="the-user-schemas"></a>

## 用户 Schema

`modules/user/schemas.py` 是最完整的示例。采用**每种用途一个 Schema**的模式，每个操作有独立结构：

```python
from datetime import datetime
from typing import Annotated
from pydantic import BaseModel, ConfigDict, EmailStr, Field

from ..common.schemas import PersistentDeletion, TimestampSchema
from .constants import USERNAME_MAX_LENGTH, USERNAME_PATTERN


# Common fields shared by create/update/full-record
class UserBase(BaseModel):
    name: Annotated[str, Field(min_length=2, max_length=30, examples=["User Userson"])]
    username: Annotated[
        str,
        Field(min_length=2, max_length=USERNAME_MAX_LENGTH, pattern=USERNAME_PATTERN, examples=["userson"]),
    ]
    email: Annotated[EmailStr, Field(examples=["user.userson@example.com"])]


# Full record (used internally — never returned to clients)
class User(TimestampSchema, UserBase, PersistentDeletion):
    hashed_password: str
    is_superuser: bool = False
    profile_image_url: str = "https://www.profileimageurl.com"
    tier_id: int | None = None

    # OAuth
    google_id: str | None = None
    github_id: str | None = None
    oauth_provider: str | None = None
    email_verified: bool = False


# API response — explicitly excludes sensitive fields
class UserRead(BaseModel):
    id: int
    name: Annotated[str, Field(min_length=2, max_length=30)]
    username: Annotated[str, Field(min_length=2, max_length=USERNAME_MAX_LENGTH, pattern=USERNAME_PATTERN)]
    email: EmailStr
    profile_image_url: str
    is_deleted: bool = False
    tier_id: int | None
    is_superuser: bool = False
    email_verified: bool = False
    oauth_provider: str | None = None


# API request body for POST /users/
class UserCreate(UserBase):
    model_config = ConfigDict(extra="forbid")  # reject unknown fields

    password: Annotated[
        str,
        Field(
            min_length=8,
            description=(
                "Password must be at least 8 characters and include a number, "
                "uppercase letter, lowercase letter, and special character"
            ),
            examples=["Str1ngst!"],
        ),
    ]

    @field_validator("password")
    def validate_password_strength(cls, v: str) -> str:
        for label, has_class in PASSWORD_CHARACTER_CLASSES:
            if not any(has_class(character) for character in v):
                raise ValueError(f"Password must include at least one {label}")
        return v
    # OAuth fields — populated when user signs up via Google/GitHub
    google_id: str | None = None
    github_id: str | None = None
    oauth_provider: str | None = None


# What the service writes to the DB (raw password replaced with hash)
class UserCreateInternal(UserBase):
    hashed_password: str
    google_id: str | None = None
    github_id: str | None = None
    oauth_provider: str | None = None
    email_verified: bool = False


# Partial update — every field optional
class UserUpdate(BaseModel):
    model_config = ConfigDict(extra="forbid")

    name: Annotated[str | None, Field(min_length=2, max_length=30, default=None)]
    username: Annotated[
        str | None,
        Field(min_length=2, max_length=USERNAME_MAX_LENGTH, pattern=USERNAME_PATTERN, default=None),
    ]
    email: Annotated[EmailStr | None, Field(default=None)]
    profile_image_url: Annotated[
        str | None,
        Field(max_length=2048, pattern=r"^(https?|ftp)://[^\s/$.?#].[^\s]*$", default=None),
    ]


# Fields only an administrator may set — the OAuth identifiers decide who a
# provider login resolves to, so they stay out of the public update schema
class UserAdminUpdate(UserUpdate):
    google_id: str | None = None
    github_id: str | None = None
    oauth_provider: str | None = None
    email_verified: bool | None = None
    oauth_updated_at: datetime | None = None


class UserUpdateInternal(UserAdminUpdate):
    updated_at: datetime  # service stamps this before persisting


class UserTierUpdate(BaseModel):
    tier_id: int


class UserDelete(BaseModel):
    model_config = ConfigDict(extra="forbid")
    is_deleted: bool
    deleted_at: datetime


# GDPR/LGPD anonymization payload
class UserAnonymize(BaseModel):
    model_config = ConfigDict(extra="forbid")
    name: str
    username: str
    hashed_password: str | None = None
    # ...other PII-clearing fields...
```

<a id="naming-conventions"></a>

### 命名约定

各模块使用一致的命名：

| 后缀 | 用途 |
|------|------|
| `Base` | 创建、更新和完整 Schema 共用的字段 |
| 无后缀，如 `User` | 完整记录，包含所有列，主要内部使用 |
| `Read` | API 响应，移除敏感、内部字段和输入限制；描述已存储记录，因此旧版本写入的数据仍可读取 |
| `Create` | POST 请求体 |
| `CreateInternal` | 服务实际存储的内容，如原始密码转换为 hashed_password |
| `Update` | PATCH 部分更新请求体，所有字段可选 |
| `AdminUpdate` | 扩展更新体，包含只有管理员可以设置的字段 |
| `UpdateInternal` | 服务更新时写入的内容，如自动设置的 `updated_at` |
| `TierUpdate`、`Anonymize`、`Delete` 等 | 特定操作使用的窄范围 Schema |

<a id="why-internal-vs-external"></a>

### 为何区分内部和外部

`Create` 与 `CreateInternal` 的区分，以及更新 Schema 的对应区分，使 API 边界清晰：

- `UserCreate` 接受客户端的 `password: str`。
- 服务计算密码哈希，构造包含 `hashed_password` 的 `UserCreateInternal`。
- 最终由 `crud_users.create(db=db, object=user_internal)` 写入数据库。

客户端无法直接指定 `hashed_password`，模型也不会接触明文密码。

<a id="field-validation"></a>

## 字段校验

<a id="annotated-field"></a>
<a id="annotated--field"></a>

### `Annotated` + `Field`

代码使用 `Annotated[T, Field(...)]` 定义规则：

| `Field` 参数 | 效果 |
|--------------|------|
| `min_length` / `max_length` | 字符串长度上下限 |
| `pattern` | 正则校验，例如用户名使用 `r"^[a-z0-9]+$"` |
| `gt` / `ge` / `lt` / `le` | 数值上下限 |
| `default` | 默认值 |
| `examples` | `/docs` 中展示的 OpenAPI 示例 |
| `description` | OpenAPI 中可见的字段说明 |

<a id="emailstr"></a>

### `EmailStr`

Pydantic 的 `EmailStr` 校验邮箱格式，并规范化大小写。

<a id="configdictextraforbid"></a>

### `ConfigDict(extra="forbid")`

用于 `UserCreate`、`UserUpdate` 等模型。客户端提交声明之外的字段会返回 422，对创建和更新请求尤其重要，避免额外字段混入。

<a id="from_attributes"></a>

### `from_attributes`

需要直接从 SQLAlchemy 模型实例构建 Pydantic Schema 时，使用 `ConfigDict(from_attributes=True)`。模板服务主要使用 FastCRUD 默认返回的字典，因此通常不需要；使用 `UserRead.model_validate(orm_user)` 时则应设置。

<a id="schema-patterns"></a>

## Schema 模式

<a id="optional-fields-in-updates"></a>

### 更新中的可选字段

`*Update` Schema 约定**全部字段可选**：

```python
class UserUpdate(BaseModel):
    model_config = ConfigDict(extra="forbid")

    name: Annotated[str | None, Field(min_length=2, max_length=30, default=None)]
    email: Annotated[EmailStr | None, Field(default=None)]
    # ...
```

服务只写入客户端实际提供的字段。

<a id="custom-validators"></a>

### 自定义校验器

跨字段规则或转换：

```python
from pydantic import field_validator, model_validator


class WidgetCreate(BaseModel):
    name: str
    color: str
    quantity: int = 1

    @field_validator("name")
    @classmethod
    def normalize_name(cls, v: str) -> str:
        if v.lower() in {"admin", "system"}:
            raise ValueError("Reserved name")
        return v.strip().lower()

    @model_validator(mode="after")
    def check_quantity(self) -> "WidgetCreate":
        if self.color == "rare" and self.quantity > 1:
            raise ValueError("Rare widgets are limited to one per request")
        return self
```

`field_validator` 校验单个字段；`model_validator(mode="after")` 在所有字段设置后运行，可校验字段组合。

<a id="computed-fields"></a>

### 计算字段

用于序列化时派生、不存储的值：

```python
from pydantic import computed_field


class UserReadWithStats(UserRead):
    created_at: datetime  # add this if your read schema doesn't already have it

    @computed_field
    @property
    def display_name(self) -> str:
        return f"@{self.username}"

    @computed_field
    @property
    def age_days(self) -> int:
        return (datetime.now(UTC) - self.created_at).days
```

<a id="multi-record-responses"></a>

## 多记录响应

分页列表接口使用 **FastCRUD 的 `PaginatedListResponse`**：

```python
from fastcrud import PaginatedListResponse, compute_offset, paginated_response


@router.get("/", response_model=PaginatedListResponse[UserRead])
async def get_users(
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

单记录接口直接返回 Schema：

```python
@router.get("/me", response_model=UserRead)
async def me(current_user: Annotated[dict[str, Any], Depends(get_current_user)]):
    return current_user
```

<a id="adding-schemas-for-a-new-module"></a>

## 为新模块添加 Schema

1. **创建文件**：`backend/src/modules/widgets/schemas.py`。
2. **定义 `WidgetBase`**：包含创建和更新请求共用的字段及输入规则。
3. **添加 `WidgetCreate`、`WidgetRead`、`WidgetUpdate`**，以及需要的内部变体。
4. **在模块的 `routes.py` 和 `service.py` 中使用**。

```python
# backend/src/modules/widgets/schemas.py
from datetime import datetime
from typing import Annotated
from pydantic import BaseModel, ConfigDict, Field

from ..common.schemas import TimestampSchema


class WidgetBase(BaseModel):
    name: Annotated[str, Field(min_length=1, max_length=50)]
    description: Annotated[str | None, Field(max_length=255, default=None)]


class WidgetCreate(WidgetBase):
    model_config = ConfigDict(extra="forbid")


class WidgetRead(BaseModel):
    model_config = ConfigDict(from_attributes=True)
    id: int
    owner_id: int
    name: str
    description: str | None
    created_at: datetime


class WidgetUpdate(BaseModel):
    model_config = ConfigDict(extra="forbid")
    name: Annotated[str | None, Field(min_length=1, max_length=50, default=None)]
    description: Annotated[str | None, Field(max_length=255, default=None)]
```

<a id="common-pitfalls"></a>

## 常见问题

<a id="dont-expose-sensitive-fields"></a>

### 不要暴露敏感字段

```python
# BAD — leaks the password hash
class UserRead(BaseModel):
    hashed_password: str

# GOOD — read-only public shape
class UserRead(BaseModel):
    id: int
    name: str
    username: str
    email: EmailStr
    profile_image_url: str
```

<a id="dont-query-the-database-in-validators"></a>

### 不要在校验器中查询数据库

```python
# BAD — every request hits the DB twice
@field_validator("email")
@classmethod
def email_must_be_unique(cls, v):
    if crud_users.exists(email=v):  # I/O in a validator
        raise ValueError("Email already exists")

# GOOD — let the DB unique constraint and service-layer logic handle it
```

模板的 `UserService.create` 已在插入前检查重复，数据库唯一约束提供最终保证。

<a id="dont-reuse-the-same-schema-for-create-and-update"></a>

### 创建和更新不要复用同一 Schema

`Create` 要求必填的字段，`Update` 应允许省略。分开定义可避免客户端漏传字段却被默认值意外覆盖的问题。

<a id="next-steps"></a>

## 后续步骤

1. **[CRUD 操作](crud.md)**：Schema 如何接入 FastCRUD。
2. **[迁移](migrations.md)**：管理底层数据库变更。
3. **[API 接口](../api/endpoints.md)**：在路由中使用 Schema。
