<a id="docker-setup"></a>

# Docker 配置

本页介绍如何在容器中运行模板。Python 项目位于 `backend/`，镜像从仓库根目录构建，需要 workspace 的 `pyproject.toml`、`uv.lock`、`cli/pyproject.toml` 和 `backend/`。`docker compose` 也在仓库根目录运行。

!!! info "Compose 文件由命令生成"
    仓库不提交 Compose 文件。`bp deploy generate local` 在仓库根目录生成文件，该文件已被 Git 忽略，因此重新生成不会与仓库冲突。`prod` 和 `nginx` 生成另外两种服务组合。见 [CLI 命令](../../cli/commands.md#bp-deploy-generate-mode)。

<a id="quick-start"></a>

## 快速开始

```bash
cp backend/.env.example backend/.env
# edit backend/.env (set SECRET_KEY, change default DB password, etc.)
uv run --no-sync bp deploy generate local
docker compose up --build
```

<a id="dockerfile-architecture"></a>

## Dockerfile 架构

原文将 `backend/Dockerfile` 描述为基于 `python:3.11-slim` 的**四阶段**构建，下表实际列出了五个命名阶段：

| 阶段 | 用途 |
|------|------|
| `requirements-stage` | 使用官方 `astral-sh/uv` 镜像，将 `uv.lock` 的锁定依赖导出为 `requirements-prod.txt` 和 `requirements-dev.txt` |
| `base` | 安装 gcc 等系统依赖及生产 Python 依赖，复制 `src/`，设置 `PYTHONPATH=/app:/app/src` |
| `dev` | 添加开发依赖和 `tests/`，使用非 root 用户 `appuser`，运行 `fastapi dev interfaces/main.py --host 0.0.0.0 --port 8000` |
| `migrate` | 添加 `migrations/` 和 `alembic.ini`，默认执行 `alembic upgrade head`，适合作为生产应用启动前的一次性作业 |
| `prod` | 基于 base，使用非 root 用户，运行 `fastapi run interfaces/main.py --host 0.0.0.0 --port 8000 --workers $WORKERS`，Worker 默认 1 个 |

构建时使用 `--target` 选择阶段。上下文是仓库根目录，Dockerfile 从这里复制 `pyproject.toml`、`uv.lock`、`backend/` 和 `cli/pyproject.toml`：

```bash
docker build --target dev -t fastapi-boilerplate:dev -f backend/Dockerfile .
docker build --target prod -t fastapi-boilerplate:prod -f backend/Dockerfile .
docker build --target migrate -t fastapi-boilerplate:migrate -f backend/Dockerfile .
```

<a id="what-bp-deploy-generate-local-writes"></a>

## `bp deploy generate local` 生成的内容

四个服务：`api`、`worker`、`postgres` 和 `redis`。API 与 Worker 从 `dev` 阶段构建，挂载源码以支持热重载：

```yaml
services:
  api:
    build:
      context: .
      dockerfile: backend/Dockerfile
      target: dev
    env_file:
      - ./backend/.env
    environment:
      POSTGRES_SERVER: postgres
      CACHE_REDIS_HOST: redis
      RATE_LIMITER_REDIS_HOST: redis
      TASKIQ_REDIS_HOST: redis
    volumes:
      - ./backend/src:/app/src
      - ./backend/tests:/app/tests
    ports:
      - "8000:8000"
    depends_on:
      postgres:
        condition: service_healthy
      redis:
        condition: service_healthy
```

`worker` 使用相同镜像，运行 `taskiq worker infrastructure.taskiq.worker:default_broker --reload`。`postgres` 和 `redis` 提供健康检查，另外两个服务等待它们就绪；数据分别保存在命名卷 `postgres-data` 和 `redis-data`。`--api-port` 修改发布端口，`--dry-run` 只打印文件而不写入。

<a id="matching-env-for-compose"></a>

### Compose 对应的 `.env`

应用在 Compose 网络中访问其他服务时，使用**服务名**作为主机名：

```env
# In backend/.env
POSTGRES_SERVER=postgres
CACHE_REDIS_HOST=redis
RATE_LIMITER_REDIS_HOST=redis
TASKIQ_REDIS_HOST=redis
```

如果还需要从宿主机直接访问 Postgres / Redis，例如使用本地开发工具，可按示例映射 `127.0.0.1:5432:5432`、`127.0.0.1:6379:6379`，以便通过 `localhost` 访问。`127.0.0.1:` 前缀避免无密码开发数据库暴露到其他网络。

<a id="service-reference"></a>

## 服务参考

<a id="api-fastapi-application"></a>
<a id="api--fastapi-application"></a>

### `api`：FastAPI 应用

从 Dockerfile 的 `dev` 阶段构建，运行支持自动重载的 `fastapi dev`。挂载 `./backend/src` 后，代码修改立即生效。

生产部署应生成 `prod` 或 `nginx` 配置，而非修改本地配置。两者均构建 `prod` 阶段，移除源码挂载，使用带 `--workers` 的 `fastapi run`。

<a id="worker-taskiq-worker"></a>
<a id="worker--taskiq-worker"></a>

### `worker`：Taskiq Worker

使用同一镜像，运行 `taskiq worker infrastructure.taskiq.worker:default_broker --reload`。使用 `docker compose up --scale worker=3` 扩容。

<a id="postgres-postgresql-16"></a>
<a id="postgres--postgresql-16"></a>

### `postgres`：PostgreSQL 16

使用 `postgres:16-alpine`，从 `backend/.env` 读取 `POSTGRES_USER`、`POSTGRES_PASSWORD` 和 `POSTGRES_DB`，数据持久化到命名卷 `postgres-data`。

<a id="redis-redis-7"></a>
<a id="redis--redis-7"></a>

### `redis`：Redis 7

用于缓存（`CACHE_REDIS_DB=0`）、限流（`RATE_LIMITER_REDIS_DB=1`）、会话（`SESSION_REDIS_DB=2`）和 Taskiq 消息代理（`TASKIQ_REDIS_DB=3`）。模板使用不同 DB 编号避免相互干扰。

<a id="optional-services"></a>

## 可选服务

按需加入生成的 `docker-compose.yml`。重新生成会丢失自定义内容，预计需要重新生成时，应将新增项放入 `docker-compose.override.yml`。

<a id="migrations-job"></a>

### 迁移作业

应用启动前运行 Alembic 迁移：

```yaml
  migrate:
    build:
      context: .
      dockerfile: backend/Dockerfile
      target: migrate
    env_file:
      - ./backend/.env
    environment:
      POSTGRES_SERVER: postgres
    depends_on:
      postgres:
        condition: service_healthy
```

```bash
docker compose run --rm migrate
```

<a id="initial-setup-job"></a>

### 初始化作业

在新数据库上创建首个管理员和默认等级：

```yaml
  setup:
    build:
      context: .
      dockerfile: backend/Dockerfile
      target: dev
    env_file:
      - ./backend/.env
    environment:
      POSTGRES_SERVER: postgres
    command: python -m scripts.setup_initial_data
    depends_on:
      postgres:
        condition: service_healthy
```

```bash
docker compose run --rm setup
```

<a id="pgadmin"></a>

### pgAdmin

需要数据库 Web 界面时添加：

```yaml
  pgadmin:
    image: dpage/pgadmin4:latest
    restart: unless-stopped
    environment:
      PGADMIN_DEFAULT_EMAIL: admin@example.com
      PGADMIN_DEFAULT_PASSWORD: admin
    ports:
      - "5050:80"
    depends_on:
      - postgres
```

访问 <http://localhost:5050> 并登录，添加服务器：主机 `postgres`，端口 `5432`，用户 `postgres`，或使用 `backend/.env` 中设置的用户。

<a id="memcached-alternative-cache-backend"></a>

### Memcached（替代缓存后端）

更倾向使用 Memcached 时：

```yaml
  memcached:
    image: memcached:1.6-alpine
    ports:
      - "11211:11211"
```

在 `.env` 中设置：

```env
CACHE_BACKEND=memcached
CACHE_MEMCACHED_HOST=memcached
```

原文称限流器始终依赖 Redis，因此仍需要 `redis` 服务。

<a id="rabbitmq-alternative-taskiq-broker"></a>

### RabbitMQ（替代 Taskiq 消息代理）

```yaml
  rabbitmq:
    image: rabbitmq:3.13-management-alpine
    environment:
      RABBITMQ_DEFAULT_USER: ${TASKIQ_RABBITMQ_USER:-guest}
      RABBITMQ_DEFAULT_PASS: ${TASKIQ_RABBITMQ_PASSWORD:-guest}
    ports:
      - "5672:5672"
      - "15672:15672"   # management UI
```

在 `.env` 中设置：

```env
TASKIQ_BROKER_TYPE=rabbitmq
TASKIQ_RABBITMQ_HOST=rabbitmq
```

<a id="common-commands"></a>

## 常用命令

```bash
# from the repository root, where the generated compose file lives

# Bring everything up (foreground, attached logs)
docker compose up

# Detached
docker compose up -d

# Rebuild after dependency changes
docker compose up --build

# Logs for a specific service
docker compose logs -f api

# Open a shell inside the app container
docker compose exec api bash

# Run a one-off command
docker compose exec api uv run --no-sync alembic upgrade head
docker compose exec postgres psql -U postgres
docker compose exec redis redis-cli

# Stop everything
docker compose down

# Stop and wipe volumes (⚠️ deletes data)
docker compose down -v
```

<a id="production-style-setup"></a>

## 接近生产的配置

需要更接近生产的本地服务组合时：

1. **生成 `prod` 配置**：`bp deploy generate prod` 构建 `prod` 阶段，移除源码挂载并运行 `fastapi run --workers`。
2. **应用启动前单独执行迁移**：使用上面的 `migrate` 服务。
3. **设置 Worker 数量**：`bp deploy generate prod --workers 4`，或在 `backend/.env` 中设置 `WORKERS=4`，`prod` 命令会读取它。
4. **需要 TLS 时添加反向代理**：单机部署中，Caddy 或 Traefik 比 nginx 更易配置。

<a id="troubleshooting"></a>

## 故障排查

<a id="container-wont-start"></a>

### 容器无法启动

```bash
docker compose logs api
docker compose build --no-cache api
```

<a id="database-connection-refused"></a>

### 数据库拒绝连接

```bash
# Is the database service up?
docker compose ps postgres

# Can the api container resolve "postgres"?
docker compose exec api python -c "import socket; print(socket.gethostbyname('postgres'))"

# Inspect its logs
docker compose logs postgres
```

<a id="code-changes-not-picking-up"></a>

### 代码修改未生效

确认 `api` 服务挂载了 `./backend/src:/app/src`，并设置 `target: dev`。`dev` 阶段运行开启重载的 `fastapi dev`，`prod` 阶段**不会**自动重载。

<a id="port-already-in-use"></a>

### 端口已被占用

```bash
lsof -i :8000
# or change the host-side port in compose:
ports:
  - "8080:8000"
```

<a id="resetting-everything"></a>

### 全部重置

```bash
docker compose down -v        # wipes volumes
docker compose build --no-cache
docker compose up
```

<a id="best-practices"></a>

## 最佳实践

<a id="development"></a>

### 开发

- 使用 `target: dev` 开启热重载。
- 将 `./src` 挂载为卷，避免每次编辑都重建。
- 暴露 Postgres / Redis 端口，便于本地调试。
- 不提交 `.env`，它已被 `.gitignore` 忽略。

<a id="production"></a>

### 生产

- 使用 `target: prod` 并移除开发卷挂载。
- 启动应用前将 `migrate` 阶段作为独立作业运行。
- 设置 `ENVIRONMENT=production`，启用安全校验器。
- 使用非 root 用户 `appuser`，Dockerfile 已配置。
- 固定镜像标签，例如 `postgres:16-alpine`，而非 `postgres:latest`。

<a id="security"></a>

### 安全

- dev / prod 阶段的容器均以非 root 用户运行。
- 生产环境不向公网暴露 Postgres / Redis 端口。
- 部署前设置强 `POSTGRES_PASSWORD`、Redis 密码（如 `CACHE_REDIS_PASSWORD`）和 `SECRET_KEY`。

<a id="see-also"></a>

## 另请参阅

- **[环境变量](environment-variables.md)**：完整变量参考。
- **[配置类](settings-classes.md)**：变量如何转换为 Python 配置。
- **[生产环境](../production.md)**：生产部署指南。
