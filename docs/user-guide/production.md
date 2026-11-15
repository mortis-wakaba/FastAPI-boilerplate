<a id="production-deployment"></a>

# 生产部署

本页是 FastAPI Boilerplate 的生产加固指南，介绍内置校验器、环境变量、多阶段 Dockerfile，以及部署时需要确定的运维策略。

<a id="the-production-validator"></a>

## 生产校验器

`ENVIRONMENT=production` 时，`infrastructure/security/production_validator.py` 在启动运行。分两级：

<a id="critical-raises-productionsecurityerror-app-exits"></a>

### 严重问题：抛 ProductionSecurityError，退出

以下情况**不会启动**：

- **SECRET_KEY 不像随机值**：为空、少于 32 字符，或大部分由占位词 / 手写词（password、secret、test、dev、default 等）、重复短块（prodprod…）、相邻键盘序列（qwertyuiop、1qaz2wsx 等）、可识别短语（如 MyCompanyApiSigningKeyForProd2026）组成。按整值比例衡量，因此随机密钥偶然包含单词仍可通过。两条绝对规则是连续八个码点、熵低于 64 位。原文测试以固定种子分别扫描生成器四种格式各十万个值：`secrets.token_hex(16/32)` 和 `secrets.token_urlsafe(24/32)`，四十万个都未拒绝。但随机值仍可能碰规则，原文估计十六进制连续八位数字约每 1260 万个一次，urlsafe 键盘序列约每千万个一次。项目内 `bp env gen-secret` 每次生成后校验，不合格重新生成。
- **数据库密码为 postgres**，公开默认值容易被尝试。
- **数据库密码为空**。
- **启用后台却缺少 ADMIN_USERNAME 或 ADMIN_PASSWORD**。
- **CORS_ORIGINS 含 `*`**。任意网站可从用户浏览器调用 API。应用会关闭 CORS_ALLOW_CREDENTIALS，返回 `Access-Control-Allow-Origin: *` 而不允许凭据，因此其他来源不能读取带 Cookie 的响应。但不需预检的请求仍发送，在 SameSite 允许时仍带 Cookie，所以通配源既不能保护接口，也不能支持需登录前端。

检查的是实际连接密码：设置 DATABASE_URL 时从 URL 读取，否则用 POSTGRES_PASSWORD。URL 完全没有密码，如 IAM / 证书认证，只告警，因为无法在这里验证。

<a id="warnings-logged-app-starts"></a>

### 警告：记录后继续启动

真实流量前应修复：

- Redis 未设密码，包括 CACHE_REDIS_PASSWORD、RATE_LIMITER_REDIS_PASSWORD 或无密码 SESSION_REDIS_URL。
- DEBUG=true，可能暴露堆栈。
- API 文档可访问，见下文。
- 会话过于宽松，如未 Secure、过长有效期。
- 可预测或常见后台凭据。

校验器只捕获常见错误，不能替代完整威胁分析，应视为部署冒烟检查。

<a id="production-env-checklist"></a>

## 生产 .env 清单

从 `backend/.env.example` 创建，至少修改：

```env
# Environment
ENVIRONMENT=production
DEBUG=false

# App
APP_NAME="Your Production App"
VERSION=1.0.0

# Secrets — generate a fresh, unique value
SECRET_KEY=<openssl rand -hex 32>

# Database — never use defaults
POSTGRES_USER=app_prod
POSTGRES_PASSWORD=<long-random-secret>
POSTGRES_SERVER=<your-db-host>
POSTGRES_PORT=5432
POSTGRES_DB=app_prod

# Auto-creating tables in prod is dangerous; use Alembic instead
CREATE_TABLES_ON_STARTUP=false

# Migrations: must be opted into, even with the right env
CONFIRM_PRODUCTION_MIGRATION=yes      # only when actively running migrations

# CORS — list the exact origins that can call your API
CORS_ORIGINS=["https://app.example.com","https://admin.example.com"]

# Cache (Redis or Memcached)
CACHE_ENABLED=true
CACHE_BACKEND=redis
CACHE_REDIS_HOST=<redis-host>
CACHE_REDIS_PASSWORD=<redis-password>

# Sessions
SESSION_BACKEND=redis                  # redis | database | memory; Redis unless SESSION_REDIS_URL is set
SESSION_SECURE_COOKIES=true            # required when serving over HTTPS
CSRF_ENABLED=true
TRUSTED_PROXY_HOPS=1                    # set to the number of proxies in front of the app
FORWARDED_ALLOW_IPS=<proxy-subnet>     # which peers uvicorn takes forwarded headers from

# Rate limiting (Redis-backed, provided by crudauth)
RATE_LIMITER_ENABLED=true
RATE_LIMITER_REDIS_HOST=<redis-host>
RATE_LIMITER_REDIS_PASSWORD=<redis-password>

# Taskiq
TASKIQ_BROKER_TYPE=redis
TASKIQ_REDIS_HOST=<redis-host>
TASKIQ_REDIS_PASSWORD=<redis-password>

# Admin panel
ADMIN_ENABLED=false                    # safest default in prod
ADMIN_USERNAME=<unique-username>
ADMIN_PASSWORD=<long-random-secret>

# Documentation
OPENAPI_URL=                           # disable /docs and /redoc

# Logging
LOG_LEVEL=INFO
LOG_FORMAT=json
```

说明：

- **EMAIL_BACKEND=smtp**：生产拒绝 console，否则完整重置 / 验证链接写日志而非投递。FRONTEND_URL 也必须可供收件人访问。无 accounts 功能时跳过这些账户邮件检查。
- **CREATE_TABLES_ON_STARTUP=false**：local / development 之外默认关闭，生产拒绝 true，结构变更走 Alembic，不在每次启动 create_all。
- **会话和限流后端不能 memory**：生产拒绝。进程本地会话重启丢失且多进程不共享；本地计数使限额随 Worker 倍增。该后端还承担登录锁定、账户变更预算、密钥窗口，无论是否选择 ratelimit 或开启 API 限流。
- **FRONTEND_URL 必须 HTTPS**，不能是仅部署自身可达主机，否则验证、重置和改邮箱链接无效，明文链路也会暴露令牌。
- **RabbitMQ 拒绝 guest / guest**：使用该代理时保留默认凭据，任何能连接的人都可能入队 Worker 执行的任务。
- **CONFIRM_PRODUCTION_MIGRATION=yes**：`migrations/env.py` 的 validate_production_migration 要求明确确认。写在部署命令，不放长期环境文件。
- **SESSION_SECURE_COOKIES=true**：仅 HTTPS 发送 Cookie，代理终止 TLS 时需要。
- **OPENAPI_URL=** 留空：完全关闭 Swagger 和 OpenAPI，生产开放会告警。

环境矩阵见[分环境配置](configuration/environment-specific.md)。

<a id="generating-a-strong-secret_key"></a>

## 生成强 SECRET_KEY

```bash
openssl rand -hex 32
```

或：

```bash
python -c "import secrets; print(secrets.token_hex(32))"
```

不复用开发密钥，不提交生产密钥。部署时从 AWS Secrets Manager、Vault、Doppler 等获取；磁盘 .env 的管理还需考虑审计。

<a id="the-production-dockerfile"></a>

## 生产 Dockerfile

多阶段 `backend/Dockerfile`：

| 阶段 | 用途 |
|------|------|
| requirements-stage | 从 uv.lock 导出锁定依赖 |
| base | 复制源码、安装生产依赖 |
| dev | 开发依赖、测试，运行 fastapi dev |
| migrate | 执行 alembic upgrade head 后退出 |
| prod | fastapi run，可配置 Worker |

构建生产镜像：

```bash
docker build --target prod -t myapp-api:1.0.0 -f backend/Dockerfile .
```

一次性迁移构建并运行 migrate 镜像：

```bash
docker build --target migrate -t myapp-migrate:1.0.0 -f backend/Dockerfile .

docker run --rm \
    --env-file backend/.env.production \
    -e CONFIRM_PRODUCTION_MIGRATION=yes \
    myapp-migrate:1.0.0
```

prod 的 CMD：

```dockerfile
CMD ["sh", "-c", "fastapi run interfaces/main.py --host 0.0.0.0 --port 8000 --workers $WORKERS"]
```

原文推荐 `fastapi run`，它封装 uvicorn，使用无 reload 等生产默认行为。通过 WORKERS 覆盖默认 1：

```bash
docker run -d \
    --env-file .env.production \
    -e WORKERS=4 \
    -p 8000:8000 \
    myapp-api:1.0.0
```

<a id="picking-a-worker-count"></a>

### 选择 Worker 数量

原文给出的粗略起点是 I/O 负载 `2 × CPU 核数 + 1`，CPU 密集更少，应按实际负载测量。每 Worker 独立进程，不共享内存，数据库池也独立，扩进程时应降低 POSTGRES_POOL_SIZE。

多数 API 不必额外使用 gunicorn，fastapi run / uvicorn 可管理进程，编排层使用 Kubernetes、ECS、systemd 或 supervisord。

<a id="running-the-background-worker"></a>

## 后台 Worker

生产使用独立容器 / 服务：

```bash
docker run -d \
    --env-file .env.production \
    --target base \
    myapp-api:1.0.0 \
    sh -c "taskiq worker src.infrastructure.taskiq.worker:default_broker --workers 4"
```

Kubernetes / ECS 作为独立 Deployment / Service 扩容，不接 HTTP，只消费代理。

任务含 schedule 标签时还需**一个且仅一个**调度器，多个会重复入队：

```bash
docker run -d \
    --env-file .env.production \
    --target base \
    myapp-api:1.0.0 \
    sh -c "taskiq scheduler src.infrastructure.taskiq.scheduler:scheduler"
```

`--workers <N>` 控制进程数，`--max-async-tasks <N>` 控制每进程异步并发。完整配置见[后台任务](background-tasks/index.md)。

<a id="database-migrations-in-production"></a>

## 生产数据库迁移

每次 Alembic 都由 `backend/migrations/env.py` 调用生产迁移校验：

```bash
# Will FAIL — refuses to run without confirmation
CONFIRM_PRODUCTION_MIGRATION=  alembic upgrade head

# OK — explicitly confirmed
CONFIRM_PRODUCTION_MIGRATION=yes alembic upgrade head
```

迁移应是明确部署步骤，而非常规启动命令：

1. 构建新镜像。
2. 带 `CONFIRM_PRODUCTION_MIGRATION=yes` 构建并运行 migrate。
3. **然后**部署 API 容器。

先部署后迁移可能让新代码访问旧结构。零停机变更分 expand / contract 两阶段，见[迁移](database/migrations.md)。

<a id="tls-reverse-proxy-and-cors"></a>

## TLS、反向代理与 CORS

模板不终止 TLS，由 Nginx、Caddy、ALB、Cloud Run 等负责。常见结构：

```text
[Client] → HTTPS → [Reverse Proxy] → HTTP → [API container]
                                  → HTTP → [API container]
                                  → HTTP → [API container]
```

代理需要转发 `X-Forwarded-Proto: https` 和真实客户端 X-Forwarded-For，保留 Set-Cookie，并正确设置 Host。是否信任转发头还需以下配置。

`TRUSTED_PROXY_HOPS` 为前方代理数量，单 nginx / Caddy 为 1，前面还有 Cloudflare 为 2。crudauth 从 X-Forwarded-For 右侧可信跳读取真实 IP，避免所有登录共用代理 IP 的预算。

**FORWARDED_ALLOW_IPS** 由 uvicorn 从进程环境读取，不经过应用配置，控制 request.client 和访问日志。通配符会取最左值，可由客户端伪造地址和 HTTPS 声明。指定网段后跳过可信网段条目，读取最后一个外部地址。生成 nginx 默认以自身看到的地址覆盖 X-Forwarded-For，只有一项；生成服务固定容器网段并配置对应 CIDR。`bp deploy generate nginx --internal-subnet 10.20.30.0/24` 同时改两处，拒绝带主机位、通配符、IPv4 比 /8 更宽或 IPv6 比 /48 更宽的范围。

!!! warning "生成配置假定 nginx 是唯一代理"
    覆盖 X-Forwarded-For 会丢弃已有信息，适合 nginx 直面客户端；若前有负载均衡、Cloudflare 或另一代理，需一起修改：恢复 `proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;`；增加 TRUSTED_PROXY_HOPS；在 FORWARDED_ALLOW_IPS 中除 Compose 网段外加入上游代理地址或网段。否则 uvicorn 停在负载均衡条目，把它当客户端；crudauth 按右侧跳数解析，因此跳数正确时其限流仍可正确。

!!! note "Docker Desktop 的 IP 限制"
    macOS / Windows Docker Desktop、rootless Docker 等经 userland proxy 时，nginx 可能只看到网关地址，所有客户端共用登录和匿名预算。Linux bridge 通常保留真实地址。逐 IP 行为应在 Linux 宿主机或真实代理前测试。

CORS_ORIGINS 应列**前端源**而非 API 源。通配符与凭据请求不兼容，生产校验拒绝。

<a id="logging-in-production"></a>

## 生产日志

JSON 方便聚合：

```env
LOG_LEVEL=INFO
LOG_FORMAT=json
```

`infrastructure/logging/` 提供关联 ID 的上下文变量、CorrelationIdFilter、set_correlation_id、add_correlation_id_filter，但未安装，默认日志没有请求 ID。需自行添加中间件逐请求设置，并启动时装过滤器，structured / json 才包含字段。错误处理器已有失败请求 support id，可关联反馈与服务端日志。

低噪声配置建议 INFO，WARNING 会跳过请求日志，不利排障。健康轮询等低信息日志在代理或聚合器采样，而非应用。

追踪和指标未内置，见[可观测性](observability.md)。本节原文建议接入 src/wiring/app.py，但该专章说明生成文件应避免手改，示例实际接入 main.py。

<a id="health-and-readiness"></a>

## 健康与就绪

应用根路径 GET /health 用作存活探针：

```yaml
# Kubernetes / Docker probe
livenessProbe:
  httpGet:
    path: /health
    port: 8000
  initialDelaySeconds: 10
  periodSeconds: 10
```

GET /health/ready 验证真实数据库、Redis 连接，执行 wiring 列出的两组检查：

- **CRITICAL_READINESS_CHECKS**：数据库、结构，以及 accounts 的登录锁定 Redis 和会话存储。任一不可达则 503，负载均衡停止分流。
- **schema** 属关键项：有迁移时比较数据库 revision 与代码 head，避免表齐全但缺列逐请求失败；没有迁移或自动建表时比较模型声明的表与实际表。它不按服务器去重，始终执行，并从源码旁而非 cwd 读取 alembic.ini。
- **INFORMATIONAL_READINESS_CHECKS**：缓存和任务代理。故障写日志但保持 200，因为请求不应等待它们。Redis ping，RabbitMQ 读取现有连接状态，正在连接或重连时不可用。

```yaml
readinessProbe:
  httpGet:
    path: /health/ready
    port: 8000
  initialDelaySeconds: 5
  periodSeconds: 10
```

响应体只含整体状态：

```json
{ "status": "ready" }
```

database、rate_limiter、sessions 故障返回 `503` 和 `{"status": "not ready"}`，具体项仅写日志：

```text
WARNING Readiness check cache failed: ConnectionError
WARNING Readiness: cache unavailable, which does not hold traffic back
ERROR   Readiness: database unavailable, holding traffic back
ERROR   Readiness check schema failed: RuntimeError
```

schema 日志说明实际差异，例如 `the database is at 9e1c2f7a4b10, the code expects b3d84c2e5f91`，无迁移时列缺表如 api_keys、key_usage、user。

每项两秒超时，并发执行，单个黑洞服务器不会长时间阻塞。同一服务器按 scheme、凭据、主机、端口去重，Redis 不同 DB 仍可共享一次检查，不同用户则分开，因为一项凭据成功不能证明另一项。凭据身份通过慢派生生成，按凭据对缓存进程内，探针不保留密码或可直接跑字典的摘要。并发探针共用一次运行，报告复用数秒，避免探针洪水抢数据库连接。

两个健康接口都在 API 前缀外且不受限流，避免 API 认证和配额影响探针。其他依赖由所属功能贡献 ReadinessCheck，在 src/wiring/hooks.py 按关键或信息项列出，见[可组合功能](composable-features.md)。

<a id="hardening-checklist"></a>

## 加固清单

- [ ] 运行环境 ENVIRONMENT=production。
- [ ] SECRET_KEY 独立随机、超过 32 字符、未复用。
- [ ] 数据库密码独立，从密钥管理器获取。
- [ ] DEBUG=false。
- [ ] CORS_ORIGINS 仅前端源，无通配符。
- [ ] OPENAPI_URL 留空，不开放 docs / redoc。
- [ ] SESSION_SECURE_COOKIES=true，代理终止 TLS。
- [ ] CSRF_ENABLED=true。
- [ ] 所有 Redis 设置密码。
- [ ] ADMIN_ENABLED=false 或网络访问限制。
- [ ] 迁移通过 migrate 阶段并显式确认。
- [ ] accounts 存在时使用 SMTP、正确 EMAIL_SMTP 配置及项目前端 FRONTEND_URL。
- [ ] CREATE_TABLES_ON_STARTUP=false。
- [ ] SESSION_BACKEND、RATE_LIMITER_BACKEND 为 redis 或 database，不能 memory。
- [ ] FRONTEND_URL 为 HTTPS。
- [ ] RabbitMQ 使用项目独立凭据。
- [ ] 每 PR 运行 pre-commit 和 CI：lint、mypy、测试。
- [ ] 数据库备份；Redis 用于不可丢会话 / 状态时也备份。
- [ ] 监控错误率、p95 / p99 延迟、数据库连接、队列深度和 Redis 内存。

<a id="scaling-considerations"></a>

## 扩容

<a id="api-instances"></a>

### API 实例

负载均衡后添加 prod 容器即可。Redis 会话使任意实例可服务任意用户。memory 不适合水平扩容，应改 redis 或 database，后者通过项目数据库共享会话和登录计数，无需 Redis。

<a id="database"></a>

### 数据库

将 `database_pool_size × api_workers + worker_concurrency × taskiq_workers` 与 Postgres max_connections 对比。4 API Worker × 10 池大小为每副本 40，两个副本加 Taskiq 很容易超过 100。

大规模使用 PgBouncer / RDS Proxy，DATABASE_URL 同样可指向池端点。托管 Postgres 方式相同，[Neon](database/neon.md) 提供 TLS、池与直连、空闲暂停调优及预览数据库分支的完整说明。

<a id="redis"></a>

### Redis

默认同实例四个 DB：缓存 0、限流 1、会话 2、Taskiq 3，适合小部署。规模扩大应将会话与缓存分离实例或集群：会话小且重持久，缓存大、容忍驱逐且高流量，共用可能使缓存内存压力危及会话。会话默认复用缓存连接，用 `SESSION_REDIS_URL` 如 `rediss://user:password@sessions-redis:6380/0` 独立配置。

限流 Redis 是登录硬依赖，锁定失败拒绝，不可达时全部登录 429，应告警，见[登录锁定](authentication/sessions.md#login-lockout)。

<a id="taskiq-workers"></a>

### Taskiq Worker

与 API 独立扩容，任务成为瓶颈时只增加 Worker Deployment。

<a id="common-production-issues"></a>

## 常见生产问题

<a id="app-fails-to-boot-with-productionsecurityerror"></a>

### 启动 ProductionSecurityError

按消息修配置，不绕过。

<a id="sessions-invalidate-after-every-deploy"></a>

### 每次部署会话失效

检查是否 memory，改 Redis（缓存连接或专用 SESSION_REDIS_URL）或 database 的 crudauth_store；Memcached 不是会话后端。

<a id="sudden-burst-of-429s-after-a-config-change"></a>

### 改配置后大量 429

检查限流路径是否仍匹配。重命名或路径规则变化可能回退到更严格的默认限制。

<a id="cache-backend-not-available-warnings-under-load"></a>

### 高负载提示 Cache backend not available

可能连接池耗尽，提高默认 10 的 CACHE_REDIS_POOL_SIZE，检查内存压力及连接泄漏。

<a id="tasks-queue-but-no-worker-picks-them-up"></a>

### 任务入队但不执行

Worker 未运行、代理不同或未导入任务，见[后台任务排查](background-tasks/index.md#troubleshooting)。

<a id="404-on-admin-after-deploy"></a>

### 部署后 /admin 404

检查 ADMIN_ENABLED；启用时加网络限制，或改用脚本。缺凭据也可能不挂载，见后台配置。

<a id="key-files"></a>

## 关键文件

| 组件 | 位置 |
|------|------|
| 生产校验 | `backend/src/infrastructure/security/production_validator.py` |
| 迁移校验 | `backend/migrations/env.py:validate_production_migration` |
| 多阶段镜像 | `backend/Dockerfile` |
| 配置 | `backend/src/infrastructure/config/settings.py` |
| 应用工厂 / 生命周期 | `backend/src/infrastructure/app_factory.py` |

<a id="next-steps"></a>

## 后续步骤

- **[分环境配置](configuration/environment-specific.md)**：环境变量矩阵。
- **[迁移](database/migrations.md)**：结构变更模式。
- **[会话](authentication/sessions.md)**：生产会话配置。
- **[测试](testing.md)**：随附测试环境。
