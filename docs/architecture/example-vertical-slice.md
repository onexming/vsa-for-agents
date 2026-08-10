# 示例：注册用户垂直切片

本示例只展示边界如何贯通，不规定其他语言复制 Python 的文件名、类型系统或运行模型。创建真实功能时，以仓库现有工具链和已加载的专题规范为准。

## 目录

```text
application/
├── features/user/
│   ├── _shared/
│   │   ├── model.py
│   │   └── events.py
│   └── register_user/
│       ├── contracts.py
│       ├── ports.py
│       ├── handler.py
│       ├── entry.py
│       └── test_register_user.py
├── shared/runtime/identifiers.py
├── composition/registry.py
└── main.py
```

## 领域定义与切片契约

```python
# features/user/_shared/model.py
from dataclasses import dataclass

@dataclass
class User:
    id: str
    email: str
    username: str
    password_hash: str
```

```python
# features/user/_shared/events.py
from dataclasses import dataclass

@dataclass(frozen=True)
class UserRegistered:
    user_id: str
    email: str
```

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
```

## 切片私有端口

```python
# features/user/register_user/ports.py
from typing import Protocol
from features.user._shared.model import User

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

## 业务处理

```python
# features/user/register_user/handler.py
from features.user._shared.events import UserRegistered
from features.user._shared.model import User
from .contracts import RegisterUserCommand, RegisterUserResult
from .ports import EventPublisher, IdentifierGenerator, PasswordHasher, UserStore

class EmailAlreadyRegisteredError(Exception):
    pass

def register_user(
    command: RegisterUserCommand,
    *,
    users: UserStore,
    password_hasher: PasswordHasher,
    events: EventPublisher,
    identifiers: IdentifierGenerator,
) -> RegisterUserResult:
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
from .contracts import RegisterUserCommand
from .handler import EmailAlreadyRegisteredError, register_user

def register_user_entry(payload: dict, dependencies) -> tuple[int, dict]:
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

HTTP、CLI 或 RPC 适配器可以调用此入口；业务处理无需知道具体框架请求对象。

## 从公共入口测试

```python
# features/user/register_user/test_register_user.py
from dataclasses import dataclass
from features.user.register_user.entry import register_user_entry

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

@dataclass
class Dependencies:
    users: InMemoryUsers
    password_hasher: FakeHasher
    events: RecordedEvents
    identifiers: FixedIdentifiers

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
    assert dependencies.users.items["user-001"].email == "test@example.com"
    assert len(dependencies.events.items) == 1

def test_reject_duplicate_email_from_public_entry():
    dependencies = make_dependencies()
    payload = {"email": "dup@example.com", "username": "first", "password": "secret"}
    assert register_user_entry(payload, dependencies)[0] == 201
    status, body = register_user_entry(payload | {"username": "second"}, dependencies)
    assert (status, body) == (409, {"error": "email_already_registered"})
```

## 在组合根装配

```python
# composition/registry.py
from dataclasses import dataclass
from shared.persistence.users import SqlUserStore
from shared.runtime.identifiers import UuidGenerator
from shared.security.passwords import ArgonPasswordHasher
from shared.transport.events import InProcessEventBus

@dataclass
class Dependencies:
    users: SqlUserStore
    password_hasher: ArgonPasswordHasher
    events: InProcessEventBus
    identifiers: UuidGenerator

def build_dependencies(database) -> Dependencies:
    return Dependencies(
        users=SqlUserStore(database),
        password_hasher=ArgonPasswordHasher(),
        events=InProcessEventBus(),
        identifiers=UuidGenerator(),
    )
```

组合根知道具体实现；切片只依赖自身声明的能力契约。示例完整性的判定是：输入能经公共入口到达业务处理，状态和事件可观察，失败被转换为公开错误，并且具体依赖只在组合根出现。
