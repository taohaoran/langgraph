# 工具节点 ToolNode（tool-node）

> 本文是 `prebuilt-agents` 域下的叶子子系统文档。域级总览见 `../prebuilt-agents.md`（输出根统一产出）。
> 本文只展开 `ToolNode` 如何把 AIMessage 里的 tool_calls 分发到具体工具并注入运行期上下文；
> 不展开它被 ReAct agent 如何挂载（见 `../chat-agent-executor/chat-agent-executor.md`）。
>
> 源码基准：`langgraph` prebuilt 1.1.0，commit `e539ac122f4126f6dd850581c1494948cf620e31`。

## 1. 功能清单

| 能力 | 说明 | 源码路径 |
| --- | --- | --- |
| `ToolNode` | 工具执行图节点（继承 `RunnableCallable`）：接收 tool_calls、分发、合并 ToolMessage | `tool_node.py:622` |
| `_func` / `_afunc` | 同步/异步执行入口：解析 tool_calls、构造 ToolRuntime、并行跑 | `tool_node.py:793`、`:828` |
| `tools_by_name` | 工具名 → 工具对象注册表 | `tool_node.py`（构造期建立） |
| `ToolRuntime` | 注入包：向工具暴露 state/store/config/context/stream_writer/tools | `tool_node.py:1663` |
| `InjectedState` / `InjectedStore` / `InjectedToolCallId` | 标注参数：运行期注入、不进工具 schema | `tool_node.py:1753` 等 |
| `_handle_tool_error` | 工具异常归一为 ToolMessage | `tool_node.py:394` |
| `tool_validator` | 工具入参校验 | `tool_validator.py` |
| `_tool_call_stream` / `_tool_call_transformer` | 工具调用流投影 | `_tool_call_stream.py`、`_tool_call_transformer.py` |
| `tools_condition` | 便捷条件边：有 tool_calls 进 tools | `tool_node.py:1582` |

## 2. 核心类型与接口清单

| 类型 / 函数 | 位置 | 职责 |
| --- | --- | --- |
| `ToolNode(RunnableCallable)` | `tool_node.py:622` | 工具节点；构造时收工具列表或 dict |
| `ToolRuntime[ContextT,StateT]` | `tool_node.py:1663` | 每次 tool_call 构造的注入对象 |
| `InjectedState(InjectedToolArg)` | `tool_node.py:1753` | 标注"从图状态注入"的参数 |
| `ToolCallWithContext` | `tool_node.py:286` | v2 Send 的载荷（tool_call + state） |
| `ToolInvocationError` | `tool_node.py:339` | 工具调用异常包装 |
| `_InjectedArgs` | `tool_node.py:567` | 内部：识别需注入的参数 |

## 3. 关键调用链

1. **接收输入**：`_func`（`tool_node.py:793`）`_parse_input` 提取 `tool_calls` 与 input_type（`:799`），`get_config_list` 为每个 call 复制 config（`:800`）。
2. **构造注入包**：对每个 tool_call，`_extract_state` 取状态（`:805`），构造 `ToolRuntime(state, tool_call_id, config, context, store, stream_writer, tools, ...)`（`:806-816`）。
3. **并行执行**：同步用 `get_executor_for_config(config)` 的 `executor.map(self._run_one, ...)`（`:821-824`）；异步用 `asyncio.gather`（`:855-858`）。
4. **合并输出**：`_combine_tool_outputs`（`:826`/`:860`）把每个工具结果包装成 ToolMessage 列表返回。
5. **错误处理**：单工具失败经 `_handle_tool_error`（`:394`）归一为错误 ToolMessage，不中断整批。

## 4. 配置项

| 配置项 | 默认 / 行为 | 位置 |
| --- | --- | --- |
| `tools` | 工具列表 / dict / 已建 ToolNode | 构造参数 |
| `handle_tool_errors` | True 时把异常包成 ToolMessage；可传自定义处理器 | `tool_node.py:394` |
| InjectedState key | `InjectedState("messages")` 指定注入哪个状态键 | `tool_node.py:1753` |
| `config` | 经 `get_config_list` 每 call 一份 | `:800` |

## 5. 错误与重试语义

- 工具抛异常：`handle_tool_errors` 默认把异常文本包成 ToolMessage（`:383` `_default_handle_tool_errors`），其他 tool_call 继续执行。
- 工具入参校验失败：`tool_validator.py` 校验，`_filter_validation_errors`（`:510`）处理。
- 不向工具暴露的注入参数缺失时抛错（如 InjectedStore 但无 store，`:1411`）。
- 重试继承自图节点 retry_policy；本叶子不自带退避。

## 6. 并发细节

- 多 tool_call 并行：同步用 `executor.map`（`:821`），异步用 `asyncio.gather`（`:858`）。
- `ToolRuntime` 每 call 独立构造，无共享可变状态。
- `config_list` 为每个 call 复制一份 config，避免并发串扰。
- 无线程泄漏；executor 由 `get_executor_for_config` 上下文管理。

## 7. 系统边界

**In-Scope（本仓库源码内）**
- `tool_node.py`、`tool_validator.py`、`_tool_call_stream.py`、`_tool_call_transformer.py`。

**Out-of-Scope（不在本仓库源码内）**
- 实际工具函数实现（用户/第三方）：不在本仓库源码内。
- langchain_core 工具协议、InjectedToolArg 基类：外部依赖。
- Pregel 如何调度节点、Send 并行：见 pregel-engine 叶子。
- LLM 产生 tool_calls：见 chat-agent-executor 叶子。

## 8. 与相邻子系统交互

- 上游 → 本叶子：ReAct agent 的 "tools" 节点由 `ToolNode` 充当；接收含 tool_calls 的 AIMessage。
- 本叶子 → 下游：按 `tools_by_name` 分发到具体工具；通过 `ToolRuntime` 把图状态/store 注入工具参数；结果回写为 ToolMessage。
- 方向：`agent(AIMessage.tool_calls) → ToolNode → ToolRuntime 注入 → 工具函数 → ToolMessage 回写 agent`。

## 9. 语言专项适配口径

按纯 Python 框架口径：
- **注册表**：`tools_by_name` 是工具名→对象的运行期注册表，构造期填充。
- **依赖注入扩展点**：`InjectedState/InjectedStore/InjectedToolCallId` 是注解式 DI——工具参数用 `Annotated[T, InjectedX]` 标记，ToolRuntime 在调用时填充，这些参数从工具 schema 中排除（对 LLM 不可见），是典型框架 DI 缝。
- **双轨实现**：sync `_func`（executor.map）/async `_afunc`（asyncio.gather）成对。
- **编排层定位**：ToolNode 不实现工具逻辑，只负责分发与注入，实际计算在工具函数。

## 10. 图表清单与质量

| 图 | 文件 | 类型 | archify 质量档 |
| --- | --- | --- | --- |
| ToolNode 架构 | `tool-node-architecture.html` | architecture | showcase |
| 工具调用时序 | `tool-node-sequence.html` | sequence | showcase |

JSON IR 源文件位于 `json/` 目录。本叶子未补 dataflow / lifecycle 图：工具分发是请求-响应时序（已由 sequence 表达），无独立数据管道，ToolRuntime 也无单实体状态机，按资源节省原则省略。
