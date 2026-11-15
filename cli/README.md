<a id="bp-fastapi-boilerplate-cli"></a>
<a id="bp--fastapi-boilerplate-cli"></a>

# bp：FastAPI-boilerplate CLI

`bp` 是基于 FastAPI 模板的项目所用的开发与运维命令行工具。它生成部署文件、辅助准备运行环境，并承载插件命令和功能生成器。

<a id="install"></a>

## 安装

该包属于 workspace。在仓库根目录运行：

```bash
uv sync --all-packages --all-extras   # bp is then available via `uv run bp`
uv run bp --help
```

如需在仓库外也能使用，将其安装为全局工具：

```bash
uv tool install --editable ./cli
bp --help
```

<a id="whats-here"></a>

## 目录内容

```
cli/src/cli/
├── app.py                 root Typer app + plugin discovery
├── plugins.py             entry-point loaders for bp.commands and bp.features
├── commands/              in-tree command sub-apps
│   ├── deploy.py          bp deploy generate <mode>
│   └── env.py             bp env gen-secret / bp env validate
├── features/              feature framework (manifest, plan, installer)
│   └── _builtins/         in-tree features
│       └── deploy/        compose/Dockerfile templates for local/prod/nginx
└── lib/                   shared helpers (project discovery, prompts, render)
```

<a id="plugin-extension-points"></a>

## 插件扩展点

两类插件有意分开：

- `bp.commands` entry-point 组：将第三方 Typer 子应用挂载为 `bp <name>`，例如 `bp aws deploy`。
- `bp.features` entry-point 组：带有清单的代码生成器，设计上由 `bp feature` 列出、安装和移除。

发现机制的契约见 `cli/src/cli/plugins.py`。`bp feature` 用户命令是否已提供，见[插件指南](../docs/cli/plugins.md)中的实现状态说明。
