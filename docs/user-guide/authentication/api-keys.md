<a id="api-keys"></a>

# API 密钥

密钥面向程序：脚本、服务器、集成无需浏览器和密码即可调用 API。密钥属于账户，具有独立 scope，可单独撤销。它随 api_keys 功能提供；不包含该功能的项目没有密钥路由、`X-API-Key` 传输和用量表，见[可组合功能](../composable-features.md)。

<a id="creating-a-key"></a>

## 创建密钥

账户登录后才能创建密钥。所有 `/api/v1/api-keys/` 路由要求会话并拒绝密钥，因此密钥不能创建另一个密钥。还要求邮箱已验证，未确认账户返回 `403`，因为地址可能属于他人；通过[恢复流程](index.md#recovery-flows-email)发送确认链接。

```bash
curl -X POST http://localhost:8000/api/v1/api-keys/ \
  -b cookies.txt \
  -H "Content-Type: application/json" \
  -H "X-CSRF-Token: <token-from-login-response>" \
  -d '{
    "name": "Nightly importer",
    "permissions": ["user.read"],
    "usage_limits": {"requests_per_minute": 60, "requests_per_day": 10000},
    "expires_at": "2027-01-01T00:00:00Z"
  }'
```

```json
{
  "id": 1,
  "api_key": "fai_7Qd3kP0r_… shown once",
  "key_prefix": "7Qd3kP0r",
  "name": "Nightly importer",
  "permissions": ["user.read"],
  "usage_limits": {"requests_per_minute": 60, "requests_per_day": 10000},
  "expires_at": "2027-01-01T00:00:00Z",
  "is_active": true,
  "last_used_at": null
}
```

完整密钥位于 `api_key`，**只在这次响应显示**。系统不存明文，遗失后只能替换，无法找回。`key_prefix` 是列表中辨认密钥的可读标识，不是凭据。

| 字段 | 用途 |
|------|------|
| `name` | 自定义名称，区分密钥 |
| `permissions` | scope，使用调用者拥有的注册权限名，见[密钥拥有的权限](#what-a-key-holds) |
| `usage_limits` | 各时间窗口的请求数，见[用量限制](#usage-limits) |
| `expires_at` | 可选到期时间，之后只能提前，不能延后 |
| `key_metadata` | 项目自定义元数据，创建响应不返回；嵌套最多 16 层，存储最多 8 KB，避免写入无法序列化读取的值 |

<a id="calling-with-a-key"></a>

## 使用密钥调用

```bash
curl http://localhost:8000/api/v1/auth/me -H "X-API-Key: fai_7Qd3kP0r_…"
```

```json
{"user_id": 1, "username": "importer", "email": "…", "is_superuser": false, "scopes": ["user.read"], "via": "apikey"}
```

请求以所有者身份认证，principal 的 `transport="apikey"`。无需 CSRF，因为 CSRF 防护针对浏览器自动附带的 Cookie，密钥则由客户端主动发送。响应不携带 Cookie，不会转化为会话。同时有 Cookie 和密钥时，会话优先。

密钥未知、格式错误、已撤销、过期、记录中有 Schema 无法读取的列，或所有者已软删除，都返回 `401`。即使接口允许匿名，也不能把存在但错误的凭据当作没有凭据。

<a id="what-a-key-holds"></a>

## 密钥拥有的权限

密钥请求具有**所有者权限与密钥 scope 的交集**。scope 只能收窄：所有者失去的权限在下次请求即从密钥消失；空 scope 只能识别所有者，没有额外权限。

```bash
curl http://localhost:8000/api/v1/permissions/me -H "X-API-Key: fai_…"
# → {"permissions": ["user.read"]}
```

创建时 scope 只能包含创建者当时拥有的权限，不能比创建它的凭据更强。**密钥永远不是超级用户**，因为超级用户标记可以绕过 scope 无法表达的归属检查。详情与理由见[权限](permissions.md#what-a-key-holds)。

<a id="what-a-key-cannot-do"></a>

## 密钥不能做什么

以下路由仅接受会话，防止密钥提升权限或抹除痕迹：

| 路由 | 原因 |
|------|------|
| `POST /api/v1/auth/change-password` | 账户登录后修改自身密码 |
| `POST /api/v1/auth/email/change-request` | 修改账户恢复地址 |
| `POST /api/v1/auth/logout`、`/logout-all` | 密钥没有可结束的会话 |
| `GET /api/v1/auth/sessions`、`DELETE /api/v1/auth/sessions/{handle}` | 密钥没有可列举或退出的设备 |
| `DELETE /api/v1/users/{username}` | 密钥不能关闭签发它的账户 |
| 全部 `/api/v1/api-keys/` | 密钥不能创建、读取、调整或撤销任何密钥，包括自身 |

归属和 scope 是独立维度，密钥可操作所有者拥有的资源，但归属不是独立许可。修改所有者自身资料也要求 scope 包含 `user.update`，避免只读密钥重命名账户；上表路由无论 scope 都拒绝。scope 又要求所有者拥有该权限，普通账户通过 rbac 角色获得；没有配置权限来源时，只有超级用户的密钥能持有它，见[API 密钥 scope](permissions.md#api-key-scope)。

<a id="usage-limits"></a>

## 用量限制

密钥可配置以下三种窗口中的任意组合：

```json
{"requests_per_minute": 60, "requests_per_hour": 1000, "requests_per_day": 10000}
```

其他名字或无效配置返回 `422`，避免存储永不执行的限制。各窗口按密钥 ID 通过登录锁定共用的限流后端计数，除 `RATE_LIMITER_BACKEND=memory` 外，各 Worker 共享，见[限流](../rate-limiting/index.md)。

窗口耗尽时返回带 `Retry-After` 的 `429`。校验在密钥认证时执行，因此覆盖密钥可访问的所有路由，包括 API 节流跳过的路由。没有限制的密钥不计数；限流后端不可达时采用失败放行，避免连带关闭 API。

多个窗口按短到长计数，第一个耗尽窗口拒绝请求，因此拒绝不会消耗更长窗口。以每分钟 1 次、每天 10 次为例，一分钟内十次请求只有一次通过，九次拒绝，当天仍剩九次额度；触发拒绝的那个窗口仍会计数。

`Retry-After` 表示所有仍耗尽窗口中最后恢复的时间，可能比实际拒绝的窗口更长。读取长窗口不额外计数，避免让客户端一分钟后重试却仍被日额度拒绝。

传输无法使用的密钥返回 `401`。启用 ratelimit 时，它消耗同地址匿名调用者在该路径的预算，从而限制垃圾密钥洪水；有效密钥仍使用所有者预算，不受同地址垃圾流量消耗。

旧版本存储的未知窗口名或非正整数值按原样读取但不生效，类似注册表不再认识的权限不会授予任何能力。在 `backend/` 执行 `uv run --no-sync python scripts/cleanup_api_key_json.py`，或容器中执行 `docker compose exec api python -m scripts.cleanup_api_key_json`，保留实际支持窗口并删除其余值。脚本记录每个密钥丢弃的内容，例如字符串 `{"requests_per_day": "1000"}` 以前没有生效，可改为数字重新设置。包含此类格式的密钥认证返回 `401` 并记录指明密钥的警告，而非每次返回 500；所有者列表仍会因该行返回 500，直到修复。

<a id="what-a-keys-usage-records"></a>

## 用量记录的内容

每个密钥认证成功的请求写一行，各限流窗口的首次拒绝也写一行：

```bash
curl http://localhost:8000/api/v1/api-keys/1/usage -b cookies.txt
curl http://localhost:8000/api/v1/api-keys/1/analytics -b cookies.txt
curl http://localhost:8000/api/v1/api-keys/summary/user -b cookies.txt
```

记录包含路径、方法、客户端实际状态、耗时、地址和 User-Agent。拒绝也记录，使所有者看到失败和节流，包括路由异常的每次 `500` 及密钥限制各窗口的首次 `429`。之后相同窗口的拒绝仍响应和计数，但不再写行，避免达到上限的客户端每次都产生数据库行与更新时间戳。会话请求不记录。

路由在密钥认证前拒绝是例外：仅会话路由只尝试会话传输，不解析密钥。是否留行取决于其他依赖的执行顺序；启用 ratelimit 时，它的逐调用者节流会解析所有传输，因此能产生记录。无论哪种情况，客户端都收到 `401`。

写入在响应后的后台任务执行，客户端不等待；同一任务更新密钥的 `last_used_at` 和 `last_used_ip`。认证本身不写数据，“最后使用”会有延迟，进程在任务前退出会丢失该次记录。

时间戳和用量行分别写入，一项失败不影响另一项，失败写日志。`last_used_at` 是请求到达时间，而非写入时间；已有更晚时间戳时保留，因此并发请求写入乱序后，仍同时保留较晚请求的时间和地址。

地址使用可信代理边界 `TRUSTED_PROXY_HOPS` 解析的结果，而非原始套接字地址，以记录真实客户端。转发头由客户端控制，因此只存列能容纳的合法地址，其他情况或无地址时记为空。Schema 拒绝的用量行记录日志后丢弃，不改变原响应状态。User-Agent 同样不可信，只保留前 512 个字符供所有者读取。

<a id="how-long-the-usage-rows-are-kept"></a>

## 用量记录保留时间

`KEY_USAGE_RETENTION_DAYS` 默认 90 天。每请求记录地址和 User-Agent，因此个人数据随请求量增长。`0` 保留全部且不清理，负值在配置加载时拒绝。

限流器在自己的存储中计数，不读取这些记录，清理不影响密钥权限。分析接口仅查询保留窗口，`days` 不超过保留期，不清理时最多 365 天。用户摘要的 `total_requests`、`total_cost_microcents` 是窗口内总量而非密钥终身总量。需要据此计费的项目应在清理前自行聚合，模板不负责计费。密钥自身的 `last_used_at` 和 `last_used_ip` 不依赖这些行，清理后保留。

启用 Taskiq 时，调度器每天 UTC 03:00 入队 `api_keys:prune_usage`：

```sh
cd backend
uv run --no-sync taskiq scheduler src.infrastructure.taskiq.scheduler:scheduler
```

必须同时运行调度器和 Worker 才会清理。没有 Taskiq 时，同样的清理可交给系统 cron：

```bash
cd backend
python scripts/prune_key_usage.py
```

两种方式均每条语句删除 10,000 行，每批提交并记录总量。保留期为 `0` 时任务不读取数据直接返回，脚本说明未配置清理。

<a id="revoking-expiring-and-rotating"></a>

## 撤销、到期与轮换

```bash
# Narrow a key, or bring its expiry forward
curl -X PATCH http://localhost:8000/api/v1/api-keys/1 \
  -b cookies.txt -H "Content-Type: application/json" -H "X-CSRF-Token: <token>" \
  -d '{"permissions": [], "expires_at": "2026-11-01T00:00:00Z"}'

# Revoke it
curl -X DELETE http://localhost:8000/api/v1/api-keys/1 -b cookies.txt -H "X-CSRF-Token: <token>"
```

撤销不可逆，过期时间只可提前、不可延后，避免静默扩大密钥能力。轮换时先创建新密钥并切换客户端，再撤销旧密钥，两者可短期并存，通过用量判断旧密钥何时停止使用。

**账户恢复会撤销旧密钥**。创建时记录所有者的凭据代次 `owner_token_version`，旧代次密钥返回 `401`。修改密码、完成密码重置、通过 OAuth 提供方认领账户都会推进代次。因此被盗会话签发的密钥在所有者重置密码后失效；在他人邮箱下签发的密钥，也会在真实所有者证明归属并认领后失效。

结束会话不会推进代次。`POST /auth/logout-all` 退出所有设备，但密钥继续工作，因为它没有会话。认为密钥泄露时应单独撤销。

<a id="how-a-key-is-stored"></a>

## 密钥存储方式

服务端通过 `secrets.token_urlsafe(48)` 生成 48 随机字节，即 384 位随机性，存储为 `sha256$<digest>`。摘要确定，因此每请求只需对 `key_hash` 做一次索引查找。

!!! note "为什么不用慢哈希"
    bcrypt、scrypt、Argon2 的工作因子用于提高猜测成本，适合人类选择的密码。密钥是服务端随机值，无可行猜测空间，慢哈希不会增加实际保护，却有成本。旧版 scrypt 使每请求付出派生开销，列表公开的 `key_prefix` 还允许持有前缀的人用垃圾请求消耗算力。扫描器可能把该列误认成密码而标记 SHA-256；账户密码仍由 crudauth 使用 bcrypt，与此无关。

旧版加盐 scrypt 密钥仍可按前缀查找并验证，首次成功使用后改存摘要。转换独立于请求事务，因此后续请求失败不回滚，转换失败则密钥仍有效，下次重试。在转换前，任何携带该前缀的请求都触发派生，不限所有者。只对仍可认证的行验证，已撤销或过期密钥不消耗派生成本，成功转换后也不再消耗。

在 `backend/` 执行 `uv run --no-sync python scripts/revoke_scrypt_api_keys.py`，或容器中执行 `docker compose exec api python -m scripts.revoke_scrypt_api_keys`，会列出仍以 scrypt 存储的密钥前缀和名称，方便通知所有者重建。加 `--revoke` 会停用它们，立即消除开销，不影响已存摘要的密钥。

<a id="x-api-key-in-the-interactive-docs"></a>

## 交互文档中的 `X-API-Key`

`/docs` 没有密钥输入框，因为该头未声明为 OpenAPI 安全方案。全局声明会错误暗示所有路由都接受密钥，逐路由声明又需要与拒绝规则手动同步。需要时，为应展示此认证方式的路由添加独立依赖：

```python
from fastapi import Security
from fastapi.security import APIKeyHeader

api_key_header = APIKeyHeader(name="X-API-Key", auto_error=False)


@router.get("/widgets/", dependencies=[Security(api_key_header)])
async def list_widgets(...): ...
```

无论是否声明，传输都照常认证。安全方案只向文档和 Swagger 说明预期请求头。

<a id="key-files"></a>

## 关键文件

| 组件 | 位置 |
|------|------|
| `X-API-Key` 传输 | `backend/src/modules/api_keys/transport.py` |
| 签发、校验和用量服务 | `backend/src/modules/api_keys/service.py` |
| 用量中间件 | `backend/src/modules/api_keys/usage.py` |
| 限流窗口 | `backend/src/modules/api_keys/limits.py` |
| 保留与清理 | `backend/src/modules/api_keys/retention.py` |
| Taskiq 定时清理 | `backend/src/modules/api_keys/tasks.py` |
| 无 Taskiq 的清理脚本 | `backend/scripts/prune_key_usage.py` |
| 旧 scrypt 密钥脚本 | `backend/scripts/revoke_scrypt_api_keys.py` |
| 路由 | `backend/src/modules/api_keys/routes.py` |
| `APIKey`、`KeyUsage` 模型 | `backend/src/modules/api_keys/models.py` |
| 传输注册 | `backend/src/wiring/transports.py` |
