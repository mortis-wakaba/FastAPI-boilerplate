<a id="background-tasks"></a>

# 后台任务

模板使用 [Taskiq](https://taskiq-python.github.io/) 执行后台任务。它是原生异步任务队列，提供类似 Celery 的 API 和可插拔消息代理，默认 Redis，也支持 RabbitMQ。

它属于 `taskiq` 功能。移除后没有 Worker、调度器和消息代理配置；原本入队的账户邮件改为响应后移交发送，见[可组合功能](../composable-features.md)。

!!! tip "构建完整 SaaS？"
    后台任务属于免费基础功能。[FastroAI](https://fastro.ai) 将其与支付、权益、邮件、前端和 AI Agent 整合。[更快交付 SaaS →](https://fastro.ai)

本页介绍 `backend/src/infrastructure/taskiq/` 的实际实现、编写和入队任务、运行 Worker。

<a id="when-to-use-a-background-task"></a>

## 何时使用后台任务

适用于：

- 超过数秒的图片处理、PDF 生成、大量数据导出。
- 调用不稳定第三方，避免 API 延迟依赖对方。
- 发邮件、webhook 或批量通知。
- 重算昂贵但无需实时的聚合。

需立即向用户返回结果，或失败可由用户当场修正时，不适合简单转为后台任务。

<a id="what-ships-out-of-the-box"></a>

## 内置内容

```text
backend/src/infrastructure/taskiq/
├── __init__.py        Docstring only: the package is kept import-free
├── brokers.py         Builds the Redis or RabbitMQ broker from settings
├── app.py             WORKER_STARTUP / WORKER_SHUTDOWN handlers (logging, engine disposal)
├── lifecycle.py       Opens and closes the broker in the API process
├── email.py           The email:send task, and the sender that enqueues it
├── scheduler.py       Scheduler entry point: the schedules tasks declare in labels
├── deps.py            DBSession dependency (TaskiqDepends-wrapped AsyncSession)
└── worker.py          Worker entry point: registers the handlers on `default_broker`
```

内置两个任务。`email:send`（`infrastructure/taskiq/email.py`）投递账户邮件，请求不等待邮件服务器。验证和重置链接中的令牌本身就是凭据，因此队列只保存由 `SECRET_KEY` 派生专用密钥加密的 Fernet 载荷，Worker 通过 `infrastructure/taskiq/sealing.py` 解开。否则能读队列或备份的人就能读待发恢复链接。密钥按用途派生，其他 SECRET_KEY 使用场景不共享该密钥。

密封载荷携带创建时间，Worker 依据链接有效期解密校验；排队超过链接寿命则丢弃，避免发送已经过期的链接。**更换 `SECRET_KEY` 前先清空队列**：旧密钥密封的消息无法解开，Worker 记录后丢弃，不重试不可能成功的消息，这些验证或重置邮件不会被收到。

以上操作不阻塞响应。发送器将消息交给 `infrastructure/handover.py` 自有任务后返回。恢复路由只向真实账户发信，若等待发布会让存在地址慢、不存在地址快，从耗时泄露账户。关闭时先排空已移交内容再关代理，避免部署丢失正在入队的消息。

**没有 Taskiq 的项目保持相同响应方式**。移交属于核心，发送器用 `infrastructure/email/senders.py` 的 `HandingOverSender` 包装，不等待邮件服务器。核心 handover 提供关闭排空生命周期，此处则由 Taskiq 生命周期承担。等待预算按实际工作不同：直接发送一次邮件，或额外经过队列。

代理不可达时，移交记录警告并自己发送，尽力保留邮件。账户邮件对调用者是尽力投递，丢恢复链接不会反馈给它。两项上限防止无限等待：无法连接主机用 `socket_connect_timeout`，连接后卡住发布用 `BROKER_CONNECT_SECONDS`。不设置 socket 读超时，因为 Worker 使用无超时 `brpop` 等新任务，读超时会在空闲时终止监听。

发布恰好超时但消息已入队时，可能既直接发又由队列发，原文选择重复优于丢失。关闭最多等待 `DRAIN_SECONDS`，这是一次合法投递的预算，放弃时记录消息数。

`api_keys:prune_usage`（`modules/api_keys/tasks.py`）按自身计划删除超过 `KEY_USAGE_RETENTION_DAYS` 的记录，随 api_keys 功能提供。除此之外可自行添加任务模块。

<a id="configuration"></a>

## 配置

相关配置属于 `TaskiqSettings`，通过 `infrastructure/config/settings.py` 读取 `backend/.env`：

```env
# Broker selection
TASKIQ_BROKER_TYPE=redis            # or "rabbitmq"

# Retries
TASKIQ_DEFAULT_RETRY_COUNT=3        # runs in all, for a task that asks; 0 turns retries off
TASKIQ_RETRY_DELAY_SECONDS=0        # how long a retry waits; 0 retries immediately

# Redis broker (when TASKIQ_BROKER_TYPE=redis)
TASKIQ_REDIS_HOST=redis             # use "localhost" without Docker
TASKIQ_REDIS_PORT=6379
TASKIQ_REDIS_DB=3                   # separate DB from CACHE / SESSION / RATE_LIMITER
TASKIQ_REDIS_PASSWORD=

# RabbitMQ broker (when TASKIQ_BROKER_TYPE=rabbitmq)
TASKIQ_RABBITMQ_HOST=localhost
TASKIQ_RABBITMQ_PORT=5672
TASKIQ_RABBITMQ_USER=guest
TASKIQ_RABBITMQ_PASSWORD=guest
TASKIQ_RABBITMQ_VHOST=/
```

`TASKIQ_REDIS_DB=3` 默认与缓存 0、限流 1、会话 2 分开，避免一个 DB 的 `FLUSHDB` 影响其他子系统。

RabbitMQ 可选依赖安装方式：

```bash
uv add taskiq-aio-pika
```

模板已包含该依赖，但按需导入 `aio_pika`，使仅 Redis 部署保持轻量。

<a id="writing-a-task"></a>

## 编写任务

任务与所属模块放在一起，如 `modules/widgets/tasks.py`：

```python
# backend/src/modules/widgets/tasks.py
import logging
from typing import Any

from ...infrastructure.taskiq.brokers import default_broker
from ...infrastructure.taskiq.deps import DBSession

logger = logging.getLogger(__name__)


@default_broker.task(task_name="widgets:rebuild_index")
async def rebuild_widget_index(
    owner_id: int,
    db: DBSession,
) -> dict[str, Any]:
    """Recompute the search index for a single owner's widgets."""
    logger.info("Rebuilding widget index for owner %s", owner_id)
    # ... do the work ...
    return {"owner_id": owner_id, "indexed": 42}
```

要点：

- **`task_name`** 可选但推荐固定。默认 `module.function_name` 在移动函数后会破坏消费者，稳定名称可避免。
- **`DBSession`** 是 `Annotated[AsyncSession, TaskiqDepends(get_db_session)]`。每任务独立会话，使用 `NullPool` 引擎，不与 API 进程共用连接，结束时关闭。
- **返回值** 默认可通过 Redis 结果后端保存一小时，即 `RESULT_KEPT_SECONDS`；不需要就不等待。
- **日志** 使用项目标准 logger，无需独立配置 Taskiq logger。

<a id="importing-tasks-for-discovery"></a>

### 导入以发现任务

Worker 只认识已导入模块的任务。将任务模块列在 `src/wiring/tasks.py`，启动入口会导入。Taskiq 包的 `__init__.py` 有意无导入，避免把应用代码拉入配置导入链。

```python
# backend/src/wiring/tasks.py
TASK_MODULES: tuple[str, ...] = (
    "src.infrastructure.taskiq.email",
    "src.modules.api_keys.tasks",
    "src.modules.widgets.tasks",
)
```

遗漏模块仍能通过 `.kiq()` 入队，但 Worker 认为未知任务并丢弃。

<a id="enqueuing-a-task"></a>

## 入队

路由、服务或其他位置可调用：

```python
from .tasks import rebuild_widget_index


@router.post("/widgets/{owner_id}/reindex", status_code=202)
async def trigger_reindex(owner_id: int) -> dict[str, str]:
    await rebuild_widget_index.kiq(owner_id=owner_id)
    return {"status": "queued"}
```

`.kiq(...)` 序列化关键字参数并发布，返回任务句柄；原文称其为 `TaskiqResult`。上面常见的发送后返回流程不等待句柄中的结果，需要等待时见[等待结果](#awaiting-results)。

发布连接由应用 lifespan 打开。Taskiq 通过 `infrastructure/taskiq/lifecycle.py` 贡献 `Lifecycle`，列于 `src/wiring/app.py`，API 服务前启动代理，关闭时释放。RabbitMQ 未启动连接会从 `NoStartupError` 转为 `SendTaskError`；Redis 在构造时建池，表现不同。

代理故障不阻止 API 启动。初次连接失败后警告，继续服务，并按 `BROKER_RETRY_SECONDS` 重试；重复失败记 DEBUG，成功记一条 INFO。每次失败都关闭半初始化连接，避免残留。

无应答的尝试在 `BROKER_CONNECT_SECONDS` 后失败，RabbitMQ 将此超时传给 aio-pika。拒绝连接会立即失败，但接受 TCP 却不说 AMQP 的主机，如卡死服务器、黑洞或故障负载均衡，若无限等待会阻止 API 和 `/health` 启动。重试同样有上限。

代理故障期间 `.kiq()` 抛 `SendTaskError`，入队路由应自行决定如何响应。`GET /health/ready` 将代理作为信息项，读取应用持有连接的实时状态；连接或重连中显示不可用，成功后就绪，但不会因此返回 503。

限制：

- **关键字参数必须 JSON 可序列化**。传 ID 而非 ORM 对象，模型包含 datetime 时先 `.model_dump(mode='json')`。
- **不传数据库会话**，任务自行通过 `DBSession` 获取。
- **不传 HTTP Request**，无法序列化，任务也不应依赖请求。
- **返回值同样 JSON 序列化**，datetime / UUID / Pydantic 返回 JSON 形式，如时间变 ISO 字符串；任意 ORM 行会在结果写入时触发 `PydanticSerializationError`。

<a id="awaiting-results"></a>

### 等待结果

确需响应前等待时，通常更适合同步计算，也可：

```python
result = await rebuild_widget_index.kiq(owner_id=owner_id)
value = await result.wait_result(timeout=30)
print(value.return_value)
```

请求持续打开直到 Worker 完成。**慢任务不要这样做**，应返回 202 并由客户端轮询；短小快速结果直接计算返回。

<a id="scheduled-tasks"></a>

### 定时任务

任务通过自身 `schedule` 标签声明时间：

```python
@default_broker.task(task_name="widgets:rebuild_index", schedule=[{"cron": "*/5 * * * *"}])
async def rebuild_widget_index(db: DBSession) -> None: ...
```

`infrastructure/taskiq/scheduler.py` 使用基于 `LabelScheduleSource` 的 `TaskiqScheduler`，与 Worker 共用代理及任务导入。设置 `TASKIQ_RETRY_DELAY_SECONDS` 时也读取待到期重试，见[重试](#error-handling-and-retries)：

```sh
cd backend
uv run --no-sync taskiq scheduler src.infrastructure.taskiq.scheduler:scheduler
```

保留 taskiq 的项目，`bp deploy generate` 会在 Worker 旁生成 `scheduler` 服务。

**只运行一个调度器**。每个调度器都会执行所有计划，两个副本就重复入队。扩容应扩 Worker，通过 `--workers` 或服务副本。

内置 `api_keys:prune_usage` 在 `0 3 * * *` 清理超过保留期的用量。没有调度器就不清，见[记录保留](../authentication/api-keys.md#how-long-the-usage-rows-are-kept)。延迟重试同样依赖调度器，否则不会执行。

计划也可使用间隔 `{'interval': 60}` 或一次时间 `{'time': datetime(...)}`，同项的 args / kwargs 传给任务。计划 ID 在启动生成，每次重启会变化。

<a id="delayed-tasks"></a>

### 延迟任务

```python
await rebuild_widget_index.kicker().with_labels(delay=60).kiq(owner_id=owner_id)
```

示例 60 秒标签是否生效取决于代理，模板当前两种都不直接支持：Redis `ListQueueBroker` 忽略标签并立即执行；RabbitMQ 若没有 delay queue 或 delayed-message-exchange 插件会报错。需要等待的任务请使用调度计划。

<a id="running-a-worker"></a>

## 运行 Worker

开发时在 API 外的终端：

```bash
cd backend
uv run --no-sync taskiq worker src.infrastructure.taskiq.worker:default_broker
```

Compose 添加相同命令的服务，连接与 API 相同的 Redis / RabbitMQ 和数据库。

调整并发：

```bash
uv run --no-sync taskiq worker src.infrastructure.taskiq.worker:default_broker --workers 4
```

`--workers` 增加进程，`--max-async-tasks` 设置每进程同时任务数。I/O 密集可单进程高并发，CPU 密集倾向多进程低并发。

<a id="reloading-on-code-changes"></a>

### 代码重载

```bash
uv run --no-sync taskiq worker src.infrastructure.taskiq.worker:default_broker --reload
```

需要 dev extra 的 `taskiq[reload]`，因此命令显式请求它。生产不要 `--reload`。

<a id="worker-lifecycle-hooks"></a>

## Worker 生命周期钩子

Worker 入口注册 `infrastructure/taskiq/app.py` 的 `WORKER_STARTUP` 和 `WORKER_SHUTDOWN`：

```python
# backend/src/infrastructure/taskiq/worker.py
from .app import configure_broker_lifecycle
from .brokers import default_broker

configure_broker_lifecycle(default_broker)
```

关闭时释放 Worker 的 NullPool 引擎。只在 worker.py 注册，API 打开代理用于发布时不会运行 Worker 处理器。Taskiq 只有由 worker CLI 启动的进程才选择 WORKER 而非 CLIENT 事件。

可在 worker.py 或其导入模块添加处理器，初始化第三方 SDK、内存缓存或关闭指标等：

```python
from taskiq import TaskiqEvents
from taskiq.state import TaskiqState

from src.infrastructure.taskiq.brokers import default_broker


async def my_startup(state: TaskiqState) -> None:
    state.metrics_client = await build_metrics_client()


default_broker.add_event_handler(TaskiqEvents.WORKER_STARTUP, my_startup)
```

`state` 在该 Worker 进程的任务间共享，适合只创建一次的连接池和客户端。

<a id="error-handling-and-retries"></a>

## 错误处理与重试

`infrastructure/taskiq/app.py` 的 `configure_broker_lifecycle` 加载 `SimpleRetryMiddleware`，但每任务需显式启用。未启用只运行一次，抛错后消息仍确认并移除。

```python
@default_broker.task(task_name="widgets:rebuild_index", retry_on_error=True)
async def rebuild_widget_index(...): ...
```

`TASKIQ_DEFAULT_RETRY_COUNT` 默认 3，指**包含首次的总执行次数**，即默认重试两次。0 对全部任务关闭。单任务也可覆盖：

```python
@default_broker.task(task_name="widgets:reconcile", retry_on_error=True, max_retries=5)
async def reconcile_widgets(...): ...
```

`TASKIQ_RETRY_DELAY_SECONDS` 默认 0，即失败后立即重新入队。限流或依赖短暂不可用时，可能几毫秒耗尽全部尝试。

设置后首次等待该时长，后续按 `* retries` 增长，封顶 `RETRY_MAX_DELAY_SECONDS`，增加最多一秒抖动。**必须同时运行调度器**：Worker 把待重试写入 `taskiq-retry` 前缀的 `ListRedisScheduleSource`，调度器到期再入队。只有 Worker 会记录却不执行，因此默认不启用延迟。还必须使用 Redis 代理；RabbitMQ 未配置 Redis 作为延迟存储，Worker 会拒绝启动，不静默忽略延迟。

死信和自定义中间件见 [Taskiq 文档](https://taskiq-python.github.io/guide/taskiq-middlewares.html)。**任务应幂等**，至少一次投递意味着部分失败时可能重复执行。

<a id="monitoring"></a>

## 监控

没有 Flower 式内置仪表盘，可使用：

- `default_broker.get_all_tasks()`：当前进程导入的名称到任务注册表，Worker 和调度器共用。
- 标准日志接入现有聚合系统。
- Redis 结果后端，默认保留一小时，可读取或 redis-cli 扫描。
- 社区 Prometheus 和管理界面，见 [Taskiq 文档](https://taskiq-python.github.io/)。

多数团队可先用结构化日志和错误率告警，需要更细时再增加逐任务计数。

<a id="common-patterns"></a>

## 常见模式

<a id="fan-out"></a>

### 扇出

一次 API 调用触发 N 个独立任务：

```python
@router.post("/widgets/reindex-all")
async def reindex_all(owner_ids: list[int]) -> dict[str, int]:
    for owner_id in owner_ids:
        await rebuild_widget_index.kiq(owner_id=owner_id)
    return {"queued": len(owner_ids)}
```

<a id="pipeline-task-chains"></a>

### 流水线：任务链

B 依赖 A 的结果时，在任务内部串联，不要让路由等待 A：

```python
@default_broker.task(task_name="widgets:fetch_then_index")
async def fetch_then_index(owner_id: int, db: DBSession) -> dict[str, int]:
    fetched = await fetch_remote_widgets(owner_id, db)
    await rebuild_widget_index.kiq(owner_id=owner_id)
    return {"fetched": fetched}
```

避免在路由写 `result = await task_a.kiq(...).wait_result(); await task_b.kiq(result, ...)`，它会持续占用请求并不必要地串行化工作。

<a id="email-and-notifications"></a>

### 邮件与通知

把耗时工作交任务，API 返回 202：

```python
@default_broker.task(task_name="users:welcome_email")
async def send_welcome_email(user_id: int, db: DBSession) -> None:
    user = await user_service.get_by_id(user_id, db)
    await email_client.send(template="welcome", to=user["email"], context={...})


# In the route:
new_user = await user_service.create(payload, db)
await send_welcome_email.kiq(user_id=new_user["id"])
return new_user
```

用户同步创建，邮件由 Worker 发，邮件故障不阻止账户建立。

<a id="troubleshooting"></a>

## 故障排查

<a id="task-is-queued-but-never-runs"></a>

### 已入队却不执行

- 确认 Worker 在运行，指向相同代理。
- 任务模块必须已导入，Taskiq 不自动发现。
- 查出队序列化错误。
- Redis 用 `redis-cli LRANGE default 0 -1` 或实际队列名看待处理消息。
- RabbitMQ 用 `rabbitmqctl list_queues` 看代理声明的 `taskiq` 队列及数量。

<a id="worker-cant-import-my-task-module"></a>

### 无法导入任务模块

导入根是 `src`，从 `backend/` 运行，保证 `sys.path` 可解析 `src.infrastructure.taskiq.worker:default_broker`。镜像设置 `PYTHONPATH=/app:/app/src`。

<a id="database-connection-errors-in-tasks"></a>

### 任务数据库连接错误

DBSession 使用独立 NullPool，每任务一连接、结束关闭。检查 Worker 的 `DATABASE_URL`，以及 Postgres `max_connections` 是否足以容纳 API 池与任务并发，粗略为 `api_pool_size + worker_concurrency`。

<a id="tasks-fail-silently"></a>

### 静默失败

未启用重试的任务抛错后确认移除。添加 `retry_on_error=True`，或 try/except 显式记录：

```python
@default_broker.task(task_name="widgets:rebuild_index")
async def rebuild_widget_index(owner_id: int, db: DBSession) -> dict[str, Any]:
    try:
        # ...
    except Exception:
        logger.exception("Widget index rebuild failed for owner %s", owner_id)
        raise
```

<a id="key-files"></a>

## 关键文件

| 组件 | 位置 |
|------|------|
| 代理工厂 | `backend/src/infrastructure/taskiq/brokers.py` |
| Worker 入口 | `backend/src/infrastructure/taskiq/worker.py` |
| Worker 生命周期 | `backend/src/infrastructure/taskiq/app.py` |
| API 代理生命周期 | `backend/src/infrastructure/taskiq/lifecycle.py` |
| 数据库依赖 | `backend/src/infrastructure/taskiq/deps.py` |
| 任务模块列表 | `backend/src/wiring/tasks.py` |
| 配置 | `backend/src/infrastructure/config/settings.py` 的 `TaskiqSettings` |

<a id="next-steps"></a>

## 后续步骤

- **[Taskiq 文档](https://taskiq-python.github.io/)**：中间件、调度和代理权威参考。
- **[生产环境](../production.md)**：运行、扩容和进程管理。
- **[缓存策略](../caching/cache-strategies.md)**：定时预热。
