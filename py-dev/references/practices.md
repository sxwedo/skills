# 各库用法与工程实践

示例假定包名为 `app`，Python 3.12+，已按 SKILL.md 的默认栈安装所需库。每节只写该库在本栈中的**推荐用法与常见坑**，不重复官方文档。FastAPI 与 SQLAlchemy 见 `web.md`，pytest 见 `testing.md`。

## 1. 配置：pydantic-settings

所有配置在启动时一次性读取并校验，缺失或格式错误立即失败，不要等到运行中途才发现。

```python
# app/settings.py
from functools import lru_cache

from pydantic import PostgresDsn, SecretStr
from pydantic_settings import BaseSettings, SettingsConfigDict

class Settings(BaseSettings):
    model_config = SettingsConfigDict(env_file=".env", env_prefix="APP_", extra="ignore")

    database_url: PostgresDsn
    api_key: SecretStr
    http_timeout: float = 10.0
    debug: bool = False

@lru_cache
def get_settings() -> Settings:
    return Settings()  # pyright: ignore[reportCallIssue]  # 必填字段来自环境变量
```

- 密钥类型用 `SecretStr`，使用时 `.get_secret_value()`，这样打印和日志里不会泄漏。
- 不在模块导入时实例化 `Settings()`，用 `get_settings()` 延迟创建，测试时可以覆盖。
- `PostgresDsn` 转字符串用 `str(settings.database_url)`。
- 提交 `.env.example`，不提交 `.env`。

## 2. 日志：structlog

```python
# app/logging_config.py
import logging

import structlog

def configure_logging(*, debug: bool = False) -> None:
    renderer = (
        structlog.dev.ConsoleRenderer() if debug else structlog.processors.JSONRenderer()
    )
    structlog.configure(
        processors=[
            structlog.contextvars.merge_contextvars,   # 带上请求级上下文
            structlog.processors.add_log_level,
            structlog.processors.TimeStamper(fmt="iso", utc=True),
            structlog.processors.StackInfoRenderer(),
            structlog.processors.format_exc_info,
            renderer,
        ],
        wrapper_class=structlog.make_filtering_bound_logger(
            logging.DEBUG if debug else logging.INFO
        ),
        cache_logger_on_first_use=True,
    )
```

使用：

```python
import structlog

log = structlog.get_logger()

log.info("order_created", order_id=order.id, amount=order.amount)
log.warning("upstream_slow", url=url, seconds=elapsed)
log.exception("sync_failed", source=source)        # 在 except 块里，自动附带堆栈
```

约定：

- **事件名**用简短的 snake_case（`order_created`），不写整句话；变量放键值对，不拼进字符串，这样日志可以按字段检索。
- 请求级上下文（`request_id`、`user_id`）在中间件里用 `structlog.contextvars.bind_contextvars(...)` 绑定一次，之后所有日志自动携带；请求结束时 `clear_contextvars()`。
- 生产输出 JSON，本地调试输出彩色控制台。
- 不记录密码、令牌、完整身份证件号、银行卡号；对象先取需要的字段再记录。
- 日志级别：`debug` 排查细节，`info` 正常业务事件，`warning` 可恢复的异常，`error`/`exception` 需要人处理的失败。

## 3. HTTP 客户端与重试：httpx + tenacity

**一个应用复用一个客户端**（连接池才有效），在 lifespan 或入口创建，退出时关闭；**每次都显式设置超时**。

```python
import httpx
from tenacity import (
    retry,
    retry_if_exception,
    stop_after_attempt,
    wait_exponential_jitter,
)

TIMEOUT = httpx.Timeout(10.0, connect=5.0)

def _is_retryable(exc: BaseException) -> bool:
    if isinstance(exc, httpx.TransportError):          # 连接、超时等网络层错误
        return True
    if isinstance(exc, httpx.HTTPStatusError):
        return exc.response.status_code in {429, 502, 503, 504}
    return False

@retry(
    retry=retry_if_exception(_is_retryable),
    stop=stop_after_attempt(3),
    wait=wait_exponential_jitter(initial=0.5, max=10),
    reraise=True,                                       # 最终仍失败时抛出原始异常
)
async def get_user(client: httpx.AsyncClient, user_id: int) -> UserOut:
    resp = await client.get(f"/users/{user_id}")
    resp.raise_for_status()
    return UserOut.model_validate(resp.json())

# 创建：base_url + timeout 一次性设好
client = httpx.AsyncClient(base_url="https://api.example.com", timeout=TIMEOUT)
```

规则：

- **只重试幂等操作**（GET、带幂等键的请求）。对 POST 创建类请求盲目重试会产生重复数据。
- 重试要有次数上限、指数退避加抖动、限定异常类型；不要无限循环。
- 4xx（除 429）是调用方的错，重试没有意义。
- 外部响应一律用 pydantic 校验，不直接信任 `resp.json()` 的结构。
- 同步脚本用 `httpx.Client`，写法与异步版一致。

## 4. 数据校验：pydantic

```python
from typing import Annotated

from pydantic import BaseModel, ConfigDict, Field, field_validator

class UserIn(BaseModel):
    model_config = ConfigDict(extra="forbid", str_strip_whitespace=True)

    name: Annotated[str, Field(min_length=1, max_length=100)]
    email: str
    tags: list[str] = Field(default_factory=list)

    @field_validator("email")
    @classmethod
    def lowercase_email(cls, v: str) -> str:
        return v.lower()

class UserOut(BaseModel):
    model_config = ConfigDict(from_attributes=True)   # 允许从 ORM 对象构造
    id: int
    name: str
```

- 解析用 `Model.model_validate(data)` 或 `model_validate_json(raw)`，导出用 `model_dump()` / `model_dump_json()`。
- 请求模型设 `extra="forbid"`，拼错的字段直接报错，不静默忽略。
- 输入模型（`UserIn`）与输出模型（`UserOut`）分开，避免把内部字段（如密码哈希）泄漏出去。
- 不要把 pydantic 模型当 ORM 模型用，两者职责不同。

## 5. 数据处理：polars

用表达式与惰性 API，让 polars 优化整条查询；不逐行遍历。

```python
import polars as pl

result = (
    pl.scan_parquet("events.parquet")              # 惰性读取，不立即载入
    .filter(pl.col("status") == "ok")
    .with_columns((pl.col("amount") * 100).alias("cents"))
    .group_by("user_id")
    .agg(pl.col("cents").sum().alias("total"))
    .sort("total", descending=True)
    .collect()                                      # 最后一步才执行
)
```

- 避免 `for row in df.iter_rows()` 和 `map_elements`（逐行调用 Python 函数，慢）；能用 `pl.col(...)` 表达式就用表达式。
- 大文件用 `scan_csv` / `scan_parquet`，存储格式优先 Parquet。
- 外部库只接受 pandas 时，在边界处 `df.to_pandas()`，不要让 pandas 渗透进内部。
- 数据文件进出系统时，先校验列与类型（`df.schema`）再处理。

## 6. 命令行：typer + rich

```python
# app/cli.py
from pathlib import Path
from typing import Annotated

import typer
from rich.console import Console
from rich.table import Table

app = typer.Typer(no_args_is_help=True, help="示例工具")
console = Console()
err_console = Console(stderr=True)

@app.command()
def sync(
    source: Annotated[Path, typer.Argument(exists=True, readable=True, help="输入文件")],
    dry_run: Annotated[bool, typer.Option("--dry-run", help="只预览，不写入")] = False,
) -> None:
    """同步数据。"""
    try:
        count = run_sync(source, dry_run=dry_run)
    except AppError as e:
        err_console.print(f"[red]失败：[/red]{e}")
        raise typer.Exit(code=1) from e
    console.print(f"[green]完成[/green]，处理 {count} 条")

if __name__ == "__main__":
    app()
```

在 `pyproject.toml` 声明入口：`[project.scripts]` 下写 `app = "app.cli:app"`，之后 `uv run app sync data.csv`。

规则：

- 参数、类型、默认值全靠类型注解声明，不手写解析。
- 正常结果输出到 stdout，错误与诊断输出到 stderr；失败返回非零退出码，便于脚本与 CI 判断。
- rich 只用于人类可读输出（表格、进度条、颜色）。需要被程序解析的输出提供 `--json` 选项，用 orjson 输出。
- 命令函数只做参数解析和展示，实际逻辑放进可测试的普通函数。

## 7. 安全

| 风险 | 做法 |
|---|---|
| 密钥泄漏 | 密钥只经环境变量进入，用 `SecretStr`；`.env` 不入库；日志不记录凭证 |
| SQL 注入 | 一律参数化；ruff 的 `S608` 会拦截字符串拼接 SQL |
| 命令注入 | `subprocess.run([...], check=True)` 用列表参数，不使用 `shell=True` 与字符串拼接命令 |
| 不可信反序列化 | 不对不可信数据使用 `pickle`、`eval`、`exec`；结构化数据用 JSON 加 pydantic 校验 |
| 路径穿越 | 用户给的文件名先 `Path(name).name` 取纯文件名，再拼到受控目录；校验解析后的路径仍在该目录内 |
| 未校验的输入 | 所有边界数据过 pydantic，请求模型 `extra="forbid"` |
| 无超时的外部调用 | 所有 httpx 与数据库调用设置超时 |
| 依赖漏洞 | 定期 `uv lock --upgrade` 并运行测试；锁文件入库，CI 用 `--locked` |
| 错误信息泄漏内部细节 | 对外返回通用错误消息与错误码，细节只写日志 |

ruff 的 `S` 规则集覆盖了上述大部分静态可检查项，保持启用，不要全局关闭；确有误报时在该行用 `# noqa: S...` 并写明原因。
