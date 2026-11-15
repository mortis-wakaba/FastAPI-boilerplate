<a id="neon-serverless-postgres"></a>

# Neon（无服务器 Postgres）

[Neon](https://neon.com) 是无服务器 Postgres：托管数据库空闲时可将计算资源缩减至零，也能像代码一样创建分支。它可直接替代本地 Postgres 容器，模板只需设置**一个环境变量**，无需改代码。

!!! info "Neon 赞助本项目"
    Neon 为 Fastro · Benav Labs FastAPI Boilerplate 的开源基础设施提供数据库额度。模板可运行于任意 Postgres 14+，包括本地、RDS、Cloud SQL、Supabase 或自建实例。Neon 只是其中一种，也是项目作者实际使用和验证本指南的服务。

<a id="when-its-a-good-fit"></a>

## 适用场景

- **不希望运行 Postgres 容器**：无需为数据库启动 Docker Compose，也不必重置数据卷。
- **预览环境**：Neon 分支是数据的写时复制克隆，数秒创建，可为每个 PR、开发者或测试运行分配独立数据库。
- **突发或低流量 API**：空闲时计算暂停，无人使用的周末预发布环境无需持续消耗计算资源。

需要离线开发或完全自托管时，继续使用随附 Postgres 容器。

<a id="what-it-costs"></a>

## 费用

提供**非试用的永久免费方案**，无需信用卡、没有到期时间，可满足模板的开发、预发布和个人项目。原文编写时，每项目额度为：

| 项目 | 免费方案 |
|------|----------|
| 存储 | 0.5 GB |
| 计算 | 每月 100 CU 小时，自动扩容最高 2 CU（8 GB RAM） |
| 项目 / 分支 | 100 个项目，每项目 10 个分支 |
| 出站流量 | 5 GB |
| 所有方案均包含 | 自动扩缩容、分支、只读副本、连接池、扩展、API 和 CLI |

计算可缩减至零，因此空闲预发布数据库几乎不消耗 CU 小时，主要在查询时计量。

!!! warning "达到额度后"
    超过月度额度会**暂停计算直到下个计费月**，而不是产生额外账单。这适合个人项目，但对重要服务有风险。生产环境应采用付费方案，或至少设置用量告警。当前额度请查看 [neon.com/pricing](https://neon.com/pricing)，以上是原文记录值，可能变化。

<a id="1-create-a-project"></a>

## 1. 创建项目

1. 在 [neon.com](https://neon.com) 注册并创建项目，选择最接近 API 部署位置的区域，所有查询都会承担该往返延迟。
2. 在控制台打开 **Connection Details（连接详情）**，复制类似以下形式的连接字符串：

```text
postgresql://neondb_owner:npg_xxxxxxxx@ep-cool-darkness-123456-pooler.us-east-2.aws.neon.tech/neondb?sslmode=require&channel_binding=require
```

<a id="2-convert-it-for-asyncpg"></a>

## 2. 转换为 asyncpg 格式

Neon 提供的连接字符串面向 **libpq**（psql、psycopg），模板使用 **asyncpg**，选项名称不同，需要两处修改：

| Neon 原值 | 替换为 | 原因 |
|-----------|--------|------|
| `postgresql://` | `postgresql+asyncpg://` | 选择 SQLAlchemy 异步驱动 |
| `?sslmode=require&channel_binding=require` | `?ssl=require` | SQLAlchemy 将未知查询参数直接传入 `asyncpg.connect()`，后者没有 `sslmode` 和 `channel_binding` 参数，会触发 `TypeError: connect() got an unexpected keyword argument 'sslmode'`，正确参数名为 `ssl` |

在 `backend/.env` 中设置：

```env
DATABASE_URL=postgresql+asyncpg://neondb_owner:npg_xxxxxxxx@ep-cool-darkness-123456-pooler.us-east-2.aws.neon.tech/neondb?ssl=require

# Let Alembic own the schema instead of creating tables at boot
CREATE_TABLES_ON_STARTUP=false
```

`DATABASE_URL` 优先于各项 `POSTGRES_*` 配置（见[配置参考](../configuration/environment-variables.md#database)），设置后其他值均忽略，包括 Compose 注入的 `POSTGRES_SERVER: postgres`。可保留它们的默认值。[生产校验器](../production.md#the-production-validator)会从 URL 提取凭据，因此不会把默认 `POSTGRES_PASSWORD` 误判为不安全部署，无需修改其他配置。

!!! warning "不要省略 `ssl` 参数"
    TCP 连接中，asyncpg 默认使用 `sslmode=prefer`：服务端支持时使用 TLS，否则静默接受明文连接，且不校验证书。明确设置 `ssl=require` 可使降级明确失败。

    与 libpq 相同，`require` 只加密连接，**不校验服务端证书**。需要完整验证时使用 `?ssl=verify-full`，并将 `PGSSLROOTCERT` 指向 CA 证书包；否则 asyncpg 查找 `~/.postgresql/root.crt`，找不到会失败。

<a id="3-run-migrations-and-start"></a>

## 3. 执行迁移并启动

```bash
cd backend
uv run --no-sync alembic upgrade head
uv run --no-sync python -m scripts.setup_initial_data   # first admin user + default tier
uv run --no-sync fastapi dev src/interfaces/main.py
```

Alembic 与应用读取同一个 `settings.DATABASE_URL`，见 `backend/migrations/env.py`，无需维护第二个连接字符串。

<a id="pooled-vs-direct-endpoints"></a>

## 连接池端点与直连端点

Neon 为每个项目提供两个主机名，区别在于 `-pooler` 后缀：

```text
ep-cool-darkness-123456-pooler.us-east-2.aws.neon.tech   # pooled — via PgBouncer
ep-cool-darkness-123456.us-east-2.aws.neon.tech          # direct — straight to Postgres
```

**API 和 Taskiq Worker 使用连接池端点**。请求和任务的连接开销正是 PgBouncer 适合处理的场景，它也能提高可用连接上限。

**迁移和管理操作使用直连端点**。Neon 的 PgBouncer 使用事务模式，会话状态不会在语句之间保留，因此 `SET` / `RESET`（包括 `SET search_path`）、`LISTEN` / `NOTIFY`、SQL 级 `PREPARE` 和会话级 advisory lock 的行为不同。包含多语句的长 DDL 使用直连更稳妥：

```bash
# One-off override for the migration only
DATABASE_URL="postgresql+asyncpg://neondb_owner:npg_xxxxxxxx@ep-cool-darkness-123456.us-east-2.aws.neon.tech/neondb?ssl=require" \
  uv run --no-sync alembic upgrade head
```

<a id="handling-scale-to-zero"></a>

## 处理空闲缩容

Neon 计算空闲后暂停，SQLAlchemy 池中的连接在唤醒后可能失效，下个请求可能报 `SSL SYSCALL error: EOF detected` 或 `connection was closed in the middle of operation`。

`POSTGRES_POOL_PRE_PING` **默认开启**，每次将连接交给请求前先检查，自动替换失效连接。配合 `POSTGRES_POOL_RECYCLE`，可在 Neon 关闭前回收连接：

```env
POSTGRES_POOL_RECYCLE=300     # seconds; keep it under your scale-to-zero timeout
```

也应缩小连接池。`POSTGRES_POOL_SIZE` 默认每进程 20，实际总量为 `pool_size × workers × replicas`，见[扩容说明](../production.md#database)。小型 Neon 计算实例通常 5–10 已足够：

```env
POSTGRES_POOL_SIZE=10
POSTGRES_MAX_OVERFLOW=5
```

Taskiq Worker 使用 `NullPool`，每个任务创建新连接，不存在池内连接陈旧的问题，无需上述配置。

<a id="docker-compose-without-the-local-database"></a>

## 不使用本地数据库的 Docker Compose

`DATABASE_URL` 指向 Neon 后，生成的 Compose 中 `postgres` 已无用途，可删除服务、数据卷，以及等待该服务的两处 `depends_on`：

```yaml hl_lines="8 9"
services:
  api:
    env_file:
      - ./backend/.env
    environment:
      CACHE_REDIS_HOST: redis
    depends_on:
      # postgres:                  ← remove
      #   condition: service_healthy
      redis:
        condition: service_healthy
```

Redis 仍本地运行，Neon 只替代 Postgres。重新执行 `uv run --no-sync bp deploy generate local` 会恢复 Postgres 服务，重新生成后需留意这项修改。

<a id="database-branching-for-previews"></a>

## 为预览创建数据库分支

分支是选择 Neon 相比普通托管 Postgres 的主要理由。它是完整数据的写时复制克隆，创建和销毁都只需数秒：

```bash
# One database per pull request
neon branches create --name pr-142 --parent main

# Print a connection string for it
neon connection-string pr-142
```

将连接字符串按步骤 2 转换后，传给预览环境的 `DATABASE_URL`，因为 CLI 输出 libpq 格式。预览环境即可使用真实且隔离的数据，PR 合并后删除分支。需要真实 Postgres 的集成测试也可采用：创建分支、迁移、测试、删除。

<a id="troubleshooting"></a>

## 故障排查

| 现象 | 原因 | 解决方式 |
|------|------|----------|
| `TypeError: connect() got an unexpected keyword argument 'sslmode'` | URL 残留 libpq 参数 | 将 `sslmode=require&channel_binding=require` 替换为 `ssl=require` |
| `TypeError: connect() got an unexpected keyword argument 'channel_binding'` | 同上 | 同上 |
| `ProductionSecurityError: Database is using default credentials` | URL 中的密码确实为 `postgres` | 在 Neon 控制台更换密码并更新 URL |
| 空闲后出现 `SSL SYSCALL error: EOF detected` | 计算暂停使池内连接失效 | 保持 `POSTGRES_POOL_PRE_PING=true` 并设置 `POSTGRES_POOL_RECYCLE` |
| 空闲后首次请求需数百毫秒 | 计算从零恢复 | 属预期行为，延迟比成本重要时可禁用该分支的空闲暂停 |
| `prepared statement "__asyncpg_stmt_x__" already exists` | 事务池中的预备语句复用 | URL 添加 `&prepared_statement_cache_size=0`，或使用直连端点 |
| 密码正确却报 `password authentication failed` | 特殊字符未做 URL 编码 | 使用百分号编码，如 `@` → `%40`、`#` → `%23` |

<a id="related"></a>

## 相关内容

- [迁移](migrations.md)：Alembic 工作流与生产确认。
- [环境变量](../configuration/environment-variables.md#database)：全部数据库配置。
- [生产部署](../production.md)：校验器、连接池大小和扩容。
- [Neon 文档](https://neon.com/docs)：分支、自动扩缩容和 `neon` CLI。
