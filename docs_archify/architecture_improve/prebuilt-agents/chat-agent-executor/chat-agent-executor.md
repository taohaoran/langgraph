# chat-agent-executor（ReAct 预构建 Agent）

> 本文是 `prebuilt-agents` 域下的叶子子系统文档。域级总览见 `../prebuilt-agents.md`。
> 本文只展开 `create_react_agent`/`create_agent` 工厂与 ReAct 循环编排；
> 工具执行节点见 `../tool-node/tool-node.md`，人机中断见 `../interrupt-command/interrupt-command.md`。
>
> 源码基准：`libs/prebuilt/langgraph/prebuilt/`（`chat_agent_executor.py`、`_tool_call_stream.py`、`_tool_call_transformer.py`）。

## 1. 功能清单

| 能力 | 说明 | 源码路径 |
|---|---|---|
| ReAct Agent 工厂 | `create_react_agent`（= `create_agent`）编译出两节点循环图 | `chat_agent_executor.py:278` |
| 静态/动态模型 | 静态模型实例或 `(state, runtime) -> BaseChatModel` 动态选择 | `chat_agent_executor.py:599` |
| prompt 适配 | str/SystemMessage/Callable/Runnable 统一转成前置 prompt runnable | `chat_agent_executor.py:137` |
| 工具绑定校验 | `_should_bind_tools` 校验 `bind_tools` 与传入 tools 一致 | `chat_agent_executor.py:173` |
| 聊天历史校验 | AIMessage 的 tool_calls 必须有对应 ToolMessage | `chat_agent_executor.py:243` |
| 步数预算 | `remaining_steps` 控制循环上限，耗尽返回兜底消息而非报错 | `chat_agent_executor.py:620` |
| 结构化输出 | `response_format` 触发 `with_structured_output` 二次调用 | `chat_agent_executor.py:744` |
| 前后钩子 | `pre_model_hook`（消息裁剪）/`post_model_hook`（护栏） | `chat_agent_executor.py:296` |
| 流式工具调用视图 | `ToolCallStream` + `ToolCallTransformer` 把 tools 通道事件投影为单调用视图 | `_tool_call_stream.py:17`、`_tool_call_transformer.py` |
| v1/v2 双版 | v1 单节点并行执行；v2 用 `Send` 按 tool_call 分发到多实例 | `chat_agent_executor.py:844` |

## 2. 核心类型与接口清单

| 类型 / 函数 | 位置 | 职责 |
|---|---|---|
| `create_react_agent` | `chat_agent_executor.py:278` | 主工厂；已 `@deprecated` 迁移到 `langchain.agents` |
| `AgentState` | `chat_agent_executor.py:57` | 默认状态：`messages`（add_messages reducer）+ `remaining_steps` |
| `call_model` / `acall_model` | `chat_agent_executor.py:661/:696` | agent 节点，同步/异步双实现 |
| `should_continue` | `chat_agent_executor.py:831` | 条件边：无 tool_calls→结束，有→tools |
| `route_tool_responses` | `chat_agent_executor.py:970` | 含 `return_direct` 工具时提前结束 |
| `ToolCallWithContext` | 引自 `tool_node` | v2 把单 tool_call + state 打包进 `Send` |
| `ToolCallStream` | `_tool_call_stream.py:17` | 单次工具调用的流式视图（deltas/output/error/completed） |
| `ToolCallTransformer` | `_tool_call_transformer.py` | 把 `tools` 通道 ProtocolEvent 投影成 `ToolCallStream` |

## 3. 关键调用链

**ReAct 循环（与 lifecycle/sequence 图对应）**：
1. 入口（可选 `pre_model_hook`）→ `agent` 节点 `call_model`：`_get_model_input_state` 取 messages、`_validate_chat_history` 校验、解析（静态或动态）模型、`model.invoke`。
2. 返回 `AIMessage`，`should_continue` 看末条消息：无 `tool_calls` → `post_model_hook`/`generate_structured_response`/`END`；有 → v2 对每个 `tool_call` 发 `Send("tools", ToolCallWithContext(...))`。
3. `tools` 节点（`ToolNode`，见 tool-node 叶子）执行工具，追加 `ToolMessage`；`route_tool_responses` 若命中 `return_direct` 工具则结束，否则回到 `agent`。
4. 循环直到无 tool_calls；`_are_more_steps_needed` 在 `remaining_steps` 不足时返回兜底 AIMessage。

**模型解析**：静态模型在编译期 `prompt_runnable | model`；动态模型在 `call_model` 内 `_resolve_model(state, runtime)` 现场选择并 `bind_tools`。

## 4. 配置项

| 配置 | 默认 / 行为 | 位置 |
|---|---|---|
| `prompt` | None 时直接用 messages；str/SystemMessage 前置；Callable/Runnable 自定义 | `chat_agent_executor.py:292` |
| `response_format` | None 关闭；给 schema 则循环后二次调用生成结构化输出 | `:293` |
| `pre_model_hook` / `post_model_hook` | None；后者仅 v2 可用 | `:296/:297` |
| `checkpointer` / `store` | None；分别持久化单线程状态与跨线程数据 | `:300/:301` |
| `interrupt_before/after` | None；可在 `agent`/`tools` 前后中断 | `:302/:303` |
| `version` | `"v2"`；v1 单节点并行，v2 `Send` 分发 | `:305` |
| `remaining_steps` | `recursion_limit - 已走步数`；<2 且有 tool_calls 时兜底 | `:627` |

## 5. 错误与重试语义

- 聊天历史不完整（tool_calls 缺 ToolMessage）抛 `ValueError(ErrorCode.INVALID_CHAT_HISTORY)`（`_validate_chat_history`）。
- 异步动态模型被同步调用：`call_model` 检测到 `is_async_dynamic_model` 抛 `RuntimeError`，提示用 `ainvoke`（`:664`）。
- 模型名传 `<provider>:<model>` 字符串需安装 `langchain`，否则 `ImportError`（`:571`）。
- 步数耗尽不抛 `GraphRecursionError`，而是返回内容为"Sorry, need more steps..."的 AIMessage（`:684`）。
- 工具执行错误由 `ToolNode` 捕获为 `ToolMessage` 错误消息回传模型，不中断循环（见 tool-node 叶子）。

## 6. 并发细节

- 本叶子编排 LangGraph 图，并发语义由 Pregel 运行时承载；v2 用 `Send` 把多个 tool_call 分发到 `tools` 节点的多个并行实例，天然并行执行独立工具。
- `ToolCallTransformer` 作为流式投影器，把 `tools` 通道上的 `tool-started`/`tool-output-delta`/`tool-finished`/`tool-error` 事件汇聚成 per-call 的 `ToolCallStream`，消费方按到达顺序迭代 deltas。
- 同步/异步双节点：`call_model`+`acall_model` 用 `RunnableCallable` 同时注册，运行时按调用方式选择。

## 7. 系统边界

**In-Scope（本仓库源码内）**
- `prebuilt/chat_agent_executor.py`：Agent 工厂、节点函数、条件边、钩子接线。
- `prebuilt/_tool_call_stream.py` / `_tool_call_transformer.py`：流式工具调用投影。

**Out-of-Scope（不在本仓库源码内）**
- LangChain `BaseChatModel` 及具体 LLM 提供商实现、`.bind_tools`/`.with_structured_output`。
- `ToolNode` 工具执行细节见 tool-node 叶子。
- Pregel 图编译/调度运行时见 core-runtime 相关分片。
- 人机中断 `interrupt`/`Command` 见 interrupt-command 叶子。

## 8. 与相邻子系统交互

- 上游：用户/应用调用编译出的 `CompiledStateGraph`（Runnable）。
- 本叶子 → 下游：调用 LangChain LLM（外部）；把工具执行委托给 `ToolNode`（tool-node 叶子）；`checkpointer`/`store` 来自 checkpoint-storage 域。
- 依赖方向：本叶子依赖 `langgraph` 核心（StateGraph/RunnableCallable/Send）与 LangChain 消息/工具抽象。

## 9. 语言专项适配口径

纯 Python 框架的**注册表与工厂 + 配置驱动**口径：`create_react_agent` 是典型工厂——输入模型/工具/prompt/钩子等配置，输出编译好的 `CompiledStateGraph`；运行时行为（动态模型选择、是否走结构化输出、v1/v2 并行策略、步数预算）全部由入参配置决定，而非硬编码。两节点条件边（`should_continue`/`route_tool_responses`）是 ReAct 范式的状态机表达，对应 lifecycle 图。图类型：lifecycle（ReAct 循环状态机）+ sequence（调用链）。

## 10. 图表清单与质量

| 图 | 文件 | 类型 | archify 质量档 |
|---|---|---|---|
| ReAct 循环状态机 | `chat-agent-executor-lifecycle.html` | lifecycle | standard（showcase 相邻短边标签重叠/回环边穿节点，迭代后仍有告警，按规则降档） |
| ReAct 调用链时序 | `chat-agent-executor-sequence.html` | sequence | showcase |

本叶子不补第三张图：ReAct 循环的组件拓扑已由 tool-node/interrupt-command 叶子的 architecture 图分别表达，且域级 architecture 图已覆盖三模块整体拓扑，按资源节省原则省略。JSON IR 源文件位于 `json/`。
