# 编码

- **破坏性更新**：_当前项目并未开发完成，未正式上线_
  - 禁止出现 “变更记录、修订说明、已替代/废弃”等历史痕迹
  - 禁止编写 向后兼容(Backward Compatibility) 、迁移(Migration) 、兜底(Fallback) 代码
- **禁止重复造轮子**：优先使用成熟的、有人维护的工具/库/包
- **禁止软约束(文字描述)**：使用强约束，例如Python必须使用：Ruff、Pyright、**ImportLinter**、Bandit、Gitleaks、deptry、pytest、pytest-cov、diff-cover，**其他语言必须使用对应的工具**
- **快速失败(Fail Fast) 原则**：禁止编写静默失败的代码，错误应该在离发生源最近的地方被发现和处理，而不是静默传播，掩盖问题
- **可测试性优先**：依赖显式注入，核心逻辑保持纯粹，副作用集中在边界，状态可控且结果可观察，减少测试时的Mock

## 测试用例

- 禁止编写依赖内部实现细节的测试用例，允许"用白盒手段验证真实行为"

# 问题&排查修复

- **禁止Band-Aid**: 排查问题，修改代码前，先回答：这是否是根因? 如果不是，根因是什么?
- 报告验证通过时的标准：真实环境(依赖数据库、测试设备、Playwright等)验证通过是，才能报告为“通过”

# 架构行动指南

[`docs/ARCHITECTURE.md`](docs/ARCHITECTURE.md) 定义本仓库的架构目标、边界与强制规则；`docs/architecture/` 将其展开为按任务加载的执行规范。本文件只负责路由、执行闭环和授权闸门。

## 任务路由

在勘察或修改前，列出任务命中的全部分支并完整读完相应文件；同时命中多个分支时全部加载。

- **架构定义：** 新增、修改或审查架构规则，或者专题规范无法覆盖当前判断时，读取 [`docs/ARCHITECTURE.md`](docs/ARCHITECTURE.md)。
- **功能边界：** 设计项目结构，或改动源码根、全局职责、领域、切片、命名及公共入口时，读取 [`docs/architecture/feature-work.md`](docs/architecture/feature-work.md)。
- **测试归属：** 改动测试、测试环境、搭建、种子或清理文件时，读取 [`docs/architecture/test-placement.md`](docs/architecture/test-placement.md)。
- **共享提取：** 改动 `_shared/`、`shared/`、Repository、公共模型、规则或工具时，读取 [`docs/architecture/shared-code.md`](docs/architecture/shared-code.md)。
- **边界协作：** 改动切片间调用、跨领域读取、事件、消息、API、RPC 或跨交付单元协作时，读取 [`docs/architecture/cross-boundary-communication.md`](docs/architecture/cross-boundary-communication.md)。
- **架构例外：** 受框架固定目录、生成代码或遗留兼容约束，或者计划偏离架构规则时，读取 [`docs/architecture/exceptions.md`](docs/architecture/exceptions.md) 及相关 ADR。
- **复合边界：** 判断跨领域相似校验、持续膨胀的切片或高扇入共享模块时，读取 [`docs/architecture/edge-cases.md`](docs/architecture/edge-cases.md)。
- **实现范例：** 创建新切片且仓库内没有可靠范例时，读取 [`docs/architecture/example-vertical-slice.md`](docs/architecture/example-vertical-slice.md)。

## 执行闭环

1. **分类：** 列出命中的路由分支及适用规则；所有目标文件均已读完且没有遗漏受影响分支时完成。
2. **勘察：** 定位交付单元、领域、切片、公共入口、公开契约、直接依赖方、组合根和相关测试；最小依赖闭包已完整列出时完成。
3. **实现：** 在闭包内完成行为，并修复本次变更触发的违规依赖；目标行为存在且闭包符合已加载规则时完成。
4. **验证：** 对行为变更，从公共入口验证成功路径、主要失败路径和可观察结果；运行适用的依赖检查、静态检查及测试。已执行检查全部通过，无法执行的检查及原因已记录时完成。
5. **交付：** 说明读取的规范、修改的边界、修复的架构冲突、验证结果及未执行检查；上述信息齐全时完成。

**最小依赖闭包**包括目标切片、为消除违规依赖所必需的直接调用方或提供方、相关公开契约、组合根、测试和边界检查。闭包合规且验证通过后停止扩展，不把未触发的遗留问题扩大为全仓库迁移。

## 授权闸门

- **直接执行：** 用户已要求实现时，可直接完成仓库内可验证、可回退且位于最小依赖闭包内的实现与架构重构。
- **先行确认：** 破坏公开 API、迁移持久化数据、改变外部系统或新增生产依赖。
- **例外处理：** 按 [`docs/architecture/exceptions.md`](docs/architecture/exceptions.md) 沿用有效 ADR；建立新例外前取得用户确认，并记录迁移目标和移除条件。

只有执行闭环全部完成、授权闸门得到满足且任何例外均有有效 ADR，才能宣布任务完成。
