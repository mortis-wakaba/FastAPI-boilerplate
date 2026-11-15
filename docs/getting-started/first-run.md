<a id="first-run-guide"></a>

# 首次运行指南

本指南介绍如何验证安装、创建管理员并测试主要功能。它接续[安装指南](installation.md)，假定应用已经运行；下方命令用于访问已启动的服务。

<a id="verification-checklist"></a>

## 验证清单

深入开发前，先确认各项功能正常。

<a id="1-check-services"></a>

### 1. 检查服务

=== "Docker Compose"

    ```bash
    docker compose ps
    ```

    应看到 `api`、`worker`、`postgres` 和 `redis` 服务处于 `running` 状态。

=== "使用 uv 在本地运行"

    确认可以连接 Postgres 和 Redis：

    ```bash
    pg_isready -h localhost -p 5432
    redis-cli ping  # should print PONG
    ```

<a id="2-test-api-documentation"></a>

### 2. 测试 API 文档

在浏览器中打开：

- **Swagger UI**：<http://localhost:8000/docs>
- **ReDoc**：<http://localhost:8000/redoc>

<a id="3-health-check"></a>

### 3. 健康检查

```bash
curl http://localhost:8000/health
```

预期响应：

```json
{
  "status": "healthy"
}
```

<a id="4-database-tables"></a>

### 4. 数据库表

检查表是否已经创建：

=== "Docker Compose"

    ```bash
    docker compose exec postgres psql -U postgres -d postgres -c "\dt"
    ```

=== "使用 uv 在本地运行"

    ```bash
    psql -h localhost -U postgres -d postgres -c "\dt"
    ```

应看到 `user`、`tiers`、`rate_limits`、`api_keys`、`key_usage` 等表。

<a id="initial-setup"></a>

## 初始设置

创建首个管理员用户和默认用户等级。

!!! warning "前置条件"
    执行前确保数据库表已经创建。`CREATE_TABLES_ON_STARTUP=true` 时（local 和 development 环境的默认值），应用首次启动会自动建表。

<a id="create-admin-user-and-default-tier"></a>

### 创建管理员和默认等级

管理员凭据来自 `backend/.env` 中的 `ADMIN_NAME`、`ADMIN_EMAIL`、`ADMIN_USERNAME` 和 `ADMIN_PASSWORD`。

=== "Docker Compose"

    ```bash
    docker compose exec api python -m scripts.setup_initial_data
    ```

=== "使用 uv 在本地运行"

    ```bash
    cd backend
    uv run --no-sync python -m scripts.setup_initial_data
    ```

这会创建：

- 默认用户等级，作为新用户的兜底等级。
- 管理员用户，其 `is_superuser=true`。

<a id="testing-core-features"></a>

## 测试核心功能

<a id="authentication-flow-sessions"></a>

### 认证流程（会话）

本模板使用**服务端会话**和 HttpOnly Cookie，不使用 JWT。

<a id="1-log-in"></a>

#### 1. 登录

```bash
curl -X POST "http://localhost:8000/api/v1/auth/login" \
  -H "Content-Type: application/x-www-form-urlencoded" \
  -d "username=admin&password=your_admin_password" \
  -c cookies.txt
```

响应设置 HttpOnly 的 `session_id` Cookie，并返回 CSRF 令牌：

```json
{ "id": 1, "username": "admin", "csrf_token": "..." }
```

`cookies.txt` 现在保存了会话，后续请求通过 `-b cookies.txt` 携带。

<a id="2-get-the-current-user"></a>

#### 2. 获取当前用户

```bash
curl http://localhost:8000/api/v1/users/me -b cookies.txt
```

<a id="3-create-a-new-user"></a>

#### 3. 创建新用户

```bash
curl -X POST "http://localhost:8000/api/v1/users/" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "John Doe",
    "username": "johndoe",
    "email": "john@example.com",
    "password": "securepassword123"
  }'
```

创建用户是开放操作，无需认证。

<a id="4-check-auth-status"></a>

#### 4. 检查认证状态

```bash
curl http://localhost:8000/api/v1/auth/check-auth -b cookies.txt
```

已登录时返回 `{"authenticated": true, "user": {...}, "session": {...}}`。

<a id="5-log-out"></a>

#### 5. 退出登录

```bash
curl -X POST "http://localhost:8000/api/v1/auth/logout" -b cookies.txt -c cookies.txt
```

<a id="api-keys"></a>

### API 密钥

如需程序化访问（机器间调用），登录后创建 API 密钥：

```bash
curl -X POST "http://localhost:8000/api/v1/api-keys/" \
  -H "Content-Type: application/json" \
  -b cookies.txt \
  -d '{
    "name": "My Integration Key",
    "permissions": ["user.read"],
    "usage_limits": {"requests_per_minute": 60}
  }'
```

⚠️ **完整 API 密钥只会在响应中显示一次。** 请妥善保存。

调用时将密钥放入 `X-API-Key` 请求头，请求会以密钥所有者的身份认证：

```bash
curl http://localhost:8000/api/v1/auth/me -H "X-API-Key: fai_..."
```

列出自己的密钥：

```bash
curl http://localhost:8000/api/v1/api-keys/ -b cookies.txt
```

<a id="tiers-and-rate-limits"></a>

### 用户等级和限流

```bash
# List tiers
curl http://localhost:8000/api/v1/tiers/

# Get a tier by name
curl http://localhost:8000/api/v1/tiers/free

# List rate limits
curl http://localhost:8000/api/v1/rate-limits/
```

<a id="caching"></a>

### 缓存

连续发送两次相同请求，观察耗时；第二次应命中 Redis：

```bash
curl http://localhost:8000/api/v1/users/johndoe -b cookies.txt -w "\nTime: %{time_total}s\n"
curl http://localhost:8000/api/v1/users/johndoe -b cookies.txt -w "\nTime: %{time_total}s\n"
```

<a id="background-tasks-taskiq"></a>

### 后台任务（Taskiq）

后台处理开箱即用，但模板没有附带任务示例接口。注册并分发自己的任务，见[后台任务](../user-guide/background-tasks/index.md)。

本地启动 Worker：

```bash
cd backend
uv run --no-sync taskiq worker src.infrastructure.taskiq.worker:default_broker
```

<a id="adding-your-first-feature-module"></a>

## 添加首个功能模块

代码库按**业务功能纵向划分模块**：每个功能的模型、Schema、CRUD、服务和路由都集中在 `backend/src/modules/` 下的同一个目录。

添加新模块的分步说明见[开发指南](../user-guide/development.md)。

<a id="debugging-common-issues"></a>

## 常见问题排查

<a id="application-logs"></a>

### 应用日志

=== "Docker Compose"

    ```bash
    docker compose logs -f api
    ```

=== "使用 uv 在本地运行"

    日志输出到运行 `fastapi dev` 的终端标准输出。

<a id="database-logs"></a>

### 数据库日志

```bash
docker compose logs -f postgres
```

<a id="run-migrations-manually"></a>

### 手动执行迁移

需要重新执行迁移时：

```bash
cd backend
uv run --no-sync alembic upgrade head
```

<a id="reset-everything-docker"></a>

### 全部重置（Docker）

在 `docker-compose.yml` 所在的仓库根目录执行：

```bash
docker compose down -v   # ⚠️ wipes the database volume
docker compose up
```

<a id="next-steps"></a>

## 后续步骤

已经验证安装并测试主要功能，接下来可以阅读：

<a id="essential-reading"></a>

### 基础阅读

1. **[项目结构](../user-guide/project-structure.md)**：代码组织方式。
2. **[数据库指南](../user-guide/database/index.md)**：模型、Schema、CRUD。
3. **[身份认证](../user-guide/authentication/index.md)**：会话、OAuth、API 密钥。

<a id="advanced-features"></a>

### 高级功能

1. **[缓存](../user-guide/caching/index.md)**：基于 Redis 的缓存。
2. **[后台任务](../user-guide/background-tasks/index.md)**：基于 Taskiq 的异步任务。
3. **[限流](../user-guide/rate-limiting/index.md)**：按用户等级限流。

<a id="development-workflow"></a>

### 开发流程

1. **[开发指南](../user-guide/development.md)**：扩展模板。
2. **[测试](../user-guide/testing.md)**：测试自己的功能。
3. **[生产环境](../user-guide/production.md)**：部署。

<a id="getting-help"></a>

## 获取帮助

- **检查日志**中的错误信息。
- **确认 `backend/.env`** 中的配置值正确。
- **搜索 [GitHub Issues](https://github.com/benavlabs/fastapi-boilerplate/issues)**，查找类似问题。
- **创建[新 Issue](https://github.com/benavlabs/fastapi-boilerplate/issues/new)**，提供详细信息。
