<a id="api-exception-handling"></a>

# API 异常处理

模板有意将异常分为两层：

1. 服务抛出的**领域异常**，位于 `modules/common/exceptions.py` 和各功能的 `exceptions.py`。
2. 应用响应使用的 **HTTP 异常**，位于 `infrastructure/http_exceptions.py`。

自动映射层连接两者，路由无需了解每个领域错误对应的 HTTP 状态。

<a id="domain-exceptions"></a>

## 领域异常

定义于 `backend/src/modules/common/exceptions.py`。服务抛出它们来描述**发生了什么问题**，而非如何转换为 HTTP。

| 异常 | 场景 |
|------|------|
| `DomainError` | 所有领域异常的基类 |
| `ResourceNotFoundError` | 请求记录不存在 |
| `ResourceExistsError` | 会违反唯一约束 |
| `ValidationError` | 输入不符合业务规则 |
| `PermissionDeniedError` | 当前用户不能执行操作 |
| `UserNotFoundError`，继承 `ResourceNotFoundError` | 用户查询失败 |
| `UserExistsError`，继承 `ResourceExistsError` | 用户名或邮箱重复 |
| `PersistenceError` | 调用者有权执行的写入未返回结果 |
| `TierNotFoundError`，位于 `modules/tier/exceptions.py` | 等级查询失败，继承 `ResourceNotFoundError` |
| `RateLimitNotFoundError`，位于 `modules/rate_limit/exceptions.py` | 限流记录缺失，继承 `ResourceNotFoundError` |

功能专属异常继承共享异常，并通过 `public_detail` 指定客户端消息，映射层无需列举具体功能错误：

```python
class UserExistsError(ResourceExistsError):
    public_detail = "A user with this email or username already exists."
```

```python
# modules/user/service.py
async def create(self, user: UserCreate, db: AsyncSession) -> dict[str, Any]:
    if await crud_users.exists(db=db, email=user.email):
        raise UserExistsError("Email already registered")

    if await crud_users.exists(db=db, username=user.username):
        raise UserExistsError("Username already taken")

    # ...
```

服务无需知道这会被转换为 HTTP `409 Conflict`，映射在其他位置完成。

<a id="http-exceptions"></a>

## HTTP 异常

由 `backend/src/infrastructure/http_exceptions.py` 从 FastCRUD 重新导出：

| 异常 | 状态 |
|------|------|
| `BadRequestException` | 400 |
| `UnauthorizedException` | 401 |
| `ForbiddenException` | 403 |
| `NotFoundException` | 404 |
| `ConflictException` | 409 |
| `DuplicateValueException` | FastCRUD 中为 422；映射层使用 `ConflictException` |
| `UnprocessableEntityException` | 422 |
| `RateLimitException` | 429 |
| `HTTPException` | FastAPI 基类 |

路由中出现无需服务参与的 HTTP 错误时使用，服务领域错误无需路由处理：

```python
from ...infrastructure.http_exceptions import BadRequestException

@router.get("/")
async def search(q: str | None = None):
    if q is None:
        raise BadRequestException("Provide ?q=")
    # ...
```

<a id="the-mapping-layer-centralized"></a>

## 集中映射层

`modules/common/utils/error_handler.py` 统一连接领域错误和 HTTP 错误。

<a id="global-handler-automatic"></a>

### 自动全局处理器

`infrastructure/app_factory.py` 启动时调用 `register_exception_handlers(app)`，安装：

- `RequestValidationError` 处理器：将 Pydantic 422 转换为通用 `Invalid request` 消息及 `support_id`。
- `DomainError` 处理器：通过 `EXCEPTION_MAPPING` 映射正确状态和消息，返回消息及 `support_id`。原始异常仅写服务端日志，不发送给客户端。
- `CatchAllErrorMiddleware`：未处理异常转换为带 `support_id` 的 500。

`infrastructure/middleware.py` 的 `NulCharacterMiddleware` 在路由前拒绝路径或查询中的 NUL 字符，返回相同校验响应。`%00` 解码后进入查询会被 Postgres 拒绝，过去因此返回 500。日志记录原因和路径，响应仅使用通用 422 内容。

请求体由继承 `EncodableText` 的 Schema 拒绝。crudauth 路由使用库拥有的模型，改由 `body_a_database_can_hold` 拒绝 JSON 中的孤立代理字符或 NUL，并由 `a_redirect_a_link_can_carry` 拒绝无法放入链接的 `redirect_to`，均返回 422。

因此，**服务中未捕获的任意 `DomainError` 都会自动成为结构正确的 HTTP 响应**。路由无需用 try/except 包裹服务：

```python
@router.post("/", response_model=UserRead, status_code=201)
async def create_user(
    user: UserCreate,
    db: AsyncSessionDep,
    user_service: UserServiceDep,
) -> dict[str, Any]:
    return await user_service.create(user, db)
```

服务抛出 `UserExistsError` 时，客户端得到 409、`"A user with this email or username already exists."` 和 `support_id`；意外错误以相同方式返回通用 500。注册路由是例外：它捕获 `EmailAlreadyRegisteredError`，返回与注册成功相同的确认，避免泄露哪些邮箱已注册，见[注册](../authentication/user-management.md#registration)。

<a id="the-default-mapping"></a>

### 默认映射

`modules/common/constants.py` 只列举共享异常：

```python
EXCEPTION_MAPPING: dict[type[DomainError], Callable[[str], HTTPException]] = {
    ResourceNotFoundError: lambda message: NotFoundException(detail="The requested resource was not found."),
    ResourceExistsError: lambda message: ConflictException(detail="This resource already exists."),
    ValidationError: lambda message: UnprocessableEntityException(detail="The request could not be processed."),
    PermissionDeniedError: lambda message: ForbiddenException(detail="You don't have permission for this action."),
    PersistenceError: lambda message: HTTPException(status_code=500, detail=GENERIC_ERROR_MESSAGE),
}
```

`map_exception` 遍历异常的 MRO，使用首个匹配项，因此功能子类继承父类状态码。消息使用映射中的固定文本，异常设置 `public_detail` 时则使用该值。不会直接返回抛出的消息，因为它可能涉及调用者无权得知的记录。原始信息连同响应中的 `support_id` 写日志，方便对应查询。

没有路由级转换辅助函数。路由要么让异常传播，要么只捕获需要特殊处理的领域错误：

```python
@router.get("/{name}/limits")
async def get_tier_limits(name: str, db: AsyncSessionDep, tier_service: TierServiceDep) -> dict[str, Any]:
    try:
        tier = await tier_service.get_by_name(name, db)
    except TierNotFoundError:
        return DEFAULT_LIMITS          # an unknown tier falls back instead of failing

    return tier["limits"]
```

<a id="response-format"></a>

## 响应格式

<a id="standard-error"></a>

### 标准错误

```json
{
  "detail": "User not found.",
  "support_id": "a1b2c3d4"
}
```

<a id="validation-error-422"></a>

### 校验错误（422）

```json
{
  "detail": "Invalid request. Please check your input and try again.",
  "support_id": "a1b2c3d4"
}
```

<a id="common-patterns"></a>

## 常见模式

<a id="check-before-create"></a>

### 创建前检查

```python
# Service method — domain layer
async def create(self, user: UserCreate, db: AsyncSession) -> dict[str, Any]:
    if await crud_users.exists(db=db, email=user.email):
        raise UserExistsError("Email already registered")
    if await crud_users.exists(db=db, username=user.username):
        raise UserExistsError("Username already taken")
    # ...
```

路由无需感知，`UserExistsError` 自动变为 409。

<a id="permission-check"></a>

### 权限检查

```python
async def update_profile(
    self, current_user: dict, target_username: str, values: UserUpdate, db: AsyncSession,
) -> None:
    if current_user["username"] != target_username and not current_user["is_superuser"]:
        raise PermissionDeniedError("You can only update your own profile")
    # ...
```

<a id="resource-lookup"></a>

### 资源查询

```python
async def get_by_username(self, username: str, db: AsyncSession) -> dict[str, Any]:
    user = await crud_users.get(db=db, username=username, is_deleted=False)
    if user is None:
        raise UserNotFoundError(f"User '{username}' not found")
    return user
```

<a id="direct-http-for-non-domain-failures"></a>

### 非领域错误直接抛 HTTP 异常

缺少某种查询参数组合等没有领域含义的错误，直接抛 HTTP 异常：

```python
from ...infrastructure.http_exceptions import BadRequestException


@router.get("/")
async def search(
    q: str | None = None,
    tag: str | None = None,
):
    if q is None and tag is None:
        raise BadRequestException("Provide either ?q= or ?tag=")
    # ...
```

<a id="adding-a-custom-domain-exception"></a>

## 添加自定义领域异常

1. 在 `modules/common/exceptions.py` **定义异常**：

    ```python
    class WidgetExceededError(DomainError):
        """Raised when a user tries to create more widgets than their tier allows."""
        pass
    ```

2. 在 `modules/common/constants.py` **添加映射**：

    ```python
    from .exceptions import WidgetExceededError

    EXCEPTION_MAPPING = {
        # ...existing entries...
        WidgetExceededError: lambda m: HTTPException(
            status_code=403, detail="You've hit your widget limit"
        ),
    }
    ```

3. **在服务中抛出**：

    ```python
    raise WidgetExceededError("Free tier limited to 10 widgets")
    ```

全局处理器自动识别新映射。已有映射异常的子类无需新增条目，只设置 `public_detail` 即可继承状态码。

<a id="adding-a-custom-http-exception"></a>

## 添加自定义 HTTP 异常

需要尚未导出的 HTTP 异常时，在 `infrastructure/http_exceptions.py` 添加：

```python
class PaymentRequiredException(HTTPException):
    """402 Payment Required."""

    def __init__(self, detail: str = "Payment required") -> None:
        super().__init__(status_code=402, detail=detail)
```

通过 `__all__` 导出，并在需要处导入。

<a id="security-considerations"></a>

## 安全事项

<a id="generic-messages-for-auth"></a>

### 认证使用通用消息

`infrastructure/auth/routes.py` 将凭据校验委托给 crudauth 的 `auth` 单例。`authenticate_password` 平衡校验耗时，凭据错误时抛出 `UnauthorizedException("Incorrect username or password")`，达到逐级锁定阈值后返回 429：

```python
# crudauth's hardened check: timing-equalized, disabled-account guard, escalating lockout
user = await auth.authenticate_password(db, form_data.username, form_data.password, request=request)
```

不区分“用户名不存在”和“密码错误”，避免泄露用户名存在性。

<a id="hide-resource-existence"></a>

### 隐藏资源存在性

用户不应知道某受保护资源存在时，返回 404 而非 403：

```python
post = await crud_posts.get(db=db, id=post_id)
if post is None:
    raise NotFoundException("Post not found")

if post["author_id"] != current_user["id"]:
    # 404, not 403 — don't reveal the post exists
    raise NotFoundException("Post not found")
```

<a id="dont-leak-internal-details"></a>

### 不泄露内部细节

全局处理器已返回通用消息，并将真实错误连同 `support_id` 写日志。用户反馈问题时，可用编号检索日志。

<a id="testing-exceptions"></a>

## 异常测试

代码库使用 `pytest-asyncio` 和 FastAPI 的 `TestClient` 测试路由：

```python
@pytest.mark.asyncio
async def test_user_not_found(client: AsyncClient):
    resp = await client.get("/api/v1/users/not-a-user")
    assert resp.status_code == 404
    body = resp.json()
    assert body["detail"]
    assert "support_id" in body


@pytest.mark.asyncio
async def test_duplicate_email(client: AsyncClient):
    payload = {
        "name": "Test User",
        "username": "test1",
        "email": "test@example.com",
        "password": "Password123!",
    }
    await client.post("/api/v1/users/", json=payload)

    payload["username"] = "test2"  # different username, same email
    resp = await client.post("/api/v1/users/", json=payload)
    assert resp.status_code == 409
```

服务测试直接断言对应的 `DomainError`：

```python
@pytest.mark.asyncio
async def test_create_duplicate_user_raises(db_session, existing_user):
    service = UserService()
    with pytest.raises(UserExistsError):
        await service.create(
            UserCreate(name="...", username=existing_user["username"], email="x@x.com", password="..."),
            db_session,
        )
```

<a id="whats-next"></a>

## 后续步骤

- **[版本管理](versioning.md)**：版本策略。
- **[CRUD 操作](../database/crud.md)**：服务如何使用 CRUD。
- **[身份认证](../authentication/index.md)**：会话、OAuth 和 API 密钥。
