# ReAct Agent 执行器（chat-agent-executor）

> 本文是 `prebuilt-agents` 域下的叶子子系统文档。域级总览见 `../prebuilt-agents.md`（输出根统一产出）。
> 本文只展开 `create_react_agent`/`create_agent` 如何组装"模型↔工具"ReAct 循环；
> 不展开 ToolNode 内部如何分发工具（见 `../tool-node/tool-node.md`）。
>
> 源码基准：`langgraph` prebuilt 1.1.0，commit `e539ac122f4126f6dd850581c1494948cf620e31`。

## 1. 功能清单

| 能力 | 说明 | 源码路径 |
| --- | --- | --- |
| `create_react_agent` | 工厂函数：把模型 + 工具组装成 ReAct StateGraph（已 @deprecated） | `chat_agent_executor.py:278` |
| `AgentState` / `AgentStatePydantic` | 默认状态 schema（`messages` 列表；Pydantic 版） | `chat_agent_executor.py:57`、`:69` |
| `call_model` / `acall_model` | "agent"节点：取 messages、调 LLM、把 AIMessage 追加进 messages | `chat_agent_executor.py:661`、`:696` |
| `should_continue` | 条件边：末条 AIMessage 有 tool_calls → 进 tools，否则 END | `chat_agent_executor.py:831` |
| 模型解析 | 静态模型 / 动态模型（按 state/runtime 运行期选模型） | `_get_model:220`、`_resolve_model:599` |
| 提示词 | `_get_prompt_runnable` 把 prompt 模板接到模型前 | `chat_agent_executor.py:137` |
| 钩子 | `pre_model_hook` / `post_model_hook` 可插节点 | `chat_agent_executor.py:796`、`:805` |
| 结构化输出 | `response_format` → `generate_structured_response` 节点 | `chat_agent_executor.py:809` |
| v2 并行派发 | 按每个 tool_call 生成 `Send("tools", ToolCallWithContext)` | `chat_agent_executor.py:849-859` |

## 2. 核心类型与接口清单

| 类型 / 函数 | 位置 | 职责 |
| --- | --- | --- |
| `create_react_agent` | `chat_agent_executor.py:278` | 对外工厂；返回编译好的 `StateGraph`（Pregel） |
| `AgentState` | `chat_agent_executor.py:57` | 默认 TypedDict 状态，仅 `messages` 一个累加键 |
| `call_model`（闭包） | `chat_agent_executor.py:661` | 节点函数；注入 `Runtime`/`RunnableConfig` |
| `should_continue`（闭包） | `chat_agent_executor.py:831` | 路由函数，返回节点名或 `list[Send]` |
| `ToolCallWithContext` | `tool_node.py` 导入 | v2 下携带 tool_call + state 的 Send 载荷 |
| `RunnableCallable` | `_internal/_runnable.py:299` | 把 call_model/acall_model 包装成图节点 |

## 3. 关键调用链

1. **组装图**：`create_react_agent(model, tools, ...)`（`:278`）→ 构造 `StateGraph(AgentState)`，`add_node("agent", RunnableCallable(call_model, acall_model))`（`:867-869`）、`add_node("tools", tool_node)`（`:872`）。
2. **模型调用**：`call_model`（`:661`）取 `_get_model_input_state(state)`（`:636`），静态模型 `static_model.invoke`（`:679`）或动态模型 `_resolve_model` 后调用（`:676-677`），返回 `{"messages": [response]}`（`:694`）。
3. **路由判断**：`should_continue`（`:831`）看 `messages[-1]`：不是 AIMessage 或无 `tool_calls` → END（或 post_model_hook / 结构化输出，`:835-841`）；有 tool_calls → v1 直接返回 `"tools"`（`:845`），v2 对每个 tool_call 返回 `Send`（`:849-859`）。
4. **工具回环**：tools 节点执行后写回 ToolMessage，图沿固定边回到 "agent"，形成 ReAct 循环，直到不再产生 tool_calls。

## 4. 配置项

| 配置项 | 默认 / 行为 | 位置 |
| --- | --- | --- |
| `model` | 必需；静态或可调用（动态选模型） | `chat_agent_executor.py` 工厂参数 |
| `tools` | 工具列表或已构造的 `ToolNode` | `:290`、`:555-560` |
| `state_schema` | 默认 `AgentState` | `:863` |
| `prompt` | 接到模型前的提示词模板 | `:137` |
| `response_format` | 结构化输出 schema | `:809` |
| `pre_model_hook` / `post_model_hook` | 可选前后钩子节点 | `:796`/`:805` |
| `version` | `"v1"` 串行 tools / `"v2"` 按 tool_call 并行 Send | `:844-859` |

## 5. 错误与重试语义

- 模型/工具调用异常沿 Pregel 节点失败路径冒泡，可由 `retry_policy` 重试（在图编译期配置）。
- 异步模型在同步 `invoke` 下调用直接抛 `RuntimeError`（`:664-670`）。
- `_are_more_steps_needed`（`:620`）判定步数将尽时，返回一条"需要更多步"的占位 AIMessage（`:684-692`），避免超 `recursion_limit`。
- 无自定义退避；重试/超时继承自 prebuilt 图与模型层。

## 6. 并发细节

- v2 用 `Send("tools", ...)` 把多个 tool_call 并行分发到 tools 节点（`:849-859`），由 Pregel 帧调度。
- sync/async 双轨：`call_model`/`acall_model` 成对，`RunnableCallable` 按调用方式选轨。
- 状态 `messages` 是累积通道，多分支写回经 reducer 合并。
- 无线程原语；并发由 Pregel 运行时与 LLM SDK 决定。

## 7. 系统边界

**In-Scope（本仓库源码内）**
- `chat_agent_executor.py`：ReAct 图组装、节点函数、路由。

**Out-of-Scope（不在本仓库源码内）**
- LLM 推理本身（外部模型/API）：不在本仓库源码内。
- ToolNode 内部工具分发与参数注入：见 tool-node 叶子。
- Pregel 帧调度、Send 并行语义：见 pregel-engine 叶子。
- checkpointer/store：见 checkpoint-storage 叶子。

## 8. 与相邻子系统交互

- 上游 → 本叶子：用户传 model + tools 调 `create_react_agent`。
- 本叶子 → 下游："tools"节点委托 `ToolNode`（tool-node 叶子）执行；模型调用委托外部 LLM；最终编译产物交给 pregel-engine 运行。
- 方向：`用户 → create_react_agent → StateGraph(agent↔tools) → Pregel 运行 → ToolNode/外部 LLM`。

## 9. 语言专项适配口径

按纯 Python 框架口径：
- **工厂 + 闭包节点**：`create_react_agent` 是工厂，`call_model`/`should_continue` 是闭包——捕获 model/tools/hooks 后注册为图节点/条件边，是 Python 框架的标准组装模式。
- **隐藏状态机在循环里**：ReAct 循环（思考→行动→思考）是隐藏在图边回环里的状态机，故补 lifecycle 图显式表达。
- **配置驱动分派**：v1/v2、动态模型、response_format、hooks 都由工厂参数决定图拓扑，是配置驱动静态分派。
- **双轨实现**：sync/async 节点成对。
- 无注册表/代码生成。

## 10. 图表清单与质量

| 图 | 文件 | 类型 | archify 质量档 |
| --- | --- | --- | --- |
| ReAct Agent 架构 | `chat-agent-executor-architecture.html` | architecture | showcase |
| ReAct 循环状态机 | `chat-agent-executor-lifecycle.html` | lifecycle | showcase |
| Agent 执行时序 | `chat-agent-executor-sequence.html` | sequence | showcase |

JSON IR 源文件位于 `json/` 目录。本叶子未补 dataflow 图：ReAct 是"模型↔工具"的消息循环，无独立数据管道，时序/生命周期图已覆盖，按资源节省原则省略。
