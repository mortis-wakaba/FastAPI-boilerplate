<h1 align="center">Fastro · Benav Labs 的 FastAPI 项目模板</h1>
<p align="center" markdown=1>
  <i><b>功能齐全的 FastAPI 起步模板</b>：按业务功能组织模块、可替换的基础设施、支持插件的 CLI。</i>
</p>

<p align="center">
  <a href="https://benavlabs.github.io/FastAPI-boilerplate">
    <picture>
      <source media="(prefers-color-scheme: dark)" srcset="docs/assets/fastro-cover-dark.png">
      <img src="docs/assets/fastro-cover-light.png" alt="Fastro：包含认证、CRUD、后台任务、缓存和限流的开源 FastAPI 模板" width="100%">
    </picture>
  </a>
</p>

<p align="center">
<a href="https://benavlabs.github.io/FastAPI-boilerplate/">文档</a> · <a href="https://deepwiki.com/benavlabs/FastAPI-boilerplate">DeepWiki</a> · <a href="https://discord.com/invite/TEmPs22gqB">Discord</a>
</p>

<p align="center">
  <a href="https://fastapi.tiangolo.com">
      <img src="https://img.shields.io/badge/FastAPI-005571?style=for-the-badge&logo=fastapi" alt="FastAPI">
  </a>
  <a href="https://www.postgresql.org">
      <img src="https://img.shields.io/badge/PostgreSQL-316192?style=for-the-badge&logo=postgresql&logoColor=white" alt="PostgreSQL">
  </a>
  <a href="https://redis.io">
      <img src="https://img.shields.io/badge/Redis-DC382D?logo=redis&logoColor=fff&style=for-the-badge" alt="Redis">
  </a>
  <a href="https://deepwiki.com/benavlabs/FastAPI-boilerplate">
      <img src="https://img.shields.io/badge/DeepWiki-1F2937?style=for-the-badge&logoColor=white" alt="DeepWiki">
  </a>
</p>

<p align="center" markdown=1>
  <i>免费、开源的 FastAPI 基础项目。正在构建 <b>SaaS</b>，无论是否包含 AI？<a href="https://fastro.ai">FastroAI</a> 在此基础上增加支付、权益管理、邮件和前端，以及 AI 功能。<a href="#fastro-vs-fastroai">查看对比 ↓</a></i>
</p>

<a id="features"></a>

## 功能

* **九项可移除功能**：accounts、rbac、tiers、tier_limits、ratelimit、api_keys、admin、cache 和 taskiq。移除时会一并移除对应的路由、模型、配置、管理视图、初始数据脚本和测试（[指南](https://benavlabs.github.io/FastAPI-boilerplate/user-guide/composable-features/)）。
* 全异步 FastAPI + SQLAlchemy 2.0。
* Pydantic v2 模型和数据校验。
* 基于 [crudauth](https://pypi.org/project/crudauth/) 的服务端会话和 CSRF 防护；已接入 Google 和 GitHub 的 OAuth；API 密钥。
* 为所有 FastAPI 依赖提供 Annotated 类型别名。
* 支持按用户等级、请求路径配置规则的限流器。
* 使用 FastCRUD 实现高效 CRUD 和分页。
* 基于 **SQLAdmin** 的管理后台，可通过环境变量启用或关闭。
* [Taskiq](https://taskiq-python.github.io/) Worker，支持 Redis 或 RabbitMQ 消息代理。
* Redis 或 Memcached 缓存，提供 `@cache` 装饰器和 provider API。
* **支持插件的 `bp` CLI**：生成 Compose 文件、检查环境变量、加载第三方命令或功能插件。
* 通过 CLI 生成适用于本地、生产及 nginx 反向代理部署的 Docker Compose 配置。
* 支持任意 Postgres：随附容器或 [Neon](https://neon.com) 提供的无服务器数据库（[指南](https://benavlabs.github.io/FastAPI-boilerplate/user-guide/database/neon/)）。

<a id="why-and-when-to-use-it"></a>

## 适用场景

**适合以下需求：**

* 需要包含认证、CRUD、后台任务、缓存和限流的实用起步模板。
* 希望有**合理的默认配置**，同时可以自由移除模块。
* 需要**可随业务成长的基础项目**：按业务功能组织模块，并通过支持插件的 CLI 扩展代码生成器。
* 希望 README 简明，把深入说明放在[文档站点](https://benavlabs.github.io/FastAPI-boilerplate/)。

> 如果需要 monorepo 微服务脚手架，本模板**不适合**；可查看[文档](https://benavlabs.github.io/FastAPI-boilerplate/user-guide/project-structure/)中的相关说明。

**提供的组件：**

* **应用**：FastAPI [应用工厂](https://benavlabs.github.io/FastAPI-boilerplate/user-guide/project-structure/)，根据环境控制 API 文档是否开放。
* **认证**：[服务端会话](https://benavlabs.github.io/FastAPI-boilerplate/user-guide/authentication/sessions/)、CSRF、[OAuth](https://benavlabs.github.io/FastAPI-boilerplate/user-guide/authentication/)、[API 密钥](https://benavlabs.github.io/FastAPI-boilerplate/user-guide/authentication/permissions/)。
* **数据库**：Postgres + SQLAlchemy 2.0；[Alembic 迁移](https://benavlabs.github.io/FastAPI-boilerplate/user-guide/database/migrations/)要求生产环境确认，可使用本地容器或 [Neon](https://benavlabs.github.io/FastAPI-boilerplate/user-guide/database/neon/)。
* **CRUD**：[FastCRUD 泛型](https://benavlabs.github.io/FastAPI-boilerplate/user-guide/database/crud/)。
* **缓存**：[装饰器和 provider API](https://benavlabs.github.io/FastAPI-boilerplate/user-guide/caching/)，支持 Redis 或 Memcached。
* **队列**：[Taskiq Worker](https://benavlabs.github.io/FastAPI-boilerplate/user-guide/background-tasks/)，支持 Redis 或 RabbitMQ。
* **限流**：[按等级和路径配置规则](https://benavlabs.github.io/FastAPI-boilerplate/user-guide/rate-limiting/)。
* **管理后台**：[SQLAdmin 视图](https://benavlabs.github.io/FastAPI-boilerplate/user-guide/admin-panel/)，可通过环境变量开关。
* **CLI**：[`bp` 工具](https://benavlabs.github.io/FastAPI-boilerplate/cli/)，用于生成 Compose 配置、检查环境变量和扩展插件。

<a id="fastro-vs-fastroai"></a>

## Fastro vs FastroAI

本模板 **Fastro** 是免费、开源的**基础项目**，提供构建生产 FastAPI 后端所需的功能。**[FastroAI](https://fastro.ai)** 是建立在相同基础上的付费模板，用于交付**完整的 SaaS**：已整合 Stripe 计费（订阅、额度、折扣）、权益管理、事务邮件和前端。如果构建 **AI** 产品，还提供 PydanticAI Agent 层；这些付费功能同样适用于普通 SaaS。

| | **Fastro**（本仓库，免费） | **FastroAI**（付费） |
|---|:---:|:---:|
| FastAPI + SQLAlchemy 2.0、Pydantic v2 | ✓ | ✓ |
| 认证：会话、OAuth、API 密钥 | ✓ | ✓ **+ JWT** |
| FastCRUD · SQLAdmin · Alembic | ✓ | ✓ |
| 缓存、限流、Taskiq 后台任务 | ✓ | ✓ |
| Docker（本地 / 生产 / nginx） | ✓ | ✓ |
| `bp` CLI：脚手架、环境检查、插件 | ✓ |  |
| **支付**：Stripe 订阅、额度、折扣、webhook |  | ✓ |
| **权益管理**：按套餐或等级开放功能 |  | ✓ |
| **事务邮件**和通知 |  | ✓ |
| **前端**：Astro 落地页 / 营销站点 |  | ✓ |
| **可观测性**：Logfire 链路追踪和指标 |  | ✓ |
| **AI Agent**：PydanticAI 记忆、工具、用量跟踪 |  | ✓ |
| 支持 | 社区、Discord | 优先支持、终身更新 |

<p align="center">
  <a href="https://fastro.ai">
    <picture>
      <source media="(prefers-color-scheme: dark)" srcset="docs/assets/fastroai-card-dark.png">
      <img src="docs/assets/fastroai-card-light.png" alt="FastroAI：在 Fastro 基础上增加支付、权益管理、邮件、前端和 AI 的完整 SaaS 模板" width="100%">
    </picture>
  </a>
</p>

如果希望在简洁、便于修改的 FastAPI 后端上二次开发，**继续使用 Fastro**。
如果正在交付 SaaS，无论是否包含 AI，并希望开箱即用地获得计费、权益、邮件、认证和前端，**[选择 FastroAI →](https://fastro.ai)**。

<a id="repo-layout"></a>

## 仓库结构

这是一个包含两个成员的 [uv workspace](https://docs.astral.sh/uv/concepts/projects/workspaces/)，共用仓库根目录下的一个虚拟环境。

```text
fastapi-boilerplate/
├── pyproject.toml          # workspace root (uv workspace metadata)
├── backend/                # the deployable application
│   ├── src/                # interfaces/, infrastructure/, modules/
│   ├── pyproject.toml
│   └── Dockerfile          # multi-stage: dev / migrate / prod
└── cli/                    # `bp` - developer/operator tool (never ships in prod)
    └── src/cli/
```

<a id="quickstart"></a>

## 快速开始

```bash
git clone https://github.com/<you>/FastAPI-boilerplate
cd FastAPI-boilerplate
uv sync --all-packages --all-extras           # one venv at the root, both members installed
```

按需要的部署方式生成 Compose 文件：

```bash
uv run --no-sync bp deploy generate local               # hot-reload dev stack
# or: uv run --no-sync bp deploy generate prod          # production single-host
# or: uv run --no-sync bp deploy generate nginx         # production behind nginx
```

配置环境变量，CLI 可帮助生成密钥和检查配置：

```bash
cp backend/.env.example backend/.env
uv run --no-sync bp env gen-secret                      # print a fresh SECRET_KEY
uv run --no-sync bp env validate                        # audit .env against the production validator
```

启动服务：

```bash
docker compose up --build
# → http://127.0.0.1:8000  (Swagger at /docs)
```

**不使用 Docker**：本地需要 Postgres 和 Redis；也可以使用 [Neon](https://benavlabs.github.io/FastAPI-boilerplate/user-guide/database/neon/) 替代本地 Postgres。

```bash
cd backend
uv run --no-sync alembic upgrade head
uv run --no-sync python -m scripts.setup_initial_data   # creates the first admin user + default tier
uv run --no-sync fastapi dev src/interfaces/main.py     # API
uv run --no-sync taskiq worker src.infrastructure.taskiq.worker:default_broker  # in a second terminal
```

> 完整安装流程、环境变量参考和各环境部署指南见[文档](https://benavlabs.github.io/FastAPI-boilerplate/getting-started/installation/)。

<a id="common-tasks"></a>

## 常用操作

```bash
# generate a fresh production-ready compose file
uv run --no-sync bp deploy generate prod --workers 8

# audit your .env against the production security validator
uv run --no-sync bp env validate

# run Alembic migrations
cd backend && uv run --no-sync alembic revision --autogenerate -m "<msg>" && uv run --no-sync alembic upgrade head

# run tests
cd backend && uv run --no-sync pytest

# install bp as a global tool (optional)
uv tool install --editable ./cli
```

更多示例，包括创建超级用户、用户等级、限流、后台管理和插件开发，见[文档](https://benavlabs.github.io/FastAPI-boilerplate/)。

<a id="sponsors"></a>

## 赞助方

<a href="https://neon.com"><img src="https://img.shields.io/badge/Neon-00E599?style=for-the-badge&logo=postgresql&logoColor=black" alt="Neon"></a>

感谢 **[Neon](https://neon.com)** 为项目的开源基础设施提供数据库额度。Neon 是无服务器 Postgres：空闲时计算资源可缩减至零，也可像代码一样创建数据库分支，适合为每个 PR 建立预览环境。

它也可以直接用于你的项目，并且**可以免费起步**：免费方案长期有效，并非试用，无需信用卡，存储和计算资源可满足开发、预发布及小规模生产负载。将 `DATABASE_URL` 指向 Neon 项目即可省去本地 Postgres 容器，无需修改代码：

```env
DATABASE_URL=postgresql+asyncpg://user:password@ep-xxx-pooler.region.aws.neon.tech/neondb?ssl=require
```

→ [完整 Neon 指南](https://benavlabs.github.io/FastAPI-boilerplate/user-guide/database/neon/)，涵盖连接字符串转换、连接池端点与直连端点、空闲缩容时的连接池配置。其他托管 Postgres 的使用方式相同。

<a id="contributing"></a>

## 参与贡献

阅读[贡献指南](CONTRIBUTING.md)。

<a id="references"></a>

## 参考项目

本项目受以下项目启发，在其基础上按作者偏好调整，并更新了 Pydantic 和 SQLAlchemy：

- @tiangolo 的 [`Full Stack FastAPI and PostgreSQL`](https://github.com/tiangolo/full-stack-fastapi-postgresql)。
- @kludex 的 [`FastAPI Microservices`](https://github.com/Kludex/fastapi-microservices)，为本模板提供了许多灵感。
- [`Async Web API with FastAPI + SQLAlchemy 2.0`](https://github.com/rhoboro/async-fastapi-sqlalchemy)：SQLAlchemy 2.0 ORM 示例。
- [`FastAPI Rocket Boilerplate`](https://github.com/asacristani/fastapi-rocket-boilerplate/tree/main)：Docker Compose 参考。

<a id="license"></a>

## 许可证

[`MIT`](LICENSE.md)

<a id="contact"></a>

## 联系方式

Benav Labs – [benav.io](https://benav.io)、[Discord 社区](https://discord.com/invite/TEmPs22gqB)

<hr>
<a href="https://benav.io">
  <picture>
    <source media="(prefers-color-scheme: dark)" srcset="docs/assets/benav-labs-banner-dark.png">
    <img src="docs/assets/benav-labs-banner-light.png" alt="Benav Labs - benav.io" width="100%"/>
  </picture>
</a>
