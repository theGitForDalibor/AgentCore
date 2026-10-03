# Dual Execution Engines Design

> 本文档深入剖析 AgentCore 的核心特色之一：**双执行引擎体系**。探讨 `RuntimeEngine`（ReAct 循环）与 `WorkflowEngine`（有向图工作流）的选型定位、内部设计与协同机制。

---

## 1. 为什么设计双执行引擎？

在现代 Agent 落地实践中，单一模式无法兼顾灵活性与确定性：
1. **自主探索场景（Open-ended / Exploratory）**：用户问题未知，步骤无法预先固化，需要依靠大模型自主思考、调用工具、观察反馈并持续调整策略。这是 **ReAct 引擎**的用武之地。
2. **确定性业务管线（Deterministic Pipeline & Human-in-the-Loop）**：金融风控、订单履约、审批发布等场景具有严格的业务前置条件、分支路由、断点人工介入或并发执行要求。这是 **Workflow 引擎**的核心价值。

AgentCore 原生同时提供了这两种引擎，并在 CLI 入口通过 `--engine react|workflow` 提供无缝切换。

---

## 2. 双引擎对比矩阵

| 特性维度 | `RuntimeEngine` (ReAct) | `WorkflowEngine` (Graph) |
| :--- | :--- | :--- |
| **底层抽象** | 隐式决策循环 + 9 状态跃迁机 (`RuntimePhase`) | 显式有向图 (`WorkflowGraph`) + 节点/边 (`Node`/`Edge`) |
| **控制流主导权** | LLM / Planner 自主决定下一步 | 开发者定义边条件 (`Edge.condition`) |
| **典型拓扑** | 动态星型循环：`Planning` $\leftrightarrow$ `Tool` | DAG 或受控循环图（由 `max_steps` 防死锁） |
| **并发支持** | 串行多轮调用 | 支持 `ParallelNode` 与 `ParallelToolNode` 线程池并发 |
| **断点暂停** | 基于回调的主动审批 (`approval_callback`) | 基于节点的前置/后置断点 (`interrupt_before/after`) |
| **上下文格式** | 强类型不可变 `State` 领域对象 | 灵活的 `dict[str, Any]` 上下文字典 |
| **审计与回放** | 检查 `state.tool_calls` 与 `observations` | 完整的时序回放轨迹 `WorkflowStepRecord` 列表 |
| **代码文件** | `runtime/engine.py` | `workflow/engine.py` |

---

## 3. ReAct 引擎设计 (`RuntimeEngine`)

ReAct 引擎以单轮对话 `Message` 或既有 `State` 为输入，驱动受控的 ReAct 循环：

* **核心参数**：
  * `max_steps`（默认 8）：单次 Run 最大规划步数，防失控刷 Token。
  * `max_retries`（默认 3）：Planner / Tool 偶发失败时的重试上限。
  * `retry_backoff_seconds`（默认 0.01s）：指数退避系数。
  * `run_timeout`（300s）、`tool_timeout`（30s）、`llm_timeout`（60s）：多层超时保护。
* **审批门控**：
  * `require_approval_for: frozenset[str]`：指定必须获得人工确认的工具集合。
  * 命中后转移至 `RuntimePhase.WAITING_APPROVAL`，调用 `approval_callback`；若被拒绝，向会话注入拒绝事实并平滑引导 LLM 重新规划。

---

## 4. 有向图工作流引擎设计 (`WorkflowEngine`)

### 4.1 图结构与条件边 (`WorkflowGraph`)

```python
graph = WorkflowGraph()
graph.add_node(FunctionNode(node_id="validate", func=validate_fn))
graph.add_node(FunctionNode(node_id="process", func=process_fn))
graph.add_node(FunctionNode(node_id="fallback", func=fallback_fn))

graph.set_entry_point("validate")
graph.add_edge("validate", "process", condition=lambda ctx: ctx.get("is_valid") is True)
graph.add_edge("validate", "fallback", condition=lambda ctx: ctx.get("is_valid") is False)
```

* **严格校验**：运行前调用 `graph.validate()`，确保 entry point 存在且所有 edge 源与目标节点合法。
* **循环保护**：执行循环记录 `step_count`，超出 `max_steps`（默认 1000）即抛出 `WorkflowValidationError`。

### 4.2 并发执行与合并策略 (`workflow.parallel`)

当需要并发运行多个独立子流程或同时调用多个外部工具时，使用并发节点：

1. **`ParallelNode`**：
   * 将子节点调度到 `ThreadPoolExecutor` 中并发运行。
   * **隔离执行**：每个子节点拷贝独立上下文，防变量污染。
   * **可定制合并策略 (`MergeStrategy`)**：
     * `default_merge_strategy`：按确定性节点顺序浅合并所有字典。
     * `namespaced_merge_strategy(namespace_key="branches")`：将各分支结果分别保存在命名空间字典下。
2. **`ParallelToolNode`**：
   * 专用于并发执行一组 `ToolCall`，无竞态安全追加至上下文 `observations` 列表。

### 4.3 状态暂停、恢复与时序回放 (Interrupt, Resume & Replay)

`WorkflowEngine` 内建全状态快照 `WorkflowSnapshot`：

* **断点挂起**：
  * `engine.run(graph, input, interrupt_before={"sensitive_node"})`：在进入敏感节点前直接将状态挂起为 `WorkflowStatus.INTERRUPTED` 并安全返回。
  * 节点执行中亦可主动返回 `context["__interrupt__"] = True` 触发后置断点。
* **恢复流转**：
  * `engine.resume(run_id, input={"approved": True})`：从上次暂停节点无缝继续执行。
* **全量回放**：
  * `engine.replay(run_id) -> list[WorkflowStepRecord]`：提供步骤索引、执行节点 ID、执行前 Context、执行后 Context 与精确时间戳，专为合规审计与调试设计。
