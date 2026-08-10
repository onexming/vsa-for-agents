# 示例：注册用户垂直切片

本示例在仓库没有可靠范例时展示一条最小完整路径：公开契约 → 私有实现 → 公共入口 → 行为测试 → 组合根。它只表达职责边界；语言、框架、文件数量和名称服从项目工具链与已加载规范。

## 边界地图

```text
application/
├── features/user/register_user/
│   ├── contracts.py               # 公开输入、输出、错误与事件契约
│   ├── entry.py                   # 公共入口
│   ├── model.py                   # 切片私有模型
│   ├── ports.py                   # 切片私有能力契约
│   ├── handler.py                 # 切片私有业务流程
│   ├── sql_user_store.py          # 切片私有持久化适配
│   └── test_register_user.py      # 从公共入口验证完整行为
├── shared/
│   ├── runtime/identifiers.py     # 无业务语义的标识符生成
│   ├── security/passwords.py      # 无业务语义的密码散列
│   └── transport/events.py        # 无业务语义的事件基础设施
├── composition/registry.py        # 具体依赖与入口注册
└── main.py
```

`User` 和数据访问先保持切片私有；只有出现符合 [`shared-code.md`](shared-code.md) 的真实复用后才逐级提升。`UserRegistered` 是切片明确公开的事件契约，不要求消费方读取处理器或模型。

## 公开契约与私有模型

```python
# features/user/register_user/contracts.py
from dataclasses import dataclass

@dataclass(frozen=True)
class RegisterUserCommand:
    email: str
    username: str
    password: str

@dataclass(frozen=True)
class RegisterUserResult:
    user_id: str

@dataclass(frozen=True)
class UserRegistered:
    user_id: str
    email: str

class EmailAlreadyRegisteredError(Exception):
    pass
```

```python
# features/user/register_user/model.py
from dataclasses import dataclass

@dataclass
class User:
    id: str
    email: str
    username: str
    password_hash: str
```

## 私有端口与业务流程

```python
# features/user/register_user/ports.py
from typing import Protocol
from .model import User

class UserStore(Protocol):
    def find_by_email(self, email: str) -> User | None: ...
    def save(self, user: User) -> None: ...

class PasswordHasher(Protocol):
    def hash(self, raw_password: str) -> str: ...

class EventPublisher(Protocol):
    def publish(self, event: object) -> None: ...

class IdentifierGenerator(Protocol):
    def new_id(self) -> str: ...
```

```python
# features/user/register_user/handler.py
from .contracts import (
    EmailAlreadyRegisteredError,
    RegisterUserCommand,
    RegisterUserResult,
    UserRegistered,
)
from .model import User
from .ports import EventPublisher, IdentifierGenerator, PasswordHasher, UserStore

def register_user(command, *, users, password_hasher, events, identifiers):
    if users.find_by_email(command.email) is not None:
        raise EmailAlreadyRegisteredError(command.email)

    user = User(
        id=identifiers.new_id(),
        email=command.email,
        username=command.username,
        password_hash=password_hasher.hash(command.password),
    )
    users.save(user)
    events.publish(UserRegistered(user_id=user.id, email=user.email))
    return RegisterUserResult(user_id=user.id)
```

## 公共入口

```python
# features/user/register_user/entry.py
from dataclasses import dataclass
from .contracts import RegisterUserCommand, EmailAlreadyRegisteredError
from .handler import register_user
from .ports import EventPublisher, IdentifierGenerator, PasswordHasher, UserStore

@dataclass
class Dependencies:
    users: UserStore
    password_hasher: PasswordHasher
    events: EventPublisher
    identifiers: IdentifierGenerator

def register_user_entry(payload: dict, dependencies: Dependencies) -> tuple[int, dict]:
    command = RegisterUserCommand(
        email=payload["email"],
        username=payload["username"],
        password=payload["password"],
    )
    try:
        result = register_user(
            command,
            users=dependencies.users,
            password_hasher=dependencies.password_hasher,
            events=dependencies.events,
            identifiers=dependencies.identifiers,
        )
    except EmailAlreadyRegisteredError:
        return 409, {"error": "email_already_registered"}
    return 201, {"user_id": result.user_id}
```

HTTP、CLI 或 RPC 适配器调用同一入口；业务流程不接触框架请求对象。

## 从公共入口测试

```python
# features/user/register_user/test_register_user.py
from .contracts import UserRegistered
from .entry import Dependencies, register_user_entry

class InMemoryUsers:
    def __init__(self): self.items = {}
    def find_by_email(self, email):
        return next((u for u in self.items.values() if u.email == email), None)
    def save(self, user): self.items[user.id] = user

class FakeHasher:
    def hash(self, raw): return f"hashed:{raw}"

class RecordedEvents:
    def __init__(self): self.items = []
    def publish(self, event): self.items.append(event)

class FixedIdentifiers:
    def new_id(self): return "user-001"

def make_dependencies():
    return Dependencies(
        InMemoryUsers(), FakeHasher(), RecordedEvents(), FixedIdentifiers()
    )

def test_register_user_from_public_entry():
    dependencies = make_dependencies()
    status, body = register_user_entry(
        {"email": "test@example.com", "username": "tester", "password": "secret"},
        dependencies,
    )
    assert (status, body) == (201, {"user_id": "user-001"})
    assert dependencies.users.items["user-001"].password_hash == "hashed:secret"
    assert dependencies.events.items == [
        UserRegistered(user_id="user-001", email="test@example.com")
    ]

def test_reject_duplicate_email_from_public_entry():
    dependencies = make_dependencies()
    payload = {"email": "dup@example.com", "username": "first", "password": "secret"}
    assert register_user_entry(payload, dependencies)[0] == 201
    status, body = register_user_entry(payload | {"username": "second"}, dependencies)
    assert (status, body) == (409, {"error": "email_already_registered"})
    assert len(dependencies.users.items) == 1
    assert len(dependencies.events.items) == 1
```

测试观察公开响应、持久化状态、领域事件和主要失败路径；内部函数测试只能补充这些行为测试。

## 在组合根装配

```python
# composition/registry.py
from features.user.register_user.entry import Dependencies, register_user_entry
from features.user.register_user.sql_user_store import SqlUserStore
from shared.runtime.identifiers import UuidGenerator
from shared.security.passwords import ArgonPasswordHasher
from shared.transport.events import InProcessEventBus

def register_user_route(router, database):
    dependencies = Dependencies(
        users=SqlUserStore(database),
        password_hasher=ArgonPasswordHasher(),
        events=InProcessEventBus(),
        identifiers=UuidGenerator(),
    )
    router.post("/users", lambda payload: register_user_entry(payload, dependencies))
```

组合根只选择实现并注册入口；注册资格、重复邮箱判断和状态变化仍在切片中。

## 示例完成标准

该示例仅在以下关系全部成立时可作为实现参照：输入经公共入口到达单一行为，成功与失败映射为公开结果，状态和事件可观察，业务代码保持切片私有，技术共享不含领域语义，具体依赖只在组合根连接。
