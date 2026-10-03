# ADR-002: 七层分层解耦架构 (Layered Architecture)

- **状态**：已接受 (Accepted)
- **日期**：2026-08-31（更新于 2026-10-03）
- **相关文档**：[System Architecture Overview](../architecture/overview.md)

---

## 1. 背景与问题 (Context)

Agent 系统是一个横跨大模型交互、多轮记忆、工具反射、工作流编排以及多种通信接口的复杂综合体。
如果缺乏严格的水平分层约束，极其容易导致：
* Runtime 内部渗透进特定工具的业务逻辑。
* 外部客户端（如 CLI 或 Web API）直接操作底层 LLM 请求对象，导致接口耦合。
* 无法对单点模块（如 Planner 或 MemoryStore）进行独立的单元测试与 Mock 替换。

---

## 2. 架构决策 (Decision)

我们决定采用**严格的七层分层架构**：

$$\text{Application} \longrightarrow \text{Runtime} \longrightarrow \text{Planning} \longrightarrow \text{Execution} \longrightarrow \text{Tools} \longrightarrow \text{LLM Adapter} \longrightarrow \text{Infrastructure}$$

### 核心约束原则
1. **单向依赖规则**：上层可以依赖下层，下层严禁反向感知上层。
2. **面向 Protocol 交互**：跨层契约（如 `Planner`、`Tool`、`MemoryStore`、`WorkflowNode`）全部通过 Python `typing.Protocol` 抽象定义，屏蔽具体实现细节。
3. **不可变数据载体**：跨层传递的数据全部封装为 `runtime.models` 下的只读不可变模型（`frozen=True`）。

---

## 3. 产生的影响 (Consequences)

### 正面收益
* **独立可测试性**：每一层均可挂载 Mock 驱动（如 `StubLLM`、`InMemoryMemoryStore`、`InMemoryToolRegistry`），测试无需网络 I/O 即可毫秒级运行。
* **清晰的代码演进路径**：功能扩展界限分明，如接入外部工具只需通过 `MCPToolAdapter` 挂载至 Tools 层，无需修改 Runtime。

### 代价与妥协
* 初始化时代码层级较多，简单的功能可能需要跨越两到三个层级的接口转换。
