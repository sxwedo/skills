# 测试：pytest 系列

目录镜像 `src/`，文件名 `test_*.py`。测试要小、快、互相独立，一个测试只验证一件事。

```python
# tests/test_users.py
import httpx
import pytest
import respx

from app.clients import get_user

@pytest.mark.parametrize(
    ("raw", "expected"),
    [("A@Example.com", "a@example.com"), (" b@x.io ", "b@x.io")],
)
def test_email_is_normalized(raw: str, expected: str) -> None:
    assert UserIn(name="n", email=raw).email == expected

@respx.mock
async def test_get_user_ok() -> None:                  # asyncio_mode="auto"，无需装饰器
    respx.get("https://api.example.com/users/1").mock(
        return_value=httpx.Response(200, json={"id": 1, "name": "Ada"})
    )
    async with httpx.AsyncClient(base_url="https://api.example.com") as client:
        user = await get_user(client, 1)
    assert user.name == "Ada"
```

测试 FastAPI 应用：

```python
import httpx
import pytest
from asgi_lifespan import LifespanManager   # uv add --dev asgi-lifespan

@pytest.fixture
async def client():
    async with LifespanManager(app):          # ASGITransport 不会触发 lifespan，需要它
        transport = httpx.ASGITransport(app=app)
        async with httpx.AsyncClient(transport=transport, base_url="http://test") as c:
            yield c

async def test_create_user(client: httpx.AsyncClient) -> None:
    resp = await client.post("/users", json={"name": "Ada", "email": "ada@example.com"})
    assert resp.status_code == 201
```

属性测试：对解析、序列化、转换这类"输入空间很大"的逻辑，描述不变量而不是列举样例：

```python
from hypothesis import given, strategies as st

@given(st.text())
def test_roundtrip(s: str) -> None:
    assert decode(encode(s)) == s
```

规则：

- **外部 HTTP 用 respx 模拟，测试里绝不访问真实网络。**
- **数据库用真实 PostgreSQL**（本地容器或 CI 服务），每个测试在事务里运行并回滚，不 mock 数据库，mock 掉的测试证明不了迁移与查询是对的。
- 时间、随机数这类不确定因素通过参数或 `Clock` 协议注入，不在测试里打补丁。
- 优先用 fixture 构造数据，`tmp_path` 处理文件，`monkeypatch` 处理环境变量。
- 覆盖率是线索不是目标：关键分支与错误路径必须覆盖，别为凑数写断言空洞的测试。
- 修 bug 时先写一个能复现的失败测试，再修复。
