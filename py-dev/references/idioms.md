# Python 语言层写法

要求 Python 3.12 及以上。

## 1. 核心原则

**EAFP（先做，出错再处理）用于"检查"和"使用"之间可能发生变化的场合**，例如文件、网络、并发。标准库已有直接方法时，直接用方法：

```python
# 好：已有方法，不必手写 try/except
value = mapping.get(key, default)

# 好：文件可能在检查后消失，用 EAFP
try:
    text = path.read_text(encoding="utf-8")
except FileNotFoundError:
    text = ""

# 坏：先检查再用，存在竞态
if path.exists():
    text = path.read_text(encoding="utf-8")
```

## 2. 类型注解

用 PEP 585/604/695 的内置语法，不再从 `typing` 导入 `List`、`Dict`、`Optional`、`Union`。

```python
from collections.abc import Callable, Iterable, Iterator, Mapping, Sequence
from typing import Any, Literal, Protocol, Self, TypedDict, override

# 内置泛型与 | 联合
def count_words(lines: Iterable[str]) -> dict[str, int]: ...
def find_user(user_id: int) -> User | None: ...

# PEP 695：type 语句定义别名，支持递归
type JSON = dict[str, JSON] | list[JSON] | str | int | float | bool | None

# PEP 695：泛型函数与泛型类，不再手写 TypeVar
def first[T](items: Sequence[T]) -> T | None:
    return items[0] if items else None

class Box[T]:
    def __init__(self, value: T) -> None:
        self.value = value

# 参数类型尽量宽（Iterable、Mapping、Sequence），返回类型尽量具体
def total(values: Iterable[float]) -> float:
    return sum(values)

# Literal 约束取值，TypedDict 描述固定键的字典
Mode = Literal["read", "write"]

class Options(TypedDict):
    retries: int
    verbose: bool

# Self 用于返回自身类型，override 标明重写（拼错方法名时 pyright 会报错）
class Base:
    def clone(self) -> Self: ...

class Child(Base):
    @override
    def clone(self) -> Self: ...
```

规则：

- 公开函数的参数与返回值必须注解；局部变量让类型检查器推断，推不出来再标注。
- 避免 `Any`。确实需要时优先用 `object` 或具体的 `Protocol`，真要用 `Any` 在边界处立刻收窄。
- 不用 `# type: ignore` 糊过去；必须用时写明具体规则并附原因，例如 `# pyright: ignore[reportGeneralTypeIssues]  # 第三方库缺少存根`。

## 3. 异常

**捕获具体类型。** 裸 `except:` 和 `except Exception: pass` 会吞掉真正的错误。

```python
# 好：具体类型，转换为领域异常并保留因果链
def load_config(path: Path) -> Config:
    try:
        raw = path.read_bytes()
        return Config.model_validate_json(raw)
    except FileNotFoundError as e:
        raise ConfigError(f"config file not found: {path}") from e
    except ValidationError as e:
        raise ConfigError(f"invalid config: {path}") from e

# 坏：静默失败，调用方拿到 None 却不知道为什么
def load_config(path):
    try:
        ...
    except:
        return None
```

**`raise ... from e`** 保留原始异常链，排查问题时能看到完整因果；确实要隐藏底层细节才用 `from None`。

**自定义异常层级。** 业务异常继承同一个根，边界处只需捕获根类：

```python
class AppError(Exception):
    """Base class for all application errors."""

class ValidationFailed(AppError): ...
class NotFound(AppError): ...
class UpstreamUnavailable(AppError): ...
```

**在边界兜底，在内部传播。** 内部函数遇到自己处理不了的错误就向上抛；HTTP 处理器、CLI 入口、后台任务循环这些边界才负责转换为状态码、退出码或重试决策，并记录日志。每个异常只记录一次，不要每层都 `log` 再 `raise`。

**TaskGroup 抛出 `ExceptionGroup`**，用 `except*` 按类型处理：

```python
try:
    async with asyncio.TaskGroup() as tg:
        ...
except* UpstreamUnavailable as group:
    log.warning("upstream_failed", count=len(group.exceptions))
```

## 4. 上下文管理器

用 `with` 管理一切需要"成对"操作的资源：文件、连接、锁、事务、临时状态。

```python
from collections.abc import AsyncIterator, Iterator
from contextlib import asynccontextmanager, contextmanager

@contextmanager
def timed(name: str) -> Iterator[None]:
    start = time.perf_counter()
    try:
        yield
    finally:  # 无论是否出错都记录
        log.info("timed", name=name, seconds=round(time.perf_counter() - start, 4))

@asynccontextmanager
async def transaction(session: AsyncSession) -> AsyncIterator[AsyncSession]:
    async with session.begin():
        yield session
```

一次管理多个资源用 `with a as x, b as y:` 或 `contextlib.ExitStack`；需要"忽略特定异常"用 `contextlib.suppress(FileNotFoundError)`，不要写空的 `try/except`。

只有当状态必须跨多个方法保持时才写成带 `__enter__`/`__exit__` 的类，其余一律用 `@contextmanager`，更短也更不容易出错。

## 5. 推导式与生成器

简单的转换和过滤用推导式；超过一个条件或一层嵌套就改写成普通循环或生成器函数。

```python
# 好
names = [u.name for u in users if u.is_active]

# 坏：难读
result = [x * 2 for x in items if x > 0 if x % 2 == 0]

# 好：拆成函数，名字说明意图
def doubled_positive_evens(items: Iterable[int]) -> Iterator[int]:
    for x in items:
        if x > 0 and x % 2 == 0:
            yield x * 2
```

数据量大或只遍历一次时用生成器，不要先造完整列表：

```python
total = sum(x * x for x in range(1_000_000))   # 好：惰性
total = sum([x * x for x in range(1_000_000)]) # 坏：先构造百万元素的列表

def read_lines(path: Path) -> Iterator[str]:
    with path.open(encoding="utf-8") as f:
        for line in f:
            yield line.rstrip("\n")
```

## 6. 数据模型

按数据所处位置选工具：

| 位置 | 工具 | 原因 |
|---|---|---|
| 进出系统的数据（请求、响应、配置、外部 API、文件） | pydantic `BaseModel` | 需要校验与解析 |
| 内部传递的值对象 | `@dataclass(slots=True, frozen=True, kw_only=True)` | 轻量、不可变、省内存 |
| 固定键的字典（与旧代码交互） | `TypedDict` | 只做类型描述，没有运行时校验 |
| 少量字段的不可变元组 | `NamedTuple` | 需要解包或作为字典键时 |

```python
from dataclasses import dataclass
from datetime import UTC, datetime

from pydantic import BaseModel, ConfigDict, EmailStr, Field

class UserIn(BaseModel):                      # 边界：校验
    model_config = ConfigDict(extra="forbid")
    email: EmailStr
    age: int = Field(ge=0, le=150)

@dataclass(slots=True, frozen=True, kw_only=True)
class Event:                                  # 内部：值对象
    name: str
    at: datetime
```

不要在 dataclass 的 `__post_init__` 里手写校验逻辑，那是 pydantic 的职责。`EmailStr` 需要额外安装 `email-validator`（`uv add email-validator`），不需要时用 `str` 加 `pattern`。

可变默认值用 `field(default_factory=list)`，不要写 `= []`。

## 7. Protocol 与鸭子类型

依赖"能做什么"，而不是"是什么"。用 `Protocol` 声明所需的最小接口，调用方不必继承任何基类，测试时也容易替换。

```python
from typing import Protocol

class Clock(Protocol):
    def now(self) -> datetime: ...

class SystemClock:
    def now(self) -> datetime:
        return datetime.now(UTC)

def is_expired(token: Token, clock: Clock) -> bool:
    return token.expires_at <= clock.now()
```

## 8. 装饰器

用 `functools.wraps` 保留原函数元信息，用 PEP 695 的 `ParamSpec` 语法保留签名：

```python
import functools
from collections.abc import Callable

def timed_call[**P, R](func: Callable[P, R]) -> Callable[P, R]:
    @functools.wraps(func)
    def wrapper(*args: P.args, **kwargs: P.kwargs) -> R:
        start = time.perf_counter()
        try:
            return func(*args, **kwargs)
        finally:
            log.info("call_finished", func=func.__qualname__,
                     seconds=time.perf_counter() - start)
    return wrapper
```

异步函数需要单独写 `async def wrapper` 并 `await`，不能用同一个装饰器同时包同步与异步函数。重试、缓存这类通用需求优先用现成库（tenacity、`functools.cache`），不要自己造。

## 9. 并发

先判断瓶颈类型：

| 瓶颈 | 方案 |
|---|---|
| 等网络、等数据库（I/O） | `asyncio` + `httpx.AsyncClient` / SQLAlchemy 异步 |
| 必须调用的阻塞函数（同步库、文件系统） | `await asyncio.to_thread(func, ...)` |
| CPU 密集计算 | `concurrent.futures.ProcessPoolExecutor`，或交给 polars 并行执行 |

```python
import asyncio
import httpx

async def fetch_all(client: httpx.AsyncClient, urls: list[str]) -> dict[str, str]:
    results: dict[str, str] = {}
    limit = asyncio.Semaphore(10)              # 限制并发数，避免打爆对端

    async def fetch(url: str) -> None:
        async with limit:
            resp = await client.get(url)
            resp.raise_for_status()
            results[url] = resp.text

    async with asyncio.timeout(30):             # 整体超时
        async with asyncio.TaskGroup() as tg:   # 任一任务失败，其余自动取消
            for url in urls:
                tg.create_task(fetch(url))
    return results
```

规则：

- 用 `TaskGroup` 代替裸 `asyncio.gather` 和 `create_task`：它保证任务不会泄漏，失败会取消同组任务。
- 事件循环里不要调用 `time.sleep`、同步的 `requests`、同步数据库驱动，这些会冻结所有协程。ruff 的 `ASYNC` 规则会拦住一部分。
- 入口只调用一次 `asyncio.run(main())`；不要用 `get_event_loop()`。
- 不要用线程池包同步 HTTP 去模拟并发，直接用异步客户端。

## 10. 内存与性能

先正确、后测量、再优化，不凭感觉改。

```python
# 字符串拼接：循环里 += 是 O(n²)
result = "".join(str(item) for item in items)       # 好

# 查找：成员判断用 set，不用 list
allowed = {"a", "b", "c"}
if value in allowed: ...

# 缓存纯函数结果
from functools import cache

@cache
def parse_rule(text: str) -> Rule: ...
```

- 大文件与大结果集用流式处理（生成器、polars `scan_*`、数据库游标分批），不要一次性读进内存。
- 实例很多的小类用 `@dataclass(slots=True)`，不要手写 `__slots__`。
- 对象需要频繁判断归属时用 `set`/`dict`，不要在列表里线性查找。
- 性能问题先用 `cProfile` 或 `py-spy` 定位热点，再动手；没有测量结果的优化不要提交。

## 11. 反模式

| 反模式 | 正确做法 |
|---|---|
| `def f(x, items=[])` | `def f(x, items=None)` 内部创建；或用 `default_factory` |
| `type(x) == list` | `isinstance(x, list)` |
| `if x == None` | `if x is None` |
| `from module import *` | 显式导入需要的名字 |
| `except:` / `except Exception: pass` | 捕获具体异常，处理或向上抛 |
| `print(...)` 调试 | structlog；排查时用调试器 |
| `assert` 做输入校验 | 抛出 `ValidationFailed`；`python -O` 会移除 assert |
| `datetime.utcnow()` / 无时区的 `datetime.now()` | `datetime.now(UTC)` |
| `open(p)` 不指定编码 | `p.read_text(encoding="utf-8")` |
| `"%s" % x`、`"{}".format(x)` | f-string（日志调用除外，日志用键值对） |
| `from typing import List, Dict, Optional` | `list`、`dict`、`X \| None` |
| `os.path.join`、`os.listdir` | `pathlib.Path` |
| 模块级可变全局状态 | 作为参数传入，或封装进对象 |
| 在 `async def` 里调用阻塞函数 | `asyncio.to_thread` 或换异步库 |
| `asyncio.gather` 不处理失败 | `TaskGroup` |
| 循环里反复拼接字符串 | `"".join(...)` |
| 一个函数既查询又修改又打日志 | 拆成单一职责的小函数 |
| 注释复述代码 | 删掉；注释写"为什么" |
