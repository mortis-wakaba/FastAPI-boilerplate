<a id="bp-the-fastapi-boilerplate-cli"></a>
<a id="bp--the-fastapi-boilerplate-cli"></a>

# bp：FastAPI-boilerplate CLI

`bp` 是随模板提供的开发与运维工具，用于生成部署文件、准备运行环境，以及承载插件命令和功能生成器。

它**不属于可部署后端**，独立放在 workspace 的 `cli/` 包，仅安装在开发环境或明确安装了该工具的机器上，使后端镜像保持精简。

<a id="whats-in-this-section"></a>

## 本章内容

- **[命令](commands.md)**：内置 deploy 和 env 子应用的完整参考。
- **[插件](plugins.md)**：bp.commands、bp.features 扩展点与编写指南。

<a id="repository-layout"></a>

## 仓库结构

```text
fastapi-boilerplate/
├── pyproject.toml          # workspace root (uv workspace metadata)
├── backend/                # deployable application — never ships bp
│   └── src/...
├── cli/                    # the bp package
│   ├── pyproject.toml      # typer + jinja2 + workspace dep on backend
│   └── src/cli/
│       ├── app.py          # Typer root + plugin discovery
│       ├── commands/       # in-tree command sub-apps
│       ├── features/       # feature framework + in-tree features
│       └── plugins.py      # entry-point loaders
└── docs/
```

两包有意分离：cli 依赖 backend，例如环境校验命令需要后端配置；backend 不反向依赖 cli，生产镜像只包含 backend。

<a id="install"></a>

## 安装

<a id="in-repo-most-common"></a>

### 仓库内使用（最常见）

```bash
git clone https://github.com/benavlabs/fastapi-boilerplate
cd fastapi-boilerplate
uv sync --all-packages --all-extras   # one venv with the backend, the cli and the dev tools
uv run --no-sync bp --help         # works from anywhere in the repo
```

workspace 共用根目录的 `.venv/`。uv 会向上查找根目录，因此任意子目录均可执行 `uv run --no-sync bp`。

<a id="machine-wide-optional"></a>

### 安装为全局工具（可选）

希望仓库外 PATH 中也有 bp 时：

```bash
uv tool install --editable ./cli
bp --help
```

`--editable` 使 cli/src/cli 的修改立即生效，无需重装。仅当 cli/pyproject.toml 的依赖或入口改变时重新安装。卸载：

```bash
uv tool uninstall fastapi-boilerplate-cli
```

<a id="quick-tour"></a>

## 快速浏览

<a id="generate-a-compose-file"></a>

### 生成 Compose 文件

```bash
uv run --no-sync bp deploy generate local                  # hot-reload dev stack
uv run --no-sync bp deploy generate prod --workers 8       # production stack
uv run --no-sync bp deploy generate nginx                  # production behind nginx
```

默认写入仓库根目录的 docker-compose.yml；nginx 模式还生成 nginx/default.conf。通过 `--output-dir` 指定其他位置。

<a id="generate-a-secret"></a>

### 生成密钥

```bash
uv run --no-sync bp env gen-secret
# → 64-char hex suitable for SECRET_KEY
```

<a id="audit-production-settings"></a>

### 检查生产配置

```bash
uv run --no-sync bp env validate
# Forces the production security validator regardless of ENVIRONMENT.
# Exits 1 if any critical issues are found.
```

<a id="command-tree"></a>

## 命令树

```text
bp
├── deploy
│   └── generate <mode> [options]    # mode ∈ {local, prod, nginx}
└── env
    ├── gen-secret [--bytes N]
    └── validate
```

插件子应用与 deploy、env 同级挂载，见[插件](plugins.md)。

<a id="design-principles"></a>

## 设计原则

- **生产运行时不依赖 bp**：它属于开发与运维工具，生产镜像不包含 cli 包。
- **扩展点分离**：命令插件是 Typer 子应用，功能插件是带清单的生成器，生命周期不同。
- **模板作为数据**：内置和插件功能都渲染 Jinja 模板，安装器统一执行计划。
- **幂等及预览**：修改仓库前支持覆盖确认和 `--dry-run`。

<a id="next-steps"></a>

## 后续步骤

- **[命令](commands.md)**：已有命令参考。
- **[插件](plugins.md)**：编写命令或功能插件。
