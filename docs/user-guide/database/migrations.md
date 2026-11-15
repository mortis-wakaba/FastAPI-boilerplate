<a id="database-migrations"></a>

# 数据库迁移

结构变更使用 [Alembic](https://alembic.sqlalchemy.org/) 管理。本指南介绍日常流程，以及模板提供的生产环境保护措施。

<a id="two-modes-auto-create-vs-migrations"></a>

## 两种模式：自动建表与迁移

模板同时支持两者，但应**二选一**，而非混用。

<a id="create_tables_on_startuptrue-auto-create"></a>
<a id="create_tables_on_startuptrue--auto-create"></a>

### `CREATE_TABLES_ON_STARTUP=true`：自动建表

启动时调用 `Base.metadata.create_all`，按当前模型创建缺失表。

| 适用场景 | 不适用场景 |
|----------|------------|
| 使用一次性数据库的本地开发 | 需要版本控制的结构变更 |
| 临时 testcontainer 测试 | 多个开发者共用数据库 |
| 快速原型 | 预发布或生产部署 |

由环境变量 `CREATE_TABLES_ON_STARTUP` 控制，`ENVIRONMENT` 为 `local` 或 `development` 时默认开启，其他环境默认关闭。生产安全校验器拒绝在 `production` 设置为 `true`。应用工厂通过 `create_application(create_tables_on_startup=...)` 应用配置。

<a id="alembic-migrations"></a>

### Alembic 迁移

提供可跟踪、可审查、可回滚的结构变更，适合本地试验之外的场景。保持 `CREATE_TABLES_ON_STARTUP` 关闭，这是 local / development 之外的默认值，也是生产环境的要求，然后显式执行迁移。

!!! note "选定功能后生成自己的初始迁移"
    `backend/migrations/versions/` 有意保持为空。项目结构取决于保留的功能，没有适合所有项目的迁移。先移除不需要的功能（见[可组合功能](../composable-features.md)），再为保留部分生成基线。

    **必须连接空数据库生成。** 自动生成比较模型与所连接数据库的差异。如果连接已由 `CREATE_TABLES_ON_STARTUP` 和 `create_all` 建好表的开发数据库，会生成 `upgrade()` 只有 `pass` 的迁移。创建一次性数据库，并在命令中指定：

    ```bash
    # from the repository root, with the local stack up
    docker compose exec postgres createdb -U postgres baseline_tmp

    cd backend
    POSTGRES_SERVER=localhost POSTGRES_PORT=5432 POSTGRES_DB=baseline_tmp \
        CREATE_TABLES_ON_STARTUP=false uv run --no-sync alembic revision --autogenerate -m baseline

    cd .. && docker compose exec postgres dropdb -U postgres baseline_tmp
    ```

    `POSTGRES_USER` 和 `POSTGRES_PASSWORD` 来自 `.env`，这里仅覆盖服务器、端口和数据库名。若发布端口不是 5432，使用 `docker compose port postgres 5432` 查询。

    `POSTGRES_SERVER` 同样重要：`.env.example` 默认使用 Compose 服务名 `postgres`，宿主机 Shell 无法解析，会触发 `socket.gaierror`。可像上面一样指定 `localhost`，或在 Compose 网络的容器中运行，并将 `backend/migrations` 挂载进去，让迁移文件写入工作区而非临时容器。

    `env.py` 导入项目保留的模型，因此基线只包含这些表；移除 tiers 的项目不会生成 `tiers` 表。审查文件，确认 `upgrade()` 确实建表后提交。它将成为每个环境执行的首个迁移。对实际服务数据库执行 `alembic upgrade head`；已经有表时则[标记基线](#applying-a-baseline-to-a-database-that-already-has-tables)。

    使用迁移后保持 `CREATE_TABLES_ON_STARTUP` 关闭，避免应用绕过 Alembic 建表。模型采用约束和索引命名约定，见 `src/infrastructure/database/session.py` 的 `Base.metadata`，确保重新生成时名称稳定。

!!! warning "统一格式之前存储的邮箱"
    `User` 在 `lower(email)` 上声明唯一索引 `ix_user_email_lower`，保证忽略大小写后每个邮箱只有一个账户。旧数据库可能同时存在 `Foo@x.com` 和 `foo@x.com`，导致创建索引失败。先在 `backend/` 执行 `uv run --no-sync python scripts/canonicalize_emails.py`，或在容器中执行 `docker compose exec api python -m scripts.canonicalize_emails`。脚本统一地址格式并添加索引；存在冲突时列出地址且不修改数据。

!!! warning "列长度限制之前存储的头像 URL"
    `User.profile_image_url` 是 `String(2048)`，管理后台和初始数据脚本与客户端受到相同长度限制。旧列没有长度限制，已有超长值会使新增限制的迁移失败。先查找：

    ```sql
    SELECT id FROM "user" WHERE length(profile_image_url) > 2048;
    ```

    这些 URL 无法正常提供服务。应用迁移前缩短它们，或恢复默认值 `https://profileimageurl.com`。

!!! warning "同一路径存在两条限流规则"
    `RateLimit` 在未软删除行的 `(tier_id, path)` 上声明唯一索引 `uq_rate_limits_tier_path`，使每个等级的每条路径只有一条规则。旧数据库可能存在两条，导致索引创建失败。先查找：

    ```sql
    SELECT tier_id, path, count(*) FROM rate_limits
    WHERE is_deleted = false GROUP BY tier_id, path HAVING count(*) > 1;
    ```

    为每个等级和路径保留应生效的规则，删除其余规则。在此之前，调用者实际受到哪条限制取决于查询先读到哪行。

!!! warning "已有数据库"
    命名约定引入之前的数据库使用 PostgreSQL 默认约束名，如 `user_pkey`、`user_tier_id_fkey`、`tiers_name_key`。首次自动迁移会用新名称删除并重建唯一约束，同时删除重复索引 `ix_api_keys_key_prefix`、`ix_key_usage_endpoint`、`ix_key_usage_status_code`，应用前需审查。Alembic 不比较主键和外键名称，因此在后续迁移按名称引用前，先手动重命名：

    ```sql
    ALTER TABLE "user" RENAME CONSTRAINT user_pkey TO pk_user;
    ALTER TABLE "user" RENAME CONSTRAINT user_tier_id_fkey TO fk_user_tier_id_tiers;
    ```

<a id="applying-a-baseline-to-a-database-that-already-has-tables"></a>

### 对已有表的数据库应用基线

旧版本通过 `CREATE_TABLES_ON_STARTUP` 建立的数据库，或希望保留的开发数据库，已经有基线要创建的表，直接应用会在首张表失败：

```text
asyncpg.exceptions.DuplicateTableError: relation "roles" already exists
```

基线描述的结构已经存在，应告诉 Alembic 该迁移已应用，而非重新执行。**标记基线自身的 revision ID，不要标记 `head`**：`head` 会把基线之后的迁移也标记为已执行，导致这些变更永远跳过。

```bash
cd backend
uv run --no-sync alembic history | tail -1      # <base> -> 866640ae8b30, baseline
uv run --no-sync alembic stamp 866640ae8b30     # records the baseline as applied, runs none of it
uv run --no-sync alembic upgrade head           # runs every revision after the baseline
uv run --no-sync alembic check                  # does the database match the models?
```

`alembic history` 按从新到旧列出迁移，最后一行是基线。ID 也在 `migrations/versions/` 的文件名前缀中。`stamp` 只将 ID 写入 `alembic_version`，不执行迁移、不修改业务记录。

`alembic check` 可验证标记后的数据库状态，应在升级**之后**运行。它通过自动生成比较差异，如果还有待执行迁移，会报 `Target database is not up to date.`，而不是报告结构差异。首次采用迁移时如果只有基线，无需升级，标记后检查即可。

它比较当前模型与实际数据库，不比较迁移历史，发现差异时列出操作并以非零退出码结束。因此回答的是“数据库是否具有当前结构”，而非“标记是否正确”。旧版本通过 `create_all` 建立的数据库通常与当前模型生成的基线有差异，`check` 会逐项列出：

```text
ERROR [alembic.util.messaging] New upgrade operations detected:
    [('add_column', None, 'user', Column('token_version', Integer(), nullable=False))]
```

应按提示手动执行 `ALTER TABLE` 修复该数据库。不要为此新增迁移，因为从基线新建的数据库已经有这些结构，再执行会失败。差异很多时，可改用创建旧数据库的版本生成基线，再通过后续迁移表达此后的变更，使历史对应这些数据库实际经历的路径。

**从自动建表切换到迁移的部署顺序：**

1. 运行两个数据修复，处理新版代码读取方式不同的旧数据。宿主机在 `backend/` 执行：

    ```bash
    uv run --no-sync python scripts/canonicalize_emails.py
    uv run --no-sync python scripts/cleanup_api_key_json.py
    ```

    或在容器内执行；容器工作目录是 `/app/src`，直接使用脚本路径会找不到：

    ```bash
    docker compose exec api python -m scripts.canonicalize_emails
    docker compose exec api python -m scripts.cleanup_api_key_json
    ```

2. 对空数据库生成基线，审查并提交，见上文。
3. 对已有表的每个数据库执行 `alembic stamp <baseline revision>`。
4. 正常执行后续迁移 `alembic upgrade head`，再用 `alembic check` 查找与当前模型仍存在的差异，并手动修复。
5. 关闭 `CREATE_TABLES_ON_STARTUP`，避免 Alembic 之外再次建表。`python -m scripts.setup_initial_data` 遵循此设置，关闭时只填充数据而不建表，因此应在步骤 4 之后运行。

<a id="configuration"></a>

## 配置

<a id="backendalembicini"></a>

### `backend/alembic.ini`

随附配置：

- `script_location = %(here)s/migrations`：迁移文件位于 `backend/migrations/`。
- `prepend_sys_path = .`：从 `backend/` 运行时可解析 `src.*`。
- `sqlalchemy.url = driver://user:pass@localhost/dbname`：占位值，实际 URL 在 `env.py` 覆盖。

<a id="backendmigrationsenvpy"></a>

### `backend/migrations/env.py`

模板的 `env.py` 完成三个关键操作：

```python
from src.infrastructure.config.settings import settings
from src.infrastructure.database.session import Base

# 1. The real DATABASE_URL is taken from app settings (which build it from POSTGRES_*)
config.set_main_option("sqlalchemy.url", settings.DATABASE_URL)


# 2. Production safety check — refuses to run unless explicitly opted in
def validate_production_migration():
    if os.getenv("ENVIRONMENT") == "production":
        if os.getenv("CONFIRM_PRODUCTION_MIGRATION") != "yes":
            raise Exception(
                "Production migration requires CONFIRM_PRODUCTION_MIGRATION=yes"
            )


# 3. Auto-import every module under src.modules so Alembic sees all models
import_models("src.modules")
target_metadata = Base.metadata
```

自动导入递归遍历 `src.modules`，无需维护模型导入列表。创建 `modules/<feature>/models.py` 即可被 `--autogenerate` 发现。导入错误会立即失败，不会跳过，因此损坏的模型模块不会静默生成大量 `drop_table`。

<a id="workflow"></a>

## 工作流

所有命令在 `backend/` 下运行：

```bash
cd backend
```

<a id="1-generate-a-migration"></a>

### 1. 生成迁移

修改模型后：

```bash
uv run --no-sync alembic revision --autogenerate -m "Add user profile fields"
```

Alembic 比较当前模型和数据库结构，在 `backend/migrations/versions/` 创建文件。

<a id="2-review-the-generated-migration"></a>

### 2. 审查迁移

应用前始终阅读。自动生成可能遗漏枚举变化、服务端默认值、计算列和复杂约束重命名。典型文件如下：

```python
"""Add user profile fields

Revision ID: abc123def456
Revises: prev_revision_id
"""

from alembic import op
import sqlalchemy as sa

revision = "abc123def456"
down_revision = "prev_revision_id"
branch_labels = None
depends_on = None


def upgrade() -> None:
    op.add_column("user", sa.Column("bio", sa.String(500), nullable=True))
    op.add_column("user", sa.Column("website", sa.String(255), nullable=True))
    op.create_index("ix_user_website", "user", ["website"])


def downgrade() -> None:
    op.drop_index("ix_user_website", "user")
    op.drop_column("user", "website")
    op.drop_column("user", "bio")
```

<a id="3-apply"></a>

### 3. 应用

```bash
# All pending migrations
uv run --no-sync alembic upgrade head

# Step forward N revisions
uv run --no-sync alembic upgrade +2

# Up to a specific revision
uv run --no-sync alembic upgrade abc123def456
```

<a id="4-inspect"></a>

### 4. 检查

```bash
uv run --no-sync alembic current      # current revision
uv run --no-sync alembic history      # full history
uv run --no-sync alembic heads        # any branched heads
uv run --no-sync alembic show <rev>   # details about a specific revision
```

<a id="5-roll-back"></a>

### 5. 回滚

```bash
uv run --no-sync alembic downgrade -1                # one step back
uv run --no-sync alembic downgrade <revision>        # to a specific point
uv run --no-sync alembic downgrade base              # all the way back
```

在开发环境测试 downgrade，是发现遗漏 `op.drop_index` 等操作的低成本方法。

<a id="common-scenarios"></a>

## 常见场景

<a id="adding-a-new-model"></a>

### 添加新模型

1. **创建包含模型的模块目录**：

    ```python
    # backend/src/modules/widgets/models.py
    from sqlalchemy import String
    from sqlalchemy.orm import Mapped, mapped_column

    from ...infrastructure.database.models import SoftDeleteMixin, TimestampMixin
    from ...infrastructure.database.session import Base


    class Widget(Base, TimestampMixin, SoftDeleteMixin):
        __tablename__ = "widgets"

        id: Mapped[int] = mapped_column(
            "id", autoincrement=True, nullable=False,
            primary_key=True, init=False,
        )
        name: Mapped[str] = mapped_column(String(100), unique=True, index=True)
    ```

2. **在 `modules/__init__.py` 注册**，供其他代码导入：

    ```python
    from .widgets.models import Widget
    ```

    迁移 `env.py` 自动导入 `src.modules`，因此无论是否注册都能发现；原文建议显式导入以明确公开 API。

3. **生成并应用**：

    ```bash
    uv run --no-sync alembic revision --autogenerate -m "Add widgets table"
    # review backend/migrations/versions/<timestamp>_add_widgets_table.py
    uv run --no-sync alembic upgrade head
    ```

<a id="adding-a-foreign-key"></a>

### 添加外键

```python
# Add to the model
class Widget(Base, ...):
    owner_id: Mapped[int] = mapped_column(ForeignKey("user.id"), index=True)
```

```bash
uv run --no-sync alembic revision --autogenerate -m "Add widget.owner_id"
```

生成的迁移包含列、外键约束和索引。

<a id="data-migrations"></a>

### 数据迁移

结构变更有时需要回填数据，可以编辑自动生成的 upgrade：

```python
def upgrade() -> None:
    # 1. Add nullable column
    op.add_column("post", sa.Column("category_id", sa.Integer(), nullable=True))

    # 2. Backfill
    bind = op.get_bind()
    bind.execute(sa.text(
        "INSERT INTO category (name, slug) VALUES ('General', 'general')"
    ))
    default_id = bind.execute(
        sa.text("SELECT id FROM category WHERE slug = 'general'")
    ).scalar_one()
    bind.execute(sa.text(
        "UPDATE post SET category_id = :cid WHERE category_id IS NULL"
    ), {"cid": default_id})

    # 3. Tighten the constraint
    op.alter_column("post", "category_id", nullable=False)
    op.create_foreign_key(
        "fk_post_category_id", "post", "category", ["category_id"], ["id"]
    )
```

<a id="renaming-a-column"></a>

### 重命名列

```python
def upgrade() -> None:
    op.alter_column("user", "full_name", new_column_name="name")


def downgrade() -> None:
    op.alter_column("user", "name", new_column_name="full_name")
```

包含外键或索引的列可能被自动生成为删除再创建。需要保留数据时，改为 `alter_column`。

<a id="dropping-a-table"></a>

### 删除表

```python
def upgrade() -> None:
    op.drop_table("old_table")


def downgrade() -> None:
    op.create_table(
        "old_table",
        sa.Column("id", sa.Integer(), nullable=False),
        sa.Column("name", sa.String(50)),
        sa.PrimaryKeyConstraint("id"),
    )
```

<a id="production-migration-strategy"></a>

## 生产迁移策略

模板通过明确确认防止误执行生产迁移。`ENVIRONMENT=production` 时，必须设置 `CONFIRM_PRODUCTION_MIGRATION=yes`，否则 Alembic 拒绝运行。

```bash
# Production migration — explicit confirmation required
ENVIRONMENT=production CONFIRM_PRODUCTION_MIGRATION=yes uv run --no-sync alembic upgrade head
```

<a id="recommended-flow"></a>

### 推荐流程

1. 对生产数据副本在本地**开发并测试迁移**。
2. **部署包含新迁移文件的代码到预发布环境**。
3. **备份预发布数据库**：

    ```bash
    pg_dump -h staging-db -U user dbname > staging_backup_$(date +%Y%m%d_%H%M%S).sql
    ```

4. **在预发布执行迁移**并进行冒烟测试。
5. 破坏性迁移应**安排维护窗口**。
6. **备份生产数据库**：

    ```bash
    pg_dump -h prod-db -U user dbname > prod_backup_$(date +%Y%m%d_%H%M%S).sql
    ```

7. 带明确确认变量**执行生产迁移**。
8. 迁移后**观察日志和指标**。

<a id="docker-migration-job"></a>

### Docker 迁移作业

`bp deploy generate prod` 和 `nginx` 会生成 `migrate` 服务，从 `backend/Dockerfile` 的 `migrate` 阶段构建，执行 `alembic upgrade head` 后退出。它位于 Compose profile 中，其他服务不等待它：

```bash
docker compose --profile migrate run --rm migrate   # then, only if it succeeded:
docker compose up -d
```

先迁移、后重启的顺序很重要：迁移失败时，旧代码继续使用旧结构提供服务，避免 API 重启到不兼容结构。运行中的服务部署结构变更也采用两步：迁移，然后 `docker compose up -d` 替换有变化的容器。

<a id="best-practices"></a>

## 最佳实践

<a id="always-review-autogenerated-migrations"></a>

### 始终审查自动迁移

自动生成可能遗漏枚举、服务端默认值、部分约束重命名和计算列。

<a id="use-descriptive-messages"></a>

### 使用描述清晰的消息

```bash
# Good
uv run --no-sync alembic revision --autogenerate -m "Add user.email_verified"

# Less useful three months later
uv run --no-sync alembic revision --autogenerate -m "Update user model"
```

<a id="adding-a-non-nullable-column-to-a-populated-table"></a>

### 向已有数据的表添加非空列

在同一迁移中分三步：

```python
def upgrade() -> None:
    op.add_column("user", sa.Column("phone", sa.String(20), nullable=True))
    op.execute("UPDATE \"user\" SET phone = ''")
    op.alter_column("user", "phone", nullable=False)
```

<a id="test-downgrades"></a>

### 测试回滚

```bash
uv run --no-sync alembic downgrade -1
uv run --no-sync alembic upgrade head
```

回滚失败时，应在合并迁移前修复。

<a id="dont-commit-auto-generated-pyc-files"></a>

### 不提交自动生成的 `.pyc`

`migrations/versions/` 只保留 `.py` 源文件，Alembic 不跟踪 `.pyc`。

<a id="troubleshooting"></a>

## 故障排查

<a id="target-database-is-not-up-to-date"></a>

### “Target database is not up to date”

数据库落后一个或多个迁移。先执行 `uv run --no-sync alembic upgrade head`，再生成新迁移。

<a id="multiple-heads-detected"></a>

### “Multiple heads detected”

两个分支从同一父迁移新增了迁移，需要合并：

```bash
uv run --no-sync alembic merge -m "merge heads" <head1> <head2>
```

<a id="autogenerate-produces-an-empty-migration"></a>

### 自动生成空迁移

模型没有结构变化，或 `target_metadata` 未发现模型。检查新模块能否从 `src.modules` 导入，`env.py` 的自动导入器会遍历该包。

<a id="migration-applies-in-dev-but-fails-in-prod"></a>

### 开发成功、生产失败

常见原因是数据不同，例如生产列存在 NULL 却要改为 NOT NULL，Postgres 版本不同，或扩展只在某环境安装。始终使用接近生产的数据测试。

<a id="next-steps"></a>

## 后续步骤

- **[CRUD 操作](crud.md)**：使用迁移后的结构。
- **[API 接口](../api/endpoints.md)**：基于新模型构建接口。
- **[生产环境](../production.md)**：生产部署指南。
