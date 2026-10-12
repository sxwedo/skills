# 新建项目与工程配置

## 1. 初始化命令

库或服务（有 `src/` 布局，可安装、可发布）：

```bash
uv init --package --build-backend hatch --python 3.12 <名称>
cd <名称>
uv add --dev ruff pyright pytest pytest-cov pre-commit
```

不需要打包的小应用用 `uv init --app --python 3.12 <名称>`。

只装工具链，运行时依赖等需要时再 `uv add`。`uv init` 会生成 `pyproject.toml`、`.python-version`、`README.md`，并初始化 git 仓库。`uv add` 首次执行时生成 `uv.lock`，该文件必须提交。

按需追加的开发依赖：

```bash
uv add --dev pytest-asyncio   # 有 async 测试时
uv add --dev hypothesis       # 有解析、序列化、转换逻辑时
uv add --dev respx            # 用了 httpx 并需要测试时
```

写完配置后启用钩子：

```bash
uv run pre-commit install
```

## 2. pyproject.toml 工具配置

`[build-system]`、`[project]`、`[dependency-groups]` 由 `uv init` 与 `uv add` 生成，不要手写依赖版本。把下面的工具配置追加到文件末尾：

```toml
[tool.ruff]
line-length = 100
target-version = "py312"
src = ["src", "tests"]

[tool.ruff.lint]
select = [
    "E", "W",   # pycodestyle
    "F",        # pyflakes
    "I",        # isort 排序
    "N",        # pep8-naming
    "UP",       # pyupgrade：自动升级到现代语法
    "B",        # bugbear：常见隐患
    "A",        # 不遮蔽内置名
    "C4",       # 推导式
    "DTZ",      # 禁止无时区的 datetime
    "SIM",      # 简化
    "PTH",      # 用 pathlib
    "PT",       # pytest 风格
    "RET",      # return 风格
    "RUF",
    "S",        # bandit 安全检查
    "T20",      # 禁止 print
    "ASYNC",    # async 中的阻塞调用
    "PERF",     # 性能反模式
    "LOG",      # logging 误用
]

[tool.ruff.lint.per-file-ignores]
"tests/**" = ["S101", "S105", "S106"]   # 测试里允许 assert 与假密码

[tool.pyright]
include = ["src", "tests"]
pythonVersion = "3.12"
venvPath = "."
venv = ".venv"
typeCheckingMode = "standard"
strict = ["src"]                         # 源码严格，测试标准

[tool.pytest.ini_options]
testpaths = ["tests"]
addopts = "-ra --strict-markers --strict-config --cov --cov-report=term-missing"
asyncio_mode = "auto"                    # 仅在装了 pytest-asyncio 时保留

[tool.coverage.run]
source = ["src"]
branch = true

[tool.coverage.report]
fail_under = 80
show_missing = true
```

说明：

- 没有启用 `TC`（类型导入检查）规则，因为 FastAPI、pydantic、SQLAlchemy 需要在运行时读取类型注解，该规则会把这些导入挪进 `TYPE_CHECKING` 而导致运行时报错。
- `asyncio_mode` 一行在没装 pytest-asyncio 时会触发 `--strict-config` 报错，此时删除该行。
- `fail_under = 80` 是起点，随项目成熟再提高，不要为了数字写无意义的测试。

## 3. 目录结构

使用 `src/` 布局，避免测试时误导入未安装的源码目录：

```
<名称>/
├── pyproject.toml
├── uv.lock
├── .python-version
├── .pre-commit-config.yaml
├── README.md
├── src/<包名>/
│   ├── __init__.py
│   ├── py.typed            # 作为库发布时需要
│   ├── settings.py         # pydantic-settings
│   ├── errors.py           # AppError 层级
│   ├── logging_config.py   # structlog 配置
│   ├── api/                # 路由与依赖（只依赖 services、schemas）
│   ├── schemas/            # pydantic 模型（进出系统的数据形状）
│   ├── services/           # 业务逻辑（不导入 fastapi）
│   └── db/                 # base.py、models.py、session.py
├── migrations/             # alembic
└── tests/
    ├── conftest.py
    └── ...                 # 目录结构镜像 src
```

依赖方向单向：`api → services → db`。业务逻辑不认识 HTTP，数据库层不认识业务规则，这样换入口（HTTP、CLI、任务）时服务层原样复用。

命令行工具在 `[project.scripts]` 声明入口，例如 `mytool = "mypkg.cli:app"`，之后 `uv run mytool` 即可运行。

## 4. pre-commit

用本地钩子调用 `uv run`，这样工具版本始终与 `uv.lock` 一致，不需要在钩子配置里再钉一遍版本。

`.pre-commit-config.yaml`：

```yaml
repos:
  - repo: local
    hooks:
      - id: ruff-check
        name: ruff check
        entry: uv run ruff check --fix
        language: system
        types: [python]
      - id: ruff-format
        name: ruff format
        entry: uv run ruff format
        language: system
        types: [python]
      - id: pyright
        name: pyright
        entry: uv run pyright
        language: system
        types: [python]
        pass_filenames: false
```

## 5. .gitignore

至少包含：

```
.venv/
__pycache__/
*.py[cod]
.env
.env.*
!.env.example
.ruff_cache/
.pytest_cache/
.coverage
htmlcov/
dist/
```

提交 `.env.example` 列出所需变量名（不含真实值）。`uv.lock` 与 `.python-version` 必须提交，不要加入忽略。

## 6. CI

CI 只做三件事，命令与本地质量闸门一致：

```bash
uv sync --locked
uv run ruff check . && uv run ruff format --check .
uv run pyright
uv run pytest
```

`--locked` 在 `uv.lock` 与 `pyproject.toml` 不一致时直接失败，防止"本地能过、CI 装了另一套版本"。

## 7. 单文件脚本

一次性脚本不需要建项目。用 PEP 723 内联元数据声明依赖，`uv run` 会自动建临时环境：

```python
# /// script
# requires-python = ">=3.12"
# dependencies = ["httpx", "rich"]
# ///
import httpx
from rich import print as rprint

rprint(httpx.get("https://example.com", timeout=10).status_code)
```

```bash
uv run script.py
uv add --script script.py orjson   # 向脚本头部追加依赖
```
