# ADR-004: 双执行引擎共存设计 (Dual Execution Engines)

- **状态**：已接受 (Accepted)
- **日期**：2026-10-03
- **相关文档**：[Dual Execution Engines Design](../design/dual-engine.md) · [Runtime Flow](../architecture/runtime-flow.md)

---

## 1. 背景与问题 (Context)

在构建复杂智能代理应用时，存在两种截然不同的执行诉求：
1. **探索式非确定性场景**：用户提出开放性问题，步骤无法预先知晓，完全依赖 LLM 与 Planner 在单步执行后根据工具反馈动态生成下一步动作（即经典的 ReAct 模式）。
2. **确定性业务流与人机协作场景**：企业级任务往往包含明确的阶段节点、条件分支（如根据校验结果进入正向流或回退流）、多子节点并发执行、以及关键节点前挂起等待人工干预（Human-in-the-Loop）。若强行使用单 Agent 循环编写，会导致 Prompt 极度冗长脆弱，且无法对控制流做出硬性保证。

---

## 2. 架构决策 (Decision)

我们决定在 AgentCore 中并列提供两个一等公民的执行引擎：

1. **`RuntimeEngine`（ReAct 状态机循环引擎）**：
   * 采用显式 9 状态跃迁机，围绕 `Planning` $\longleftrightarrow$ `Executing` $\longleftrightarrow$ `WaitingTool` 运转。
   * 内建多层超时、指数退避重试、以及敏感工具阻断回调。
2. **`WorkflowEngine`（有向图工作流编排引擎）**：
   * 采用节点与有向条件边构成的 `WorkflowGraph`。
   * 支持多分支并发（`ParallelNode`）与多工具并发（`ParallelToolNode`）。
   * 支持任意节点前置/后置断点挂起（`interrupt`）、上下文注入恢复（`resume`）以及全生命周期时序回放轨迹（`replay`）。
3. **顶层统一组合**：
   * CLI 与外部客户端可通过统一参数 `--engine react|workflow` 按需调度底层引擎。

---

## 3. 产生的影响 (Consequences)

### 正面收益
* **兼顾灵活性与确定性**：开发者可根据实际场景自由选择自主规划（ReAct）或硬编码业务流（Workflow Graph）。
* **可审计与企业级落地**：工作流引擎提供的 `replay()` 审计机制为严肃商业应用提供了完备的过程追溯能力。

### 代价与妥协
* 需同时维护两套引擎的核心单元测试，增加了维护成本。
* 开发者在初期需要理解何时选择 ReAct、何时选择 Workflow Graph。
