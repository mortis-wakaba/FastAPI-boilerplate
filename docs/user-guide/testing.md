<a id="testing"></a>

# 测试

模板在 `backend/tests/` 配置 pytest，通过 `testcontainers[postgres]` 使用真实数据库，通过 httpx 在 HTTP 层测试 FastAPI。已有各模块单元测试及真实 Postgres 集成测试；`tools/removal_drill.py` 还会移除各功能后重新构建并运行检查。本页介绍这些模式。

<a id="whats-configured"></a>

## 已有配置

`backend/pyproject.toml`：

```toml
[tool.pytest.ini_options]
testpaths = ["tests"]
python_files = ["test_*.py"]
python_functions = ["test_*"]
python_classes = ["Test*"]
asyncio_mode = "auto"
addopts = ["-v", "--strict-markers", "--tb=short"]
markers = [
    "unit: Unit tests that don't require external dependencies",
    "integration: Integration tests that may require external services",
    "asyncio: Tests that use asyncio",
    "slow: marks tests as slow running",
]
```

提供：

- **`asyncio_mode = 'auto'`**：所有 `async def test_*` 自动运行在 pytest-asyncio 下，无需装饰器。
- **`--strict-markers`**：标记拼写错误直接失败。
- **unit / integration / slow 标记**：已声明，需自行应用；当前无测试标记 unit，因此 `pytest -m unit` 暂不选择任何测试。

`backend/tests/conftest.py` 将 backend 加入 `sys.path`，加载 `backend/tests/wiring.py` 列出的 fixture。测试不强制 ENVIRONMENT，合法值为 local / development / staging / production，校验器仅在 production 执行对应检查。

通过 `uv sync --all-packages --all-extras` 安装的 dev extra 包含 pytest、pytest-asyncio、pytest-mock、httpx、faker、testcontainers[postgres] 和支持并行的 pytest-xdist[psutil]。

未包含 pytest-cov。开发工具属于 extra 而非 dependency group，开始收集覆盖率时使用 `uv add --optional dev pytest-cov`。

<a id="test-layout"></a>

## 测试布局

已有测试及新增位置：

```text
backend/tests/
├── conftest.py                  # the Postgres container, the session, the HTTP clients
├── wiring.py                    # the fixture modules the selected features contribute
├── fixtures/
│   ├── accounts.py              # a registered user, a logged-in client
│   └── tiers.py
├── unit/
│   ├── modules/
│   │   ├── user/
│   │   │   ├── test_service.py
│   │   │   ├── test_schemas.py
│   │   │   └── test_models.py        # e.g. User.is_active derives from not is_deleted
│   │   └── tier/
│   │       └── test_service.py
│   └── infrastructure/
│       └── auth/
│           └── test_dependencies.py  # get_current_user / _superuser / _optional_user
└── integration/
    ├── api/
    │   ├── test_auth.py
    │   ├── test_users.py
    │   └── test_tiers.py
    └── db/
        └── test_migrations.py
```

分层是指导而非硬规则：单元测试通常模拟 FastCRUD 数据库边界，速度快；集成测试使用真实 Postgres，可通过 testcontainers 或本地数据库，覆盖 HTTP 层。

<a id="a-working-conftestpy"></a>

## 可用的 conftest.py

以下提供三层 fixture：FastAPI app、异步 db_session，以及将数据库依赖替换成测试会话的 httpx.AsyncClient。

```python
# tests/conftest.py
from collections.abc import AsyncGenerator

import pytest_asyncio
from httpx import ASGITransport, AsyncClient
from sqlalchemy.ext.asyncio import AsyncSession, async_sessionmaker, create_async_engine
from testcontainers.postgres import PostgresContainer

from src.infrastructure.database.session import Base
from src.infrastructure.database.session import async_session
from src.interfaces.main import app


@pytest_asyncio.fixture(scope="session")
async def postgres_container() -> AsyncGenerator[PostgresContainer, None]:
    container = PostgresContainer("postgres:16-alpine", driver="asyncpg")
    container.start()
    try:
        yield container
    finally:
        container.stop()


@pytest_asyncio.fixture(scope="session")
async def db_engine(postgres_container):
    url = postgres_container.get_connection_url()
    engine = create_async_engine(url, echo=False, future=True)
    async with engine.begin() as conn:
        await conn.run_sync(Base.metadata.create_all)
    yield engine
    await engine.dispose()


@pytest_asyncio.fixture
async def db_session(db_engine) -> AsyncGenerator[AsyncSession, None]:
    factory = async_sessionmaker(db_engine, expire_on_commit=False)
    async with factory() as session:
        try:
            yield session
        finally:
            await session.rollback()


@pytest_asyncio.fixture
async def client(db_session) -> AsyncGenerator[AsyncClient, None]:
    async def override_db():
        yield db_session

    app.dependency_overrides[async_session] = override_db
    transport = ASGITransport(app=app)
    async with AsyncClient(transport=transport, base_url="http://test") as ac:
        yield ac
    app.dependency_overrides.clear()
```

要点：

- `PostgresContainer` 按测试会话启动真实 Postgres，首个测试承担数秒启动，后续复用。
- 每测试结束回滚 db_session，容器复用但数据隔离。
- `app.dependency_overrides[async_session]` 将生产依赖换为测试事务。
- `ASGITransport` 在进程内运行，不启动真实 HTTP 服务器。

不依赖 Docker 时，可连接已有本地 Postgres，但使用独立测试数据库如 test_db，并在测试会话结束删除。

<a id="writing-unit-tests"></a>

## 单元测试

不接触数据库，在 **CRUD 层**模拟。服务契约是调用 `crud_widgets.get` 返回字典或 None：

```python
# tests/unit/modules/user/test_service.py
from unittest.mock import AsyncMock

import pytest

from src.modules.common.exceptions import ResourceNotFoundError
from src.modules.user.service import UserService


@pytest.mark.unit
async def test_get_by_id_returns_user(mocker):
    mock_crud = mocker.patch("src.modules.user.service.crud_users")
    mock_crud.get = AsyncMock(return_value={"id": 1, "username": "alice"})

    user = await UserService().get_by_id(user_id=1, db=AsyncMock())

    assert user["username"] == "alice"
    mock_crud.get.assert_awaited_once()


@pytest.mark.unit
async def test_get_by_id_raises_when_missing(mocker):
    mock_crud = mocker.patch("src.modules.user.service.crud_users")
    mock_crud.get = AsyncMock(return_value=None)

    with pytest.raises(ResourceNotFoundError):
        await UserService().get_by_id(user_id=999, db=AsyncMock())
```

pytest-mock 的 `mocker` 自动清理，AsyncMock 匹配异步接口。

<a id="writing-integration-tests"></a>

## 集成测试

通过 client 和 db_session 使用真实数据库。httpx.AsyncClient 在调用间保存 Cookie，登录一次后复用即可遵循会话流程。

下方 `build_user_create_payload` 是需自行添加的 helper，模板不提供工厂，其文件在后面给出。

```python
# tests/integration/api/test_users.py
import pytest

from src.modules.user.service import UserService
from tests.helpers.factories import build_user_create_payload


@pytest.mark.integration
async def test_register_login_and_fetch_me(client, db_session):
    # Register
    payload = build_user_create_payload(email="alice@example.com", username="alice")
    register_response = await client.post("/api/v1/users/", json=payload)
    assert register_response.status_code == 201
    user_data = register_response.json()
    assert user_data["username"] == "alice"

    # Log in — sets session cookie on the client
    login_response = await client.post(
        "/api/v1/auth/login",
        json={"username": "alice", "password": payload["password"]},
    )
    assert login_response.status_code == 200

    # Authenticated request reuses the cookie automatically
    me_response = await client.get("/api/v1/users/me/")
    assert me_response.status_code == 200
    assert me_response.json()["username"] == "alice"
```

小型数据工厂：

```python
# tests/helpers/factories.py
from faker import Faker

fake = Faker()


def build_user_create_payload(**overrides) -> dict:
    return {
        "name": fake.name(),
        "username": fake.user_name(),
        "email": fake.email(),
        "password": "TestPass123!",
        **overrides,
    }
```

<a id="authenticating-as-a-specific-user"></a>

### 指定身份认证

测试无需走注册时，可通过服务直接创建超级用户，然后登录：

```python
# tests/conftest.py (additional fixture)
@pytest_asyncio.fixture
async def superuser_client(client, db_session):
    service = UserService()
    await service.create(
        payload={
            "name": "Super",
            "username": "super",
            "email": "super@test.com",
            "password": "SuperPass123!",
        },
        db=db_session,
    )
    # Manually flip is_superuser via crud_users.update if your service doesn't expose it
    await db_session.commit()

    await client.post("/api/v1/auth/login",
                      json={"username": "super", "password": "SuperPass123!"})
    yield client
```

`superuser_client` 可供需要管理员权限的测试使用。

<a id="resetting-the-session-between-tests"></a>

### 测试间重置会话

AsyncClient 在 fixture 生命周期保存 Cookie，函数级 client 使每测试从无会话开始。测试中退出可调用 `await client.post('/api/v1/auth/logout/')`，或 `client.cookies.clear()`。

<a id="csrf-in-tests"></a>

## 测试 CSRF

默认开启时，状态变更请求使用双重提交：服务端设 Cookie，客户端在头中回传。可选择：

- 非 CSRF 重点的服务集成测试，设置 `CSRF_ENABLED=false`。
- 验证完整流程时携带令牌：

  ```python
  # Hit a GET first so the server sets the CSRF cookie
  await client.get("/api/v1/auth/me/")  # any safe endpoint
  csrf_token = client.cookies["csrf_token"]   # cookie name from your config
  response = await client.post(
      "/api/v1/widgets/",
      json={...},
      headers={"X-CSRF-Token": csrf_token},
  )
  ```

详情见[会话](authentication/sessions.md)。

<a id="testing-cached-endpoints"></a>

## 测试缓存接口

装饰器使用 CACHE_BACKEND 指向的后端。一般测试关闭 `CACHE_ENABLED`，使装饰器不缓存；专门验证缓存和失效时，使用本地 Redis 或 testcontainer，测试放 integration。

<a id="testing-background-tasks"></a>

## 测试后台任务

一般测试不启动实际 Worker，可用 Taskiq InMemoryBroker 在测试进程执行 `.kiq()`：

```python
# tests/conftest.py (additional)
from taskiq import InMemoryBroker

from src.infrastructure.taskiq.brokers import default_broker as real_broker

@pytest_asyncio.fixture(autouse=True)
async def in_memory_broker(monkeypatch):
    test_broker = InMemoryBroker()
    monkeypatch.setattr("infrastructure.taskiq.brokers.default_broker", test_broker)
    monkeypatch.setattr("infrastructure.taskiq.default_broker", test_broker)
    yield test_broker
```

若仅断言任务已安排、不执行任务体，则使用记录调用的模拟代理。

<a id="running-the-suite"></a>

## 运行测试

```bash
cd backend

# Run everything (--no-sync skips uv's implicit sync; the environment stays as you synced it)
uv run --no-sync pytest

# Just the unit tests (skip the slower ones that need the container)
uv run --no-sync pytest tests/unit

# Just the integration tests: by path, or by the marker they carry
uv run --no-sync pytest tests/integration
uv run --no-sync pytest -m integration

# Stop on first failure
uv run --no-sync pytest -x

# Keep running on failures, show output for tests matching a name
uv run --no-sync pytest -k "user_login" -v

# Parallel via pytest-xdist
uv run --no-sync pytest -n auto

# With coverage (after `uv add --optional dev pytest-cov`)
uv run --no-sync pytest --cov=src --cov-report=term-missing
```

<a id="continuous-integration"></a>

## 持续集成

`.github/workflows/tests.yml` 在 PR 运行测试，另有 lint、类型检查及所有移除预设矩阵。工作流在根目录 `uv sync --all-packages --all-extras --locked`，再进入 backend 使用 `uv run --no-sync pytest`，与本地一致。

ubuntu-latest 提供 Docker，因此 testcontainers 自行启动 Postgres，无需 `services:` 块。CI 运行完整集成测试。

没有 Docker 的 runner（如某些容器镜像）会使数据库 fixture 自行跳过，绿色结果不能代表这些测试通过。应提供自行启动的 Postgres，或使用支持 Docker 的 runner。

<a id="common-mistakes"></a>

## 常见问题

<a id="my-test-isnt-actually-using-the-test-database"></a>

### 未使用测试数据库

覆盖的 `async_session` 必须与路由依赖为**同一个 callable**，路由用了其他依赖则不生效，应查看路由源码。

<a id="tests-pass-individually-but-fail-when-run-together"></a>

### 单独通过、合跑失败

通常是数据库共享状态。确保每测试回滚，或测试间 truncate；大结构下 truncate 比反复建表更快。

<a id="async-tests-hang"></a>

### 异步测试挂起

检查 `asyncio_mode='auto'`，以及 async fixture 是否正确使用 `pytest_asyncio.fixture`，两者需匹配。

<a id="cookies-arent-persisting-between-test-calls"></a>

### Cookie 不保留

两次请求必须使用同一个 AsyncClient，不要每请求新建。

<a id="fastcrud-returns-dicts-not-models-in-tests-too"></a>

### FastCRUD 测试也返回字典

这是设计。断言 `result['name']` 等键，不断言 ORM Widget 类型。

<a id="test-database-has-stale-schema-after-model-changes"></a>

### 模型变更后结构仍旧

示例使用 create_all，每测试会话重建；模块级重建 fixture 时需重启测试会话。长期测试库应在 fixture 中运行 Alembic。

<a id="key-files"></a>

## 关键文件

| 组件 | 位置 |
|------|------|
| pytest 配置 | `backend/pyproject.toml` 的 `[tool.pytest.ini_options]` |
| 测试根目录 | `tests/`，位于 backend 下 |
| 被测参考模块 | `backend/src/modules/user/`、`backend/src/modules/tier/` |
| 测试环境配置 | `backend/src/infrastructure/config/settings.py` |
| 模型 / Base | 原文列 `backend/src/infrastructure/database/models.py`，Base 定义位置另见数据库章节 |

<a id="next-steps"></a>

## 后续步骤

- **[开发指南](development.md)**：完整开发流程。
- **[生产环境](production.md)**：部署及 CI 考虑。
- **[会话](authentication/sessions.md)**：测试中的 Cookie 和 CSRF。
