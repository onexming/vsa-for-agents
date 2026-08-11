> [!important] 成熟项目、已经在生产环境中运行的项目，去除 AGENTS.md 中: "_破坏性更新...、兜底(Fallback) 代码_"

# Agent 友好的架构: 垂直切片架构（VSA）

垂直切片架构（Vertical Slice Architecture），按特性组织代码，每个切片自成一体，AI 友好的架构模式，最大化了上下文隔离。AI 工具可以理解并修改一个自包含的特性，而无需了解整个代码库

## 约束(提高AI代码质量的下限)

以严格的**强制约束**限制AI写出符合VSA规范的代码，确保代码质量。

### 代码质量约束

| 约束类型   | 约束目标            | Python        | TypeScript                 |
| ------ | --------------- | ------------- | -------------------------- |
| 格式化    | 自动统一排版与格式       | Ruff format   | Prettier                   |
| 静态质量   | 代码风格、常见错误与团队规则  | Ruff          | ESLint + typescript-eslint |
| 类型检查   | 静态类型与接口一致性      | Pyright       | `tsc --noEmit`             |
| 架构边界   | 模块分层、依赖方向与循环依赖  | Import Linter | dependency-cruiser         |
| 代码安全   | 危险用法与可扩展安全规则    | Bandit        | 暂不选                        |
| 密钥治理   | 密码、令牌与 API 密钥泄露 | Gitleaks      | Gitleaks                   |
| 依赖声明治理 | 漏声明、未使用与误用依赖    | deptry        | Knip                       |
| 本地提交入口 | 提交前自动运行快速检查     | pre-commit    | Husky + lint-staged        |

### 功能交互约束

| 约束类型 | 约束目标 | Python | TypeScript |
|---|---|---|---|
| 功能测试 | 功能正确性与回归测试 | pytest | Vitest |
| 覆盖率 | 行与分支覆盖率采集 | pytest-cov | `@vitest/coverage-v8` |
| 新增代码覆盖率 | Git 改动行是否被测试覆盖 | diff-cover | diff-cover |

# SKILL 推荐

- https://github.com/mattpocock/skills.git
    - /grill-me 帮助思考、整理思路、为AI提供边界
    - /wait-what 让AI说人话，帮助自己理解AI的回复
    - /improve-codebase-architecture 帮助提升代码质量、可维护性
- https://github.com/markdown-viewer/skills.git 生成漂亮的图表

# Links

- https://github.com/AmazingAng/old-coder.git
- https://x.com/unclebobmartin/status/2080257779395154409
- `docs/ARCHITECTURE.md` 为 `AGENTS.md` 事实来源
