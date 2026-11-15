<a id="commands"></a>

# 命令

本页为内置 bp 命令参考。插件命令有自己的帮助文本，见[插件](plugins.md)。

<a id="bp-deploy"></a>

## `bp deploy`

生成部署文件，当前唯一子命令为 generate。

<a id="bp-deploy-generate"></a>

### `bp deploy generate <mode>`

根据部署方式渲染 docker-compose.yml，nginx 模式另生成 nginx/default.conf。

```text
Usage: bp deploy generate [OPTIONS] MODE

Arguments:
  MODE                          {local|prod|nginx}  [required]

Options:
  -o, --output-dir DIRECTORY    Where to write the compose file (default: repo root).
  --api-port INTEGER            Host port to publish the API on. [default: 8000]
  --workers INTEGER             Number of API workers (prod / nginx only). [default: 4]
  --internal-subnet TEXT        Subnet for the compose network, and the only peers uvicorn
                                takes forwarded headers from (nginx only).
                                [default: 172.31.240.0/24]
  -f, --force                   Overwrite existing files without asking.
  -y, --yes                     Assume yes for all prompts.
  --dry-run                     Show what would be written, don't touch disk.
```

<a id="modes"></a>

#### 模式

| 模式 | 服务组合 | 用途 |
|------|----------|------|
| local | dev 阶段 API、Postgres、Redis，挂载源码并热重载 | 本地开发 |
| prod | prod 阶段 API、WORKERS 环境变量、Postgres、Redis、一次性 migrate | 无代理的单机生产 |
| nginx | prod 加 80 端口 nginx，API 仅内网开放 | 反向代理后的生产 |

保留 taskiq 功能时，每种模式还包含 Worker 和一个调度器；移除后均不生成。所有模式共用多阶段 backend/Dockerfile，不生成各自 Dockerfile。

migrate 位于 Compose profile，其他服务不等待它，因此 docker compose up 不自动迁移。先迁移再重启：

```bash
docker compose --profile migrate run --rm migrate
docker compose up -d
```

这样迁移失败时旧容器继续使用旧结构，避免新 API 启动到无法使用的结构。

<a id="examples"></a>

#### 示例

```bash
# Generate the dev stack at the repo root
uv run --no-sync bp deploy generate local

# Override workers and port
uv run --no-sync bp deploy generate prod --workers 8 --api-port 9000

# Dry run — print what would be written, don't touch disk
uv run --no-sync bp deploy generate nginx --output-dir /tmp/scratch --dry-run

# Generate into a separate directory (for staging configs in CI, etc.)
uv run --no-sync bp deploy generate prod --output-dir ./deploy/prod --yes

# Put the stack on another range when 172.31.240.0/24 clashes with a network you use
uv run --no-sync bp deploy generate nginx --internal-subnet 10.89.0.0/24
```

`--internal-subnet` 必须是合法网络地址：IPv4 前缀长度至少 /8，IPv6 至少 /48，不能包含主机位、通配符或更宽范围。它同时写入 Compose ipam 子网及 API 的 FORWARDED_ALLOW_IPS，决定信任哪些来源的 X-Forwarded-For 和 X-Forwarded-Proto。`172.31.240.1/24` 等非法值在写入前拒绝。

> 译注：原文用 “/8 or smaller” 描述范围，此处明确为子网范围不能更宽，避免把前缀长度方向理解反。

<a id="what-it-writes"></a>

#### 生成文件

| 模式 | 文件 |
|------|------|
| local | `<output-dir>/docker-compose.yml` |
| prod | `<output-dir>/docker-compose.yml` |
| nginx | `<output-dir>/docker-compose.yml`<br>`<output-dir>/nginx/default.conf` |

目标已存在时提示覆盖，非交互执行可用 `--force` 或 `--yes`。

<a id="generated-compose-conventions"></a>

#### Compose 约定

服务名和网络约定相同：

- **api**：FastAPI 应用。
- **worker**：保留 taskiq 时，运行 `taskiq worker src.infrastructure.taskiq.worker:default_broker`。
- **scheduler**：保留 taskiq 时，单实例运行 `taskiq scheduler src.infrastructure.taskiq.scheduler:scheduler`。
- **postgres**：Postgres 16 alpine，带健康检查。
- **redis**：Redis 7 alpine，带健康检查。
- **migrate**：prod / nginx 中带 `CONFIRM_PRODUCTION_MIGRATION=yes` 执行一次 alembic upgrade head，位于 migrate profile。其他服务不依赖它，部署先运行再重启其余服务。
- **nginx**：仅 nginx 模式，使用 Nginx 1.27 alpine，只读挂载生成配置。

Compose 引用 `./backend/.env`，启动前先执行 `cp backend/.env.example backend/.env` 并配置。

<a id="bp-env"></a>

## `bp env`

检查和准备运行环境，当前提供两个子命令。

<a id="bp-env-gen-secret"></a>

### `bp env gen-secret`

生成适合 SECRET_KEY 的高熵十六进制密钥。

```text
Usage: bp env gen-secret [OPTIONS]

Options:
  --bytes INTEGER RANGE [16 ≤ x ≤ 128]   Number of random bytes (hex output is 2x). [default: 32]
```

```bash
$ uv run --no-sync bp env gen-secret
af97045f600bf988041ec4b6fd891763d8f79f01f0a0a4a7ed2022e57f771a9e

$ uv run --no-sync bp env gen-secret --bytes 16
c042a8aa0d678a9c73dc371e3e0d6a5e
```

默认输出 64 个十六进制字符，即 256 位熵，可用于 SECRET_KEY 和签名 Cookie 密钥等。

项目内会按应用自身密钥规则检查，不合格重新生成。原文估计每 1260 万个十六进制密钥约一个出现连续八个递增码点，生产会拒绝。项目外没有规则可查询时，直接输出首次生成结果。

可通过管道直接写密钥管理器：

```bash
uv run --no-sync bp env gen-secret | gh secret set SECRET_KEY --repo my-org/my-app
```

<a id="bp-env-validate"></a>

### `bp env validate`

无论当前 ENVIRONMENT 如何，都用生产规则检查配置。

```text
Usage: bp env validate
```

```bash
$ uv run --no-sync bp env validate
Critical (2):
  • SECRET_KEY is using default or insecure value. ...
  • Database is using default credentials (POSTGRES_PASSWORD='postgres'). ...
```

执行步骤：

1. 导入应用启动所用 ProductionSecurityValidator。
2. 对 ENVIRONMENT 被设为 production 的配置副本审查，确保实际运行生产检查，错误中的环境名也表示审查环境。
3. 分组输出严重错误和警告，存在严重问题则非零退出。

退出码：

| 码 | 含义 |
|----|------|
| 0 | 无严重问题，可能仍有提示性警告 |
| 1 | 至少一个严重问题 |

可在 CI 阻止不安全部署：

```yaml
# .github/workflows/deploy.yml (excerpt)
- run: uv run --no-sync bp env validate
  env:
    SECRET_KEY: ${{ secrets.SECRET_KEY }}
    POSTGRES_PASSWORD: ${{ secrets.POSTGRES_PASSWORD }}
    # ...
```

具体规则见[生产校验器](../user-guide/production.md#the-production-validator)。

<a id="global-options"></a>

## 全局选项

Typer 提供：

| 选项 | 作用 |
|------|------|
| --help | 显示帮助后退出 |
| --install-completion | 安装 bash / zsh / fish 补全 |
| --show-completion | 输出当前 Shell 补全脚本 |

<a id="exit-codes"></a>

## 退出码

| 码 | 含义 |
|----|------|
| 0 | 成功 |
| 1 | 校验或操作失败 |
| 2 | 参数解析错误，如非法选项或缺参数 |

<a id="discoverability"></a>

## 查找帮助

所有命令都有 --help：

```bash
uv run --no-sync bp --help
uv run --no-sync bp deploy --help
uv run --no-sync bp deploy generate --help
uv run --no-sync bp env --help
uv run --no-sync bp env validate --help
```

根帮助列出所有已挂载组，包括插件。插件未出现时见[插件排查](plugins.md#troubleshooting)。
