<a id="user-guide"></a>

# 用户指南

本指南全面介绍 FastAPI Boilerplate 的使用方式与设计。从创建第一个 API 到理解高级功能，都可以在这里找到相关说明。

<a id="what-youll-learn"></a>

## 内容概览

<a id="project-understanding"></a>

### 理解项目

- **[项目结构](project-structure.md)**：了解代码组织与架构决策。
- **[可组合功能](composable-features.md)**：各功能拥有的内容、如何在不直接导入的情况下扩展其他功能，以及移除方式。
- **[配置](configuration/index.md)**：针对不同环境配置应用。

<a id="core-components"></a>

### 核心组件

<a id="database-operations"></a>

### 数据库操作

- **[数据库概览](database/index.md)**：了解数据层架构。
- **[模型](database/models.md)**：定义和使用 SQLAlchemy 模型。
- **[Schema](database/schemas.md)**：创建用于数据校验的 Pydantic Schema。
- **[CRUD 操作](database/crud.md)**：实现增删改查。
- **[迁移](database/migrations.md)**：使用 Alembic 管理数据库结构变更。

<a id="api-development"></a>

### API 开发

- **[API 概览](api/index.md)**：使用 FastAPI 构建可靠的 REST API。
- **[接口](api/endpoints.md)**：创建和组织 API 接口。
- **[分页](api/pagination.md)**：实现高效的数据分页。
- **[异常处理](api/exceptions.md)**：妥善处理错误。
- **[API 版本管理](api/versioning.md)**：管理 API 版本与向后兼容性。

<a id="security-authentication"></a>
<a id="security--authentication"></a>

### 安全与认证

- **[认证概览](authentication/index.md)**：使用会话认证、OAuth 和 API 密钥保护 API。
- **[会话](authentication/sessions.md)**：服务端会话、HttpOnly Cookie 与 CSRF 防护。
- **[用户管理](authentication/user-management.md)**：用户注册、登录和资料管理。
- **[权限](authentication/permissions.md)**：实现基于角色的访问控制。

<a id="admin-panel"></a>

### 管理后台

由 [SQLAdmin](https://github.com/smithyhq/sqladmin) 驱动，可根据 SQLAlchemy 模型生成灵活的管理界面。

- **[管理后台概览](admin-panel/index.md)**：通过 Web 界面管理数据库。
- **[配置](admin-panel/configuration.md)**：安装配置、会话后端与环境变量。
- **[添加模型](admin-panel/adding-models.md)**：注册模型、Schema 及自定义行为。
- **[用户管理](admin-panel/user-management.md)**：管理员、认证和安全。

<a id="performance-caching"></a>
<a id="performance--caching"></a>

### 性能与缓存

- **[缓存概览](caching/index.md)**：通过 Redis 缓存提升性能。
- **[Redis 缓存](caching/redis-cache.md)**：服务端 Redis 缓存。
- **[客户端缓存](caching/client-cache.md)**：HTTP 缓存头与浏览器缓存。
- **[缓存策略](caching/cache-strategies.md)**：高级缓存模式与失效处理。

<a id="background-processing"></a>

### 后台处理

- **[后台任务](background-tasks/index.md)**：通过 Taskiq 处理耗时操作，支持 Redis 或 RabbitMQ 消息代理。

<a id="rate-limiting"></a>

### 限流

- **[限流](rate-limiting/index.md)**：使用基于 Redis 的限流保护 API。

<a id="operations"></a>

### 运维

- **[可观测性](observability.md)**：项目输出的遥测数据，以及如何接入 Logfire 或任意 OpenTelemetry 收集器。

<a id="prerequisites"></a>

## 前置条件

阅读前请确保：

- 已完成[入门指南](../getting-started/index.md)。
- 有一个正在运行的 FastAPI Boilerplate 实例。
- 基本了解 Python、FastAPI 和 REST API。
- 熟悉 SQL 数据库，了解 PostgreSQL 会更有帮助。

<a id="next-steps"></a>

## 推荐阅读路径

<a id="for-new-users"></a>

### 新用户

1. 从[项目结构](project-structure.md)了解代码库。
2. 学习[数据库模型](database/models.md)和 [Schema](database/schemas.md)。
3. 创建第一个 [API 接口](api/endpoints.md)。
4. 添加[身份认证](authentication/index.md)以保护 API。

<a id="for-experienced-developers"></a>

### 有经验的开发者

1. 阅读[数据库 CRUD 操作](database/crud.md)，了解高级模式。
2. 使用[缓存策略](caching/index.md)提升性能。
3. 设置[后台任务](background-tasks/index.md)，实现异步处理。
4. 为生产环境配置[限流](rate-limiting/index.md)。

<a id="for-production-deployment"></a>

### 生产部署

1. 理解[缓存策略](caching/cache-strategies.md)。
2. 根据用户等级配置[限流](rate-limiting/index.md)。
3. 设置[后台任务处理](background-tasks/index.md)。
4. 阅读[生产指南](production.md)，了解部署注意事项。

根据需求和经验选择阅读路径。各章节在前面概念的基础上展开，也可以独立查阅。
