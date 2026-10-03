# Observability & Configuration Design

> 本文档描述 AgentCore 的工程化生产加固能力：集中配置系统、结构化日志、上下文追踪与性能指标度量。

---

## 1. 集中配置管理 (`infra.config`)

AgentCore 采用 Pydantic `BaseModel` 实现强类型集中配置 `JarvisConfig`，全面支持环境变量重载：

| 配置属性 | 对应环境变量 | 默认值 | 作用说明 |
| :--- | :--- | :--- | :--- |
| `llm_api_key` | `JARVIS_API_KEY` | `None` | 大模型访问令牌（敏感信息，严禁硬编码） |
| `llm_base_url` | `JARVIS_BASE_URL` | `https://api.openai.com/v1` | OpenAI 兼容接口地址 |
| `llm_model` | `JARVIS_MODEL` | `gpt-4o-mini` | 默认调用的模型名 |
| `llm_timeout` | `JARVIS_LLM_TIMEOUT` | `60.0` | 单次 LLM 调用超时秒数 |
| `runtime_engine` | `JARVIS_ENGINE` | `react` | 默认执行引擎：`react` 或 `workflow` |
| `runtime_max_steps` | `JARVIS_MAX_STEPS` | `8` | 状态机最大执行步数限制 |
| `storage_db_path` | `JARVIS_DB_PATH` | `None` | SQLite 存储文件路径，若留空使用内存 |
| `log_level` | `JARVIS_LOG_LEVEL` | `INFO` | 日志级别：DEBUG, INFO, WARNING, ERROR |
| `log_format` | `JARVIS_LOG_FORMAT` | `text` | 日志输出格式：`text` 或 `json` |
| `trace_enabled` | `JARVIS_TRACE_ENABLED` | `True` | 是否启用 Tracer 记录 |
| `metrics_enabled` | `JARVIS_METRICS_ENABLED` | `True` | 是否收集统计指标 |

* **获取方式**：业务代码通过 `get_config()` 获取单例配置，亦支持通过 `get_config(**overrides)` 进行运行时按需覆写。

---

## 2. 结构化日志 (`infra.telemetry.StructuredLogger`)

* **基于 ContextVars 的上下文透传**：
  通过 `bind_context(run_id=..., engine=...)` 绑定的变量在当前异步/同步调用链中自动传递，所有日志行自动打印上下文标签，便于跨函数排查问题。
* **双模式输出**：
  * **Text 模式（开发调试）**：
    `[2026-10-03T03:53:57.123456+00:00] [INFO] (runtime.engine) run_id=123-abc engine=react - Starting ReAct engine run`
  * **JSON 模式（生产日志采集）**：
    `{"timestamp": "...", "level": "INFO", "logger": "runtime.engine", "message": "...", "context": {"run_id": "123-abc", "engine": "react"}}`

---

## 3. 轻量级调用链路追踪 (`infra.telemetry.Tracer`)

无需引入庞大的外部 Agent 追踪平台，AgentCore 内建了开箱即用的轻量级线程安全 Tracer：

```python
with tracer.start_span("react_engine_run", tags={"run_id": state.id}):
    # 嵌套子 Span
    with tracer.start_span("tool_execution", tags={"tool": "echo"}):
        ...
```

* **父子跨度串联**：自动通过 `_ACTIVE_SPAN` ContextVar 获取父 Span 的 `span_id` 作为当前 Span 的 `parent_span_id`，继承相同的 `trace_id`。
* **生命周期耗时计算**：精准记录 `start_time` 与 `end_time`，提供精确到浮点秒级的 `duration_seconds`。
* **异常捕捉标注**：若代码块发生未捕获异常，Span 自动记录 `span.error = str(exc)` 并将 `tags["error"] = True`。

---

## 4. 运行时指标系统 (`infra.telemetry.MetricsRegistry`)

AgentCore 收集核心业务关键指标，通过 `snapshot()` 输出瞬时快照字典：

1. **Counters 计数器**：
   * `engine_runs_total{engine=react|workflow}`：引擎启动总次数。
   * `engine_completions_total{engine=react|workflow}`：成功终态总次数。
   * `engine_failures_total{engine=react|workflow}`：失败终态总次数。
   * `tool_calls_total{tool=xxx}`：各工具调用频次。
   * `tool_errors_total{tool=xxx}`：各工具错误次数。
2. **Observations 延迟分布**：
   * `engine_run_duration_seconds`：端到端执行耗时分布（统计 `count`, `total`, `avg`, `min`, `max`）。
   * `tool_call_duration_seconds`：各工具调用延迟分布。
