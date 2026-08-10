# 功能边界工作规范

本文件用于单交付单元项目的结构设计，以及功能切片的新增、修改、移动、拆分和审查。名称与示例只表达职责；实际语法、文件扩展名和源码集布局服从项目工具链。多交付单元协作读取 [`cross-boundary-communication.md`](cross-boundary-communication.md)。

## 执行流程

1. **定位坐标：** 找出仓库根、源码根、交付单元、目标领域、现有切片、公共入口、组合关系和相关测试；每个受影响构件都已归入唯一职责位置时完成。
2. **划分意图：** 写明行为的触发、输入、结果和主要失败分支。能够独立变化的行为建立独立切片；切片名称能唯一表达该意图时完成。
3. **确定契约：** 枚举入口、输入、输出、错误、事件及调用方兼容性；公开契约不暴露框架请求对象、内部持久化模型或私有目录时完成。
4. **贯通路径：** 把校验、业务决策、状态访问、映射、展示和专属适配保留在切片内；从入口到可观察结果的完整路径存在，且没有跨切片内部依赖时完成。
5. **处理分支：** 测试归属、共享提取、边界协作或平台约束出现时，分别加载 [`test-placement.md`](test-placement.md)、[`shared-code.md`](shared-code.md)、[`cross-boundary-communication.md`](cross-boundary-communication.md) 或 [`exceptions.md`](exceptions.md)；所有命中分支均按对应规范解决时完成。
6. **显式装配：** 在组合根注册入口并绑定具体依赖；注册无需隐式导入副作用，组合根只包含发现、连接和启动逻辑时完成。
7. **验证行为与边界：** 从公共入口覆盖成功、主要失败、边界条件和全部改变的可观察结果，并运行适用的架构检查、静态检查及测试；检查全部通过，或缺失自动化检查时已逐条记录人工边界结论，方可完成。

修改已有切片时，先记录契约与调用方的当前状态，再执行以上流程。若改动必须同时进入多个无关切片，先检查协作契约或领域边界；切片出现多意图信号时读取 [`edge-cases.md`](edge-cases.md)。

## 目标蓝图

```text
<repository>/
├── AGENTS.md
├── README.*
├── <toolchain-files>
├── docs/architecture/
│   └── decisions/                    # 有架构决定时出现
├── <source-root>/
│   ├── features/
│   │   └── <domain>/
│   │       ├── _shared/              # 有真实领域共享时出现
│   │       └── <verb-object>/        # 一个完整行为
│   ├── shared/                       # 有跨领域纯技术复用时出现
│   ├── composition/                  # 也可映射为入口附近的文件
│   └── <application-entry>.*
├── <automation>/                     # 按需
├── <deployment>/                     # 按需
└── <ci-config>/                      # 按需
```

蓝图规定职责映射，不是目录初始化清单。只有出现真实内容时才建立按需区域；一个职责可以由单个文件承担，名称和位置可以按语言生态调整。

## 全局职责

- `<toolchain-files>` 是依赖、构建、格式、静态检查和测试命令的环境事实来源。
- `docs/architecture/` 保存架构规范和决策。
- `<source-root>/features/` 保存全部业务语义，先按领域、再按用户意图组织。
- `<source-root>/shared/` 保存无业务语义的技术基础设施；提升条件见 [`shared-code.md`](shared-code.md)。
- `<source-root>/composition/` 及 `<application-entry>.*` 只负责发现、注册、装配和启动。
- `<automation>`、`<deployment>` 和 `<ci-config>` 分别容纳自动化、部署与持续集成资产，位置服从工具链。

## 切片边界

领域以共同业务语言划分，不以数据库表或框架模块命名。切片表达一个用户意图、业务命令、查询或外部事件处理目标，并同时具备：

- 可识别的公共入口；
- 完成行为所需的私有实现；
- 返回值、状态、事件、消息、文件或界面变化等可观察结果；
- 无需读取其他切片内部代码即可理解和测试的边界。

```text
features/<domain>/
├── _shared/                         # 提升规则见 shared-code.md
├── <verb-object>/
│   ├── <entry>.*                    # HTTP、RPC、CLI、UI、消息或任务入口
│   ├── <contracts>.*                # 输入、输出和错误契约
│   └── <implementation>.*           # 流程、状态变化和私有适配
└── <another-verb-object>/
```

目录表达边界，不规定固定文件清单。简单切片可以只有一个实现文件；复杂切片仍在自身目录内按职责拆分。

## 命名与放置

- 领域使用业务名词；单复数、大小写和分隔符在同一作用域保持一致。
- 切片使用“动词 + 对象或结果”，例如 `create_order`、`approve_order`、`get_order_detail`。
- 切片专属契约、流程、数据访问、映射、展示、资源和适配均与切片共置。
- 所有业务代码按领域和意图归属切片；`controllers/`、`services/`、`repositories/`、`components/` 或 `stores/` 仅在工具强制时作为薄适配层。受强制目录约束时读取 [`exceptions.md`](exceptions.md)。
- 聚合多个意图的 `Service`、`Manager`、`Processor` 或 `Coordinator` 按用户意图拆成切片。
- 跨领域且无业务语义的手写资源归入相应技术共享区域；生成物和部署资源遵循工具链，但不承载隐藏业务规则。

## 完成标准

只有执行流程全部完成、每个受影响路径都能唯一映射到上述职责、单个行为的大部分变化位于一个切片内，且框架或部署变化不改变业务边界时，功能边界工作才完成。
