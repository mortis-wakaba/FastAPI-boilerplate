<a id="observability"></a>

# 可观测性

模板未内置链路追踪或指标导出器，但提供导出所需信息：结构化日志、失败请求的支持编号、依赖就绪状态和 API 密钥请求记录。本页说明如何接入 [Logfire](https://logfire.pydantic.dev)，即带 Pydantic SDK 的 OpenTelemetry，而不改动这些基础能力。

<a id="what-the-project-already-emits"></a>

## 项目已有信号

| 信号 | 来源 | 内容 |
|------|------|------|
| 应用日志 | `infrastructure/logging/`、`get_logger()` | `LOG_FORMAT=json` 每行一个 JSON，级别和控制台 / 文件处理器可配置，见[生产日志](production.md#logging-in-production) |
| 失败请求支持编号 | 全局错误处理器 | 500 响应的 ID 出现在同次请求日志，关联用户反馈 |
| 登录拒绝 | `infrastructure/auth/audit.py` | 每次拒绝和锁定一条 WARNING，含原因和地址，见[审计](authentication/sessions.md#what-a-refusal-records) |
| 依赖健康 | `GET /health/ready` | 每轮就绪状态和不可达项日志，见[健康与就绪](production.md#health-and-readiness) |
| API 密钥流量 | `key_usage` 表 | 路径、方法、状态、耗时、地址和 User-Agent，可逐密钥查询，见[密钥用量](authentication/api-keys.md#what-a-keys-usage-records) |

两处缺口：日志默认**没有请求 ID**，虽然提供 `set_correlation_id()` 和 `add_correlation_id_filter()`，但未安装；也没有逐路由延迟指标。追踪可补充每请求 span 及其下的数据库查询。

<a id="adding-logfire"></a>

## 添加 Logfire

安装依赖并调用三个入口，以下是接入示例，并非内置功能：

```bash
cd backend && uv add "logfire[fastapi,sqlalchemy,redis,httpx,asyncpg]"
```

在独立模块 `src/infrastructure/observability.py` 配置：

```python
"""Tracing, off until LOGFIRE_TOKEN is set."""

import logging

import logfire
from fastapi import FastAPI

from .config.settings import settings
from .database.session import get_engine


def configure_tracing() -> None:
    """Configure Logfire and instrument the clients this app talks through."""
    logfire.configure(
        service_name="api",
        environment=settings.ENVIRONMENT.value,
        send_to_logfire="if-token-present",
        console=False,
    )
    logfire.instrument_sqlalchemy(engine=get_engine())
    logfire.instrument_redis()
    logfire.instrument_httpx()
    logging.getLogger().addHandler(logfire.LogfireLoggingHandler())


def install(app: FastAPI) -> None:
    """Trace every request this app serves, except the probes."""
    logfire.instrument_fastapi(app, excluded_urls=["/health", "/health/ready"])
```

在 `src/interfaces/main.py` 组装：

```python
from ..infrastructure.observability import configure_tracing, install as tracing_install

configure_tracing()

app = create_application(
    router=router,
    settings=settings,
    lifespan=lifespan_with_security,
    root_routers=ROOT_ROUTERS,
    installers=(*INSTALLERS, tracing_install),
    docs_guard=DOCS_GUARD,
)
```

不要手动写入 `src/wiring/app.py`，它由 `tools/removal_drill.py` 生成，测试会比较生成结果。`main.py` 不会被重新生成，适合接入。需要生成器感知时，可[做成可移除功能](#if-you-turn-this-into-a-feature)。

`send_to_logfire="if-token-present"` 使未设置 `LOGFIRE_TOKEN` 时不向进程外发送数据。`LOGFIRE_TOKEN`、`LOGFIRE_SERVICE_NAME`、`LOGFIRE_ENVIRONMENT` 从环境读取，上述默认值可由部署覆盖。

!!! note "Worker 是独立进程"
    `taskiq worker src.infrastructure.taskiq.worker:default_broker` 不导入 main.py，因此需要在 `src/infrastructure/taskiq/worker.py` 也调用 `configure_tracing()`，使用 `service_name='worker'` 区分。请求入队任务不会自动关联追踪，需要自行通过 `logfire.propagate` 传播上下文。

<a id="what-not-to-send"></a>

## 不应发送的内容

插桩可看到应用接触的数据，因此示例采用以下默认值：

- **不启用 `capture_headers`**：避免将 Cookie、`X-API-Key` 等有效凭据发送第三方。
- **不捕获查询参数**：`instrument_sqlalchemy` 不记录绑定参数，避免启用 Redis `capture_statement=True` 和 asyncpg `capture_parameters=True`，否则可能记录邮箱、密码哈希、会话 ID。
- **按模式脱敏**：Logfire 默认识别 password、secret、`api[._ -]?key` 等，可通过 `logfire.configure(scrubbing=logfire.ScrubbingOptions(extra_patterns=['token', 'csrf']))` 扩展。但模式只是兜底，无法判断 `identifier` 实际是用户名。
- **审计标识属于个人数据**：只记录真实账户的标识仍可能是用户名和邮箱，经 `LogfireLoggingHandler` 转发会导出。禁止导出时修改 `infrastructure/auth/audit.py` 两个函数，或不安装日志处理器，只导出追踪。
- **排除探针噪声**：`/health`、`/health/ready` 被频繁轮询，`excluded_urls` 可排除。依赖故障更适合看就绪日志。规则以 `re.search` 匹配不带查询的完整 `scheme://host/path`；`'/health'` 匹配两者及其他含该片段的路径，开头 `^` 会与 scheme 不匹配。仅排除这两个用 `excluded_urls=[r'/health(/ready)?$']`。

<a id="sampling-and-cost"></a>

## 采样与成本

每请求和每查询都产生 span，数据量很大。`logfire.configure(sampling=...)` 接受 [SamplingOptions](https://logfire.pydantic.dev/docs/guides/advanced/sampling/)，可头部采样，或尾部保留全部错误及其余部分比例，后者适合作为起点。`min_level` 在构建前过滤低级别记录。

<a id="doing-without-logfire"></a>

## 不使用 Logfire

除 SDK 外，此方案基于 OpenTelemetry。可用 `opentelemetry-instrumentation-fastapi` 和 OTLP exporter 接任意 collector；或保持插桩，设置 `send_to_logfire=False` 以及 `OTEL_EXPORTER_OTLP_ENDPOINT`、`OTEL_EXPORTER_OTLP_HEADERS`。只需指标时，`logfire.instrument_system_metrics()` 覆盖进程；就绪接口及代理访问日志已提供可用性与响应速度信号。

<a id="if-you-turn-this-into-a-feature"></a>

## 做成可移除功能

在 `tools/removal_drill.py` 的 `FEATURES` 添加 observability 模块、测试及对 `src/wiring/app.py` 的贡献，未设置时默认关闭，见[可组合功能](composable-features.md)。

<a id="key-files"></a>

## 关键文件

| 组件 | 位置 |
|------|------|
| 日志配置和格式化器 | `backend/src/infrastructure/logging/` |
| 就绪检查与报告 | `backend/src/infrastructure/readiness.py`、`backend/src/wiring/hooks.py` |
| 登录拒绝审计 | `backend/src/infrastructure/auth/audit.py` |
| 密钥用量 | `backend/src/modules/api_keys/usage.py` |
| 全局错误与编号 | `backend/src/modules/common/utils/error_handler.py` |
| 本示例接入点 | `backend/src/interfaces/main.py` |
| 功能安装器接入点 | `backend/src/wiring/app.py` |

---

[← 生产环境](production.md){ .md-button } [可组合功能 →](composable-features.md){ .md-button .md-button--primary }
