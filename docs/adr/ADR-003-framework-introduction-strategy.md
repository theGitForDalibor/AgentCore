# ADR-003: 外部第三方框架引入策略 (Framework Strategy)

- **状态**：已接受 (Accepted)
- **日期**：2026-08-31（更新于 2026-10-03）
- **相关文档**：[ADR-001](./ADR-001-runtime-self-build.md)

---

## 1. 背景与问题 (Context)

尽管 AgentCore 确立了自研运行时的核心方向，但在工业界与开源社区中，诸如 LangGraph、LlamaIndex 等框架仍在不断输出特定领域的设计范式。若完全自我封闭，可能丧失与业界标准方案的对照参考机会。

---

## 2. 架构决策 (Decision)

明确界定第三方框架与协议的引入边界：

1. **核心内核严禁依赖重型 Agent 框架**：
   * `main` 分支的核心模块（`runtime/`, `planner/`, `tools/`, `memory/`, `workflow/`）绝对不直接将 LangChain、LangGraph 或 AutoGen 作为依赖引入。
2. **开放接纳标准化外部交互协议**：
   * 支持并积极集成标准化协议（如 Anthropic 主导的 **Model Context Protocol (MCP)**）。通过官方协议桥接外部工具生态，而非依赖专有 SDK。
3. **隔离实验与对比评估**：
   * 仅允许在独立的探索分支或 `examples/` 目录下引入框架进行横向基准对照与设计对比，评估自研引擎与商业框架的性能及灵活性差异。

---

## 3. 产生的影响 (Consequences)

### 正面收益
* 保证了 AgentCore 核心内核的纯粹性与极小依赖面，避免了由于外部框架深层依赖冲突导致的破坏性升级。
* 拥抱 MCP 标准协议，使得 AgentCore 原生具备了与现代 AI 生态（如 Claude Desktop、Cursor MCP servers）互联互通的能力。
