<a id="composable-features"></a>

# 可组合功能

模板的每个功能都可以连同路由、模型、配置、管理视图、初始数据脚本和测试一起删除。剩余项目仍应可导入，通过 lint、类型检查和保留的测试。

实现方式是功能之间不通过直接导入来接入彼此：各功能在稳定路径导出普通对象，由统一包收集。

<a id="the-composition-root"></a>

## 组装入口

<p align="center">
  <img src="../assets/composition-root-light.png#only-light" alt="功能向 src/wiring/ 提供列表，应用读取列表，功能不直接导入其他功能" width="100%">
  <img src="../assets/composition-root-dark.png#only-dark" alt="功能向 src/wiring/ 提供列表，应用读取列表，功能不直接导入其他功能" width="100%">
</p>

`backend/src/wiring/` 是功能唯一集中汇合的位置：

| 文件 | 列出的内容 |
|------|------------|
| `app.py` | `ROUTER_MOUNTS`、`ROOT_ROUTERS`、`API_THROTTLE`、`LIFECYCLES`、`INSTALLERS`、`DOCS_GUARD` |
| `settings.py` | 功能配置 mixin，核心放最后 |
| `hooks.py` | `PERMISSION_SOURCES`、`RATE_LIMIT_RESOLVERS`、`TIER_DELETE_GUARDS`、`TIER_DELETE_RELEASES`、`CRITICAL_READINESS_CHECKS`、`INFORMATIONAL_READINESS_CHECKS` |
| `models.py` | `UserModelExtensions`、`UserSchemaExtensions` |
| `admin.py` | `ADMIN_VIEWS`，移除 admin 后该文件也不存在 |
| `email.py` | `EMAIL_SENDER`，账户邮件发送器；独立成文件以避免发送器导入 accounts 与 hooks 形成依赖环 |
| `tasks.py` | `TASK_MODULES`，Worker 注册的任务模块 |
| `tables.py` | `TABLE_MODULES`，在 src/modules 之外向 Base.metadata 声明表的模块 |
| `transports.py` | `EXTRA_TRANSPORTS`，与会话并列的认证传输 |
| `cache.py` | `cached`，重新导出装饰器，使路由不直接依赖缓存功能 |

另外两项位于使用它们的代码旁：`backend/scripts/seeders.py` 的 `SEEDERS` 和 `backend/tests/wiring.py` 的 pytest fixture 模块列表。

文件只包含导入和名称，每项是功能导出的值或挂载路径字面值，不进行条件决策，ruff、mypy、IDE 可直接分析。OAuth 路由是否存在等条件逻辑由所属功能决定，再把结果交给 wiring。

!!! note "这些文件由生成器写入"
    `tools/removal_drill.py` 根据功能清单生成全部十二项，`tests/unit/test_removal_drill.py` 逐树比较生成结果与提交文件。因此新增功能要同时更新生成器和已提交文件，遗漏一边会失败。

<a id="the-shapes-a-feature-contributes"></a>

## 功能贡献的类型

`backend/src/infrastructure/composition.py` 定义：

- `RouterMount(router, prefix, throttled)`：挂载在 API 前缀下的路由。
- `Lifecycle(name, startup, shutdown)`：按 wiring 顺序启动、逆序关闭。
- `PermissionSource`：查询用户权限。
- `RateLimitResolver`：返回请求限制，或 None 表示不处理。
- `TierDeleteGuard`：返回禁止删除等级的原因，None 表示允许。
- `TierDeleteRelease`：永久删除已软删除等级前，释放其他功能持有的相关行。
- `ReadinessCheck(name, check)`：应用提供服务所需依赖检查。

扩展 `User` 模型时提供 mixin。例如 `modules/tier/contrib.py` 添加 `tier_id`、`tier`，用户模块无需知道 tiers 存在。

<a id="what-each-feature-owns"></a>

## 各功能归属

| 功能 | 依赖 | 拥有内容 |
|------|------|----------|
| core | 无 | 下列以外全部内容，包括权限注册表 |
| accounts | 无 | `infrastructure/auth/`、`modules/user/`、首个超级用户初始化 |
| rbac | accounts | `modules/role/` |
| ratelimit | accounts | `infrastructure/ratelimit/` |
| tiers | accounts | `modules/tier/`、默认等级初始化 |
| tier_limits | tiers、ratelimit | `modules/rate_limit/` |
| api_keys | accounts | `modules/api_keys/` |
| admin | accounts | `interfaces/admin/`、`wiring/admin.py` |
| cache | 无 | `infrastructure/cache/` |
| taskiq | 无 | `infrastructure/taskiq/` |

部分文件属于功能组合：`modules/user/admin.py` 要求 accounts + admin，`modules/tier/admin.py` 要求 tiers + admin，`modules/role/admin.py` 要求 rbac + admin。

<a id="checking-that-removal-still-works"></a>

## 验证移除后仍能工作

<p align="center">
  <img src="../assets/removal-drill-light.png#only-light" alt="复制仓库、删除未选功能、重新生成 wiring，并对各预设执行六项检查" width="100%">
  <img src="../assets/removal-drill-dark.png#only-dark" alt="复制仓库、删除未选功能、重新生成 wiring，并对各预设执行六项检查" width="100%">
</p>

`tools/removal_drill.py` 复制仓库，删除未选功能路径，重新生成 wiring，检查应用和各模块导入、对 `src scripts migrations tests` 运行 ruff / mypy，并运行剩余测试。

`tests/unit/test_removal_drill.py` 根据当前功能重建 wiring、seeders 和测试 wiring，忽略 docstring 和格式后逐树比较。手动添加贡献但未更新生成器会失败。

检查失败时打印末尾输出，包括测试、规则或类型错误，并保留临时项目，方便手动重跑。构建异常同样保留副本。

使用当前调用解释器，因此通过 uv 运行以获取依赖，并保持 Docker 可用：测试需要 Postgres，包含 taskiq 时还需要 RabbitMQ。

```sh
uv run --no-sync python tools/removal_drill.py                 # every preset
uv run --no-sync python tools/removal_drill.py accounts-only   # one of them
uv run --no-sync python tools/removal_drill.py --keep          # keep the scratch copies
```

CI 以矩阵运行预设：

| 预设 | 验证内容 |
|------|----------|
| `core-only` | 所有功能可同时移除 |
| `core-taskiq` | Worker 无 accounts 也能运行，不注册账户邮件 |
| `accounts-only` | 无 rbac 时权限检查退化为仅超级用户 |
| `accounts-rbac` | 无等级、节流、缓存、Taskiq 时权限仍可用 |
| `accounts-admin` | 后台只注册保留功能拥有的视图 |
| `accounts-api-keys` | 密钥无 rbac、等级、缓存也可认证、收窄、计数 |
| `accounts-rbac-tiers` | 无逐等级限制时等级和 API 节流仍可用 |
| `accounts-cache-taskiq` | 基础设施功能可独立组合 |
| `everything` | 重新生成能还原当前仓库 |

若测试暗中依赖不属于自己的功能，会在移除该功能的预设暴露。

<a id="writing-a-feature-that-can-be-removed"></a>

## 编写可移除功能

1. 自有内容放入自己的路径，并加入验证脚本清单。
2. 只有表中声明依赖的功能可以直接导入。
3. 扩展其他功能时通过钩子，而非直接导入。
4. 配置放自己的 `settings.py`，fixture 放 `tests/fixtures/`，两者加入 wiring。
5. 运行移除验证；排除自身功能后仍失败，通常说明内容放到了不会随功能删除的路径。
