<a id="getting-started"></a>

# 入门指南

欢迎使用 FastAPI Boilerplate！本指南帮助你在几分钟内启动项目。

<a id="quick-start-5-minutes"></a>

## 快速开始（5 分钟）

选择适合自己的方式：

=== "使用 uv 在本地运行"

    <a id="prerequisites"></a>

    ### 前置条件

    - [uv](https://docs.astral.sh/uv/getting-started/installation/)（0.4+）
    - 本地运行 PostgreSQL 和 Redis，或改用 Docker 选项卡中的方式。

    <a id="1-get-the-template"></a>

    ### 1. 获取模板

    1. 在 [GitHub 仓库](https://github.com/benavlabs/fastapi-boilerplate)中点击 **Use this template（使用此模板）**。
    2. 以自己的项目名创建新仓库。
    3. 克隆新仓库，并使用 `cd` 进入后端目录：

    ```bash
    git clone https://github.com/yourusername/your-project-name
    cd your-project-name/backend
    ```

    <a id="2-install-dependencies"></a>

    ### 2. 安装依赖

    ```bash
    uv sync --all-packages --all-extras
    ```

    在仓库根目录执行时，`--all-packages` 安装两个 workspace 成员，`--all-extras` 添加开发工具（ruff、mypy、pytest）。

    <a id="3-environment-setup"></a>

    ### 3. 配置环境

    ```bash
    cp .env.example .env
    # then edit .env to set your database creds, SECRET_KEY, etc.
    ```

    !!! warning "SECRET_KEY"
        使用 `python -c "import secrets; print(secrets.token_urlsafe(64))"` 生成安全密钥，替换 `.env` 中的默认值。

    <a id="4-run-the-server"></a>

    ### 4. 启动服务

    ```bash
    uv run --no-sync fastapi dev src/interfaces/main.py
    ```

=== "Docker Compose"

    <a id="prerequisites_1"></a>

    ### 前置条件

    - [Docker](https://docs.docker.com/get-docker/)（20.10+）
    - [Docker Compose](https://docs.docker.com/compose/install/)（v2+）
    - [uv](https://docs.astral.sh/uv/getting-started/installation/)（0.4+），用于运行生成 Compose 文件的 CLI。

    <a id="1-get-the-template_1"></a>

    ### 1. 获取模板

    ```bash
    git clone https://github.com/yourusername/your-project-name
    cd your-project-name
    ```

    <a id="2-generate-the-compose-file"></a>

    ### 2. 生成 Compose 文件

    ```bash
    uv sync --all-packages --all-extras
    uv run --no-sync bp deploy generate local
    ```

    `docker-compose.yml` 生成在仓库根目录，不提交到版本控制，因为所需服务组合取决于部署方式。`local` 提供热重载开发环境；另外两种模式为 `prod` 和 `nginx`，见 [CLI → 命令](../cli/commands.md)。

    <a id="3-environment-setup_1"></a>

    ### 3. 配置环境

    ```bash
    cp backend/.env.example backend/.env
    # then edit backend/.env to set your DB password, SECRET_KEY, etc.
    ```

    <a id="4-start-everything"></a>

    ### 4. 启动所有服务

    在仓库根目录运行：

    ```bash
    docker compose up
    ```

    这会启动：

    - 监听 8000 端口的 **FastAPI 应用**。
    - **PostgreSQL** 数据库。
    - 用于缓存、限流和会话的 **Redis**。

<a id="verify-its-running"></a>

## 验证运行状态

访问：

- **Swagger UI**：[http://localhost:8000/docs](http://localhost:8000/docs)
- **ReDoc**：[http://localhost:8000/redoc](http://localhost:8000/redoc)
- **健康检查**：[http://localhost:8000/health](http://localhost:8000/health)

<a id="youre-ready"></a>

## 准备就绪

现在你已经有一个可运行的 FastAPI 应用，包含：

- 自动生成 OpenAPI 文档的 REST API。
- 使用 Alembic 迁移的 PostgreSQL 数据库。
- 基于 Redis 的缓存和限流。
- 基于会话的认证，以及可选的 OAuth（Google、GitHub）。
- 支持独立权限配置的 API 密钥。
- 位于 `/admin` 的 SQLAdmin 管理界面。
- 基于 Taskiq 的异步后台任务支持。

<a id="test-your-api"></a>

## 测试 API

<a id="1-health-check"></a>

### 1. 健康检查

```bash
curl http://localhost:8000/health
```

<a id="2-create-a-user"></a>

### 2. 创建用户

```bash
curl -X POST "http://localhost:8000/api/v1/users/" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "John Doe",
    "username": "johndoe",
    "email": "john@example.com",
    "password": "securepassword"
  }'
```

<a id="3-log-in-session-cookie"></a>

### 3. 登录（会话 Cookie）

```bash
curl -X POST "http://localhost:8000/api/v1/auth/login" \
  -H "Content-Type: application/x-www-form-urlencoded" \
  -d "username=johndoe&password=securepassword" \
  -c cookies.txt
```

响应会设置 HttpOnly 会话 Cookie，并返回 CSRF 令牌。后续请求使用 `-b cookies.txt` 携带会话。

<a id="4-get-the-current-user"></a>

### 4. 获取当前用户

```bash
curl http://localhost:8000/api/v1/users/me -b cookies.txt
```

<a id="next-steps"></a>

## 后续步骤

<a id="essential-reading"></a>

### 基础阅读

- **[配置指南](configuration.md)**：环境变量与设置。
- **[项目结构](../user-guide/project-structure.md)**：代码组织方式。
- **[身份认证](../user-guide/authentication/index.md)**：会话、OAuth 和 API 密钥。

<a id="popular-features"></a>

### 常用功能

- **[数据库操作](../user-guide/database/index.md)**：模型、Schema 和 CRUD。
- **[缓存](../user-guide/caching/index.md)**：基于 Redis 的缓存。
- **[后台任务](../user-guide/background-tasks/index.md)**：基于 Taskiq 的异步任务。
- **[限流](../user-guide/rate-limiting/index.md)**：防止 API 被滥用。

<a id="development-deployment"></a>
<a id="development--deployment"></a>

### 开发与部署

- **[开发指南](../user-guide/development.md)**：扩展与定制。
- **[测试](../user-guide/testing.md)**：为 API 编写测试。
- **[生产部署](../user-guide/production.md)**：部署到生产环境。

<a id="alternative-setup-methods"></a>

## 其他安装方式

- **[手动安装](installation.md)**：详细的分步安装说明。

<a id="need-help"></a>

## 获取帮助

- 加入 **[Discord 社区](../community.md)**，向其他开发者求助。
- 在 **[GitHub](https://github.com/benavlabs/fastapi-boilerplate/issues)** 上反馈问题。

---

**准备深入了解？** 继续阅读[安装指南](installation.md)，或直接进入[用户指南](../user-guide/index.md)。
