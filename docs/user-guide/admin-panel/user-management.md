<a id="admin-user-management"></a>

# 后台用户管理

后台认证有意保持简单：环境变量定义一个管理员，使用 Starlette 会话 Cookie，没有管理员表或现成多操作员流程。本页介绍取舍、加固和扩展。

> 译注：原文称 Cookie 被 `SECRET_KEY` 加密，但配置专章说明实际为签名，内容可读。

<a id="how-it-works"></a>

## 工作方式

`interfaces/admin/auth.py` 比较提交值与 `ADMIN_USERNAME`、`ADMIN_PASSWORD`：

```python
class AdminAuth(AuthenticationBackend):
    async def login(self, request: Request) -> bool:
        form = await request.form()
        username = form.get("username")
        password = form.get("password")

        settings = get_settings()
        if username == settings.ADMIN_USERNAME and password == settings.ADMIN_PASSWORD:
            request.session.update({"admin_authenticated": True})
            return True
        return False
```

成功后在 Starlette 会话中设置 `admin_authenticated=True`，后续请求检查。没有管理员模型、独立密码哈希列或管理员表，凭据来自环境。

<a id="initial-setup"></a>

## 初始配置

在 `backend/.env`：

```env
ADMIN_USERNAME=admin
ADMIN_PASSWORD=Change-Me-Now-1!
SECRET_KEY=<openssl rand -hex 32>
```

重启后访问 <http://localhost:8000/admin> 登录。

同一组 `ADMIN_*` 也被 `scripts/setup_initial_data.py` 用于创建**首个应用超级用户**，除此之外两个系统独立。

<a id="two-separate-user-systems"></a>

## 两套独立用户系统

| 项目 | 后台登录 | 应用用户 |
|------|----------|----------|
| 身份 | SQLAdmin 操作员 | 应用终端用户 |
| 存储 | 环境变量 | 数据库 `user` 表 |
| 认证 | 与 `ADMIN_PASSWORD` 比较 | 验证 bcrypt 密码后创建会话 |
| 多账户 | 不支持，仅一个用户名 | 每用户一行 |
| 使用位置 | 仅 `/admin` | `/api/v1/*`，以及后台展示用户模型 |
| 登录 URL | `/admin/login` | `/api/v1/auth/login` |

数据库 `is_superuser=true` 可调用超级用户 API，如匿名化接口，但除非凭据恰好匹配环境值，否则不能登录后台，两者不共享认证状态。

<a id="managing-application-users-via-the-admin"></a>

## 通过后台管理应用用户

登录 `/admin` 后：

- **Users** 创建、编辑、删除 `user` 表记录。
- **Tiers** 管理等级及名称、描述。
- 密码通过 `on_model_change` 自动哈希。
- 编辑表单可直接修改 `is_superuser`。

这是为已有用户授予超级用户身份的简便方式。

<a id="hardening-for-production"></a>

## 生产加固

<a id="option-1-disable-in-production"></a>

### 方案一：禁用

不在生产暴露后台：

```env
ADMIN_ENABLED=false
```

`create_admin_interface()` 提前返回，不挂载路由。管理操作使用脚本或数据库工具。

<a id="option-2-network-restrict-the-path"></a>

### 方案二：网络限制

保持启用，在负载均衡或反向代理仅允许 VPN / 办公网段访问 `/admin/*`。应用不变，网络阻止公网访问，适合偶尔使用且不想新增认证逻辑的情况。

<a id="option-3-strong-credentials-tls"></a>
<a id="option-3-strong-credentials--tls"></a>

### 方案三：强凭据与 TLS

必须开放互联网时：

- 使用长高熵 `ADMIN_PASSWORD`，部署时从密钥管理器获取。
- 代理终止 HTTPS。
- HTTPS 使用安全 Cookie，见[配置](configuration.md#session-cookies)。
- 定期轮换密码，需要重新部署。

生产校验器拒绝后台启用却缺少用户名密码的配置，对可预测用户名、短或常见密码告警，但不能完整判断强度。

<a id="recovering-from-a-lost-admin-password"></a>

## 管理员密码遗失

1. 修改 `backend/.env`、密钥管理器或编排配置中的 `ADMIN_USERNAME` / `ADMIN_PASSWORD`。
2. 重启应用。

无需修复数据库，也没有邮件重置，因为它不属于用户自助凭据。

<a id="when-you-need-multiple-admins"></a>

## 多管理员需求

单凭据适合个人或小团队，需要真实多操作员时可选：

<a id="option-a-use-application-superusers-a-dedicated-admin-route"></a>
<a id="option-a-use-application-superusers--a-dedicated-admin-route"></a>

### A：使用应用超级用户

自定义 `AuthenticationBackend`，仅允许已认证且 `is_superuser=true` 的应用用户进入后台：

```python
# interfaces/admin/auth.py
from sqladmin.authentication import AuthenticationBackend
from starlette.requests import Request

# Pseudocode — wire to your existing session backend
class AdminAuth(AuthenticationBackend):
    async def login(self, request: Request) -> bool:
        # Reuse your /api/v1/auth/login flow:
        # validate credentials, look up the user, check is_superuser
        ...

    async def authenticate(self, request: Request) -> bool:
        # Read the app's session_id cookie, validate it, confirm is_superuser
        ...
```

代价是所有应用超级用户均可登录，且需要考虑 CSRF：应用采用双重提交，而 SQLAdmin 单独提交表单。

<a id="option-b-add-an-adminuser-model"></a>

### B：新增 `AdminUser` 模型

包含 `username`、`hashed_password`、`is_active`，覆盖 `AdminAuth.login` 查询，提供一次性脚本初始化。

<a id="option-c-external-auth-oidc-saml"></a>
<a id="option-c-external-auth-oidc--saml"></a>

### C：外部认证（OIDC / SAML）

大型组织可把后台放在 Authelia、Pomerium、AWS ALB + Cognito 等 SSO 代理后，后台信任代理提供的认证头。

这些方案都未内置，应按环境实现。扩展参考 [SQLAdmin 认证文档](https://github.com/smithyhq/sqladmin)。

<a id="auditing-admin-activity"></a>

## 审计后台活动

默认不记录每项后台操作，需要审计时：

- 使用已有日志与关联 ID，SQLAdmin 请求也经过相同链路。
- 覆盖视图的 `on_model_change`、`after_model_change`、`delete_model` 显式记录：

```python
from src.infrastructure.logging import get_logger

logger = get_logger()


class UserAdmin(DataclassModelMixin, ModelView, model=User):
    async def on_model_change(self, data, model, is_created, request):
        action = "created" if is_created else "updated"
        logger.info(f"Admin {action} user", extra={"user_id": data.get("id"), "actor": "admin"})
```

更完整的审计可写独立日志流或 SIEM。

<a id="key-files"></a>

## 关键文件

| 组件 | 位置 |
|------|------|
| 认证后端 | `backend/src/interfaces/admin/auth.py` |
| 应用工厂 | `backend/src/interfaces/admin/initialize.py` |
| 配置类 | `backend/src/infrastructure/config/settings.py` 的 `AdminSettings`、`SQLAdminSettings` |
| 初始数据脚本 | `backend/scripts/setup_initial_data.py` |

<a id="next-steps"></a>

## 后续步骤

- **[配置](configuration.md)**：环境变量和 Cookie。
- **[添加模型](adding-models.md)**：注册视图。
- **[权限](../authentication/permissions.md)**：应用角色与超级用户。
- **[生产环境](../production.md)**：加固清单。
