<a id="installation-guide"></a>

# 安装指南

本指南介绍如何根据需求选择 FastAPI Boilerplate 的安装方式。

<a id="system-requirements"></a>

## 系统要求

- **Python**：3.11 或更高版本。
- **操作系统**：Linux、macOS 或 Windows（推荐 WSL2）。
- **内存**：至少 4 GB，推荐 8 GB。
- **磁盘空间**：约 2 GB 可用空间。

<a id="method-1-docker-compose-recommended"></a>

## 方式一：Docker Compose（推荐）

Docker Compose 会管理全部服务：Postgres、Redis 和应用。

<a id="prerequisites"></a>

### 前置条件

- [Docker](https://docs.docker.com/get-docker/)（20.10+）
- [Docker Compose](https://docs.docker.com/compose/install/)（v2+）
- [uv](https://docs.astral.sh/uv/getting-started/installation/)，用于运行生成 Compose 文件的 CLI。

<a id="steps"></a>

### 步骤

1. **克隆仓库**：

    ```bash
    git clone https://github.com/benavlabs/fastapi-boilerplate
    cd fastapi-boilerplate
    ```

2. **安装 workspace**：

    ```bash
    uv sync --all-packages --all-extras
    ```

3. **生成 Compose 文件**：

    ```bash
    uv run --no-sync bp deploy generate local
    ```

    `docker-compose.yml` 生成在仓库根目录，不提交到版本控制，因为所需服务组合取决于部署方式。`local` 提供热重载开发环境，包含 API、Taskiq Worker、调度器、Postgres 和 Redis。其他模式为 `prod` 和 `nginx`，见 [CLI → 命令](../cli/commands.md)。

4. **设置环境文件**：

    ```bash
    cp backend/.env.example backend/.env
    # Edit backend/.env: set SECRET_KEY (`uv run --no-sync bp env gen-secret` prints one),
    # change ADMIN_PASSWORD and the default DB password.
    ```

5. 在仓库根目录**启动服务**：

    ```bash
    docker compose up
    ```

6. **验证运行状态**：

    ```bash
    curl http://localhost:8000/health
    ```

<a id="what-gets-installed"></a>

### 安装的服务

- **应用**（FastAPI + Uvicorn）：8000 端口。
- **PostgreSQL**：5432 端口。
- **Redis**：6379 端口。

<a id="method-2-manual-installation-local-with-uv"></a>

## 方式二：手动安装（使用 uv 在本地运行）

适用于在宿主机上开发，需要自行提供 PostgreSQL 和 Redis。

<a id="prerequisites_1"></a>

### 前置条件

1. **Python 3.11+**：

    ```bash
    # Ubuntu/Debian
    sudo apt update && sudo apt install python3.11

    # macOS
    brew install python@3.11

    # Windows: download from python.org
    ```

2. **uv**（快速的 Python 包管理器）：

    ```bash
    # macOS / Linux
    curl -LsSf https://astral.sh/uv/install.sh | sh

    # or via pip
    pip install uv
    ```

3. **PostgreSQL**：

    ```bash
    # Ubuntu/Debian
    sudo apt install postgresql postgresql-contrib

    # macOS
    brew install postgresql
    ```

4. **Redis**：

    ```bash
    # Ubuntu/Debian
    sudo apt install redis-server

    # macOS
    brew install redis
    ```

<a id="steps_1"></a>

### 步骤

1. **克隆仓库**：

    ```bash
    git clone https://github.com/benavlabs/fastapi-boilerplate
    cd fastapi-boilerplate
    ```

    仓库采用 [uv workspace](https://docs.astral.sh/uv/concepts/projects/workspaces/)，可部署应用位于 `backend/`，开发者 CLI（`bp`）位于 `cli/`。两者共用仓库根目录的一个虚拟环境。

2. **安装 Python 依赖**：

    ```bash
    uv sync --all-packages --all-extras
    ```

    这会把整个 workspace（后端、CLI 和开发工具）同步到仓库根目录下的 `.venv/`。之后可在任意子目录运行 `uv run --no-sync <cmd>`；该参数跳过 uv 隐式执行的依赖同步，使命令立即启动，并保持环境和锁文件不变。如果重新运行不带 `--all-extras` 的 `uv sync`，ruff、mypy 和 pytest 将再次被移除。

3. **设置环境变量**：

    ```bash
    cp backend/.env.example backend/.env
    # Edit backend/.env: point POSTGRES_SERVER and CACHE_REDIS_HOST at localhost,
    # set a strong SECRET_KEY (or run `uv run --no-sync bp env gen-secret`), etc.
    ```

    CLI 提供了辅助命令，见 [CLI → 命令](../cli/commands.md)：

    ```bash
    uv run --no-sync bp env gen-secret      # print a fresh SECRET_KEY
    uv run --no-sync bp env validate        # audit the .env against the production validator
    ```

4. **设置 PostgreSQL**（尚未配置时）：

    ```bash
    sudo -u postgres psql
    CREATE DATABASE myapp;
    CREATE USER myuser WITH PASSWORD 'mypassword';
    GRANT ALL PRIVILEGES ON DATABASE myapp TO myuser;
    \q
    ```

5. **执行数据库迁移**：在 `backend/` 下运行，Alembic 会在当前工作目录查找 `alembic.ini`。

    ```bash
    cd backend
    uv run --no-sync alembic upgrade head
    ```

6. **创建初始管理员和用户等级**：

    ```bash
    uv run --no-sync python -m scripts.setup_initial_data
    ```

7. **启动应用**：

    ```bash
    uv run --no-sync fastapi dev src/interfaces/main.py
    ```

8. **可选：启动 Taskiq Worker**，在第二个终端运行：

    ```bash
    uv run --no-sync taskiq worker src.infrastructure.taskiq.worker:default_broker
    ```

<a id="method-3-development-setup"></a>

## 方式三：开发环境配置

适用于贡献者以及需要修改模板本身的开发者。

<a id="steps_2"></a>

### 步骤

1. **Fork 并克隆仓库**：

    ```bash
    # Fork on GitHub first
    git clone https://github.com/yourusername/fastapi-boilerplate
    cd fastapi-boilerplate
    ```

2. **安装开发依赖**：

    ```bash
    uv sync --all-packages --all-extras
    ```

3. **安装 pre-commit 钩子**，在仓库根目录运行：

    ```bash
    uv run --no-sync pre-commit install
    ```

4. **设置环境**：

    ```bash
    cp backend/.env.example backend/.env
    ```

5. **运行测试套件验证配置**：

    ```bash
    cd backend && uv run --no-sync pytest
    ```

<a id="configuration"></a>

## 配置

<a id="environment-variables"></a>

### 环境变量

应用从 `backend/.env` 读取配置，完整参考见 `backend/.env.example`。常用变量如下：

```env
# Environment
ENVIRONMENT=development

# Database
POSTGRES_USER=postgres
POSTGRES_PASSWORD=postgres
POSTGRES_DB=postgres
POSTGRES_SERVER=postgres  # use "localhost" without Docker
POSTGRES_PORT=5432

# Cache (Redis)
CACHE_BACKEND=redis
CACHE_REDIS_HOST=redis    # use "localhost" without Docker
CACHE_REDIS_PORT=6379

# Security
SECRET_KEY=insecure-change-this-in-production

# Initial admin
ADMIN_NAME=Admin User
ADMIN_EMAIL=admin@example.com
ADMIN_USERNAME=admin
ADMIN_PASSWORD=Change-Me-Now-1!
```

完整参考见[环境变量](../user-guide/configuration/environment-variables.md)。

<a id="using-a-hosted-database-instead"></a>

### 使用托管数据库

无需自行运行 Postgres。设置 `DATABASE_URL` 后，它会覆盖上面所有 `POSTGRES_*` 变量，本地容器或本地安装就不再必需：

```env
DATABASE_URL=postgresql+asyncpg://user:password@host.example.com/dbname?ssl=require
```

[Neon](../user-guide/database/neon.md) 是本项目的赞助方，提供无服务器 Postgres，可免费起步，免费方案长期有效且无需信用卡。对应指南包含详细步骤，以及 asyncpg 所需的连接字符串转换。其他托管 Postgres 的使用方式相同。

<a id="verification"></a>

## 验证

安装后检查以下项目：

1. **API 文档**：<http://localhost:8000/docs>
2. **健康检查**：`curl http://localhost:8000/health`
3. **管理后台**：<http://localhost:8000/admin>
4. **数据库**：检查应用日志中的 `Database connected`，以及启动时是否出现错误。

<a id="troubleshooting"></a>

## 故障排查

<a id="port-already-in-use"></a>

### 端口已被占用

```bash
# Find what's on port 8000
lsof -i :8000
kill -9 <PID>
```

<a id="database-connection-error"></a>

### 数据库连接错误

```bash
# Linux
sudo systemctl status postgresql
sudo systemctl restart postgresql

# macOS (Homebrew)
brew services restart postgresql
```

<a id="redis-connection-error"></a>

### Redis 连接错误

```bash
redis-cli ping  # should print PONG
# If not, start it:
redis-server
```

<a id="docker-clean-reset"></a>
<a id="docker--clean-reset"></a>

### Docker：清理并重建

在 `docker-compose.yml` 所在的仓库根目录执行：

```bash
docker compose down       # stop containers
docker compose down -v    # also delete volumes (⚠️ wipes data)
docker compose build --no-cache
docker compose up
```

<a id="next-steps"></a>

## 后续步骤

1. **[配置指南](configuration.md)**：配置运行环境。
2. **[首次运行](first-run.md)**：测试安装结果。
3. **[项目结构](../user-guide/project-structure.md)**：了解代码库。

<a id="need-help"></a>

## 获取帮助

- 在 [GitHub Issues](https://github.com/benavlabs/fastapi-boilerplate/issues) 查找已知问题。
- 创建[新 Issue](https://github.com/benavlabs/fastapi-boilerplate/issues/new)，提供详细信息。
