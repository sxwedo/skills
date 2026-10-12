# Web 服务：FastAPI + SQLAlchemy

示例包名 `app`，Python 3.12+。横切实践（配置、日志、httpx、pydantic、安全）见 `practices.md`，测试见 `testing.md`。

## 1. FastAPI

```python
# app/main.py
from collections.abc import AsyncIterator
from contextlib import asynccontextmanager

import httpx
from fastapi import FastAPI, Request
from fastapi.responses import JSONResponse

from app.api.routes import router
from app.db.session import engine
from app.errors import AppError, NotFound
from app.logging_config import configure_logging
from app.settings import get_settings

@asynccontextmanager
async def lifespan(app: FastAPI) -> AsyncIterator[None]:
    settings = get_settings()
    configure_logging(debug=settings.debug)
    app.state.http = httpx.AsyncClient(timeout=httpx.Timeout(settings.http_timeout))
    try:
        yield
    finally:                                   # 退出时释放资源
        await app.state.http.aclose()
        await engine.dispose()

app = FastAPI(lifespan=lifespan)
app.include_router(router)

@app.exception_handler(AppError)
async def handle_app_error(request: Request, exc: AppError) -> JSONResponse:
    status = 404 if isinstance(exc, NotFound) else 400
    return JSONResponse({"error": type(exc).__name__, "detail": str(exc)}, status_code=status)
```

路由与依赖：

```python
# app/api/routes.py
from typing import Annotated

from fastapi import APIRouter, Depends, status
from sqlalchemy.ext.asyncio import AsyncSession

from app.db.session import get_session
from app.schemas.user import UserIn, UserOut
from app.services import users

router = APIRouter(prefix="/users", tags=["users"])

SessionDep = Annotated[AsyncSession, Depends(get_session)]   # 依赖用 Annotated 别名

@router.post("", response_model=UserOut, status_code=status.HTTP_201_CREATED)
async def create_user(payload: UserIn, session: SessionDep) -> UserOut:
    user = await users.create(session, payload)
    return UserOut.model_validate(user)

@router.get("/{user_id}", response_model=UserOut)
async def read_user(user_id: int, session: SessionDep) -> UserOut:
    user = await users.get(session, user_id)       # 找不到时服务层抛 NotFound
    return UserOut.model_validate(user)
```

规则：

- **用 lifespan 管理启动与关闭**，不用已被取代的 `@app.on_event`。
- **路由处理器保持很薄**：解析参数、调用服务层、返回模型。业务逻辑放在 `services/`，不在路由里写 SQL 和规则。
- 处理器用 `async def`；若必须调用同步阻塞代码，用 `await asyncio.to_thread(...)`，或把该处理器声明为普通 `def`（FastAPI 会放进线程池）。
- 显式声明 `response_model` 与 `status_code`，文档与行为才一致。
- 业务异常用统一的异常处理器转成响应，错误响应保持同一结构，不要每个路由各自拼。
- 分页接口返回 `items`、`total`、`limit`、`offset`，并对 `limit` 设上限。
- 生产启动：`uvicorn app.main:app --host 0.0.0.0 --port 8000`；本地开发加 `--reload`。

## 2. 数据库：SQLAlchemy + Alembic

```python
# app/db/base.py
from datetime import datetime

from sqlalchemy import MetaData, func
from sqlalchemy.orm import DeclarativeBase, Mapped, mapped_column

class Base(DeclarativeBase):
    metadata = MetaData(naming_convention={       # 约束命名固定，迁移才能稳定生成与回滚
        "ix": "ix_%(column_0_label)s",
        "uq": "uq_%(table_name)s_%(column_0_name)s",
        "ck": "ck_%(table_name)s_%(constraint_name)s",
        "fk": "fk_%(table_name)s_%(column_0_name)s_%(referred_table_name)s",
        "pk": "pk_%(table_name)s",
    })

class User(Base):
    __tablename__ = "users"

    id: Mapped[int] = mapped_column(primary_key=True)
    email: Mapped[str] = mapped_column(unique=True)
    created_at: Mapped[datetime] = mapped_column(server_default=func.now())
```

```python
# app/db/session.py
from collections.abc import AsyncIterator

from sqlalchemy.ext.asyncio import AsyncSession, async_sessionmaker, create_async_engine

from app.settings import get_settings

engine = create_async_engine(str(get_settings().database_url), pool_pre_ping=True)
SessionLocal = async_sessionmaker(engine, expire_on_commit=False)

async def get_session() -> AsyncIterator[AsyncSession]:
    async with SessionLocal() as session:
        yield session
```

数据库 URL 使用 `postgresql+psycopg://user:pass@host/db`。

查询与事务：

```python
from sqlalchemy import select
from sqlalchemy.orm import selectinload

async def get_by_email(session: AsyncSession, email: str) -> User | None:
    result = await session.scalars(select(User).where(User.email == email))
    return result.one_or_none()

async def create(session: AsyncSession, payload: UserIn) -> User:
    user = User(email=payload.email)
    session.add(user)
    await session.commit()          # 事务边界放在服务层，不放在底层查询函数里
    await session.refresh(user)
    return user

# 避免 N+1：一次性预加载关联
stmt = select(Order).options(selectinload(Order.items)).where(Order.user_id == uid)
```

规则：

- 只用 2.x 风格：`select()`、`Mapped[...]`、`mapped_column()`；不用 `session.query()`。
- 异步会话下**不要依赖懒加载**（会触发隐式 I/O 而报错），需要关联数据时用 `selectinload` / `joinedload` 显式预加载。
- SQL 一律参数化，不用 f-string 拼接。需要原生 SQL 时用 `text("... WHERE id = :id")` 加绑定参数。
- 一个请求一个会话，通过依赖注入创建与关闭，不全局共享会话。
- **表结构变更只通过 Alembic 迁移**：

```bash
uv run alembic init -t async migrations          # 首次，生成异步模板
# 编辑 migrations/env.py：target_metadata = Base.metadata，URL 取自 get_settings()
uv run alembic revision --autogenerate -m "add users"
# 打开生成的文件人工审查，autogenerate 会漏掉一部分变更（如重命名、数据迁移）
uv run alembic upgrade head
```

- 迁移文件进版本库，已发布的迁移不再修改，要改就新增一个迁移。
