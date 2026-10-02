# prebuilt-agents（预构建 Agent）域总览

> 本域包含以下叶子子系统；各叶子详情见对应文档。
> 源码基准：langgraph monorepo main 分支，commit `e539ac12`；主语言纯 Python（libs/prebuilt）。

## 1. 域职责

prebuilt-agents 域（`libs/prebuilt`）提供开箱即用的高级 Agent 构造器：`create_agent`/`create_react_agent`
工厂生成 ReAct 循环图，`ToolNode` 统一工具执行与状态注入，`interrupt()`/`Command` 实现人机交互（human-in-the-loop）。
覆盖 3 个叶子：

- **chat-agent-executor**：ReAct 循环的图构造（`chat_agent_executor.py`）；
- **tool-node**：工具调用执行、结果转换与依赖注入（`tool_node.py` 等）；
- **interrupt-command**：中断挂起与恢复语义（`interrupt.py`）。

## 2. 叶子索引

| 叶子 | 文档 | 架构图 | 第二图 | 职责一句话 |
|------|------|--------|--------|-----------|
| chat-agent-executor | [chat-agent-executor.md](chat-agent-executor/chat-agent-executor.md) | [架构图](chat-agent-executor/chat-agent-executor-architecture.html) | [生命周期图](chat-agent-executor/chat-agent-executor-lifecycle.html) · [时序图](chat-agent-executor/chat-agent-executor-sequence.html) | ReAct 循环图构造 |
| tool-node | [tool-node.md](tool-node/tool-node.md) | [架构图](tool-node/tool-node-architecture.html) | [时序图](tool-node/tool-node-sequence.html) | 工具执行与注入 |
| interrupt-command | [interrupt-command.md](interrupt-command/interrupt-command.md) | [架构图](interrupt-command/interrupt-command-architecture.html) | [生命周期图](interrupt-command/interrupt-command-lifecycle.html) · [时序图](interrupt-command/interrupt-command-sequence.html) | 人机交互中断恢复 |

## 3. 域级机制细节

- **工厂模式**：`create_react_agent` 按配置（模型/工具/系统提示词）组装节点边，返回编译后的
  `CompiledStateGraph`——配置驱动静态分派的典型；`AgentState` 继承 `MessagesState`（消息累加）。
- **ReAct 隐藏状态机**：思考→行动循环（call_model → should_continue → ToolNode）本身是循环内状态机，
  已显式补 lifecycle 图（chat-agent-executor 与域级各一张）；`recursion_limit` 约束循环深度。
- **注入机制**：`InjectedState`/`InjectedStore`/`InjectedToolCallId` 由 `tool_node.py` 解析注入，
  与 langgraph-core 的 `channels`/`managed-values` 联动。
- **人机交互**：`interrupt()` 在超级步边界挂起并持久化，`Command(resume/goto/update)` 恢复执行——
  中断状态机见 interrupt-command-lifecycle 图。
- **外部边界**：LLM 提供商、LangChain 工具、langgraph 引擎（另一域）不在本域源码内。

## 4. 域级图（均 standard 档，render 退出码 0）

![prebuilt-agents 域架构图](prebuilt-agents-architecture.html)

![prebuilt-agents 域时序图](prebuilt-agents-sequence.html)

![prebuilt-agents 域生命周期图](prebuilt-agents-lifecycle.html)

> 本域未生成 dataflow 图：Agent 执行是消息交互与状态流转而非数据管道语义，与已有图信息重复，按资源节省原则省略。
