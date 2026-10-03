# Engineering Standards & Guidelines

> 本文档规范 AgentCore 的工程开发规范、代码风格、类型安全、Git 提交约定与质量门禁。

---

## 1. 代码质量与格式化规范

项目通过 `pyproject.toml` 统一配置了静态检查与代码格式化工具链：

### 1.1 Ruff 静态检查
* **规则集**：激活 `E`（PEP8 语法）、`F`（Pyflakes）、`I`（isort 导入排序）、`N`（命名规范）、`W`（警告）、`UP`（语法现代化升级）。
* **行宽限制**：88 字符。
* **执行命令**：
  ```bash
  uv run ruff check .
  # 自动安全修复
  uv run ruff check --fix .
  ```

### 1.2 Black 格式化
* **执行命令**：
  ```bash
  uv run black .
  # 检查是否符合格式
  uv run black --check .
  ```

### 1.3 Pre-commit 本地自动化门禁
在提交 Git 之前自动触发 Ruff、Black 与 Pytest 检查：
```bash
uv run pre-commit install
# 手动触发全量检查
uv run pre-commit run --all-files
```

---

## 2. 类型安全规范 (Type Safety)

1. **不可变核心领域对象**：
   * 所有在状态机与会话间传递的事件类模型（如 `Message`、`ToolCall`、`Observation`、`MemoryRecord`），必须显式配置 `model_config = ConfigDict(frozen=True)`。
   * 状态变化通过 Pydantic 的 `.model_copy(update={...})` 进行防御性拷贝，严禁原地直接修改属性。
2. **面向协议编程 (Protocol-first)**：
   * 跨层通信核心契约必须使用 `typing.Protocol` 并标注 `@runtime_checkable`（如 `Planner`、`Tool`、`MemoryStore`、`WorkflowNode`、`LLMAdapter`）。
   * 实现类无需继承具体父类，只需满足鸭子类型契约，确保纯粹的解耦与极简的 Mock 测试。

---

## 3. Git 提交规范 (Conventional Commits)

所有 Commit Message 严格遵守 [Conventional Commits](https://www.conventionalcommits.org/) 规范：

```text
<type>(<scope>): <short description>
```

### 常见 Type
* `feat`: 新增特性或能力模块
* `fix`: 缺陷修复
* `docs`: 文档变更
* `refactor`: 重构（在不改变外部契约的前提下调整内部结构）
* `test`: 补充或修正测试用例
* `chore`: 依赖调整、工程配置修改

### 核心 Scope 清单
* `runtime`: `runtime/` 模块（状态机、引擎、循环）
* `workflow`: `workflow/` 模块（图、节点、边、中断）
* `planner`: `planner/` 模块（规划器）
* `tools`: `tools/` 模块（工具契约、注册中心、执行器）
* `memory`: `memory/` 模块（记忆存储、上下文构建、持久化）
* `mcp`: `integrations/mcp` 模块（MCP 客户端与适配器）
* `cli`: `integrations/cli` 模块（命令行工具）
* `infra`: `infra/` 模块（配置、遥测监控）
