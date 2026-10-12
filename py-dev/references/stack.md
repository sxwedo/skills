# 选型细则与迁移

## 1. 选型细则

SKILL.md 的默认栈给出"选什么"。这里补充每一项的联动安装与使用边界。安装统一用 `uv add <包>`（开发依赖加 `--dev`）。

| 需求 | 安装 | 联动与边界 |
|---|---|---|
| HTTP 客户端 | `httpx` | 同步用 `httpx.Client`，异步用 `httpx.AsyncClient`，两者接口一致。测试时 `uv add --dev respx` |
| Web 服务 | `fastapi uvicorn` | pydantic 随 fastapi 自动带入，不必单独声明。表单上传需要 `python-multipart` |
| 数据校验 | `pydantic` | 只用 v2 的 API（`model_validate`、`model_dump`、`field_validator`），不要写 v1 风格的 `parse_obj`、`validator` |
| 配置 | `pydantic-settings` | 与 pydantic 同系，配置类继承 `BaseSettings` |
| ORM 与迁移 | `sqlalchemy alembic` | 用 2.x 风格：`DeclarativeBase`、`Mapped`、`select()`；不用 1.x 的 `session.query()` |
| PostgreSQL | `"psycopg[binary]"` | 不带 `[binary]` 在没有 libpq 的机器上装不上。SQLAlchemy 连接串用 `postgresql+psycopg://` |
| 数据处理 | `polars` | 优先惰性 API（`scan_*`、`.lazy()`）。只有某个第三方库强制要求 pandas 时，才在边界用 `.to_pandas()` 转换 |
| JSON | `orjson` | `orjson.dumps` 返回 `bytes` 而不是 `str`，需要字符串时 `.decode()` |
| 命令行 | `typer` | 用类型注解声明参数，不手写 argparse |
| 日志 | `structlog` | structlog 可以构建在标准库 logging 之上，配置阶段用 logging 无妨，业务代码只调用 structlog |
| 重试 | `tenacity` | 只用于幂等操作，必须限定次数与异常类型 |
| 终端输出 | `rich` | 只用在命令行入口的人类可读输出，不用来写生产日志 |
| 测试 | `--dev pytest pytest-cov` | 异步测试加 `pytest-asyncio`，属性测试加 `hypothesis`，HTTP 模拟加 `respx` |
| 检查与类型 | `--dev ruff pyright` | ruff 同时取代 black、isort、flake8，不要再装这三个 |
| 提交钩子 | `--dev pre-commit` | 配置见 setup.md |

## 2. 旧库迁移速查

用户点名旧库，或已有代码需要迁移时使用。**已有项目未经用户同意不要迁移**。

### requests → httpx

| requests | httpx | 注意 |
|---|---|---|
| `requests.get(url)` | `httpx.get(url, timeout=10)` | httpx 默认 5 秒超时，requests 默认永不超时，迁移后行为会变，显式写出超时 |
| `requests.Session()` | `httpx.Client()` | 复用连接，用 `with` 管理 |
| `allow_redirects=True` | `follow_redirects=True` | httpx 默认**不**跟随重定向，参数名也不同 |
| `proxies={...}` | `proxy="http://..."` | 参数名与形式不同 |
| `r.json()`、`r.text` | 相同 | |
| `r.raise_for_status()` | 相同 | 异常类型是 `httpx.HTTPStatusError` |
| `requests.exceptions.RequestException` | `httpx.HTTPError` | 网络层错误是 `httpx.TransportError` |
| `data=` 传 JSON | `json=` | 表单用 `data=`，JSON 用 `json=`，两者别混 |

### aiohttp → httpx

`aiohttp.ClientSession` 换成 `httpx.AsyncClient`，用 `async with` 管理，整个应用复用一个实例，不要每次请求新建。

### pandas → polars

| pandas | polars |
|---|---|
| `pd.read_csv(p)` | `pl.read_csv(p)`，大文件用 `pl.scan_csv(p)` |
| `df[df["a"] > 1]` | `df.filter(pl.col("a") > 1)` |
| `df["b"] = df["a"] * 2` | `df.with_columns((pl.col("a") * 2).alias("b"))` |
| `df.groupby("k").agg(...)` | `df.group_by("k").agg(...)` |
| `df.apply(f, axis=1)` | 改写为表达式；实在不行用 `map_elements`，它很慢 |
| 索引（`df.index`） | polars 没有行索引，需要的话显式加列 |

### 其他

| 旧写法 | 新写法 |
|---|---|
| `black .` + `isort .` + `flake8` | `ruff format .` + `ruff check --fix .` |
| `mypy src/` | `pyright` |
| `argparse` / `click` | `typer` |
| `json.dumps(x)` | `orjson.dumps(x).decode()`（缩进用 `option=orjson.OPT_INDENT_2`） |
| `logging.getLogger(__name__).info("x %s", v)` | `structlog.get_logger().info("x_happened", v=v)` |
| `psycopg2.connect(...)` | `psycopg.connect(...)`，占位符仍是 `%s`；要字典行用 `row_factory=psycopg.rows.dict_row` |
| `load_dotenv()` + `os.environ[...]` | `BaseSettings` 子类，在启动时一次性校验 |
| `pip install` / `poetry add` | `uv add` |
| `requirements.txt` | `pyproject.toml` + `uv.lock` |
| `session.query(User).filter(...)` | `session.execute(select(User).where(...))` |

## 3. 何时偏离默认栈

默认栈适用于新项目。出现下列情况时，沿用现有技术并向用户说明：

| 情况 | 做法 |
|---|---|
| 项目已经基于 Django | 沿用 Django ORM 与其生态，不引入 FastAPI、SQLAlchemy |
| 已有大量 pandas 代码 | 新代码可以用 polars，旧代码不迁移，两者在边界用 `to_pandas()` / `pl.from_pandas()` 转换 |
| 团队已用 poetry 或 pip-tools | 沿用，建议迁移到 uv 但不强推 |
| 某个 SDK 内部依赖 requests | 允许它作为传递依赖存在，自己的代码仍用 httpx |
| 下游库要求 mypy 插件或特定类型检查器 | 以该库要求为准 |
| 只需发一两次 HTTP 请求的脚本 | 用 PEP 723 脚本，不必建完整项目 |

## 4. 未覆盖的需求

下表是我对常见需求的候选，**没有经过与默认栈的联合解析验证**。选用前向用户说明候选与理由，装完后用 `uv lock` 确认没有冲突。

| 需求 | 候选 | 理由 |
|---|---|---|
| 缓存、队列后端 | `redis`（用 `redis.asyncio`） | 官方客户端，同时提供同步与异步接口 |
| 后台任务 | `arq`（asyncio 场景）或 `celery`（复杂编排） | 简单场景用 arq 依赖更少，需要复杂路由与调度时才上 celery |
| 认证 | `pyjwt`、`authlib` | 前者只处理 JWT，后者覆盖 OAuth 流程 |
| 浏览器自动化与动态页面抓取 | `playwright` | 自带浏览器管理，API 以 async 为主 |
| 数值计算 | `numpy` | 事实标准，polars 可与其互转 |
| 可视化 | `plotly` | 交互式图表，可导出 HTML |
| 机器学习 | 视任务而定 | 先确认框架（PyTorch、scikit-learn 等），不要自行预设 |

遇到表外的库，按下面的标准评估后再推荐：近期仍有发布与维护、有完整类型注解、依赖少、与 asyncio 兼容、许可证允许使用。拿不准时先搜索确认，再告诉用户。
