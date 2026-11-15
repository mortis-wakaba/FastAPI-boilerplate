<a id="plugins"></a>

# 插件

bp 有两个刻意分离的扩展点，生命周期不同，不共用同一执行机制：

- **命令插件**（bp.commands）：挂载到根命令的第三方 Typer 子应用，如 bp aws deploy；提供命令行接口，不通过功能安装器修改仓库。
- **功能插件**（bp.features）：带清单的代码生成器，设计上由 bp feature 列出、安装和移除，可写文件、改配置、运行钩子。

两者通过 [Python entry points](https://packaging.python.org/en/latest/specifications/entry-points/) 发现，因此插件就是用户机器上安装的普通 Python 包。

<a id="how-discovery-works"></a>

## 发现机制

bp 启动时：

1. 挂载内置 deploy、env 子应用。
2. 调用 cli.plugins 的 discover_command_plugins，遍历 `entry_points(group='bp.commands')`，挂载可正常加载的结果。
3. 按需构建功能注册表，合并内置功能和从 bp.features 发现的插件。

加载逐项包裹 try/except，损坏插件产生 RuntimeWarning 后跳过，其余仍可用：

```python
# cli/src/cli/plugins.py (abridged)
def _safe_load(ep: EntryPoint) -> object | None:
    try:
        return ep.load()
    except Exception as exc:
        warnings.warn(f"Failed to load plugin {ep.name!r}: {exc}", RuntimeWarning)
        return None
```

名称冲突时内置命令和功能优先，插件不能静默覆盖。

<a id="command-plugins"></a>

## 命令插件

希望不修改模板就为 bp 添加顶层命令时使用，如 bp fly deploy、bp pulumi up、bp lint custom-rules。

<a id="authoring-a-command-plugin"></a>

### 编写命令插件

创建含 typer.Typer 实例的 Python 包，并注册到 bp.commands 组。

<a id="1-package-layout"></a>

#### 1. 包结构

```text
bp-deploy-fly/
├── pyproject.toml
└── src/
    └── bp_deploy_fly/
        ├── __init__.py
        └── cli.py
```

<a id="2-clipy-define-the-sub-app"></a>
<a id="2-clipy--define-the-sub-app"></a>

#### 2. 在 cli.py 定义子应用

```python
import typer

app = typer.Typer(
    name="fly",
    help="Deploy this app to Fly.io.",
    no_args_is_help=True,
)


@app.command("deploy")
def deploy(
    region: str = typer.Option("ord", help="Fly region to target."),
    yes: bool = typer.Option(False, "--yes", "-y"),
) -> None:
    """Build and deploy the current branch to Fly.io."""
    typer.echo(f"deploying to fly region={region}")
    # ... real work here ...
```

<a id="3-pyprojecttoml-declare-the-entry-point"></a>
<a id="3-pyprojecttoml--declare-the-entry-point"></a>

#### 3. 在 pyproject.toml 声明入口

```toml
[project]
name = "bp-deploy-fly"
version = "0.1.0"
requires-python = ">=3.11"
dependencies = ["typer>=0.12"]

[project.entry-points."bp.commands"]
fly = "bp_deploy_fly.cli:app"

[tool.setuptools.packages.find]
where = ["src"]
```

键 fly 成为子命令名，值 bp_deploy_fly.cli:app 指向 Typer 实例的导入路径。

<a id="4-install-and-use"></a>

#### 4. 安装并使用

```bash
uv pip install bp-deploy-fly
uv run --no-sync bp --help
# ╭─ Commands ────────────────────────────────────────────────╮
# │ deploy  Generate deployment artifacts ...                 │
# │ env     Inspect and prepare the runtime environment.      │
# │ fly     Deploy this app to Fly.io.            ← new       │
# ╰───────────────────────────────────────────────────────────╯

uv run --no-sync bp fly deploy --region ord --yes
```

<a id="best-practices-for-command-plugins"></a>

### 命令插件最佳实践

- **名称唯一**：与 deploy、env、feature 冲突会告警并忽略。优先 fly、aws、pulumi 等具体名称，而非 cloud、tools。
- **子应用独立**：只用公开 Typer 接口，不依赖宿主 CLI 内部实现，cli.app 不是公共 API。
- **遵循交互约定**：非交互支持 --yes，适用时支持 --dry-run，与内置命令一致。
- **妥善报告外部依赖缺失**：需要 flyctl、kubectl 时先检查并提供清晰错误，不直接暴露 subprocess 堆栈。

<a id="feature-plugins"></a>

## 功能插件

需要**修改仓库**时使用，例如添加模块、OAuth 提供方、支付集成或迁移。功能通过清单、计划、安装器管理应用、列举和未来的撤销。

!!! note "功能框架已实现，尚未接入 bp feature"
    Feature、FeatureManifest、FeaturePlan、FeatureInstaller 已实现，注册表也能发现插件，但面向用户的 bp feature 子应用尚未发布。当前只能由 bp deploy generate 等内置命令使用功能框架。原文将下述契约标为稳定，并计划在 list / add / remove / info 发布后直接复用插件。

<a id="the-feature-contract"></a>

### 功能契约

继承 cli.features.base.Feature：

```python
from cli.features.base import Feature, FeatureManifest, FeaturePlan, FileOp
from cli.lib.project import ProjectContext


class MyFeature(Feature):
    def manifest(self) -> FeatureManifest:
        return FeatureManifest(
            name="my-feature",
            version="1.0",
            summary="One-line description shown by `bp feature list`.",
        )

    def plan(self, params: dict, project: ProjectContext) -> FeaturePlan:
        return FeaturePlan(
            manifest=self.manifest(),
            templates_root=Path(__file__).parent / "templates",
            template_context={"project_name": project.repo_root.name, **params},
            files=(
                FileOp(template="my_module.py.j2",
                       target=project.backend_dir / "src/modules/my_feature/__init__.py"),
                # ... more files ...
            ),
        )
```

两个方法都必须实现：

- **manifest()**：静态元数据，注册表和列表可调用，不实际执行安装。
- **plan(params, project)**：接受运行参数和 ProjectContext，返回具体 FeaturePlan。

<a id="the-plan-schema"></a>

### 计划结构

```python
@dataclass(frozen=True)
class FeaturePlan:
    manifest: FeatureManifest
    templates_root: Path | None
    template_context: dict[str, Any] = field(default_factory=dict)
    files: tuple[FileOp, ...] = ()
    codemods: tuple[Codemod, ...] = ()    # reserved for v2 — installer raises NotImplementedError
    hooks: tuple[Hook, ...] = ()          # reserved for v2 — installer raises NotImplementedError
```

结构覆盖当前的 Jinja 渲染写盘及未来代码修改、迁移钩子。提前定义 Codemod、Hook 类型，使未来安装器支持它们时不破坏插件契约；当前声明这些操作会触发 NotImplementedError。

<a id="fileop"></a>

#### FileOp

```python
@dataclass(frozen=True)
class FileOp:
    template: str          # path relative to templates_root
    target: Path           # absolute path on disk
    overwrite: bool = False
    skip_if_exists: bool = False
    executable: bool = False
```

安装器使用计划的 template_context 渲染 template，再写入 target。目标已存在时：

| 设置 | 行为 |
|------|------|
| 默认 | 提示确认覆盖 |
| overwrite=True | 不询问直接覆盖 |
| skip_if_exists=True | 静默跳过，保留用户定制 |

executable=True 为所有者、组及其他用户添加执行权限 +x。

<a id="authoring-a-feature-plugin"></a>

### 编写功能插件

<a id="1-package-layout_1"></a>

#### 1. 包结构

```text
bp-feature-audit-log/
├── pyproject.toml
└── src/
    └── bp_feature_audit_log/
        ├── __init__.py
        ├── feature.py
        └── templates/
            └── audit_log_module.py.j2
```

<a id="2-featurepy-define-the-feature"></a>
<a id="2-featurepy--define-the-feature"></a>

#### 2. 在 feature.py 定义功能

```python
from pathlib import Path

from cli.features.base import Feature, FeatureManifest, FeaturePlan, FileOp
from cli.lib.project import ProjectContext


class AuditLogFeature(Feature):
    def manifest(self) -> FeatureManifest:
        return FeatureManifest(
            name="audit-log",
            version="1.0",
            summary="Drop in an audit-log module that records mutating requests.",
        )

    def plan(self, params: dict, project: ProjectContext) -> FeaturePlan:
        templates_root = Path(__file__).parent / "templates"
        module_path = (
            project.backend_dir
            / "src/modules/audit_log/models.py"
        )
        return FeaturePlan(
            manifest=self.manifest(),
            templates_root=templates_root,
            template_context={
                "retention_days": params.get("retention_days", 90),
            },
            files=(
                FileOp(template="audit_log_module.py.j2", target=module_path),
            ),
        )


# Entry-point target — can be a Feature instance or a callable that returns one.
feature = AuditLogFeature()
```

插件应写入当前布局中实际存在的目录。因此示例采用 src/modules/<name>/，而非旧 auth/oauth/providers/。OAuth 不再按提供方拆文件，而是在 infrastructure/auth/setup.py 通过 crudauth OAuthCredentials 配置。因此添加提供方应修改该文件，例如幂等补丁，而非新建旧式模块。

<a id="3-pyprojecttoml-declare-the-entry-point_1"></a>
<a id="3-pyprojecttoml--declare-the-entry-point_1"></a>

#### 3. 在 pyproject.toml 声明入口

```toml
[project]
name = "bp-feature-audit-log"
version = "0.1.0"
requires-python = ">=3.11"
dependencies = ["fastapi-boilerplate-cli"]

[project.entry-points."bp.features"]
audit-log = "bp_feature_audit_log.feature:feature"

[tool.setuptools]
include-package-data = true

[tool.setuptools.packages.find]
where = ["src"]

[tool.setuptools.package-data]
"*" = ["*.j2"]
```

必须配置 include-package-data 和 package-data 通配规则，否则 Jinja 模板不会打入 wheel。

<a id="4-install"></a>

#### 4. 安装

```bash
uv pip install bp-feature-audit-log
# Once `bp feature` ships:
# uv run --no-sync bp feature add audit-log --retention-days 90
```

<a id="best-practices-for-feature-plugins"></a>

### 功能插件最佳实践

- **幂等**：重复运行应安全。用户可能修改的文件使用 skip_if_exists=True。
- **尊重项目结构**：使用 ProjectContext.backend_dir / repo_root，不写死路径。
- **声明全部文件**：不在计划外写文件，否则 dry-run、幂等和覆盖确认无法生效。
- **模板放功能旁**：采用 `Path(__file__).parent / 'templates'`，保持包独立。
- **使用 StrictUndefined**：内置 Renderer 已开启，缺上下文变量明确失败，而非渲染空字符串。
- **记录必要参数**：未来 feature 命令的 manifest summary 是用户入口，应清楚说明 params。

<a id="discovery-diagnosis"></a>
<a id="discovery--diagnosis"></a>

## 发现与诊断

<a id="listing-plugins"></a>

### 列出插件

```bash
# In-tree + plugin commands
uv run --no-sync bp --help

# (Once shipped) — list features:
# uv run --no-sync bp feature list
```

<a id="inspecting-entry-points"></a>

### 检查 entry points

```bash
uv run --no-sync python -c "
from importlib.metadata import entry_points
for ep in entry_points(group='bp.commands'):
    print(f'command  {ep.name:>15s}  ->  {ep.value}')
for ep in entry_points(group='bp.features'):
    print(f'feature  {ep.name:>15s}  ->  {ep.value}')
"
```

<a id="troubleshooting"></a>

### 故障排查

<a id="my-plugin-doesnt-appear-in-bp-help"></a>
<a id="my-plugin-doesnt-appear-in-bp---help"></a>

#### 插件未出现在 bp --help

1. **确认安装到正确虚拟环境**：运行上面的入口检查脚本，未列出表示没有装到 bp 使用的环境。
2. **检查加载警告**：导入失败向 stderr 输出 warning: Failed to load plugin，可运行 `bp --help 2>&1` 确保没有隐藏。
3. **检查入口类型**：命令必须解析为 typer.Typer；功能为 Feature 实例或返回实例的 callable。

<a id="my-plugin-shadows-a-built-in"></a>

#### 插件与内置名称冲突

bp 会告警并忽略占用 deploy、env、feature 的插件，改成唯一名称。

<a id="my-templates-arent-shipping-in-the-wheel"></a>

#### wheel 中没有模板

pyproject.toml 设置 include-package-data=true，并添加如 `"*" = ["*.j2"]` 的 package-data 通配规则，否则代码旁的模板不会打包。

<a id="roadmap"></a>

## 路线图

- **bp feature {list,add,remove,info}**：功能框架的用户命令，目前只有注册表。
- **Codemod 执行**：应用 FeaturePlan.codemods 声明的 AST 修改，适合添加模块并注册路由。
- **安装后钩子**：执行 FeaturePlan.hooks 中的命令，如生成迁移后运行 alembic upgrade head。

Codemod、Hook 已在 cli.features.base 定义，插件作者可以声明而不改变未来结构，但不代表当前安装器能够执行。
