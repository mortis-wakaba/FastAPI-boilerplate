<a id="user-management"></a>

# 用户管理

用户管理覆盖注册、认证、资料更新和删除。本页介绍模板已有接口和使用模式。

<a id="endpoints-at-a-glance"></a>

## 接口速览

定义于 `modules/user/routes.py`，前缀为 `/api/v1/users/`：

| 方法 | 路径 | 说明 | 认证 |
|------|------|------|------|
| `POST` | `/api/v1/users/` | 创建用户 | 开放 |
| `GET` | `/api/v1/users/` | 分页用户列表 | 本表原文写超级用户，下文写 `user.read` |
| `GET` | `/api/v1/users/me` | 当前用户资料 | 会话 |
| `GET` | `/api/v1/users/{username}` | 按用户名读取公开展示字段 | 会话 |
| `GET` | `/api/v1/users/active-and-inactive/{username}` | 包含软删除用户 | 超级用户 |
| `PATCH` | `/api/v1/users/{username}` | 更新自身或管理员可管理的资料 | 会话 |
| `DELETE` | `/api/v1/users/{username}` | 软删除自身或管理员可管理的用户 | 会话 |
| `DELETE` | `/api/v1/users/db/{username}` | GDPR 匿名化 | 超级用户 |
| `GET` | `/api/v1/users/{username}/rate-limits` | 用户等级对应的限流 | 会话 |
| `GET` | `/api/v1/users/{username}/tier` | 等级详情 | 会话 |
| `PATCH` | `/api/v1/users/{username}/tier` | 修改等级 | 超级用户 |

`/api/v1/auth/` 接口见[会话](sessions.md)。

<a id="registration"></a>

## 注册

`POST /api/v1/users/` 开放，无需认证，任何人可注册。

```bash
curl -X POST http://localhost:8000/api/v1/users/ \
  -H "Content-Type: application/json" \
  -d '{
    "name": "John Doe",
    "username": "johndoe",
    "email": "john@example.com",
    "password": "Str1ngst!"
  }'
```

**每个被接受的注册返回相同内容**：`201`、固定消息、不包含账户信息。已有邮箱与未占用邮箱响应一致，已有账户会收到通知，避免接口枚举注册邮箱。**用户名**已占用仍返回 `409`，因为用户名可通过公开资料 URL 发现，注册表单也需要明确反馈。响应不返回新资料，否则会泄露邮箱是否可用。登录后通过 `GET /api/v1/users/{username}` 读取。

路由委托 `UserService.create`：

1. 查找凭据前，先通过 `get_password_hash_async` 执行 bcrypt。它是注册最慢的步骤，只对可用邮箱执行会造成时间侧信道。
2. 检查邮箱唯一性，重复抛 `EmailAlreadyRegisteredError`，路由返回固定消息，通知邮件交后台发送，不等待邮件服务器。邮箱去空白、转小写存储，`ix_user_email_lower` 在绕过服务写入时仍保证大小写不重复。
3. 检查用户名唯一性，重复抛 `UserExistsError`，映射 409。
4. 构造以 `hashed_password` 代替 `password` 的 `UserCreateInternal`。
5. 通过 `crud_users.create` 写入。并发注册或旧格式邮箱导致数据库唯一约束拒绝时，以 asyncpg 错误中完整索引名裁决：`ix_user_email` / `ix_user_email_lower` 返回固定消息，`ix_user_username` 返回 409。完整匹配避免后来相似名称的索引被误认。

```python
# modules/user/service.py
async def create(self, user: UserCreate, db: AsyncSession) -> dict[str, Any]:
    if await crud_users.exists(db=db, email=user.email):
        raise EmailAlreadyRegisteredError("Email already registered")
    if await crud_users.exists(db=db, username=user.username):
        raise UserExistsError("Username already taken")

    payload = user.model_dump()
    payload["hashed_password"] = get_password_hash(payload.pop("password"))
    user_internal = UserCreateInternal(**payload)

    try:
        return await crud_users.create(db=db, object=user_internal, schema_to_select=UserRead)
    except IntegrityError as collision:
        await db.rollback()
        raise UserExistsError("Email or username already taken") from collision
```

`UserCreate` 执行输入校验：

```python
class UserCreate(UserBase):
    model_config = ConfigDict(extra="forbid")

    password: Annotated[
        str,
        Field(
            min_length=8,
            examples=["Str1ngst!"],
        ),
    ]

    @field_validator("password")
    def validate_password_strength(cls, v: str) -> str:
        """Reject a password missing any of the four character classes."""
        ...
    # OAuth fields (filled when user signs up via Google)
    google_id: str | None = None
    github_id: str | None = None
    oauth_provider: str | None = None
```

`extra="forbid"` 拒绝客户端未知字段。

<a id="authentication"></a>

## 身份认证

通过 `POST /api/v1/auth/login` 登录，完整流程见[会话](sessions.md)。路由委托 `infrastructure/auth/setup.py` 的 crudauth `auth` 单例查用户、验密码和建会话，不再提供 `authenticate_user` 辅助函数。

- 同一字段接受**用户名或邮箱**。
- **软删除用户不能登录**：crudauth 读取 `User.is_active = not is_deleted`，密码检查前拒绝停用账户。

<a id="password-hashing-bcrypt"></a>

### 密码哈希（bcrypt）

辅助函数来自 crudauth：

```python
from crudauth import get_password_hash, verify_password

hashed = get_password_hash("plaintext")          # bcrypt, salt generated automatically
ok = await verify_password("plaintext", hashed)  # constant-time compare
```

bcrypt 自动加盐并提供较高计算成本，增加大规模暴力破解难度。本节原文称 `UserService.create` 持久化新账户时使用 `get_password_hash`，[注册](#registration)流程上文则写异步版本。

<a id="profile-operations"></a>

## 资料操作

<a id="get-current-user"></a>

### 获取当前用户

```bash
curl http://localhost:8000/api/v1/users/me -b cookies.txt
```

`get_current_user` 已返回用户字典，路由很简单：

```python
@router.get("/me", response_model=UserRead)
async def get_current_user_profile(
    current_user: Annotated[dict[str, Any], Depends(get_current_user)],
) -> dict[str, Any]:
    return current_user
```

<a id="get-user-by-username"></a>

### 按用户名获取用户

本节原文描述为无需认证的公开接口，并过滤软删除用户。

> 译注：与本页接口速览中要求会话的描述不一致。

```bash
curl http://localhost:8000/api/v1/users/johndoe
```

不存在或软删除返回 404。管理员 `/active-and-inactive/{username}` 也能返回软删除用户。

<a id="update-profile"></a>

### 更新资料

用户可修改自身资料。带 scope 的凭据，包括 API 密钥，也需 `user.update`，避免只读密钥重命名账户。修改他人需超级用户或 `user.update`，等级更新走独立接口，见[权限](permissions.md)。

```bash
curl -X PATCH http://localhost:8000/api/v1/users/johndoe \
  -b cookies.txt \
  -H "Content-Type: application/json" \
  -H "X-CSRF-Token: <token>" \
  -d '{"name": "John Updated"}'
```

服务执行规则：

```python
# modules/user/service.py
async def verify_update_permission(
    self,
    requester_user: dict[str, Any],
    target_username: str,
    permissions: Collection[str],
    *,
    is_superuser: bool,
    scoped: bool,
) -> None:
    if not scoped and self.is_self_or_superuser(requester_user, target_username, is_superuser=is_superuser):
        return

    if UserPermission.UPDATE.value in permissions:
        return

    raise PermissionDeniedError("You don't have permission to update profile on this user")
```

非超级用户的 `user.update` 持有者还需通过 `verify_no_privilege_escalation`：目标不是超级用户，且权限不超出调用者，见[资源归属](permissions.md#resource-ownership)。修改 `username` 时重新检查唯一性。

<a id="changing-your-email-address"></a>

#### 修改邮箱

不在资料接口修改，传入 `email` 返回 `422`。账户通过确认流程变更：发送新地址确认链接，使用后通知旧地址：

```bash
curl -X POST http://localhost:8000/api/v1/auth/email/change-request \
  -b cookies.txt \
  -H "Content-Type: application/json" \
  -H "X-CSRF-Token: <token>" \
  -d '{"new_email": "new@example.com", "password": "<current password>"}'
```

确认会结束全部会话，包括发起会话。邮箱是恢复凭据，确认来自新邮箱链接，不携带可保留会话，所有者需用新地址重新登录。

见[恢复流程](index.md#recovery-flows-email)。后台表单基于 `UserAdminUpdate`，包含邮箱、OAuth 标识和验证标记，管理员可直接修正，无需所有者确认。

`UserUpdate` 全部字段可选，支持部分更新；邮箱、OAuth 标识、验证标记不在其中，提交返回 422：

```python
from .constants import (
    NAME_MAX_LENGTH,
    PROFILE_IMAGE_URL_MAX_LENGTH,
    USERNAME_MAX_LENGTH,
    USERNAME_PATTERN,
)


class UserUpdate(BaseModel):
    model_config = ConfigDict(extra="forbid")

    name: Annotated[str | None, Field(min_length=2, max_length=NAME_MAX_LENGTH, default=None)]
    username: Annotated[
        str | None,
        Field(min_length=2, max_length=USERNAME_MAX_LENGTH, pattern=USERNAME_PATTERN, default=None),
    ]
    profile_image_url: Annotated[
        str | None,
        Field(
            max_length=PROFILE_IMAGE_URL_MAX_LENGTH,
            pattern=r"^(https?|ftp)://[^\s/$.?#].[^\s]*$",
            default=None,
        ),
    ]
```

头像会提供给读取资料的已登录账户，因此限制为 URL 长度 2048。OAuth 提供方头像也受同样限制，超长则忽略而非写入。

<a id="deletion"></a>

## 删除

模板区分三种删除方式。

<a id="soft-delete"></a>

### 软删除

`DELETE /api/v1/users/{username}` 设置 `is_deleted=True` 和 `deleted_at=now()`，保留数据库行和数据，用户无法再登录。

```bash
curl -X DELETE http://localhost:8000/api/v1/users/johndoe \
  -b cookies.txt \
  -H "X-CSRF-Token: <token>"
```

规则：

- 用户可软删除自身。
- 超级用户可软删除任何人。
- 必须通过会话登录，拒绝 API 密钥。
- 设置 `FRESH_SIGN_IN_SECONDS` 时，登录时间必须在窗口内；未设置则不要求，见[近期重新登录](sessions.md#asking-for-a-fresh-sign-in)。

<a id="hard-delete-database"></a>

### 数据库硬删除

有意不提供公开硬删除接口，直接删除 `user` 行可能影响会话、API 密钥等关联数据。确有需要时，从脚本或管理任务调用 `crud_users.db_delete(...)`，并理解外键影响。

<a id="gdpr-anonymization"></a>

### GDPR 匿名化

`DELETE /api/v1/users/db/{username}` 仅超级用户可用。替换个人信息为中性值，保留行和外键关系。

```bash
curl -X DELETE http://localhost:8000/api/v1/users/db/johndoe \
  -b superuser_cookies.txt \
  -H "X-CSRF-Token: <token>"
```

服务实现：

```python
async def anonymize_user(self, user_id: int, db: AsyncSession) -> None:
    anonymize_data = UserAnonymize(
        name="[DELETED]",
        username=f"deleted-{user_id}-{uuid.uuid4().hex[:12]}",
        hashed_password="DELETED_INVALID_HASH",
        profile_image_url="https://deleted.com/deleted.jpg",
        tier_id=None,
        is_superuser=False,
        google_id=None,
        github_id=None,
        oauth_provider=None,
        email_verified=False,
        oauth_created_at=None,
        oauth_updated_at=None,
    )
    # anonymize the row, then soft-delete it
    await crud_users.update(db=db, object=anonymize_data, commit=False, id=user_id)
    await crud_users.delete(db=db, id=user_id)
```

匿名用户名含连字符，不符合注册和改名的 `USERNAME_PATTERN`，普通用户无法提前占用。若管理员通过后台占用了名称，则生成新十六进制值重试，最多 `ANONYMIZATION_ATTEMPTS`。失败返回 500，不误报成功。只对用户名索引拒绝重试，其余错误直接抛出。

原文说明有意保留邮箱用于审计和法律记录；这是一项原模板的数据保留设计。

保留邮箱也意味着**该地址退役**：

- 再次注册不创建账户，返回通用消息；软删除行和 `ix_user_email_lower` 仍占用地址。
- 验证或重置请求不发邮件，与不存在账户响应一致，因为账户 inactive。
- OAuth 登录返回 `account_inactive`，不关联账户，不能认领退役行。

其他人不会继承地址，也不会向其投递。如果业务要求同地址重新注册，应在请求处理中自行清空旧行邮箱，此后匿名化行不再记录其原身份。

<a id="administrative-operations"></a>

## 管理操作

<a id="list-all-users"></a>

### 列出全部用户

`GET /api/v1/users/` 分页，要求 `dependencies=[require_permissions("user.read")]`。由角色授权，超级用户始终通过，见[角色权限](permissions.md#role-based-permissions)。

```bash
curl "http://localhost:8000/api/v1/users/?page=1&items_per_page=10" \
  -b cookies.txt
```

`paginated_response` 响应：

```json
{
  "data": [
    { "id": 1, "name": "Admin User", "username": "admin", "email": "admin@example.com", ... }
  ],
  "total_count": 42,
  "has_more": true,
  "page": 1,
  "items_per_page": 10
}
```

完整模式见[分页](../api/pagination.md)。

<a id="view-a-users-tier"></a>

### 查看用户等级

```bash
curl http://localhost:8000/api/v1/users/johndoe/tier -b cookies.txt
```

返回用户与等级关联结果，仅本人或超级用户可读。

<a id="change-a-users-tier"></a>

### 修改用户等级

`PATCH /api/v1/users/{username}/tier` 仅超级用户：

```bash
curl -X PATCH http://localhost:8000/api/v1/users/johndoe/tier \
  -b superuser_cookies.txt \
  -H "Content-Type: application/json" \
  -H "X-CSRF-Token: <token>" \
  -d '{"tier_id": 2}'
```

服务先验证等级存在。`tier_id` 超出 `1..2147483647` 返回 422，避免超出列范围。

<a id="view-a-users-rate-limits"></a>

### 查看用户限流

```bash
curl http://localhost:8000/api/v1/users/johndoe/rate-limits -b cookies.txt
```

返回用户等级的限流配置，仅本人或超级用户可读。

<a id="user-model-reference"></a>

## 用户模型参考

实际位于 `modules/user/models.py`，节选：

```python
class User(Base, TimestampMixin, SoftDeleteMixin):
    __tablename__ = "user"
    __table_args__ = (
        Index("ix_user_email_lower", text("lower(email)"), unique=True),
    )

    id: Mapped[int] = mapped_column(
        "id", autoincrement=True, nullable=False,
        primary_key=True, init=False,
    )
    name: Mapped[str] = mapped_column(String(30))
    username: Mapped[str] = mapped_column(String(32), unique=True, index=True)
    email: Mapped[str] = mapped_column(String(50), unique=True, index=True)
    hashed_password: Mapped[str] = mapped_column(String(100))
    profile_image_url: Mapped[str] = mapped_column(
        String(2048), default="https://profileimageurl.com",
    )

    tier_id: Mapped[int | None] = mapped_column(
        Integer, ForeignKey("tiers.id"), index=True, default=None,
    )
    tier: Mapped["Tier | None"] = relationship(
        "Tier", back_populates="users", lazy="selectin", init=False,
    )

    is_superuser: Mapped[bool] = mapped_column(default=False)

    # OAuth (filled when user signs in via Google/GitHub)
    google_id: Mapped[str | None] = mapped_column(String(50), unique=True, index=True, default=None)
    github_id: Mapped[str | None] = mapped_column(String(50), unique=True, index=True, default=None)
    oauth_provider: Mapped[str | None] = mapped_column(String(20), default=None)
    email_verified: Mapped[bool] = mapped_column(default=False)

    @property
    def is_active(self) -> bool:
        # Derived, not stored. is_deleted stays the single source of truth.
        return not self.is_deleted
```

!!! note "已有数据库"
    `username` 为 32 字符，以容纳 crudauth 生成的 OAuth 用户名。旧版 20 字符列在手动修改前仍保持原长度：

    ```sql
    ALTER TABLE "user" ALTER COLUMN username TYPE VARCHAR(32);
    ```

    使用 Alembic 时，`alembic revision --autogenerate` 可发现该变化。

`infrastructure/database/models` 提供 mixin：

- `TimestampMixin`：`created_at`、`updated_at`。
- `SoftDeleteMixin`：`is_deleted`、`deleted_at`。

`is_active` **派生自** `not is_deleted`，没有独立列，crudauth 据此拒绝软删除用户；状态以 `is_deleted` 为唯一来源。

表名为单数 **`user`**。

<a id="common-crud-tasks"></a>

## 常见 CRUD 操作

[CRUD 操作](../database/crud.md)中的 FastCRUD 用法同样适用于用户：

```python
from src.modules.user.crud import crud_users

# Existence checks
await crud_users.exists(db=db, email="user@example.com")
await crud_users.exists(db=db, username="johndoe")

# Counts
total_active = await crud_users.count(db=db, is_deleted=False)
admin_count = await crud_users.count(db=db, is_superuser=True)

# Filtered queries
result = await crud_users.get_multi(db=db, tier_id=1, is_deleted=False, limit=20)

# Search by username substring
result = await crud_users.get_multi(db=db, username__icontains="ad")
```

<a id="frontend-integration"></a>

## 前端集成

默认采用 Cookie 而非 bearer，设置 `credentials: 'include'` 后浏览器自动携带会话 Cookie：

```javascript
class UserClient {
    constructor(baseUrl = '/api/v1') {
        this.baseUrl = baseUrl;
        this.csrfToken = null;
    }

    async register(userData) {
        const res = await fetch(`${this.baseUrl}/users/`, {
            method: 'POST',
            headers: { 'Content-Type': 'application/json' },
            body: JSON.stringify(userData),
        });
        if (!res.ok) throw new Error((await res.json()).detail);
        return await res.json();
    }

    async login(username, password) {
        const res = await fetch(`${this.baseUrl}/auth/login`, {
            method: 'POST',
            credentials: 'include',
            headers: { 'Content-Type': 'application/x-www-form-urlencoded' },
            body: new URLSearchParams({ username, password }),
        });
        if (!res.ok) throw new Error((await res.json()).detail);
        const { csrf_token } = await res.json();
        this.csrfToken = csrf_token;
        return csrf_token;
    }

    async getProfile() {
        const res = await fetch(`${this.baseUrl}/users/me`, {
            credentials: 'include',
        });
        if (!res.ok) throw new Error('Failed to get profile');
        return await res.json();
    }

    async updateProfile(username, updates) {
        const res = await fetch(`${this.baseUrl}/users/${username}`, {
            method: 'PATCH',
            credentials: 'include',
            headers: {
                'Content-Type': 'application/json',
                'X-CSRF-Token': this.csrfToken,
            },
            body: JSON.stringify(updates),
        });
        if (!res.ok) throw new Error((await res.json()).detail);
        return await res.json();
    }

    async deleteAccount(username) {
        const res = await fetch(`${this.baseUrl}/users/${username}`, {
            method: 'DELETE',
            credentials: 'include',
            headers: { 'X-CSRF-Token': this.csrfToken },
        });
        if (!res.ok) throw new Error((await res.json()).detail);
        this.csrfToken = null;
        return await res.json();
    }

    async logout() {
        await fetch(`${this.baseUrl}/auth/logout`, {
            method: 'POST',
            credentials: 'include',
            headers: { 'X-CSRF-Token': this.csrfToken },
        });
        this.csrfToken = null;
    }
}
```

跨源 Cookie 发送和存储还需服务端 `CORS_ALLOW_CREDENTIALS=true` 与明确 `CORS_ORIGINS` 列表，不使用 `*`。

<a id="security-considerations"></a>

## 安全事项

<a id="server-side-validation"></a>

### 服务端校验

所有输入通过 Pydantic 在服务端校验，客户端校验仅改善体验，不能替代。

<a id="login-lockout"></a>

### 登录锁定

crudauth 自动按 IP 和标识逐级锁定，返回带 `Retry-After` 的 429。本节原文称没有环境变量可调整，与[会话](sessions.md#login-lockout)列出的 `LOGIN_*` 不一致。

<a id="generic-auth-error-messages"></a>

### 通用认证错误

用户名或密码错误均返回 `Incorrect username or password`，不区分。

<a id="soft-delete-for-accounts"></a>

### 账户软删除

默认删除接口软删除。原文将 GDPR 类请求描述为通过匿名化保留外键完整性，而不是直接移除用户行。

<a id="next-steps"></a>

## 后续步骤

1. **[权限](permissions.md)**：角色访问控制。
2. **[会话](sessions.md)**：Cookie、CSRF、生命周期。
3. **[生产指南](../production.md)**：安全加固。
