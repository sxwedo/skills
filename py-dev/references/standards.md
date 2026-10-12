# 规范与标准

## 1. 命名与风格

**格式化与导入排序交给 ruff**，不人工争论，也不手动调整。风格以 PEP 8 为基础，行宽取 100（PEP 8 的 79 是历史限制，现代屏幕与 ruff format 下 100 更实用）。

| 对象 | 约定 | 例 |
|---|---|---|
| 模块、包 | 短小的小写，包名避免下划线 | `utils`、`billing` |
| 函数、方法、变量 | `snake_case` | `get_user`、`retry_count` |
| 类、类型别名、协议 | `PascalCase` | `UserService`、`Clock` |
| 常量 | `UPPER_SNAKE_CASE` | `DEFAULT_TIMEOUT` |
| 内部使用 | 前缀单下划线 | `_parse_header` |
| 类型变量（PEP 695 泛型） | 单个大写字母或简短名 | `T`、`K`、`V` |
| 布尔值 | 用 `is_`、`has_`、`can_` 开头 | `is_active` |

命名原则：

- 名字表达含义而不是类型：`users` 而不是 `user_list`，`timeout_seconds` 而不是 `t`。
- 带单位的量把单位写进名字：`timeout_seconds`、`size_bytes`。
- 不遮蔽内置名（`id`、`type`、`list`、`input`），ruff 的 `A` 规则会报。
- 函数名用动词，类名用名词；返回布尔值的函数读起来像一个问句。
- 缩写只用通用的（`url`、`id`、`db`、`http`），其余写全。

函数与类的大小：

- 一个函数做一件事，读完不需要滚动屏幕；参数超过五个时考虑用数据类或 pydantic 模型打包。
- 关键字参数多的函数用 `*` 强制关键字传参：`def send(to: str, *, retries: int = 3)`。
- 嵌套超过三层就提前返回或拆函数。

## 2. 导入与模块

- 导入顺序为标准库、第三方、本项目，三组之间空一行；由 ruff 的 `I` 规则自动维护。
- 项目内使用**绝对导入**：`from app.services import users`，不用 `from ..services import users`。
- 一律在文件顶部导入；仅为避免循环导入或延迟加载重依赖时才在函数内导入，并写一行注释说明。
- 不用 `import *`。
- 包的公开接口在 `__init__.py` 里用 `__all__` 显式声明，其余视为内部实现。
- 避免循环依赖：依赖方向保持单向（`api → services → db`），出现循环说明职责划分有问题，抽出共同的下层模块。
- 不在模块导入时执行有副作用的代码（读环境变量、连数据库、发请求），把它们放进函数或 lifespan。

## 3. 注释与 docstring

- 注释解释**为什么**这样做（约束、取舍、陷阱），不复述代码做了什么。
- 公开的模块、类、函数写 docstring，第一行是一句概述，以句号结尾（PEP 257）。简单函数一行即可。
- 参数、返回值、异常只在不显而易见时才写，用 Google 风格：

```python
def fetch_page(client: httpx.AsyncClient, url: str, *, max_bytes: int) -> str:
    """Fetch a page and return its text.

    Args:
        client: Shared HTTP client; the caller owns its lifecycle.
        url: Absolute URL to fetch.
        max_bytes: Abort when the body exceeds this size.

    Raises:
        UpstreamUnavailable: The server kept failing after retries.
    """
```

- 类型已经写在签名里，docstring 里不要重复类型。
- 没有人维护的注释比没有注释更糟：改代码时同步改注释，或直接删除。
- TODO 写明负责人或 issue 编号，不留无主的 TODO。

## 4. Python 标准（PEP）对照

| 主题 | 标准 | 本项目的做法 |
|---|---|---|
| 代码风格 | PEP 8 | 由 `ruff format` 与 `ruff check` 强制，行宽 100 |
| Docstring | PEP 257 | 公开 API 写一句话概述，复杂函数用 Google 风格 |
| 类型注解 | PEP 484、526 | 全部函数签名注解，pyright strict |
| 内置泛型 | PEP 585 | 用 `list[str]`、`dict[str, int]` |
| 联合类型语法 | PEP 604 | 用 `X \| None`，不用 `Optional`、`Union` |
| 类型参数语法与 `type` 语句 | PEP 695（3.12） | 用 `def f[T]()`、`class C[T]`、`type X = ...` |
| `@override` | PEP 698（3.12） | 重写方法时标注，防止方法名拼错 |
| 类型包标记 | PEP 561 | 发布库时包含空文件 `py.typed` |
| 构建系统声明 | PEP 517、518 | `[build-system]` 用 hatchling |
| 项目元数据 | PEP 621 | 元数据写在 `[project]`，不用 setup.py |
| 依赖组 | PEP 735 | 开发依赖放 `[dependency-groups]`，不放 `optional-dependencies` |
| 脚本内联元数据 | PEP 723 | 单文件脚本用 `# /// script` 声明依赖 |
| 许可证表达 | PEP 639 | `license = "MIT"` 用 SPDX 表达式字符串 |
| 版本号 | PEP 440 + SemVer | 版本形如 `1.4.2`，破坏性变更升主版本 |
| 依赖说明符 | PEP 508 | 只写下界，如 `httpx>=0.28`；精确版本由 `uv.lock` 记录 |

说明：

- 运行时依赖用 `[project].dependencies`，开发期工具（测试、检查）用 `[dependency-groups]`，二者不要混用。`optional-dependencies` 只用于"用户安装时可选的功能"，例如 `pkg[postgres]`。
- 应用与库的依赖约束不同：应用靠锁文件固定版本；发布到 PyPI 的库只写宽松下界，不写上界（除非确知不兼容），否则会让下游无法解析。

## 5. 打包与版本

- 版本号只在 `pyproject.toml` 一处维护；运行时需要时用 `importlib.metadata.version("app")` 读取，不在代码里再写一份 `__version__`。
- 使用 `src/` 布局，`tests/` 放在包外。
- `requires-python` 写下界（如 `>=3.12`），`.python-version` 固定开发使用的小版本。
- 提交 `uv.lock`；发布库用 `uv build`，需要时 `uv publish`。
- 公开 API 的破坏性变更（删除、改签名、改行为）升主版本，并在 `CHANGELOG.md` 记录。

## 6. 完成的定义

一项变更同时满足以下条件才算完成：

1. SKILL.md 的质量闸门全部通过。
2. 新行为有测试；修复的 bug 有能复现的回归测试。
3. 涉及表结构的变更附带 alembic 迁移，且迁移经过人工审查。
4. 没有新增未经说明的依赖，`uv.lock` 与 `pyproject.toml` 同步并一起提交。
5. 没有遗留的 `print`、调试代码、被注释掉的代码、无主的 TODO。
6. 对外行为变化（接口、配置、命令行参数）已同步更新 README 或变更记录。

## 7. 代码审查清单

审查 Python 代码时按下列顺序检查，每条给出具体位置与建议，而不是笼统地说"可以更好"。

**正确性与边界**

- 进出系统的数据是否经过 pydantic 校验？请求模型是否 `extra="forbid"`？
- 所有外部调用（HTTP、数据库、子进程）是否有超时？重试是否只作用于幂等操作且有上限？
- 异常是否捕获具体类型、保留因果链？有无裸 `except` 或被吞掉的错误？
- 时间是否带时区？边界值（空列表、`None`、零、超长输入）是否处理？

**并发与资源**

- `async def` 里有没有阻塞调用（`time.sleep`、同步 HTTP、同步数据库驱动）？
- 是否用 `TaskGroup` 与 `timeout` 管理并发任务？有无泄漏的任务？
- 客户端、连接、文件是否用 `with` 管理并复用，而不是每次新建？

**数据库**

- SQL 是否全部参数化？有无 N+1 查询？事务边界是否清晰？
- 表结构变更是否有迁移？迁移能否回滚？

**类型与设计**

- 签名是否完整注解？有无不必要的 `Any` 与 `# type: ignore`？
- 业务逻辑是否放在服务层，而不是路由或命令行入口？依赖方向是否单向？
- 内部值对象是否用轻量的 dataclass，而不是到处用 pydantic 或裸字典？

**安全**

- 有无硬编码密钥、被记录进日志的敏感信息？
- 有无 `shell=True`、`eval`、`exec`、对不可信数据的 `pickle`？
- 文件路径是否来自用户输入而未做限制？

**依赖与风格**

- 是否引入了不在默认栈中的库或"不要用"一列的库？理由是否充分？
- 有无重复造轮子（自己写重试、配置解析、命令行解析）？
- 命名是否清晰，注释是否解释"为什么"？格式问题交给 ruff，不在审查中讨论。

**测试**

- 新行为与错误路径是否有测试？外部 HTTP 是否用 respx 模拟？
- 测试是否独立、可重复，有无依赖执行顺序或真实网络？
