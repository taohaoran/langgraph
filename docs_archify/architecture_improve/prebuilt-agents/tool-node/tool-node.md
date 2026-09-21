# tool-node（工具执行节点与依赖注入）

> 本文是 `prebuilt-agents` 域下的叶子子系统文档。域级总览见 `../prebuilt-agents.md`。
> 本文只展开 `ToolNode` 工具执行节点、依赖注入注解与 `ValidationNode`；
> 它在 ReAct 循环中的位置见 `../chat-agent-executor/chat-agent-executor.md`。
>
> 源码基准：`libs/prebuilt/langgraph/prebuilt/tool_node.py`（约 2031 行）、`tool_validator.py`。

## 1. 功能清单

| 能力 | 说明 | 源码路径 |
|---|---|---|
| 工具执行节点 | `ToolNode(RunnableCallable)`，接收 tool_calls 并行执行工具 | `tool_node.py:622` |
| 按名查表 | `tools_by_name` 属性把工具名映射到 `BaseTool` 实例 | `tool_node.py:789` |
| 依赖注入 | `InjectedState`/`InjectedStore`/`ToolRuntime` 注解向工具注入图状态/存储/运行时 | `tool_node.py:1753/:1829/:1663` |
| 错误处理 | `handle_tool_errors` 把工具异常转为错误 `ToolMessage` 回传模型 | `tool_node.py:394` |
| 命令式返回 | 工具可返回 `Command` 实现跳转/状态更新；多 PARENT goto 合并 | `tool_node.py:862` |
| 并行执行 | 同步 `get_executor_for_config.map`；异步 `asyncio.gather` | `tool_node.py:821/:858` |
| 工具调用校验器 | `ValidationNode` 只校验不执行，pydantic schema 校验 tool_calls | `tool_validator.py:47` |
| 条件路由 | `tools_condition` 按是否有 tool_calls 路由 | `tool_node.py:1582` |

## 2. 核心类型与接口清单

| 类型 / 函数 | 位置 | 职责 |
|---|---|---|
| `ToolNode` | `tool_node.py:622` | 主类；`_func`（同步）/`_afunc`（异步）为节点入口 |
| `_run_one` / `_arun_one` | `tool_node.py:1014/:1161` | 单 tool_call 的同步/异步执行单元 |
| `_execute_tool_sync` / `_execute_tool_async` | `tool_node.py:922/:1069` | 真正调用工具并包成 ToolMessage |
| `_inject_tool_args` | `tool_node.py:1315` | 按注解从 state/store/runtime 提取注入参数 |
| `_get_all_injected_args` | `tool_node.py:1967` | 静态分析工具签名，识别 state/store/runtime 注入点 |
| `InjectedState(field)` | `tool_node.py:1753` | 注解：把图状态（或某字段）注入工具参数 |
| `InjectedStore` | `tool_node.py:1829` | 注解：注入 `BaseStore` |
| `ToolRuntime` | `tool_node.py:1663` | 打包 state/context/config/stream_writer/tool_call_id/store 的运行时对象 |
| `ToolCallRequest` | `tool_node.py:133` | 工具请求可被覆写（handler 模式） |
| `ValidationNode` | `tool_validator.py:47` | 已废弃；只校验 tool_calls 符合 pydantic schema 不执行 |

## 3. 关键调用链

**一次工具执行（`_func`）**：
1. `_parse_input(input)` 从末条 AIMessage 解析出 `tool_calls` 与 input 类型（list/dict/tool_calls）。
2. 为每个 call 构造 `ToolRuntime`（含 state、tool_call_id、config、context、store、stream_writer）。
3. `get_executor_for_config(config)` 得线程池，`executor.map(self._run_one, ...)` 并行执行；异步版 `asyncio.gather`。
4. `_run_one` → `_validate_tool_call`（未知名工具直接返回错误 ToolMessage）→ `_inject_tool_args` 按注解提取注入参数 → `_execute_tool_sync` 调工具。
5. `_combine_tool_outputs` 合并：纯 ToolMessage 直接返回；含 `Command` 时把多个 `Command(PARENT, goto=[Send])` 合并为一条 parent 命令。

**依赖注入识别**：`_get_all_injected_args` 遍历工具类型注解，`_is_injected_arg_type` 识别 `InjectedToolArg` 子类，分别记录 state_args/store_arg/runtime_arg，运行时由 `_inject_tool_args` 把对应值塞进工具入参。

## 4. 配置项

| 配置 | 默认 / 行为 | 位置 |
|---|---|---|
| `tools` | 工具列表或 `ToolNode`；空列表则 agent 无工具节点 | `tool_node.py:743` |
| `handle_tool_errors` | True/str/Callable；True 用默认错误文案 | `tool_node.py:394` |
| `format_error` | 自定义错误文案格式化 | `tool_validator.py:34` |
| `retry_on` | 工具异常重试（继承 BaseTool 机制） | BaseTool 外部 |
| `messages_key` | dict 输入时输出挂载的 key（默认 `messages`） | `tool_node.py` |

## 5. 错误与重试语义

- 工具执行异常：`_handle_tool_error` 按 `handle_tool_errors` 配置转成错误 `ToolMessage`，回传模型让其自我修正，不中断图。
- 未识别工具名：`_validate_tool_call` 直接返回错误 `ToolMessage`，不抛异常（`tool_node.py:1268`）。
- 工具返回的 `Command` 列表校验：`_validate_tool_command_list`/`_validate_tool_command` 保证 Command 形态合法（`:1455/:1503`）。
- `ValidationNode` 校验失败返回含错误信息的 `ToolMessage`（默认文案提示"修复后再响应"）。

## 6. 并发细节

- 同步多工具并行：`get_executor_for_config` 返回的线程池 `map` 分发到 `_run_one`（`:821`）。
- 异步多工具并行：`asyncio.gather` 并发 `_arun_one`（`:858`）。
- v2 ReAct 中，多个 tool_call 还会被 `Send` 分发到 `tools` 节点的多个独立实例（见 chat-agent-executor 叶子），与节点内并行叠加。
- 每个并行任务独立持有自己的 `ToolRuntime` 副本，无共享可变状态。

## 7. 系统边界

**In-Scope（本仓库源码内）**
- `prebuilt/tool_node.py`：ToolNode、注入注解、ToolRuntime、错误处理、Command 合并。
- `prebuilt/tool_validator.py`：ValidationNode（已废弃）。

**Out-of-Scope（不在本仓库源码内）**
- LangChain `BaseTool` 抽象、`@tool` 装饰器、工具 schema 生成。
- 工具的具体业务实现与外部系统调用。
- 并行执行器 `get_executor_for_config` 来自 LangChain 运行时。
- Pregel 节点调度见 core-runtime 分片。

## 8. 与相邻子系统交互

- 上游：`create_react_agent` 把 `ToolNode` 注册为图的 `tools` 节点；agent 节点产出含 tool_calls 的 AIMessage 后条件边路由到这里。
- 本叶子 → 下游：调用用户注册的工具函数；工具可返回 `Command` 反哺图状态/跳转；通过 `InjectedStore` 访问 checkpoint-storage 域的 `BaseStore`。
- 依赖方向：本叶子依赖 `langgraph` 核心（RunnableCallable/Runtime/Command/Send）与 LangChain 工具/消息抽象。

## 9. 语言专项适配口径

纯 Python 框架的**能力缝（依赖注入）+ 注册表**口径：`ToolNode` 是"工具执行"能力缝——`tools_by_name` 注册表把字符串工具名路由到 `BaseTool` 实例；`InjectedState`/`InjectedStore`/`ToolRuntime` 是声明式依赖注入注解，工具作者只需标注类型，运行时由 `_inject_tool_args` 自动满足。这是框架级"控制反转"扩展点。图类型以 architecture（查表→注入→执行→合并的组件流）为主。

## 10. 图表清单与质量

| 图 | 文件 | 类型 | archify 质量档 |
|---|---|---|---|
| ToolNode 执行结构 | `tool-node-architecture.html` | architecture | showcase |
| ToolNode 工具调用执行链 | `tool-node-sequence.html` | sequence | showcase |

第二张时序图（本次迭代新增）刻画工具调用解析→按名查表→注入参数并行执行→`_combine_tool_outputs` 合并回写的完整链路，对应第 3 节"一次工具执行（`_func`）"调用链。JSON IR 源文件位于 `json/`。
