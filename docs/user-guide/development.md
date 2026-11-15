<a id="development-guide"></a>

# 开发指南

本页介绍日常开发流程：运行应用、使用随附工具、添加模块，以及调试模板各组件。

从头到尾添加一个实体，可配合阅读：

- **[数据库 → 模型](database/models.md)**：定义继承 `Base` 的 dataclass 模型。
- **[数据库 → Schema](database/schemas.md)**：Pydantic 请求与响应模型。
- **[API → 接口](api/endpoints.md)**：将路由连接到服务。
- **[管理后台 → 添加模型](admin-panel/adding-models.md)**：在管理界面展示模型。

本页将这些步骤串联起来。

<a id="running-the-app"></a>

## 运行应用

```bash
cd backend
uv run --no-sync fastapi dev src/interfaces/main.py
```

`--reload` 监视文件系统，并在 Python 文件变更时重启。只用于开发，**不要用于生产环境**。

使用 Docker 时，在仓库根目录运行：

```bash
docker compose up -d                # API + Postgres + Redis
docker compose logs -f api          # tail the api logs
docker compose exec api bash        # shell into the api container
```

服务名取决于 `docker-compose.yml`，通常使用 `api`。

<a id="the-background-worker"></a>

## 后台 Worker

应用使用 Taskiq 任务时，在第二个终端运行 Worker：

```bash
cd backend
uv run --no-sync taskiq worker src.infrastructure.taskiq.worker:default_broker --reload
```

如果任务声明了 `schedule` 标签，还需要运行一个调度器：

```bash
cd backend
uv run --no-sync taskiq scheduler src.infrastructure.taskiq.scheduler:scheduler
```

`--reload` 需要 `dev` extra 中的 `taskiq[reload]`。使用 `--all-extras` 同步的 workspace 已经包含它；`--extra dev` 可将其安装到此前未安装开发依赖的环境中。生产环境去掉这两个参数。详细说明见[后台任务](background-tasks/index.md)。

<a id="the-dev-toolchain"></a>

## 开发工具链

项目通过 `backend/pyproject.toml` 配置了 `ruff`、`mypy` 和 `pytest`：

```bash
cd backend

# Lint + format (ruff handles both)
uv run --no-sync ruff check .
uv run --no-sync ruff format .
uv run --no-sync ruff check --fix .          # auto-fix what ruff can

# Type check
uv run --no-sync mypy src scripts migrations tests

# Tests
uv run --no-sync pytest
uv run --no-sync pytest -k "test_user"       # run tests matching a name
uv run --no-sync pytest -x                   # stop on first failure
uv run --no-sync pytest -n auto              # parallel via pytest-xdist
```

Ruff 配置位于 `pyproject.toml:[tool.ruff]`，后端配置位于 `backend/pyproject.toml`：

- `line-length = 128`。
- 规则集为 `E`、`F`、`I`、`UP`，分别对应 pycodestyle、pyflakes、isort、pyupgrade。
- 后端额外启用 `UP006`、`UP007`、`UP035`、`UP039` 和 `PLC0415`，后者要求导入放在模块顶部。
- `known-first-party = ["src"]` 确保 `src.*` 导入正确分组。

Mypy 对类型注解有意采用宽松设置（`disallow_untyped_defs = false`），可随新模块逐步提高严格度。CI 和 pre-commit 钩子检查 `src`、`scripts`、`migrations` 和 `tests`，并加载 Pydantic 插件（`plugins = ["pydantic.mypy"]`）识别 Schema 的必填字段。插件只能识别赋值一侧的默认值，因此应写为：

```python
name: Annotated[str | None, Field(max_length=50)] = None
```

不要仅把 `default=None` 放在 `Field(...)` 内，否则每次构造时省略该字段都会被报告为错误。

<a id="pre-commit"></a>

## Pre-commit

仓库的 `.pre-commit-config.yaml` 与 CI 执行相同的检查，避免相互冲突：

| 钩子 | 作用 |
|------|------|
| `ruff-check --fix`、`ruff-format` | 与 CI 相同的 lint 和格式检查 |
| `pyupgrade` | 更新为 Python 3.11+ 语法 |
| `yesqa` | 删除不再需要的 `noqa` |
| 基础检查钩子 | 文件末尾换行、行尾空白、YAML、大文件、私钥、合并标记 |
| `mypy_backend`、`mypy_cli` | 本地钩子，调用方式与 CI 相同 |
| `unit_test` | 本地手动阶段运行 `pytest tests/unit` |

`unit_test` 只运行单元测试，不需要容器，数秒即可完成。集成测试通过 testcontainers 启动 Postgres，会在 CI 和手动运行 `pytest` 时执行。

项目没有 docstring 或 Markdown 格式化钩子。格式化由 `ruff-format` 负责；以前的 `mdformat` 会破坏 MkDocs 内容选项卡，`blacken-docs` 会把代码示例改成含义不同的代码，因此已移除。

`check-docstring-first` 也已移除。项目在模块级名称下方使用属性 docstring，该钩子将它误认成第二个模块 docstring，曾因此拒绝 40 个文件。

`pre-commit` 包含在 `dev` extra 中，`uv sync --all-packages --all-extras` 会安装它。Git 钩子只需安装一次：

```bash
uv run --no-sync pre-commit install
```

之后 `git commit` 会自动运行钩子。也可以手动执行：

```bash
uv run --no-sync pre-commit run --all-files
uv run --no-sync pre-commit run unit_test --hook-stage manual --all-files
```

<a id="adding-a-new-module"></a>

## 添加新模块

模板将领域代码按功能组织在 `backend/src/modules/<name>/`。添加 `widgets` 模块：

```text
backend/src/modules/widgets/
├── __init__.py
├── models.py        # SQLAlchemy model (Base + dataclass)
├── schemas.py       # Pydantic request/response models
├── crud.py          # FastCRUD instance
├── service.py       # Business logic — calls CRUD, raises domain errors
└── routes.py        # FastAPI router — wraps the service, handles HTTP
```

完整代码见[数据库 → 模型](database/models.md)和 [API → 接口](api/endpoints.md)。简要流程：

1. **编写模型**：在 `models.py` 中继承 `Base`，按需使用 `TimestampMixin`、`SoftDeleteMixin`、`UUIDMixin`。
2. **编写 Schema**：在 `schemas.py` 中定义 `WidgetBase`、`WidgetCreate`、`WidgetRead`、`WidgetUpdate`，以及 FastCRUD 的 `schema_to_select` 使用的 `WidgetSelect`。
3. **连接 FastCRUD**，在 `crud.py` 中：

   ```python
   from fastcrud import FastCRUD
   from .models import Widget
   crud_widgets = FastCRUD(Widget)
   ```

4. **实现服务**：在 `service.py` 中通过类方法调用 `crud_widgets`，状态不合法时抛出 `DomainError` 子类。
5. **定义路由**：在 `routes.py` 中调用服务并返回字典，FastAPI 通过 `response_model=WidgetRead` 序列化；领域错误交给全局处理器。
6. **注册路由**：在 `interfaces/main.py` 或顶层路由汇总处注册：

   ```python
   from src.modules.widgets.routes import router as widgets_router
   api_v1.include_router(widgets_router, prefix="/widgets")
   ```

7. **生成迁移**：

   ```bash
   cd backend
   uv run --no-sync alembic revision --autogenerate -m "Add widget model"
   uv run --no-sync alembic upgrade head
   ```

   注意：`env.py` 开始时会运行 `validate_production_migration`。生产环境必须设置 `CONFIRM_PRODUCTION_MIGRATION=yes` 才能应用迁移，本地开发不受影响。
8. **可选**：添加 `WidgetAdmin` 视图，见[管理后台 → 添加模型](admin-panel/adding-models.md)。

Alembic 环境（`backend/migrations/env.py`）通过 `import_models("src.modules")` 自动发现模型。只要模型位于 `modules/<name>/models.py`，`--autogenerate` 就能识别，无需手动导入。

<a id="adding-custom-middleware"></a>

## 添加自定义中间件

中间件位于 `backend/src/infrastructure/middleware.py`，也可创建同级文件：

```python
import time

from fastapi import Request, Response
from starlette.middleware.base import BaseHTTPMiddleware, RequestResponseEndpoint
from starlette.types import ASGIApp


class TimingMiddleware(BaseHTTPMiddleware):
    async def dispatch(self, request: Request, call_next: RequestResponseEndpoint) -> Response:
        start = time.perf_counter()
        response = await call_next(request)
        elapsed_ms = (time.perf_counter() - start) * 1000
        response.headers["X-Process-Time-Ms"] = f"{elapsed_ms:.1f}"
        return response
```

在 `infrastructure/app_factory.py` 或自定义 `create_application` 中注册：

```python
application.add_middleware(TimingMiddleware)
```

顺序很重要：后添加的中间件在请求路径中**先执行**。模板按特定顺序添加 `SecurityHeadersMiddleware`、`ClientCacheMiddleware`、`CatchAllErrorMiddleware`、`GZipMiddleware` 和 `CORSMiddleware`，见 `app_factory.py:create_application`。管理后台的 `SessionMiddleware` 不在其中，它只挂载到后台路由。

<a id="adding-a-custom-dependency"></a>

## 添加自定义依赖

依赖应与所属功能放在一起。需要感知会话的依赖，可参考 `infrastructure/auth/dependencies.py:get_current_user`。

<a id="define-the-factory"></a>

### 定义工厂

```python
# modules/workspace/dependencies.py
from fastapi import Request

from ...infrastructure.auth.dependencies import get_current_user
from ...infrastructure.auth.deps import CurrentUserDep


def get_workspace(
    request: Request,
    current_user: CurrentUserDep,
) -> str:
    workspace = request.headers.get("X-Workspace")
    if not workspace:
        raise PermissionDeniedError("Missing workspace header")
    # validate workspace membership against current_user...
    return workspace
```

<a id="register-the-alias-optional"></a>

### 注册别名（可选）

将其添加到模块的 `dependencies.py`，避免路由每次都写 `Depends(get_workspace)`：

```python
# modules/workspace/dependencies.py (extended)
from typing import Annotated

from fastapi import Depends

from ...infrastructure.auth.deps import CurrentUserDep

WorkspaceDep = Annotated[str, Depends(get_workspace)]
```

<a id="use-it"></a>

### 使用依赖

```python
from ..dependencies import WorkspaceDep


@router.get("/workspace/items")
async def list_workspace_items(
    workspace: WorkspaceDep,
    db: AsyncSessionDep,
) -> list[dict[str, Any]]:
    ...
```

各模块的服务别名采用相同模式，实际示例见 `modules/{user,tier,rate_limit,api_keys}/dependencies.py`。

<a id="debugging-tips"></a>

## 调试建议

<a id="see-every-sql-query"></a>

### 查看每条 SQL 查询

项目没有专门开关：引擎使用 `echo=False` 和 `hide_parameters=True` 构建，见 `infrastructure/database/session.py`。提高 SQLAlchemy 自身日志级别即可查看语句：

```python
logging.getLogger("sqlalchemy.engine").setLevel(logging.INFO)
```

日志有意隐藏参数值，避免失败的插入把密码哈希写入日志。需要临时查看参数时，可以通过 `build_engine(hide_parameters=False, echo=True)` 创建自己的引擎。

<a id="inspect-rate-limit-and-cache-state"></a>

### 检查限流和缓存状态

```bash
docker compose exec redis redis-cli
> SELECT 0                           # cache DB
> KEYS '*'
> SELECT 1                           # rate-limiter DB
> KEYS 'ratelimit:*'
> SELECT 3                           # taskiq queue DB
> LRANGE default 0 -1                # pending tasks
```

每个子系统使用不同的 Redis DB，约定见 `.env.example`：`CACHE_REDIS_DB=0`、`RATE_LIMITER_REDIS_DB=1`、`SESSION_REDIS_DB=2`、`TASKIQ_REDIS_DB=3`。

<a id="watch-sessions-live"></a>

### 实时查看会话

会话存储由 `crudauth` 库管理，模板没有可调用的 `SessionManager`。用户意外退出登录时，可直接检查 Redis 中的会话存储：

```bash
redis-cli
> SELECT 2                           # session backend DB (SESSION_REDIS_DB)
> KEYS 'session:*'
```

详情见[认证 → 会话](authentication/sessions.md)。

<a id="use-the-interactive-docs"></a>

### 使用交互式文档

`http://localhost:8000/docs`（Swagger UI）和 `http://localhost:8000/redoc` 根据路由自动生成。可直接在界面发送请求，带 Pydantic 请求体的接口都有 **Try it out** 表单。默认仅非生产环境开放，通过 `OPENAPI_URL` 控制。

<a id="production-validators"></a>

### 生产环境校验器

`ENVIRONMENT=production` 时，`infrastructure/security/` 在启动阶段检查以下问题并明确报错：

- 不安全或占位用的 `SECRET_KEY`。
- 默认或空数据库密码。
- 启用管理后台但没有 `ADMIN_USERNAME` / `ADMIN_PASSWORD`。
- `CORS_ORIGINS` 包含 `*`。

空的 `CORS_ORIGINS` 并非错误，它表示不允许跨域请求。

生产启动因这些问题失败时，应修正配置，而不是绕过校验器。

<a id="testing"></a>

## 测试

测试位于 `backend/tests/`，`backend/pyproject.toml` 中的 pytest 配置如下：

```toml
[tool.pytest.ini_options]
testpaths = ["tests"]
asyncio_mode = "auto"
addopts = ["-v", "--strict-markers", "--tb=short"]
```

测试套件不强制指定 `ENVIRONMENT`，保持 `local` 或 `development` 即可。生产校验器只在 `production` 运行，因此测试不会被仅生产环境需要的变量阻塞。

一个合理的 `tests/conftest.py` 起点：

```python
# tests/conftest.py
import pytest
import pytest_asyncio
from httpx import ASGITransport, AsyncClient
from sqlalchemy.ext.asyncio import AsyncSession, async_sessionmaker, create_async_engine

from src.infrastructure.database.session import Base
from src.infrastructure.database.session import async_session
from src.interfaces.main import app

TEST_DATABASE_URL = "postgresql+asyncpg://postgres:postgres@localhost:5432/test_db"


@pytest_asyncio.fixture
async def db_engine():
    engine = create_async_engine(TEST_DATABASE_URL)
    async with engine.begin() as conn:
        await conn.run_sync(Base.metadata.create_all)
    yield engine
    async with engine.begin() as conn:
        await conn.run_sync(Base.metadata.drop_all)
    await engine.dispose()


@pytest_asyncio.fixture
async def db_session(db_engine) -> AsyncSession:
    factory = async_sessionmaker(db_engine, expire_on_commit=False)
    async with factory() as session:
        yield session


@pytest_asyncio.fixture
async def client(db_session):
    async def override_db():
        yield db_session

    app.dependency_overrides[async_session] = override_db
    transport = ASGITransport(app=app)
    async with AsyncClient(transport=transport, base_url="http://test") as ac:
        yield ac
    app.dependency_overrides.clear()
```

然后添加冒烟测试：

```python
# tests/test_smoke.py
async def test_health(client):
    response = await client.get("/health")
    assert response.status_code == 200
```

真正依赖 Postgres 语义的测试，如外键约束、ARRAY 类型和 JSONB，使用开发依赖中已有的 `testcontainers[postgres]`，在 fixture 中启动真实 Postgres。

服务单元测试应在 **CRUD 层**模拟，而非数据库层。服务的契约是调用 `crud_widgets.get` 并获得字典或 None，这就是合适的模拟边界。

<a id="customizing-the-settings"></a>

## 自定义配置

配置位于 `backend/src/infrastructure/config/settings.py`。添加环境变量驱动的配置值：

1. 在对应配置类中添加字段，或新建配置类：

   ```python
   class WidgetSettings(BaseSettings):
       WIDGET_BATCH_SIZE: int = config("WIDGET_BATCH_SIZE", default=100, cast=int)
   ```

2. 将其加入 `settings.py` 底部 `Settings` 的 mixin 列表。
3. 在 `backend/.env.example` 中记录变量。

之后通过 `get_settings().WIDGET_BATCH_SIZE` 读取。

完整模式见[配置 → 配置类](configuration/settings-classes.md)。

<a id="disabling-subsystems"></a>

## 禁用子系统

多数主要子系统可通过环境变量关闭，无需改代码：

| 子系统 | 开关 | 效果 |
|--------|------|------|
| 缓存 | `CACHE_ENABLED=false` | `@cache` 不执行缓存操作 |
| 客户端缓存 | `CLIENT_CACHE_ENABLED=false` | 不挂载中间件 |
| 限流器 | `RATE_LIMITER_ENABLED=false` | crudauth API 限流器直接返回 |
| 后台任务 | 不运行 Worker | 仍创建消息代理，但没有消费者 |
| 管理后台 | `ADMIN_ENABLED=false` | 不挂载 `/admin` |
| 文档 | `OPENAPI_URL=` | 关闭 `/docs` 和 `/redoc` |

原文建议通常优先禁用子系统，而不是删除代码；保留禁用状态不会增加运行开销。

<a id="common-mistakes"></a>

## 常见错误

<a id="auto-import-gotchas"></a>

### 自动导入的限制

Alembic 使用 `import_models("src.modules")` 发现模型，**只查找 `modules/<name>/models.py`**。如果把模型放在 `modules/<name>/sub/inner.py`，自动迁移生成不会找到它。应保留在 `models.py` 或手动导入文件。

<a id="forgetting-lazyselectin-on-a-relationship"></a>

### 关联关系忘记设置 `lazy="selectin"`

SQLAdmin 在异步上下文运行。关联关系未设置 `lazy="selectin"` 时，后台渲染可能触发 `MissingGreenlet`。模板的 `User.tier` 等关联已采用此模式，可以参考。`Tier.users` 是有意的例外：使用 `lazy="select"`，避免加载一个等级时加载其所有用户。

<a id="dataclass-models-without-initfalse-on-relationships"></a>

### dataclass 模型的关联缺少 `init=False`

`Base = DeclarativeBase + MappedAsDataclass`。关联字段必须设置 `init=False`，否则会进入 dataclass 的 `__init__`，导致插入失败。参考 `modules/user/models.py:User.tier`。

<a id="catching-exceptions-too-broadly-in-routes"></a>

### 路由捕获范围过大的异常

服务抛出 `ResourceNotFoundError`、`PermissionDeniedError` 等领域错误，路由无需转换。`modules/common/utils/error_handler.py` 的 `register_exception_handlers(app)` 注册统一处理器，通过 `EXCEPTION_MAPPING` 将任意 `DomainError` 映射为状态码，并将其他错误转换为带支持编号的 500 响应。只有需要返回不同响应时，路由才捕获领域错误。

<a id="cache-decorators-without-request-request"></a>

### 缓存装饰器缺少 `request: Request`

`@cache` 检查 `request.method` 来决定读取还是失效缓存。每个被装饰路由的首个参数必须是 `request: Request`。其余约定见[缓存 → 服务端缓存](caching/redis-cache.md)。

<a id="key-files"></a>

## 关键文件

| 组件 | 位置 |
|------|------|
| 应用工厂 / 中间件顺序 | `backend/src/infrastructure/app_factory.py` |
| 配置 | `backend/src/infrastructure/config/settings.py` |
| 生命周期 / 启动 | `backend/src/infrastructure/app_factory.py:lifespan_factory` |
| 数据库会话 | `backend/src/infrastructure/database/session.py` |
| 模块参考 | `backend/src/modules/user/` |
| Pre-commit | `.pre-commit-config.yaml` |
| lint / 类型 / 测试配置 | `backend/pyproject.toml` |

<a id="next-steps"></a>

## 后续步骤

- **[项目结构](project-structure.md)**：完整目录说明。
- **[测试](testing.md)**：测试模式与基础设施。
- **[生产环境](production.md)**：部署与安全加固清单。
