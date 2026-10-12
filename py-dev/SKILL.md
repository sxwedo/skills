---
name: py-dev
description: Python 项目的默认栈与写法规范。用于新建 Python 项目、编写或审查 Python 代码、挑选或新增依赖、配置 pyproject.toml 与 ruff/pyright/pytest/uv、搭建 FastAPI 服务；用户点名 requests、pandas、flask、click、poetry、pip、mypy、black、psycopg2 等旧库时同样适用。用户没有明确要求选型时也使用本 skill。
---

# Python 开发：默认栈与写法规范

本 skill 把"用哪个库、怎么配置、怎么写"固定下来，让每个 Python 项目走同一条路径。一条总原则：**工具最少，配置唯一，版本交给锁文件**。

## 工作流程

1. **判断场景**。读 `pyproject.toml`、`uv.lock`、`requirements.txt`。已有项目按铁律 1 沿用现状；空目录或新项目走第 2 步。
2. **新建项目**。读 `references/setup.md`：uv 初始化命令、pyproject 工具配置、`src/` 目录结构、pre-commit、.gitignore、CI、单文件脚本。只装工具链，不预装运行时依赖。
3. **选依赖**。先查下方默认栈，再读 `references/stack.md` 对应条目：联动安装、使用边界、旧库迁移对照表、未覆盖需求的候选。用 `uv add` 安装。用户点名"不要用"一列的库，直接换成"选择"一列，用一句话说明原因，不征求同意。
4. **写代码**。遵循下方代码要点，按任务读对应参考：

   | 写什么 | 读 |
   |---|---|
   | 语言层写法：类型、异常、上下文管理器、数据模型、Protocol、装饰器、并发、性能、反模式表 | `references/idioms.md` |
   | 配置、日志、HTTP 与重试、pydantic、polars、命令行、安全 | `references/practices.md` |
   | FastAPI 路由与 lifespan、SQLAlchemy 2.x、Alembic 迁移 | `references/web.md` |
   | pytest、respx、FastAPI 测试客户端、hypothesis | `references/testing.md` |

5. **交付**。跑质量闸门，三项全绿。再对照 `references/standards.md` 的"完成的定义"逐条确认，每条给出结论，不整体宣称"已完成"。审查他人代码时改用同文件末尾的审查清单，逐条给出具体位置与修改建议。

## 铁律

1. **已有项目沿用现有选择。** 为统一风格做大面积迁移，会制造与任务无关的巨大 diff 和回归风险。发现差距时提建议，等用户决定，不擅自迁移。
2. **只装用到的库。** 每个依赖都是维护成本和攻击面。默认栈是"需要时怎么选"，不是"必装清单"。
3. **不手写版本号。** `uv add <包>` 解析当天最新且互不冲突的版本。真实版本以 `uv.lock` 为准：必须提交，不手改，升级走 `uv lock --upgrade`。
4. **配置只放 `pyproject.toml`。** 不创建 setup.py、setup.cfg、requirements.txt、.flake8、mypy.ini、pytest.ini。例外是工具强制自带的文件：`alembic.ini`、`.pre-commit-config.yaml`、`.python-version`。
5. **所有命令走 `uv run`。** 不直接调用 `python`、`pytest`、`pip`，这样才能保证用的是项目环境里的解释器和版本。
6. **不熟的 API 先查证再写。** 库接口变化快，0.x 更甚。查官方文档（有 context7 一类文档工具时优先用），或读 `.venv/` 里已安装包的源码，不凭记忆。

## 默认栈：需求 → 选择

| 需求 | 选择 | 不要用 |
|---|---|---|
| HTTP 客户端 | httpx（测试加 respx） | requests、aiohttp、urllib3 |
| Web 服务 | fastapi + uvicorn | flask、bottle |
| 数据校验与模型 | pydantic v2 | marshmallow、手写校验 |
| 配置与环境变量 | pydantic-settings | python-dotenv、environs |
| 数据库 ORM | sqlalchemy 2.x（迁移用 alembic） | peewee、手拼 SQL |
| PostgreSQL 驱动 | psycopg 3（装 `psycopg[binary]`） | psycopg2 |
| 数据处理 | polars | pandas |
| JSON 序列化 | orjson | 标准库 json、ujson |
| 命令行 | typer | click、argparse、fire |
| 日志 | structlog | 直接使用 logging、loguru |
| 失败重试 | tenacity | backoff、retrying、手写重试循环 |
| 终端输出 | rich | colorama、termcolor |
| 测试 | pytest（按需加 pytest-cov、pytest-asyncio、hypothesis） | unittest、nose |
| 检查与格式化 | ruff | black、flake8、isort、pylint |
| 类型检查 | pyright | mypy |
| 包管理与环境 | uv | pip、poetry、pipenv、conda |
| 构建后端 | hatchling | setuptools（新项目） |
| 提交钩子 | pre-commit | 手写 git hook |

表外需求（缓存、任务队列、认证、浏览器自动化、科学计算等）：读 `references/stack.md` 的"未覆盖需求"，向用户说明候选与理由后再选。

## 质量闸门

```bash
uv run ruff check --fix . && uv run ruff format .
uv run pyright
uv run pytest
```

三项全部通过才可以提交。CI 跑同样三条命令的 `--locked` 变体，见 `references/setup.md` 第 6 节。

## 代码要点

1. **类型**：所有函数签名写注解，pyright 零报错。用现代语法：`list[str]`、`X | None`、`type` 别名、泛型 `def f[T](...)`；不用 `typing.List`、`Dict`、`Optional`、`Union`。
2. **边界用 pydantic，内部用轻量类型。** 请求、响应、配置、外部 API、文件内容这些进出系统的数据，用 pydantic 校验；内部传递用 `@dataclass(slots=True, frozen=True)` 或普通类型。
3. **I/O 默认 async。** 用 `httpx.AsyncClient`、SQLAlchemy 异步会话。CPU 重活交给进程池，必须调用的阻塞函数放进 `asyncio.to_thread`，不阻塞事件循环。
4. **并发用结构化写法。** 用 `asyncio.TaskGroup` 管理一组任务，用 `asyncio.timeout()` 控制超时，用 `Semaphore` 限制并发数。
5. **外部调用必须有超时。** 可重试操作用 tenacity：只重试幂等操作，设次数上限，指数退避加抖动。
6. **异常**：捕获具体类型，用 `raise ... from e` 保留因果链，定义 `AppError` 层级，只在边界（HTTP 处理器、CLI 入口、任务循环）统一兜底。
7. **日志**：用 structlog，事件名是简短的 snake_case 短语，数据放键值对，不拼接字符串；不记录密钥和个人信息。
8. **配置**：用 pydantic-settings，密钥用 `SecretStr`，不硬编码，不提交 `.env`。
9. **资源与流**：资源用 `with` / `async with` 管理，大数据用生成器或 polars 的惰性 API；读写文本显式写 `encoding="utf-8"`；时间一律带时区，用 `datetime.now(UTC)`。
10. **数据库**：表结构变更必须写 alembic 迁移，不手改；SQL 一律参数化。
11. **测试**：新行为先写测试；外部 HTTP 用 respx 模拟，不访问真实网络；数据库用真实 PostgreSQL，不 mock；解析、转换、序列化这类逻辑补 hypothesis 属性测试。
