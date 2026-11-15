<a id="admin-panel"></a>

# 管理后台

模板内置基于 [SQLAdmin](https://github.com/smithyhq/sqladmin) 的后台，可通过 Web 浏览和编辑数据库，无需另写 CRUD 接口。

后台属于 `admin` 功能。移除后没有 `/admin`、`wiring/admin.py` 和管理视图；各功能的视图也随功能移除，见[可组合功能](../composable-features.md)。

!!! tip "构建完整 SaaS？"
    后台属于免费基础功能。**[FastroAI](https://fastro.ai)** 将其与 Stripe 支付、权益管理、事务邮件、前端及 AI Agent 整合。原文将其定位为可直接用于生产的完整组合。[更快交付 SaaS →](https://fastro.ai)

<a id="accessing-the-admin-panel"></a>

## 访问后台

默认挂载 `/admin`，需要显式开启，未配置登录凭据时不会挂载：

```env
ADMIN_ENABLED=true   # unset or false leaves /admin unmounted
ADMIN_USERNAME=admin
ADMIN_PASSWORD=Change-Me-Now-1!
```

后台认证**独立于应用会话认证**，从环境变量读取用户名和密码：

```env
ADMIN_USERNAME=admin
ADMIN_PASSWORD=Change-Me-Now-1!
SECRET_KEY=<used for admin session encryption>
```

访问 <http://localhost:8000/admin> 并输入凭据。

!!! warning "配置凭据前无法登录"
    `ADMIN_USERNAME` 和 `ADMIN_PASSWORD` 默认空。任一未设置，所有登录尝试均失败，空表单也不能认证。

<a id="what-youll-learn"></a>

## 内容概览

- **[配置](configuration.md)**：环境变量和部署。
- **[添加模型](adding-models.md)**：注册自定义模型。
- **[用户管理](user-management.md)**：后台认证和安全。

<a id="whats-included"></a>

## 包含的视图

各功能自带视图，由 `src/wiring/admin.py` 列出，默认五个：

| 视图 | 来源 | 说明 |
|------|------|------|
| **Users** | `modules/user/admin.py` | 增删改用户，自动密码哈希，感知软删除；仅 tiers 功能提供等级列时展示 |
| **Tiers** | `modules/tier/admin.py` | 管理等级，仅列举和统计未删除记录；调用 `TierService.permanent_delete` 避免遗留用户或限流引用 |
| **Roles** | `modules/role/admin.py` | 创建、改名、删除角色，并列出权限 |
| **Role permissions** | `modules/role/admin.py` | 每行一个权限，字段通过注册表选择 |
| **User roles** | `modules/role/admin.py` | 账户角色；列表、计数和账户选择器均排除软删除账户 |

均位于 **Users & Access** 分类。Users 和 Tiers 提供搜索、排序、过滤及 CSV 导出。三个角色视图随 rbac 提供，各视图随表所属功能提供，因此移除功能后数量减少。

需要 `RateLimit`、`APIKey` 等视图时，见[添加模型](adding-models.md)。

<a id="common-operations"></a>

## 常用操作

<a id="creating-a-user"></a>

### 创建用户

进入 **Users → Create**。`Password` 接受明文，`UserAdmin.on_model_change` 保存前调用 `get_password_hash()`，数据库只接收哈希。

<a id="editing-a-user"></a>

### 编辑用户

点击用户行再选 **Edit**，可改姓名、用户名、邮箱、等级、OAuth 标识和 `is_superuser`。编辑表单由 `UserAdminUpdate` 的 `form_edit_rules` 生成，不含密码，重置需使用 API 改密路由或直接写新哈希。

等级选择器只显示未软删除等级。保存时仍引用已删除等级会被拒绝，提示 `That tier has been deleted. Pick another one.`。

<a id="giving-an-account-a-role"></a>

### 为账户分配角色

先 **Roles → Create** 创建名称，再 **Role permissions → Create** 逐行添加权限。选择器只包含注册名称，绕过表单的写入也被模型校验。权限授予和角色分配都是关系行，没有其他内容可编辑，所以只提供创建和删除；更换权限时先删旧行再加新行。

**User roles → Create** 分配角色。列表、分页总数、账户选择器排除软删除账户，但 API `GET /api/v1/users/{user_id}/roles` 仍可读取旧分配。

!!! warning "后台不受普通委派检查约束"
    API 不允许授予比自己更强的权限或操作更强账户，见[权限](../authentication/permissions.md#managing-roles)。后台通过独立 `ADMIN_*` 登录，已能设置任意人的 `is_superuser`，没有较弱操作员可约束。能登录后台的人可授予任意注册权限。

<a id="deleting-a-tier"></a>

### 删除等级

调用 `TierService.permanent_delete`，仍有活跃用户或限流引用时**失败**，需先重新分配或移除依赖。永久删除会释放软删除用户的等级引用；`DELETE /api/v1/tiers/{name}` 软删除则保留，恢复时仍可找到原关系。

<a id="how-authentication-works"></a>

## 认证工作方式

后台通过 Starlette `SessionMiddleware` 使用独立会话：

1. `AdminAuth.login` 校验 `ADMIN_USERNAME` / `ADMIN_PASSWORD`。
2. 成功设置 `request.session["admin_authenticated"] = True`。
3. 后续请求检查标记。

后台面向少量可信操作员，因此比主应用简单。Cookie 由 `SECRET_KEY` **签名而非加密**，持有者可读内容，签名防篡改。路径限定为 `ADMIN_BASE_URL` 加请求 `root_path`，API 不接收它，后台会话也不能访问 API。

<a id="how-its-wired"></a>

## 组装方式

后台在 `src/interfaces/admin/initialize.py` 创建，启动时挂载：

```python
# interfaces/admin/initialize.py
from fastapi import FastAPI
from sqladmin import Admin

from ...infrastructure.config.settings import get_settings
from ...infrastructure.database.session import get_engine
from ...wiring.admin import ADMIN_VIEWS
from .auth import AdminAuth


def create_admin_interface(app: FastAPI) -> Admin | None:
    settings = get_settings()
    if not settings.ADMIN_ENABLED:
        return None

    admin = Admin(
        app=app,
        engine=get_engine(),
        authentication_backend=AdminAuth(secret_key=settings.SECRET_KEY),
        title="Admin",
    )
    for view in ADMIN_VIEWS:
        admin.add_view(view)
    return admin
```

admin 功能向 wiring 的安装器列表贡献 `install(app)`，由应用工厂调用，`main.py` 无需直接引用后台。`ADMIN_ENABLED=false` 不挂载。后台会话中间件只作用于自己的路由，API 不解码后台 Cookie。

<a id="disabling-in-production"></a>

## 在生产禁用

不希望生产开放后台时：

```env
ADMIN_ENABLED=false
```

也可启用，但在负载均衡或代理仅允许 VPN CIDR 访问 `/admin`。

<a id="key-files"></a>

## 关键文件

| 组件 | 位置 |
|------|------|
| 后台工厂 | `backend/src/interfaces/admin/initialize.py` |
| 认证后端 | `backend/src/interfaces/admin/auth.py` |
| dataclass 模型 mixin | `backend/src/interfaces/admin/mixins.py` |
| 用户视图 | `backend/src/modules/user/admin.py` |
| 等级视图 | `backend/src/modules/tier/admin.py` |
| 角色、权限、分配视图 | `backend/src/modules/role/admin.py` |
| 视图注册表 | `backend/src/wiring/admin.py` |

<a id="next-steps"></a>

## 后续步骤

1. **[配置](configuration.md)**：环境变量和部署选项。
2. **[添加模型](adding-models.md)**：注册自定义视图。
3. **[用户管理](user-management.md)**：强化生产登录。
