# 编码规范：垂直切片架构

## 目录

1. [架构原则](#1-架构原则)
2. [目录结构规范](#2-目录结构规范)
3. [三层共享模型](#3-三层共享模型)
4. [切片间通信规范](#4-切片间通信规范)
5. [强制规则](#5-强制规则)
6. [新增功能 Checklist](#6-新增功能-checklist)
7. [修改已有功能 Checklist](#7-修改已有功能-checklist)
8. [提取共享模块 Checklist](#8-提取共享模块-checklist)
9. [典型代码示例](#9-典型代码示例)
10. [反模式识别](#10-反模式识别)
11. [边界情况决策指南](#11-边界情况决策指南)

---

## 1. 架构原则

本项目采用**垂直切片架构（Vertical Slice Architecture）**。

每个功能需求、用户意图、业务命令、查询或外部事件处理都应建模为一个独立的“切片”。切片包含实现该行为所需的完整代码路径：从外部入口开始，经过输入校验、业务决策、状态读取或变更，直到产生可观察结果。

外部入口可以是 HTTP、RPC、消息、命令行、图形界面操作、定时任务、批处理节点、设备指令或其他触发方式。架构规范不依赖具体编程语言、框架、协议和部署形态。

### 1.1 核心目标

- **高内聚：** 与同一行为有关的代码集中在同一切片中。
- **低耦合：** 切片不依赖其他切片的内部实现。
- **可定位：** 能通过“领域 + 用户意图”快速找到全部相关代码。
- **可验证：** 能从切片公共入口验证完整行为及其可观察结果。
- **可替换：** 技术框架、存储方式或交互形式变化时，不破坏业务边界。
- **可演进：** 共享代码按实际复用逐级提升，不预先设计大而全的公共层。

### 1.2 核心思想对比

| 横向分层架构 ❌ | 垂直切片架构 ✅ |
|----------------|----------------|
| 按控制器、服务、仓储、组件等技术职责分目录 | 按领域和用户意图组织代码 |
| 修改一个行为需要跨多个技术目录 | 修改一个行为主要进入一个切片目录 |
| 公共 Service、Manager 持续聚合无关用例 | 每个切片只负责一个明确行为 |
| 内部函数成为主要复用与测试边界 | 公共入口和可观察行为成为主要边界 |
| 依赖方向由调用便利性决定 | 依赖方向由领域边界和公开契约决定 |
| 共享层在实现前被预先设计 | 共享层在真实复用发生后逐级提取 |

### 1.3 切片的定义

一个合格的切片应满足：

1. 表达一个明确的用户意图、业务命令、查询或事件处理目标。
2. 具有可识别的公共入口。
3. 拥有完成该行为所需的私有实现。
4. 对外产生可观察结果，例如返回值、状态变化、事件、文件、消息或界面变化。
5. 可以在不读取其他切片内部代码的情况下理解和测试。

> 切片不是单个函数或单个接口，也不是按数据库实体划分的模块。切片是一个完整行为的代码边界。

### 1.4 依赖方向

依赖只能从具体行为指向稳定抽象：

```text
外部入口
   ↓
功能切片
   ↓
领域共享定义
   ↓
全局技术基础设施
```

反向依赖不允许出现：

- 全局共享模块不得依赖具体领域。
- 领域 `_shared/` 不得依赖任何切片内部实现。
- 一个切片不得直接依赖另一个切片的内部文件。
- 组合根可以引用各切片公共入口，但不得实现业务规则。

---

## 2. 目录结构规范

本节只规定**代码的业务边界、依赖方向和职责位置**，不规定具体编程语言、框架、文件扩展名或代码组织形式。

目录结构以“交付单元 → 领域 → 功能切片”为主轴。无论项目是服务端、客户端、桌面应用、命令行工具、嵌入式程序、批处理任务或消息驱动系统，都应使用同一套组织原则。

### 2.1 抽象层级

| 层级 | 含义 | 典型实例 |
|------|------|---------|
| 交付单元（Delivery Unit） | 可独立构建、运行、发布或部署的程序单元 | 服务、应用、可执行程序、后台任务 |
| 领域（Domain） | 一组具有共同业务语义的能力 | 订单、用户、支付、库存 |
| 功能切片（Feature Slice） | 对应一个用户意图、业务命令、查询或外部事件处理 | 创建订单、取消订单、查询订单详情 |
| 领域共享（Domain Shared） | 同一领域内多个切片共同依赖的被动定义或工具 | 领域模型、公共契约、领域错误、纯规则 |
| 全局共享（Global Shared） | 跨领域复用且不包含业务语义的技术基础设施 | 存储、通信、安全、日志、配置 |
| 组合根（Composition Root） | 注册切片、装配依赖并启动交付单元的位置 | 应用入口、处理器注册、依赖绑定 |

### 2.2 通用目录模板

下列名称使用语义占位符表示职责，并非要求创建同名文件。`*` 表示由具体语言和工具链决定的文件扩展名。

```text
<delivery-unit>/
├── features/
│   ├── <domain>/
│   │   ├── _shared/                         # 本领域内多切片共享的被动模块
│   │   │   ├── <domain-model>.*             # 领域模型或持久化映射
│   │   │   ├── <domain-contract>.*          # 公共类型、数据契约或协议定义
│   │   │   ├── <domain-error>.*             # 领域错误、错误码或异常定义
│   │   │   ├── <domain-rule>.*              # 无副作用的领域规则或计算
│   │   │   └── <domain-event>.*             # 领域事件及其载荷定义
│   │   │
│   │   ├── <verb-object>/                   # 一个用户意图或业务能力切片
│   │   │   ├── <entry-adapter>.*            # HTTP、RPC、CLI、UI、消息、任务等入口
│   │   │   ├── <input-output-contract>.*    # 本切片输入、输出及校验契约
│   │   │   ├── <use-case-implementation>.*  # 本切片业务流程与状态变更
│   │   │   ├── <slice-local-support>.*      # 私有端口、映射、查询或辅助代码
│   │   │   └── <slice-test>.*               # 从公共入口覆盖完整行为的测试
│   │   │
│   │   └── <another-verb-object>/
│   │       └── ...
│   │
│   └── <another-domain>/
│       └── ...
│
├── shared/                                  # 跨领域技术基础设施，不含业务逻辑
│   ├── persistence/                         # 数据库、文件、缓存等存储能力
│   ├── transport/                           # HTTP、RPC、消息、进程间通信等
│   ├── security/                            # 认证、授权、加密、凭据管理
│   ├── observability/                       # 日志、指标、追踪、审计
│   ├── runtime/                             # 配置、时间、调度、运行环境适配
│   └── presentation/                        # 无业务语义的通用展示能力
│
├── composition/                             # 仅负责注册、装配和启动
│   ├── <feature-registry>.*                 # 切片或模块注册
│   ├── <interface-registry>.*               # 路由、命令、页面、消息订阅等注册
│   └── <dependency-wiring>.*                # 依赖绑定与运行时装配
│
└── <application-entry>.*                    # 交付单元启动入口
```

### 2.3 目录职责边界

#### `features/`

存放所有具有业务语义的代码。代码首先按领域分组，再按用户意图或业务能力拆分为切片，不按控制器、服务、仓储、组件、状态管理等技术职责横向分组。

#### `features/<domain>/`

表示一个稳定的业务领域边界。领域名称应来自业务语言，而不是数据库表名、框架模块名或技术实现名。

#### `features/<domain>/<verb-object>/`

表示一个完整的功能切片。切片应包含实现该行为所需的入口、契约、业务流程、数据访问、展示逻辑和测试；这些职责可以放在一个文件中，也可以拆成多个文件，但不得因此移动到全局横向目录。

切片入口不限于 Web 路由，也可以是：

- RPC 方法或消息处理器
- 命令行命令或桌面操作
- 页面、屏幕、对话框或交互流程
- 定时任务、批处理任务或工作流节点
- 设备指令、中断处理或其他外部输入适配器

#### `features/<domain>/_shared/`

只存放同一领域内多个切片复用的**被动模块**。允许定义数据、类型、契约、错误、事件和无副作用规则；不得在此编排业务流程，也不得调用任一切片的内部实现。

#### `shared/`

只存放跨领域复用的技术基础设施。目录中的模块不得依赖具体领域，也不得包含“订单”“用户”“支付”等业务概念。

#### `composition/`

只负责发现、注册、连接和启动切片，不实现业务规则。依赖关系必须在此处或等价的组合根中显式装配，避免通过隐式全局状态建立耦合。

### 2.4 切片内部组织规则

切片目录规定的是**边界**，不是固定文件清单：

1. 简单切片可以只包含一个实现文件和一个测试文件。
2. 复杂切片可以按入口、契约、流程、展示、持久化适配等职责拆分多个文件。
3. 所有切片私有文件必须保留在切片目录内。
4. 不得为了“整齐”把各切片的同类文件集中到全局 `controllers/`、`services/`、`repositories/`、`components/`、`stores/` 等横向目录。
5. 切片对外只能暴露明确的公共入口或契约，内部文件默认不可被其他切片直接依赖。
6. 测试应从切片的公共入口进入，并验证可观察结果，而不是只测试内部函数。

### 2.5 多交付单元项目

当一个仓库包含多个应用、服务或可执行程序时，每个交付单元独立应用上述结构：

```text
<workspace>/
├── applications/
│   ├── <delivery-unit-a>/
│   │   ├── features/
│   │   ├── shared/
│   │   ├── composition/
│   │   └── <application-entry>.*
│   └── <delivery-unit-b>/
│       └── ...
│
├── libraries/                               # 可独立复用的纯技术库
│   └── <technical-library>/
│
└── contracts/                               # 可选：跨进程或跨语言公开协议
    └── <protocol-or-schema>/
```

跨交付单元共享时遵循以下规则：

- 业务流程不通过源码目录直接共享，而通过公开协议、消息或 API 协作。
- `libraries/` 只容纳无业务语义的技术库。
- `contracts/` 只定义公开通信契约，不承载任一交付单元的内部实现。
- 每个交付单元拥有自己的组合根和启动入口，不共享隐式运行时状态。

### 2.6 通用命名规范

| 对象 | 语言无关规则 | 示例语义 |
|------|-------------|---------|
| 领域目录 | 使用业务名词；单复数形式由项目统一约定 | `order`、`user`、`payment` |
| 切片目录 | 使用“动词 + 对象/结果”的意图短语 | `create_order`、`approve_order`、`list_orders` |
| 领域共享目录 | 使用项目约定的保留名称，推荐 `_shared` | `<domain>/_shared/` |
| 全局共享目录 | 使用 `shared` 或项目统一的等价名称 | `<delivery-unit>/shared/` |
| 组合根目录 | 使用能表达装配职责的名称，推荐 `composition` | `composition/` |
| 切片内文件 | 按单一职责命名，名称应说明角色或行为 | `entry`、`contracts`、`handler`、`ports`、`test` |

具体大小写和分隔符遵循目标语言及其生态的主流约定，例如 snake_case、kebab-case、camelCase 或 PascalCase；架构规范不强制某一种格式，但同一作用域内必须保持一致。

> **强制语义规则：切片名称必须表达行为或用户意图，而不是仅表达数据实体。**
>
> ✅ `create_order`、`approve_order`、`get_order_detail`  
> ❌ `order`、`order_data`、`order_service`

### 2.7 结构合规判定

一个目录结构满足本规范，至少应同时满足：

- 能通过“领域 + 用户意图”快速定位功能代码。
- 修改单个功能时，大部分变更限制在一个切片目录内。
- 切片拥有明确的公共入口和可独立验证的行为。
- 切片私有实现不会被其他切片直接引用。
- 共享代码按切片私有、领域共享、全局共享逐级提升。
- 全局共享目录不包含业务语义。
- 组合根只装配依赖，不承载业务流程。
- 语言、框架或部署形态变化时，目录边界和依赖规则无需改变。

---

## 3. 三层共享模型

共享代码遵循**就近原则**。代码最初属于具体切片，只有发生真实复用后才允许逐级提升，并且每次最多提升一层：

```text
切片私有  ──→  领域 _shared  ──→  全局 shared
   ↑                ↑                  ↑
单一行为需要      同领域多个行为需要    跨领域纯技术复用
```

### 3.1 切片私有

位置：`features/<domain>/<slice>/`

允许包含：

- 该行为独有的输入、输出和校验契约
- 业务流程、状态变更和错误分支
- 切片专属的数据访问端口和适配器
- 切片专属的映射、展示、序列化或交互逻辑
- 切片专属事件处理和副作用
- 从公共入口覆盖完整行为的测试

示例：

```text
features/order/create_order/
├── contracts.py
├── handler.py
├── ports.py
├── entry.py
└── test_create_order.py
```

默认规则：代码只被一个切片使用时，必须保持切片私有，即使存在未来可能复用的猜测。

### 3.2 领域共享

位置：`features/<domain>/_shared/`

允许放入的内容必须同时满足：

1. 已被同一领域的多个切片实际使用。
2. 只表达稳定的领域概念或无副作用规则。
3. 自身不编排完整业务流程。
4. 不调用任何切片内部实现。
5. 不替其他领域转发能力。

Python 示例：

```text
features/order/_shared/
├── model.py          # Order、OrderLine 等稳定领域模型
├── contracts.py      # 同领域公共值对象或数据契约
├── errors.py         # 领域错误定义
├── rules.py          # 纯计算、纯判断、无副作用规则
└── events.py         # 领域事件及载荷定义
```

允许：

```python
# features/order/_shared/rules.py
from decimal import Decimal


def calculate_total(unit_price: Decimal, quantity: int) -> Decimal:
    if quantity <= 0:
        raise ValueError("quantity must be positive")
    return unit_price * quantity
```

禁止：

```python
# features/order/_shared/order_service.py  # ❌
class OrderService:
    def create_order(self, command): ...
    def cancel_order(self, command): ...
    def approve_order(self, command): ...
```

`_shared/` 应当是被动依赖：切片调用它，它不反过来驱动切片。

### 3.3 全局共享

位置：`shared/`

只允许放置不含业务语义的技术基础设施，例如：

```text
shared/
├── persistence/      # 连接、事务、通用存储适配
├── transport/        # 通用 HTTP、RPC、消息客户端与服务器能力
├── security/         # 认证、授权、加密
├── observability/    # 日志、指标、追踪
├── runtime/          # 配置、时钟、ID 生成、任务调度
└── testing/          # 无业务语义的测试基础设施
```

Python 示例：

```python
# shared/runtime/clock.py
from datetime import datetime, timezone


def utc_now() -> datetime:
    return datetime.now(timezone.utc)
```

以下内容不得进入全局共享：

- `OrderValidator`、`UserManager` 等包含领域名称的模块
- 特定业务状态机或业务错误分支
- 只服务于一个领域的查询或持久化逻辑
- 跨领域业务流程编排

### 3.4 提升时机

| 当前状态 | 判断 | 行动 |
|---------|------|------|
| 仅 1 个切片使用 | 尚未形成复用 | 保持切片私有 |
| 同领域 2 个切片使用 | 可能形成稳定领域概念 | 评估后可提升到 `_shared/` |
| 同领域 3 个及以上切片使用 | 已形成明显复用 | 应审查并提升稳定部分到 `_shared/` |
| 跨领域使用且无业务语义 | 属于技术能力 | 提升到全局 `shared/` |
| 跨领域使用且含业务语义 | 属于独立业务能力或边界问题 | 新建切片、公开协议或重新划分领域 |

### 3.5 逐级提升规则

不得从切片私有代码直接跳到全局 `shared/`。必须先回答：

1. 该代码是否仍含当前领域语义？
2. 它是否只是同领域多个切片共享？
3. 移除领域名称后，它是否仍然成立？
4. 它是否会产生业务副作用？

只有完全无领域语义、可跨领域独立使用的技术能力，才允许进入全局共享。

---

## 4. 切片间通信规范

切片之间**禁止直接引用对方的内部文件**。通信必须通过稳定边界完成，调用方不得依赖被调用方的内部数据结构、处理函数或持久化实现。

### 4.1 允许方式一：使用全局技术基础设施

切片可以共同依赖 `shared/` 中的技术能力，但不能借助 `shared/` 转发业务调用。

```python
# ✅ 使用通用时钟和事件基础设施
from shared.runtime.clock import utc_now
from shared.transport.events import EventPublisher
```

### 4.2 允许方式二：发布领域事件或消息

当一个切片完成后需要触发其他行为，尤其是通知、审计、同步、索引、异步处理等副作用时，应发布事件，而不是直接调用另一个切片。

```python
# features/order/create_order/handler.py
from features.order._shared.events import OrderCreated


def create_order(command, orders, events) -> str:
    order = orders.create(command)
    events.publish(OrderCreated(order_id=order.id, user_id=order.user_id))
    return order.id
```

```python
# features/notification/send_order_confirmation/handler.py
from features.order._shared.events import OrderCreated


def handle_order_created(event: OrderCreated, mailer) -> None:
    mailer.send_order_confirmation(
        user_id=event.user_id,
        order_id=event.order_id,
    )
```

事件订阅关系必须在组合根中注册：

```python
# composition/event_registry.py

def register_event_handlers(event_bus, send_confirmation):
    event_bus.subscribe("order.created", send_confirmation)
```

### 4.3 允许方式三：通过公开协议或进程边界调用

当需要同步查询其他领域，或能力属于其他交付单元时，应通过公开 API、RPC、消息请求或其他明确协议调用。

```python
# features/order/create_order/inventory_client.py
from dataclasses import dataclass


@dataclass(frozen=True)
class InventoryAvailability:
    available: bool
    remaining: int


class InventoryClient:
    def __init__(self, transport, base_url: str):
        self._transport = transport
        self._base_url = base_url

    def check(self, product_id: str, quantity: int) -> InventoryAvailability:
        payload = self._transport.get(
            f"{self._base_url}/inventory/{product_id}/availability",
            params={"quantity": quantity},
        )
        return InventoryAvailability(**payload)
```

调用方只依赖公开响应契约，不依赖库存领域的模型、仓储或内部处理函数。

### 4.4 禁止直接引用切片内部实现

```python
# ❌ approve_order 直接导入 create_order 的内部校验
from features.order.create_order.handler import validate_order_input
```

```python
# ❌ 一个事件处理切片直接调用另一个切片的私有函数
from features.notification.send_order_confirmation.handler import _render_email
```

正确处理：

- 真正稳定的同领域纯逻辑：提升到领域 `_shared/`。
- 触发其他业务副作用：发布事件。
- 查询其他领域或交付单元：调用公开协议。
- 两个切片总是同步变化：重新评估是否应合并切片或调整领域边界。

### 4.5 禁止架构逃避模式

禁止通过 `*_adapter`、`*_bridge`、`*_facade`、`*_proxy` 等模块重新导出其他切片或领域的内部实现，以规避依赖规则。

```python
# features/campaign/_shared/device_adapter.py  # ❌
from features.device.register_device.handler import register_device

__all__ = ["register_device"]
```

这种文件没有消除耦合，只是隐藏耦合。判断标准不是文件名，而是它是否：

- 导入另一个切片或领域的内部实现；
- 不增加稳定协议，仅进行转发或重新导出；
- 使调用方继续依赖被调用方的内部变化。

### 4.6 公开契约规则

跨边界通信所使用的契约必须满足：

- 字段和语义明确，避免直接暴露内部对象。
- 由提供能力的一方维护，或由独立协议包维护。
- 变更时遵循兼容性策略。
- 不携带数据库连接、框架请求对象、界面组件等运行时内部对象。
- 不要求调用方了解提供方的内部目录结构。

---

## 5. 强制规则

### 5.1 必须做

1. **新增独立行为时，必须创建或明确归属一个功能切片。**
2. **切片名称必须表达用户意图、业务命令、查询目标或事件处理目的。**
3. **切片必须具有明确公共入口。**入口可以是函数、处理器、命令、页面、消息消费者或等价形式。
4. **切片必须包含从公共入口验证完整行为的测试。**测试应验证返回结果、状态变化、事件、消息、文件或其他可观察结果。
5. **切片必须在组合根中显式注册或装配。**不得依赖隐式导入副作用完成注册。
6. **优先允许切片内的小规模重复。**只有真实复用出现后才提取共享代码。
7. **领域 `_shared/` 必须保持被动。**不得承载完整业务流程或调用切片内部实现。
8. **全局 `shared/` 只能包含无业务语义的技术基础设施。**
9. **切片之间的协作必须通过领域事件、公开协议或明确的组合关系完成。**
10. **架构边界必须可由静态检查、依赖测试或代码审查验证。**

### 5.2 禁止做

| 禁止行为 | 原因 |
|---------|------|
| 按 `controllers/`、`services/`、`repositories/` 等横向目录组织全部业务代码 | 同一行为被拆散，修改需要跨目录 |
| 新建聚合多个用例的 `*Service`、`*Manager`、`*Coordinator` | 容易演变为上帝对象和隐形分层 |
| 直接导入另一个切片的内部文件 | 形成隐性编译期耦合 |
| 在全局 `shared/` 中放置领域模型或业务规则 | 破坏领域边界并扩大影响范围 |
| 将多个独立用户意图合并到同一切片 | 切片职责不再单一 |
| 仅因代码相似就立即提取公共模块 | 相似不等于稳定复用 |
| 从切片私有代码直接提升到全局共享 | 跳过领域边界判断 |
| 用 adapter、bridge、facade 等文件重新导出跨边界内部实现 | 只隐藏耦合，没有建立稳定协议 |
| 在组合根中实现业务判断 | 装配代码与业务规则混合 |
| 只测试内部函数而不测试公共入口 | 无法验证完整行为和集成边界 |

### 5.3 允许的例外

只有满足以下条件并在架构决策记录中说明时，才允许例外：

- 受语言或平台强制目录约束；
- 由代码生成器产生且不承载手写业务逻辑；
- 为兼容遗留系统而设置的临时边界；
- 已明确迁移目标、负责人和移除条件。

例外不能成为新代码复制的模板。

---

## 6. 新增功能 Checklist

在实现任何新行为之前，逐条确认：

### 6.1 边界与命名

- [ ] 已确认该功能属于哪个交付单元。
- [ ] 已确认该功能属于哪个业务领域。
- [ ] 切片名称是否使用动词短语表达明确意图。
- [ ] 该行为是否与现有切片具有独立输入、结果或失败分支。
- [ ] 若与现有切片不同，是否创建了新的切片目录。

### 6.2 切片内容

- [ ] 是否定义了公共入口。
- [ ] 是否定义了输入、输出和错误契约。
- [ ] 业务流程是否保留在切片内部。
- [ ] 切片专属的数据访问、映射和适配代码是否保留在切片内。
- [ ] 是否避免创建聚合多个行为的 Service 或 Manager。

### 6.3 依赖与通信

- [ ] 是否直接导入了其他切片的内部文件；若有，必须消除。
- [ ] 若需要触发其他行为，是否改为发布事件或消息。
- [ ] 若需要查询其他领域，是否使用公开协议。
- [ ] 新增共享代码是否已有至少两个切片真实复用。
- [ ] 跨领域共享内容是否确实不含业务语义。

### 6.4 测试与装配

- [ ] 是否从切片公共入口覆盖成功路径。
- [ ] 是否覆盖主要失败路径和边界条件。
- [ ] 是否验证状态变化、事件、消息或其他可观察结果。
- [ ] 是否在组合根中显式注册入口和依赖。
- [ ] 是否更新架构依赖检查或模块边界测试。

---

## 7. 修改已有功能 Checklist

- [ ] 是否定位到与该行为对应的唯一切片。
- [ ] 大部分改动是否限制在该切片目录内。
- [ ] 修改是否改变了公共输入、输出、错误或事件契约。
- [ ] 若契约变化，是否评估所有调用方和兼容性。
- [ ] 是否出现需要同时修改多个无关切片的情况。
- [ ] 若多个切片必须同步修改，是否需要调整领域边界或通信契约。
- [ ] 是否新增了对其他切片内部实现的依赖。
- [ ] 是否误将切片私有逻辑提升到 `_shared/` 或全局 `shared/`。
- [ ] 完整行为测试是否仍从公共入口进入。
- [ ] 是否验证新增或改变的可观察结果。
- [ ] 组合根是否仍只负责装配而不包含业务判断。

---

## 8. 提取共享模块 Checklist

在决定提取共享模块前，逐条确认：

- [ ] 是否已有 **2 个及以上**切片实际使用该代码。
- [ ] 复用的是相同业务概念，而不仅是代码形状相似。
- [ ] 提取后是否能给出清晰、稳定、单一职责的名称。
- [ ] 该代码是否为被动定义、纯计算、纯判断或无副作用技术能力。
- [ ] 该代码是否不调用任何切片内部实现。
- [ ] 该代码是否不编排完整业务流程。
- [ ] 该代码是否不产生业务副作用。
- [ ] 是否确认应提升到领域 `_shared/`，而不是全局 `shared/`。
- [ ] 若跨领域复用，是否确认它完全不含领域语义。
- [ ] 是否避免使用 `utils`、`helpers`、`common` 等模糊命名。

### 8.1 停止提取的信号

出现以下任一信号时，应停止提取，改为新建切片、公开协议或重新划分边界：

- 共享逻辑有独立的输入校验和错误分支。
- 共享逻辑会写数据库、发消息、调用外部服务或产生其他副作用。
- 共享逻辑需要知道多个切片的内部状态。
- 调用方必须按不同场景传入大量标志位控制流程。
- 模块名称开始出现 `service`、`manager`、`processor`、`coordinator` 等泛化词。
- 一个 `_shared/` 文件被大量切片引用并持续增长。
- 修改共享模块时需要同时修改多个无关领域。

### 8.2 提取后的验证

- [ ] 原切片测试是否仍全部通过。
- [ ] 共享模块是否具有针对其稳定契约的测试。
- [ ] 共享模块是否保持无副作用或纯技术职责。
- [ ] 是否引入了循环依赖。
- [ ] 是否可以在不理解调用切片内部流程的情况下使用该模块。

---

## 9. 典型代码示例

本节使用 Python 作为示例语言。示例用于说明目录边界和依赖关系，不要求其他语言复制 Python 的文件名、语法或运行模型。

### 示例一：新增“注册用户”切片

#### 目录结构

```text
application/
├── features/
│   └── user/
│       ├── _shared/
│       │   ├── model.py
│       │   └── events.py
│       └── register_user/
│           ├── contracts.py
│           ├── ports.py
│           ├── handler.py
│           ├── entry.py
│           └── test_register_user.py
├── shared/
│   └── runtime/
│       └── identifiers.py
├── composition/
│   └── registry.py
└── main.py
```

#### 领域共享模型

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

#### 切片契约

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

#### 切片私有端口

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

#### 业务处理

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

#### 公共入口适配器

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

`entry.py` 可以被 HTTP、CLI、RPC 或其他技术入口调用。切片内部业务处理不依赖具体框架请求对象。

#### 从公共入口测试完整行为

```python
# features/user/register_user/test_register_user.py
from dataclasses import dataclass

from features.user.register_user.entry import register_user_entry


class InMemoryUsers:
    def __init__(self):
        self.items = {}

    def find_by_email(self, email):
        return next((user for user in self.items.values() if user.email == email), None)

    def save(self, user):
        self.items[user.id] = user


class FakeHasher:
    def hash(self, raw_password):
        return f"hashed:{raw_password}"


class RecordedEvents:
    def __init__(self):
        self.items = []

    def publish(self, event):
        self.items.append(event)


class FixedIdentifiers:
    def new_id(self):
        return "user-001"


@dataclass
class Dependencies:
    users: InMemoryUsers
    password_hasher: FakeHasher
    events: RecordedEvents
    identifiers: FixedIdentifiers


def make_dependencies():
    return Dependencies(
        users=InMemoryUsers(),
        password_hasher=FakeHasher(),
        events=RecordedEvents(),
        identifiers=FixedIdentifiers(),
    )


def test_register_user_from_public_entry():
    dependencies = make_dependencies()

    status, body = register_user_entry(
        {
            "email": "test@example.com",
            "username": "testuser",
            "password": "secure123",
        },
        dependencies,
    )

    assert status == 201
    assert body == {"user_id": "user-001"}
    assert dependencies.users.items["user-001"].email == "test@example.com"
    assert len(dependencies.events.items) == 1


def test_reject_duplicate_email_from_public_entry():
    dependencies = make_dependencies()
    payload = {
        "email": "dup@example.com",
        "username": "first",
        "password": "secret",
    }

    assert register_user_entry(payload, dependencies)[0] == 201
    status, body = register_user_entry(payload | {"username": "second"}, dependencies)

    assert status == 409
    assert body == {"error": "email_already_registered"}
```

#### 在组合根中装配

```python
# composition/registry.py
from dataclasses import dataclass

from shared.runtime.identifiers import UuidGenerator
from shared.security.passwords import ArgonPasswordHasher
from shared.transport.events import InProcessEventBus
from shared.persistence.users import SqlUserStore


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

组合根知道具体实现，切片只知道所需能力的契约。

---

### 示例二：通过领域事件连接两个切片

场景：创建订单后需要发送确认通知。创建订单切片不应直接导入通知切片。

#### 事件定义

```python
# features/order/_shared/events.py
from dataclasses import dataclass


@dataclass(frozen=True)
class OrderCreated:
    order_id: str
    user_id: str
```

#### 发布事件

```python
# features/order/create_order/handler.py
from features.order._shared.events import OrderCreated


def create_order(command, *, orders, events):
    order = orders.create(
        user_id=command.user_id,
        lines=command.lines,
    )
    events.publish(OrderCreated(order_id=order.id, user_id=order.user_id))
    return order.id
```

#### 独立通知切片

```python
# features/notification/send_order_confirmation/handler.py
from features.order._shared.events import OrderCreated


def send_order_confirmation(event: OrderCreated, *, mailer, users) -> None:
    recipient = users.get_email(event.user_id)
    mailer.send(
        recipient=recipient,
        template="order_confirmation",
        context={"order_id": event.order_id},
    )
```

#### 组合根注册订阅关系

```python
# composition/event_registry.py
from features.notification.send_order_confirmation.handler import (
    send_order_confirmation,
)


def register_event_handlers(event_bus, dependencies):
    event_bus.subscribe(
        event_type="order.created",
        handler=lambda event: send_order_confirmation(
            event,
            mailer=dependencies.mailer,
            users=dependencies.user_directory,
        ),
    )
```

创建订单和发送通知各自保持独立。只有组合根知道二者如何连接。

---

### 示例三：领域 `_shared/` 的正确使用

场景：`create_order` 和 `approve_order` 都需要计算订单总额并检查金额上限。

#### 正确做法：提取纯领域规则

```python
# features/order/_shared/rules.py
from decimal import Decimal


class OrderAmountExceededError(Exception):
    pass


def calculate_total(lines) -> Decimal:
    return sum(
        (line.unit_price * line.quantity for line in lines),
        start=Decimal("0"),
    )


def assert_amount_within_limit(total: Decimal, limit: Decimal) -> None:
    if total > limit:
        raise OrderAmountExceededError(
            f"order amount {total} exceeds limit {limit}"
        )
```

切片可以调用这些纯规则：

```python
# features/order/approve_order/handler.py
from features.order._shared.rules import (
    assert_amount_within_limit,
    calculate_total,
)


def approve_order(command, *, orders, approval_limit):
    order = orders.get(command.order_id)
    total = calculate_total(order.lines)
    assert_amount_within_limit(total, approval_limit)
    order.approve()
    orders.save(order)
```

#### 错误做法：在 `_shared/` 编排多个用例

```python
# features/order/_shared/order_processing_service.py  # ❌
class OrderProcessingService:
    def create(self, command): ...
    def approve(self, command): ...
    def cancel(self, command): ...
    def notify_customer(self, order): ...
```

该类包含多个独立行为和副作用，应拆回对应切片。

---

## 10. 反模式识别

### 反模式一：伪装成共享模块的服务类

```text
features/order/_shared/
└── order_processing_service.py  ❌
```

识别信号：

- 名称含 `service`、`manager`、`processor`、`coordinator`。
- 暴露多个与不同用户意图对应的方法。
- 被多个切片调用并持续增长。
- 内部同时读写存储、发事件和调用外部系统。

解决方案：将每个独立行为拆为切片，只把稳定纯规则或契约留在 `_shared/`。

---

### 反模式二：切片目录只使用名词

```text
features/order/
├── order/          ❌
└── order_data/     ❌
```

名词只说明数据对象，不能说明行为边界。

正确命名：

```text
features/order/
├── create_order/
├── cancel_order/
└── get_order_detail/
```

---

### 反模式三：只测试内部函数

```python
# ❌ 只测试私有校验，未验证完整行为
def test_validate_quantity():
    assert validate_quantity(-1) is False
```

问题：无法发现入口映射、依赖装配、状态写入、事件发布或错误转换中的问题。

正确做法：从切片公共入口进入，验证输入到可观察结果的完整路径。

---

### 反模式四：一个处理器聚合多个用例

```python
# features/order/manage_order/handler.py  # ❌
class OrderHandler:
    def create_order(self, command): ...
    def cancel_order(self, command): ...
    def approve_order(self, command): ...
    def refund_order(self, command): ...
```

解决方案：拆为 `create_order`、`cancel_order`、`approve_order`、`refund_order` 四个切片。

---

### 反模式五：跨切片直接导入

```python
# features/order/approve_order/handler.py  # ❌
from features.order.create_order.handler import validate_customer
```

解决方案：

- 纯且稳定的同领域规则提升到 `_shared/`；
- 副作用通过事件触发；
- 跨领域查询使用公开协议；
- 高度同步的行为重新评估切片边界。

---

### 反模式六：伪装成 adapter 的跨领域耦合

```python
# features/campaign/_shared/device_adapter.py  # ❌
from features.device.register_device.handler import register_device

__all__ = ["register_device"]
```

识别信号：

- 文件只做导入、包装和重新导出。
- 没有定义独立、稳定的公开协议。
- 目的是让跨领域内部调用看起来“合法”。

解决方案：删除转发层，改为事件、公开 API 或重新划分领域边界。

---

### 反模式七：组合根包含业务逻辑

```python
# composition/registry.py  # ❌
def build_application(config):
    if config.customer_level == "vip":
        approval_limit = 100000
    else:
        approval_limit = 10000
```

客户等级对应审批额度属于业务规则，不应位于组合根。组合根只能选择和连接实现；业务决策应进入相应切片或领域规则。

---

### 反模式八：全局共享目录业务化

```text
shared/
├── order_validator.py     ❌
├── payment_manager.py     ❌
└── user_repository.py     ❌
```

这些模块包含明确领域语义，应归入对应领域或功能切片。

---

## 11. 边界情况决策指南

### Q1：两个不同领域需要相似校验，是否应该共享？

先判断校验是否含业务语义：

- **纯格式或技术校验**，例如 UUID、邮箱语法、编码格式：可以进入全局技术共享。
- **领域规则**，例如订单最低金额、账户提现额度：各领域独立实现，除非它们确实属于同一个更高层领域概念。
- **需要读取状态或产生副作用的校验**：不是简单共享函数，应建模为切片或公开能力。

不要因为两段代码当前相似就创建公共模块。

---

### Q2：多个切片都需要发送通知，如何处理？

建立独立通知切片，由其他切片发布领域事件触发：

```python
# 发布方

events.publish(OrderCreated(order_id=order.id, user_id=order.user_id))
```

```python
# 通知切片

def send_order_confirmation(event, *, mailer, users):
    recipient = users.get_email(event.user_id)
    mailer.send(recipient=recipient, template="order_confirmation")
```

通用邮件客户端可以位于 `shared/transport/`，但“发送订单确认邮件”仍是具有业务语义的独立切片。

---

### Q3：一个切片越来越复杂，应该如何拆分？

常见拆分信号：

- 出现多个完全不同的输入或触发入口。
- 存在多个可独立失败或重试的子流程。
- 处理器中大量条件分支对应不同用户意图。
- 一个切片同时产生多种不相关结果。
- 测试开始分成互不相关的场景组。
- 修改某一分支时经常影响其他分支。

拆分时按用户意图或业务结果划分，而不是简单按文件数量划分。文件超过 6～8 个只是审查信号，不是机械规则。

---

### Q4：入口声明和注册应该放在哪里？

切片目录负责声明自己的公共入口；组合根负责统一注册和装配。

Python 示例：

```python
# features/order/create_order/entry.py

def create_order_entry(payload, dependencies):
    ...
```

```python
# composition/command_registry.py
from features.order.create_order.entry import create_order_entry
from features.user.register_user.entry import register_user_entry


def build_command_registry(dependencies):
    return {
        "order.create": lambda payload: create_order_entry(payload, dependencies),
        "user.register": lambda payload: register_user_entry(payload, dependencies),
    }
```

入口实现留在切片内，统一注册留在组合根中。

---

### Q5：`_shared/` 文件被 5 个以上切片引用，正常吗？

可能正常，但必须审查。高引用数通常意味着以下情况之一：

1. 它是稳定且简单的领域模型、值对象、错误或纯规则。
2. 它已经聚合过多职责，正在演变为隐形 Service 层。
3. 当前领域边界过大，需要进一步拆分。
4. 多个切片实际上属于同一更高层行为。

处理步骤：

- 检查是否有副作用和业务流程编排。
- 检查是否存在大量可选参数或场景标志。
- 将主动行为剥离为独立切片。
- 仅保留稳定模型、契约、事件和纯规则。

---

### Q6：一个切片需要读取另一个领域的数据，怎么办？

优先通过提供方的公开查询协议，而不是导入其模型或仓储。

```python
class CustomerStatusClient:
    def get_status(self, customer_id: str) -> dict:
        return self._transport.get(f"/customers/{customer_id}/status")
```

调用方只依赖公开结果。若读取发生频繁且强一致要求很高，应重新评估：

- 两个概念是否属于同一领域；
- 是否需要本地投影或事件同步；
- 是否应合并到同一交付单元；
- 当前服务边界是否划分错误。

---

### Q7：跨交付单元能否共享业务代码库？

默认不能。跨交付单元应共享公开协议，而不是内部业务实现。

允许共享：

- 序列化协议和公开 Schema；
- 无业务语义的技术库；
- 生成客户端所需的接口描述。

不允许共享：

- 某个服务的领域模型实现；
- 某个切片的处理器；
- 内部仓储或事务逻辑；
- 为避免网络调用而复制出的业务 Service。

---

### Q8：多个切片使用相同的数据访问代码，是否应创建 Repository？

仅在数据访问表达稳定领域查询且被同领域多个切片复用时，才可放入领域 `_shared/`。它必须保持被动，不编排业务流程。

```python
# features/order/_shared/queries.py

def find_open_orders_by_customer(connection, customer_id: str):
    return connection.fetch_all(
        "SELECT * FROM orders WHERE customer_id = ? AND status = 'open'",
        [customer_id],
    )
```

若查询只服务一个切片，应保留在该切片内。若模块同时包含创建、审批、取消等业务流程，则不是查询工具，应拆回切片。

---

### Q9：纯领域函数产生异常，是否仍可放 `_shared/`？

可以。抛出领域错误不等于产生副作用。纯函数可以根据输入返回结果或抛出稳定领域异常，只要它：

- 不读写外部状态；
- 不调用其他切片；
- 不发送消息或访问网络；
- 不编排完整用例。

---

### Q10：语言或框架要求固定目录时怎么办？

保留框架要求的最薄适配层，将业务切片边界放在其后。

例如框架要求所有入口位于统一目录时：

```text
framework_routes/              # 框架强制目录，只做转发
└── create_order.py

features/order/create_order/   # 实际切片
├── entry.py
├── handler.py
└── test_create_order.py
```

适配层不得实现业务规则，只负责把框架输入转换为切片契约，并调用切片公共入口。

---

## 参考资料

- [Vertical Slice Architecture — Jimmy Bogard](https://jimmybogard.com/vertical-slice-architecture/)
- [Organizing Code by Feature](https://khalilstemmler.com/articles/software-design-architecture/feature-driven/)
