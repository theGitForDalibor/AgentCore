# Developer Getting Started Guide

> 本文档为初次接触 AgentCore 的开发者提供环境搭建、依赖同步、测试运行与 CLI 使用指南。

---

## 1. 前置环境要求

* **操作系统**：Linux / macOS / Windows
* **Python 版本**：Python $\ge$ 3.12（推荐使用 3.12 或更高版本）
* **包管理工具**：推荐使用极速包管理工具 [uv](https://github.com/astral-sh/uv)

---

## 2. 快速安装与依赖同步

克隆仓库后，在项目根目录下同步环境依赖：

```bash
# 同步主依赖与 dev 依赖环境（包含 pytest、ruff、black、pre-commit）
uv sync --extra dev
```

> [!TIP]
> `uv sync --extra dev` 会在项目根目录创建 `.venv` 虚拟环境，并安装全量开发依赖。
> 若未安装 `uv`，亦可使用传统方式：`python -m venv .venv && source .venv/bin/activate && pip install -e ".[dev]"`。

---

## 3. 运行测试套件

AgentCore 拥有完备的自动化测试体系，当前包含 142 个用例，涵盖模型、状态机、执行循环、并发、持久化、MCP 协议与配置：

```bash
# 运行全部单元测试与集成测试
uv run pytest

# 安静输出模式
uv run pytest -q

# 运行特定模块测试（如工作流引擎）
uv run pytest tests/workflow/
```

---

## 4. CLI 命令行使用体验

项目内置了功能丰富的命令行工具（`jarvis` 命令入口映射自 `integrations.cli:app`）：

### 4.1 离线体验（无需 API Key）

无需连接大模型即可验证状态机与工具链闭环：

```bash
# 基础问答
uv run jarvis chat --demo "你好，AgentCore"

# 触发内置 echo 工具调用
uv run jarvis chat --demo "echo Hello AgentCore"
```

### 4.2 离线工作流引擎体验

体验基于有向图与条件边的执行：

```bash
uv run jarvis chat --demo --engine workflow "echo Testing Workflow Graph"
```

### 4.3 SQLite 记忆与多轮会话持久化

在离线或在线模式下持久化存储状态和会话上下文：

```bash
# 第一轮：告诉 Agent 名字，落盘到 agent.db
uv run jarvis chat --demo --db-path "agent.db" --session-id "user-001" "我的名字是小李"

# 第二轮：同一 session_id 恢复上下文
uv run jarvis chat --demo --db-path "agent.db" --session-id "user-001" "echo 状态恢复测试"
```

### 4.4 接入真实大模型 (OpenAI-compatible)

配置环境变量或显式传递参数：

```bash
# 方式 A：通过环境变量
export JARVIS_API_KEY="sk-YOUR_API_KEY"
export JARVIS_BASE_URL="https://api.openai.com/v1" # 或第三方兼容接口
export JARVIS_MODEL="gpt-4o-mini"
uv run jarvis chat "请解释什么是有限状态机"

# 方式 B：通过命令行参数直接指定
uv run jarvis chat --api-key "sk-YOUR_API_KEY" --model "gpt-4o-mini" "你好！"
```
