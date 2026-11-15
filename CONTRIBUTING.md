<a id="contributing-to-fastapi-boilerplate"></a>

# 参与 FastAPI-boilerplate 贡献

感谢你关注 FastAPI-boilerplate！本指南帮助你开始贡献。报告问题、补充文档和回答问题同样有价值。新增功能可按以下流程进行。

<a id="setting-up-your-development-environment"></a>

## 配置开发环境

<a id="cloning-the-repository"></a>

### 克隆仓库

先 Fork 并克隆 FastAPI-boilerplate：

1. **Fork 仓库**：访问 https://github.com/igormagalhaesr/FastAPI-boilerplate 并点击 Fork。
2. **创建功能分支**：运行 `git checkout -b feature/fooBar`。
3. **测试修改**：从 `backend/` 执行 `uv run --no-sync pytest`，确认没有破坏现有功能。

<a id="using-uv-for-dependency-management"></a>

### 使用 uv 管理依赖

项目使用 uv，安装方式见 [uv 官方文档](https://docs.astral.sh/uv/)。

安装后进入克隆的仓库，安装依赖：

```sh
cd FastAPI-boilerplate
uv sync --all-packages --all-extras
```

`--all-packages` 安装两个 workspace 成员，`--all-extras` 添加开发工具。单独运行 `uv sync` 不会保留 ruff、mypy 和 pytest 等开发依赖。

<a id="activating-the-virtual-environment"></a>

### 激活虚拟环境

uv 会创建项目虚拟环境，可通过以下命令激活：

```sh
source .venv/bin/activate
```

也可不激活，直接用 `uv run`：

```sh
uv run --no-sync python your_script.py
```

<a id="making-contributions"></a>

## 开发贡献

<a id="coding-standards"></a>

### 编码规范

- 遵循 PEP 8。
- 为新功能或缺陷修复编写有意义的测试。

<a id="testing-with-pytest"></a>

### 使用 pytest 测试

从 `backend/` 运行：

```sh
cd backend
uv run --no-sync pytest
```

`--no-sync` 跳过 uv 的隐式同步，立即执行命令，不改变环境和锁文件。普通 `uv run` 会按锁文件补充依赖，但不删除已有包；不带 `--all-extras` 的 `uv sync` 则会移除开发工具。

<a id="linting"></a>

### 静态检查

使用 mypy 检查类型：

```sh
mypy src scripts migrations tests
```

使用 ruff 检查和格式化代码：

```sh
ruff check --fix
ruff format
```

提交前确保检查通过。

<a id="using-pre-commit-for-better-code-quality"></a>

### 使用 pre-commit 改善代码质量

pre-commit 在代码审查前自动发现常见问题，保持质量和风格一致。

1. **安装钩子**：执行 `pre-commit install`，将钩子安装到 `.git/`，之后提交自动检查。
2. **提交修改**：原文示例为 `git commit -am 'Add some fooBar'`，提交时运行 pre-commit。检查可能阻止提交，应修复后重新提交。
3. **推送并创建 PR**：执行 `git push origin feature/fooBar`，在 GitHub 的 Fork 中向主仓库发起 Pull Request。

<a id="additional-notes"></a>

### 补充说明

**保持同步**：定期获取并合并上游变更，减少冲突。

**遵守项目约定**：遵循代码风格、规范和提交消息要求。

**公开交流**：通过 Issue 或 Discussions 提问和讨论。

<a id="submitting-your-contributions"></a>

## 提交贡献

<a id="creating-a-pull-request"></a>

### 创建 Pull Request

修改完成后：

- 推送到自己的 Fork。
- 创建描述清晰的 PR。
- 必要时更新 README.md。

<a id="code-reviews"></a>

### 代码审查

- 处理审查反馈。
- 通过后合并到主分支。

<a id="code-of-conduct"></a>

## 行为准则

请遵守[行为准则](CODE_OF_CONDUCT.md)，共同维护友好、包容的社区。

感谢你为 FastAPI-boilerplate 作出贡献！🚀
