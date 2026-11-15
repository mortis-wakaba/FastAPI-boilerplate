<a id="project-structure"></a>

# 项目结构

代码库采用三层架构（**interfaces / infrastructure / modules**），并按业务功能**纵向划分模块**：每个功能的模型、Schema、CRUD、服务和路由集中在同一个目录。本指南介绍代码组织方式及新增代码的放置位置。

<a id="repository-root"></a>

## 仓库根目录

```text
fastapi-boilerplate/
├── backend/                  # Python project root (see below)
├── docs/                     # zensical documentation
├── .github/                  # CI workflows
├── README.md
└── LICENSE.md
```

Python 项目全部位于 `backend/`。将来添加前端时，可在同级创建 `frontend/`。

<a id="backend-layout"></a>

## 后端结构

```text
backend/
├── pyproject.toml            # Dependencies and tooling config
├── uv.lock                   # Locked dependency versions
├── Dockerfile                # Container image for the app
├── alembic.ini               # Alembic migration config
├── .env.example              # Reference for environment variables
├── migrations/               # Alembic migrations
│   ├── env.py
│   ├── script.py.mako
│   └── versions/
├── scripts/                  # Setup and maintenance scripts
│   ├── canonicalize_emails.py
│   ├── cleanup_api_key_json.py
│   ├── create_first_superuser.py
│   ├── create_first_tier.py
│   ├── create_tables.py
│   ├── prune_key_usage.py    # The usage prune, for a project with no worker to schedule it
│   ├── revoke_scrypt_api_keys.py
│   ├── seeders.py            # The seeders the selected features contribute
│   ├── setup_initial_data.py
│   └── unlock_account.py
├── src/                      # Application source (the three layers below)
└── tests/                    # Test suite (unit + integration)
```

<a id="configuration-files"></a>

### 配置文件

| 文件 | 用途 |
|------|------|
| `pyproject.toml` | 项目元数据、依赖（`[project]`）、ruff / mypy / pytest 配置 |
| `uv.lock` | 锁定精确依赖版本，实现可重复安装 |
| `Dockerfile` | 多阶段构建：导出依赖 → 基础镜像 → dev / prod / migrate 阶段 |
| `alembic.ini` | Alembic 配置，包括脚本位置和日志 |
| `.env.example` | 全部环境变量的说明与参考 |

<a id="the-three-layers-src"></a>

## 三层架构（`src/`）

```text
src/
├── interfaces/               # HOW the world talks to the app (HTTP, admin UI)
├── infrastructure/           # WHAT the app uses (DB, cache, auth, taskiq, config)
├── modules/                  # WHAT the app IS (vertical-slice feature modules)
└── wiring/                   # WHICH features this project selected
```

调用方向是 **interfaces → modules → infrastructure**：

- `interfaces` 挂载路由、中间件和管理界面。
- `modules` 表达业务功能，各模块独立完整。
- `infrastructure` 提供上层共用的基础设施。
- `wiring` 是组装入口，是唯一集中列出所有功能的位置。

除共享的 `common` 模块外，业务模块之间不直接导入。接口层不包含业务逻辑，基础设施层不感知具体业务功能。

<a id="srcinterfaces"></a>

### `src/interfaces/`

```text
interfaces/
├── main.py                   # FastAPI app instance + lifespan + middleware setup
├── api/
│   ├── __init__.py           # Mounts /api router
│   └── v1/
│       └── __init__.py       # Mounts /v1 + each module's router
└── admin/
    ├── initialize.py         # SQLAdmin setup (mounted at /admin by its installer)
    ├── auth.py               # Admin auth backend, with its own session middleware
    ├── settings.py           # The admin feature's settings mixin
    └── mixins.py
```

`main.py` 是入口，`uv run --no-sync fastapi dev src/interfaces/main.py` 从这里启动，并提供 `/health` 和 `/health/ready`。`v1/__init__.py` 按 `wiring/app.py` 中的列表挂载路由，不直接列举功能。每个功能的管理视图与功能放在一起，位于 `modules/<feature>/admin.py`。

<a id="srcinfrastructure"></a>

### `src/infrastructure/`

```text
infrastructure/
├── app_factory.py            # Builds the FastAPI app (CORS, GZip, middleware, lifespan)
├── middleware.py             # ClientCache, SecurityHeaders, etc.
├── config/                   # Settings + Pydantic-driven env loading
│   ├── settings.py
│   └── enums.py
├── composition.py            # The shapes features contribute (RouterMount, Lifecycle, …)
├── permissions.py            # The permission registry every module declares into
├── dependencies.py           # AsyncSessionDep — the deps every project has
├── database/                 # SQLAlchemy engine, session, base model, readiness
├── auth/                     # The accounts feature: crudauth setup, deps, route handlers
│   ├── setup.py              # The `auth = CRUDAuth(...)` singleton
│   ├── dependencies.py       # get_current_user / _superuser / _optional_user
│   ├── deps.py               # The annotated caller dependencies (CurrentUserDep, …)
│   ├── authorization.py      # require_permissions and the permission resolution
│   ├── routes.py             # /auth/login, /logout, /oauth, /check-auth
├── cache/                    # Redis/Memcached cache + decorator + readiness
│   └── backends/
├── ratelimit/                # The API throttle dependency
├── redis.py                  # Shared Redis clients injected into crudauth and the cache
├── handover.py               # Work run after the answer, which a shutdown drains
├── taskiq/                   # Async task queue (broker, its app lifecycle, worker and scheduler entry points)
├── security/                 # Production security validator
└── logging/                  # Centralized logging configuration
```

认证是各功能共同依赖的基础能力，因此 `infrastructure/auth/routes.py` 有意放在这里，而不是 `modules/auth/`。它仍是可以移除的功能：accounts 拥有的内容全部位于 `auth/` 和 `modules/user/`。

<a id="srcmodules-vertical-slice-features"></a>
<a id="srcmodules--vertical-slice-features"></a>

### `src/modules/`：按功能纵向划分

```text
modules/
├── common/                   # Cross-module shared schemas, exceptions, utils
│   ├── collisions.py         # Which constraint refused a row, read off the driver's error
│   ├── constants.py
│   ├── exceptions.py
│   ├── schemas.py
│   └── utils/
├── user/
│   ├── models.py             # SQLAlchemy User model
│   ├── schemas.py            # Pydantic UserCreate, UserRead, UserUpdate, etc.
│   ├── crud.py               # FastCRUD wrapper (crud_users)
│   ├── service.py            # Business logic (UserService)
│   ├── routes.py             # APIRouter with /users endpoints
│   ├── permissions.py        # UserPermission StrEnum (user.read, user.update, ...)
│   └── enums.py              # OAuthProvider, etc.
├── role/                     # RBAC: Role, RolePermission, UserRole
├── tier/                     # Subscription tiers (model + simple CRUD)
├── rate_limit/               # Per-tier rate limit definitions
└── api_keys/                 # API keys, key usage, key permissions
```

每个模块都**独立完整**，添加或移除时尽量减少对其他部分的影响。只有 `src/wiring/` 知道项目选择了哪些模块；`tools/removal_drill.py` 验证移除后项目确实仍可工作，见[可组合功能](composable-features.md)。

<a id="common-module-files"></a>

### 模块常见文件

| 文件 | 用途 |
|------|------|
| `models.py` | SQLAlchemy ORM 模型，即表结构 |
| `schemas.py` | Pydantic 请求与响应模型 |
| `crud.py` | 模型对应的 FastCRUD 实例 |
| `service.py` | 业务逻辑，编排 CRUD 调用并应用业务规则 |
| `routes.py` | 包含模块接口的 `APIRouter` |
| `permissions.py` | 可选：模块权限的 `StrEnum`，通过 `@register_permissions` 注册 |
| `enums.py` | 可选：模块需要的 StrEnum 类型 |
| `settings.py` | 可选：模块配置 mixin，由 `wiring/settings.py` 组合 |
| `contrib.py` | 可选：模块为其他模型添加的列和 Schema 字段 |
| `hooks.py` | 可选：模块为其他功能扩展点提供的内容 |
| `admin.py` | 可选：模块的 SQLAdmin 视图，在 `wiring/admin.py` 中列出 |

<a id="srcwiring-the-composition-root"></a>
<a id="srcwiring--the-composition-root"></a>

### `src/wiring/`：组装入口

```text
wiring/
├── app.py                    # ROUTER_MOUNTS, ROOT_ROUTERS, API_THROTTLE, LIFECYCLES, INSTALLERS, DOCS_GUARD
├── settings.py               # Each feature's settings mixin, core last
├── hooks.py                  # PERMISSION_SOURCES, RATE_LIMIT_RESOLVERS, TIER_DELETE_GUARDS, the readiness checks
├── models.py                 # What features add to the User model and its schemas
├── email.py                  # EMAIL_SENDER, the sender account mail goes out through
├── tasks.py                  # TASK_MODULES, whose background tasks the worker registers
├── tables.py                 # TABLE_MODULES, declaring tables outside src/modules
├── transports.py             # EXTRA_TRANSPORTS, registered beside the session transport
├── cache.py                  # cached, the decorator a route caches its response with
└── admin.py                  # ADMIN_VIEWS (absent without the admin feature)
```

这些文件只包含导入和名称，不包含决策逻辑，因此 ruff、mypy 和 IDE 都能清楚看到项目组装了哪些内容。`tools/removal_drill.py` 根据所选功能生成这些文件，并由测试对比。因此移除功能意味着删除对应目录并重新生成这些内容，而非只手动编辑导入。[可组合功能](composable-features.md)介绍了扩展方式和移除验证。

<a id="migrations-backendmigrations"></a>

## 迁移（`backend/migrations/`）

```text
migrations/
├── env.py                    # Alembic environment (loads all models)
├── script.py.mako            # Template for new migrations
└── versions/                 # One file per migration revision
```

在 `backend/` 中运行：

```bash
uv run --no-sync alembic revision --autogenerate -m "add foo"
uv run --no-sync alembic upgrade head
```

<a id="scripts-backendscripts"></a>

## 脚本（`backend/scripts/`）

```text
scripts/
├── setup_initial_data.py     # All-in-one: tables + tier + admin
├── create_first_superuser.py # Just the admin user
├── create_first_tier.py      # Just the default tier
├── create_tables.py          # Just the database tables
├── cleanup_api_key_json.py   # One-off repair for legacy API key rows
├── canonicalize_emails.py    # One-off repair for addresses stored with case or spaces
├── unlock_account.py         # Clears the login lockout one account is held in
├── revoke_scrypt_api_keys.py # Names the API keys still stored as scrypt, revokes them on request
└── prune_key_usage.py        # Deletes the API-key usage rows past KEY_USAGE_RETENTION_DAYS
```

最常用的入口是 `setup_initial_data`，它会调用上述三个脚本。

```bash
uv run --no-sync python -m scripts.setup_initial_data
```

每个脚本也可在 `backend/` 下独立运行：

```bash
uv run --no-sync python scripts/create_first_tier.py
uv run --no-sync python scripts/create_first_superuser.py
```

脚本无法完成操作时，例如管理员配置不完整、密码不符合策略、用户名已被占用或数据库不可达，会记录一行日志并以退出码 `1` 结束。

`cleanup_api_key_json.py` 不属于新项目初始化步骤。它修复旧版本在 `permissions` 或 `usage_limits` 中写入 JSON null 的 API 密钥记录；读取这些记录并校验时会返回 `500`。升级后在 `backend/` 下执行一次：

```bash
# from backend/ on the host
uv run --no-sync python scripts/cleanup_api_key_json.py

# or in the container, whose working directory is /app/src
docker compose exec api python -m scripts.cleanup_api_key_json
```

脚本报告每列修复的行数，不修改其他内容，可以重复运行。

`canonicalize_emails.py` 也是一次性修复脚本，用于尚未统一邮箱格式的旧数据库。它把邮箱改为所有查询使用的形式：去除首尾空白并转为小写，同时创建唯一索引 `ix_user_email_lower`：

```bash
# from backend/ on the host
uv run --no-sync python scripts/canonicalize_emails.py

# or in the container, whose working directory is /app/src
docker compose exec api python -m scripts.canonicalize_emails
```

如果忽略大小写和空白后两个账户使用同一邮箱，脚本不会修改数据，而会列出冲突地址并以退出码 `1` 结束。应由人工决定邮箱归属，合并或改名后重新运行。

<a id="tests-backendtests"></a>

## 测试（`backend/tests/`）

```text
tests/
├── conftest.py               # Postgres testcontainer, db session, client, mocks
├── wiring.py                 # The fixture modules pytest loads for the selected features
├── fixtures/                 # Each feature's own fixtures
├── unit/                     # Unit tests (no external deps)
│   ├── infrastructure/
│   └── modules/
└── integration/              # Integration tests (real Postgres via testcontainers)
```

在 `backend/` 中运行：

```bash
uv run --no-sync pytest tests/unit         # fast, no Docker
uv run --no-sync pytest tests/integration  # spins up Postgres in Docker via testcontainers
uv run --no-sync pytest                    # everything
```

<a id="architectural-patterns"></a>

## 架构模式

<a id="three-layer-architecture"></a>

### 三层架构

1. **接口层**（`interfaces/`）：HTTP 路由、管理界面和 FastAPI 应用实例。
2. **业务模块层**（`modules/`）：按业务功能纵向划分的模块。
3. **基础设施层**（`infrastructure/`）：数据库、缓存、认证、队列、配置和日志等共用能力。

依赖向下流动：接口层依赖模块层和基础设施层；模块层依赖基础设施层及 `modules/common`；基础设施层不反向依赖上层。

`wiring/` 位于三层之上，从各层导入内容；其他层只读取它提供的列表。例如 `api/v1/__init__.py` 读取 `ROUTER_MOUNTS`，`auth/authorization.py` 读取 `PERMISSION_SOURCES`，`modules/user/models.py` 读取 `UserModelExtensions`。因此一个功能可以扩展另一个功能，而无需直接导入它。

<a id="vertical-slices"></a>

### 按业务功能纵向划分

每个 `modules/<feature>/` 目录拥有对应功能的完整实现。添加新功能通常意味着新增**一个**目录，而非分别修改五个顶层目录。

<a id="dependency-injection"></a>

### 依赖注入

项目广泛使用 FastAPI 的 `Depends`：

- **数据库会话**：`infrastructure.database.session` 中的 `Depends(async_session)`。
- **当前用户**：`infrastructure.auth.dependencies` 中的 `Depends(get_current_user)`。
- **仅超级用户**：`Depends(get_current_superuser)`。
- **权限要求**：在路由的 `dependencies` 列表中使用 `require_permissions("user.read")`。
- **服务实例**：每个模块的 `routes.py` 定义自己的 `get_<feature>_service()` 工厂。

`infrastructure/dependencies.py` 保存所有项目都需要的别名 `AsyncSessionDep`。解析调用者身份的别名随 accounts 功能提供，位于 `infrastructure/auth/deps.py`：`CurrentUserDep`、`CurrentSuperUserDep`、`OptionalUserDep`、`CurrentPrincipalDep`（crudauth 的 `Principal`）和 `CurrentPermissionsDep`（每个请求只解析一次的调用者有效权限）。

<a id="configuration"></a>

### 配置

配置从 `.env` 读取，并通过 mixin 组装：

- 核心 mixin（`DatabaseSettings`、`CORSSettings`、`SecuritySettings` 等）位于 `infrastructure/config/base.py`。
- 各功能在自己的代码旁提供 mixin，如 `infrastructure/cache/settings.py`、`modules/tier/settings.py`。
- `wiring/settings.py` 将所选 mixin 组合为 `Settings`；核心配置放在最后，允许功能覆盖核心默认值。
- `infrastructure/config/settings.py` 是稳定入口，无论安装了哪些功能，都从这里导入 `settings` 或 `get_settings()`。

<a id="error-handling"></a>

### 错误处理

- `modules/common/exceptions.py` 定义领域异常：`DomainError` 以及各功能共用的 `ResourceNotFoundError`、`ResourceExistsError`、`ValidationError`、`PermissionDeniedError`、`PersistenceError`。
- 功能通过继承这些异常定义自己的错误，并用 `public_detail` 指定向客户端公开的信息，映射层无需列举各功能的异常。
- `infrastructure/http_exceptions.py` 定义 HTTP 异常。
- `modules/common/utils/error_handler.register_exception_handlers(app)` 注册统一处理器，通过 `map_exception` 映射任意 `DomainError`；路由只抛出异常，无需自行转换。

<a id="adding-a-new-feature"></a>

## 添加新功能

推荐步骤：

1. **创建模块目录**：`mkdir backend/src/modules/widgets`。
2. **定义模型**：`backend/src/modules/widgets/models.py`。
3. **添加 Schema**：`backend/src/modules/widgets/schemas.py`。写入 Schema 继承 `modules/common/schemas.py` 中的 `EncodableText`，拒绝任意嵌套层级中无法存入数据库列的文本，如孤立代理字符和 NUL 字符；`tests/unit/interfaces/test_request_text.py` 会检查所有路由请求体。
4. **封装 FastCRUD**：`backend/src/modules/widgets/crud.py`。
5. **编写服务**：`backend/src/modules/widgets/service.py`。
6. **提供路由**：`backend/src/modules/widgets/routes.py`。
7. **注册路由**：在 `backend/src/wiring/app.py` 的 `ROUTER_MOUNTS` 中添加 `RouterMount`。
8. **组装其他内容**：按需在 `wiring/settings.py` 添加配置 mixin，在 `wiring/hooks.py` 添加钩子，在 `wiring/admin.py` 添加管理视图，在 `tests/wiring.py` 添加 fixture。
9. **加入移除验证**：在 `tools/removal_drill.py` 中列出路径，检查不包含该功能的构建。
10. **生成迁移**：`uv run --no-sync alembic revision --autogenerate -m "add widgets"`。
11. **应用迁移**：`uv run --no-sync alembic upgrade head`。

完整示例见[开发指南](development.md)。

<a id="data-flow"></a>

## 数据流

```text
HTTP Request
    → interfaces/api/v1/__init__.py (mounting what wiring/app.py lists)
    → modules/<feature>/routes.py
    → modules/<feature>/service.py
    → modules/<feature>/crud.py (FastCRUD)
    → infrastructure/database/session.py
    → PostgreSQL

HTTP Response ← Pydantic schema ← service ← CRUD result ← DB query
```

这种分层将 HTTP 处理、业务逻辑和数据访问分开，便于阅读、测试与扩展。
