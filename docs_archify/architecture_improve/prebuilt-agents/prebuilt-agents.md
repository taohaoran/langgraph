# prebuilt-agents 域（预构建 Agent 与控制原语）

> 本域是 LangGraph 面向应用的预构建层：把"模型 + 工具 + 人机交互"组装成可直接运行的 Agent 图。

## 域职责

本域在 core-runtime 的 `StateGraph` 之上提供开箱即用的 Agent 模式：

- **ReAct Agent 工厂**：`create_react_agent`/`create_agent` 把模型与工具编译成"调模型→执行工具→再调模型"的循环图。
- **工具执行节点**：`ToolNode` 标准化工具调用的执行、并行与依赖注入。
- **人机交互原语**：`interrupt()`/`Command` 提供暂停-恢复与图控制能力。

## 叶子索引

| 叶子 | 职责 | 关键产物 |
|---|---|---|
| [chat-agent-executor](chat-agent-executor/chat-agent-executor.md) | ReAct Agent 工厂与循环编排、动态模型、钩子、结构化输出 | `create_react_agent`/`call_model`/`should_continue` |
| [tool-node](tool-node/tool-node.md) | 工具执行节点、依赖注入、错误处理、Command 合并 | `ToolNode`/`InjectedState`/`InjectedStore`/`ToolRuntime` |
| [interrupt-command](interrupt-command/interrupt-command.md) | 人机中断与 Command 控制原语 | `interrupt()`/`Command`/`Interrupt` |

## 域级机制

**1. 预构建 Agent 工厂**
`create_react_agent` 是配置驱动的工厂：输入模型（静态实例或 `(state, runtime)` 动态选择）、工具、prompt、前后钩子、response_format、v1/v2 版本，输出编译好的 `CompiledStateGraph`。图由两节点条件边构成 ReAct 循环——`agent`（call_model）与 `tools`（ToolNode），`should_continue` 按末条 AIMessage 是否含 tool_calls 决定继续或退出。步数预算 `remaining_steps` 防止无限循环。

**2. ToolNode 标准化与依赖注入**
`ToolNode` 用 `tools_by_name` 注册表把模型产出的字符串 tool_call 路由到 `BaseTool`；`InjectedState`/`InjectedStore`/`ToolRuntime` 是声明式依赖注入注解，运行时自动把图状态/存储/运行时塞进工具入参。同步用线程池、异步用 `asyncio.gather` 并行执行多个 tool_call；工具异常被转成错误 `ToolMessage` 回传模型而非中断图。

**3. interrupt/Command 人机交互**
`interrupt(value)` 在节点内抛 `GraphInterrupt` 暂停图并把值抛给客户端；恢复时用 `Command(resume=...)` 重跑节点，scratchpad 按中断序号匹配恢复值。`Command` 是统一返回值原语，同时承载状态更新（update）、跳转（goto/Send）、恢复（resume）三类意图，跨图用 `Command.PARENT`。该机制必须配合 checkpointer（checkpoint-storage 域）才能持久化与恢复。

## 与相邻域交互

- 上游：应用代码调用编译出的 `CompiledStateGraph`（Runnable）。
- 下游：调用 LangChain `BaseChatModel`（外部）；checkpointer/store 来自 checkpoint-storage 域；图编译/调度由 core-runtime（Pregel）承载。
- 依赖方向：本域依赖 `langgraph` 核心（StateGraph/RunnableCallable/Send/Command）与 LangChain 消息/工具抽象。

## 域级图

| 图 | 文件 | 类型 | 说明 |
|---|---|---|---|
| 域组件拓扑 | [prebuilt-agents-architecture.html](prebuilt-agents-architecture.html) | architecture | ChatAgentExecutor/ToolNode/interrupt 三模块与外部 LLM、checkpointer 边界 |
| 一次 Agent 执行时序 | [prebuilt-agents-sequence.html](prebuilt-agents-sequence.html) | sequence | 用户输入→chat 模型→工具→中断检查→返回的核心链路 |
| ReAct 循环状态机 | [prebuilt-agents-lifecycle.html](prebuilt-agents-lifecycle.html) | lifecycle | 启动→调模型→调工具→完成的循环状态机（含继续循环回环） |

域级 JSON IR 源文件位于 `json/`。
