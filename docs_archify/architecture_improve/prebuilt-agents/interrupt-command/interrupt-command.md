# interrupt-command（人机中断与 Command 控制原语）

> 本文是 `prebuilt-agents` 域下的叶子子系统文档。域级总览见 `../prebuilt-agents.md`。
> 本文只展开 `interrupt()` 人机中断机制与 `Command` 控制原语及其与 Pregel 的联动；
> ReAct 循环中如何被使用见 `../chat-agent-executor/chat-agent-executor.md`。
>
> 源码基准：`libs/prebuilt/langgraph/prebuilt/interrupt.py`（人机交互类型）与 `libs/langgraph/langgraph/types.py`（`interrupt`/`Command` 核心实现）。

## 1. 功能清单

| 能力 | 说明 | 源码路径 |
|---|---|---|
| 人机中断 | `interrupt(value)` 在节点内暂停图、向客户端抛出值 | `langgraph/types.py:851` |
| 恢复原语 | `Command(resume=...)` 携带恢复值继续执行 | `langgraph/types.py:799` |
| 状态更新命令 | `Command(update=...)` 在工具/节点返回时更新图状态 | `langgraph/types.py:821` |
| 跳转命令 | `Command(goto=...)` / `Send` 跳转到指定节点 | `langgraph/types.py:813` |
| 跨图命令 | `Command(graph=Command.PARENT, goto=[Send])` 派发父图 | `langgraph/types.py:848` |
| 中断值对象 | `Interrupt(value, id)` 序列化抛给客户端 | `langgraph/types.py:575` |
| 人机交互类型 | `HumanInterrupt`/`ActionRequest`/`HumanInterruptConfig`/`HumanResponse`（已废弃，迁至 langchain.agents） | `prebuilt/interrupt.py:11/:33/:51/:87` |

## 2. 核心类型与接口清单

| 类型 / 函数 | 位置 | 职责 |
|---|---|---|
| `interrupt(value)` | `langgraph/types.py:851` | 节点内调用；首次抛 `GraphInterrupt`，恢复时返回 resume 值 |
| `Command` | `langgraph/types.py:799` | 控制原语，聚合 `graph`/`update`/`resume`/`goto` |
| `Command.PARENT` | `langgraph/types.py:848` | 指向最近父图的字面量 |
| `Interrupt` | `langgraph/types.py:575` | 带 id 的中断值，`from_ns` 生成 |
| `GraphInterrupt` / `NodeInterrupt` | `langgraph/errors.py` | 中断异常族 |
| `HumanInterrupt` / `HumanResponse` | `prebuilt/interrupt.py:51/:87` | 人在环请求/响应 TypedDict（已废弃） |
| `HumanInterruptConfig` | `prebuilt/interrupt.py:11` | allow_ignore/respond/edit/accept 四开关 |

## 3. 关键调用链

**中断-恢复生命周期（与 sequence 图对应）**：
1. 首次执行：Pregel 调度节点 → 节点内调 `interrupt(value)`。
2. `interrupt` 从 scratchpad 取 `interrupt_counter()` 作为本次中断序号 `idx`；无 resume 值 → 构造 `Interrupt.from_ns(value, ns)` 并 `raise GraphInterrupt(...)`，节点执行被打断。
3. Pregel 捕获中断 → 经 checkpointer 持久化当前检查点 → 把 `Interrupt(value, id)` 作为 `__interrupt__` 输出抛给客户端。
4. 客户端用 `Command(resume=...)` 再次 `invoke` 同一 thread → Pregel 加载检查点 → **从头重执行该节点**（而非从中断点继续）。
5. 重执行时 `interrupt` 再次被调，scratchpad 已含 `resume` 列表：`idx < len(resume)` 时返回 `resume[idx]` 并 `CONFIG_KEY_SEND([(RESUME, ...)])`，不再抛异常；节点拿到 resume 值后正常返回状态更新。
6. 节点可在一个节点内多次 `interrupt`，按调用顺序（`idx`）与恢复值一一对应。

## 4. 配置项

| 配置 | 默认 / 行为 | 位置 |
|---|---|---|
| checkpointer | **必须启用**，否则 interrupt 无法持久化/恢复 | `types.py:870` 文档 |
| `Command.resume` | 可为 `{interrupt_id: value}` 映射，或单值按序匹配 | `types.py:808` |
| `Command.goto` | 节点名 / 节点名序列 / `Send` / `Send` 序列 | `types.py:813` |
| `Command.graph` | `None` 当前图；`Command.PARENT` 父图 | `types.py:821` |
| `HumanInterruptConfig` | allow_ignore/respond/edit/accept 四开关 | `prebuilt/interrupt.py:23` |

## 5. 错误与重试语义

- 中断不是错误：`GraphInterrupt` 被 Pregel 正常捕获为"暂停"，不记失败。
- 恢复值不匹配：`interrupt` 中 `assert len(scratchpad.resume) == idx` 保证恢复值数量与中断调用一致，不匹配会抛断言错误。
- 无 checkpointer 时使用 interrupt：行为未定义（文档明确要求启用 checkpointer）。
- 重执行幂等要求：节点被从头重跑，作者需保证 `interrupt` 之前的副作用可重入（通常把可重副作用放在 `interrupt` 之后）。

## 6. 并发细节

- 恢复值列表 `scratchpad.resume` 是**任务级**的（文档明确"scoped to the specific task, not shared across tasks"），多任务并行时各自维护自己的 interrupt 序号。
- 跨图命令 `Command.PARENT + goto=[Send]` 在 ToolNode 中被合并为单条 parent 命令（见 tool-node 叶子 `_combine_tool_outputs`）。
- 中断通过抛异常实现控制流反转，不阻塞线程；Pregel 暂停后释放执行资源。

## 7. 系统边界

**In-Scope（本仓库源码内）**
- `langgraph/types.py`：`interrupt`/`Command`/`Interrupt` 定义与 scratchpad 交互。
- `prebuilt/interrupt.py`：人机交互 TypedDict（已废弃迁移）。

**Out-of-Scope（不在本仓库源码内）**
- Pregel 对 `GraphInterrupt` 的捕获、检查点持久化与重调度逻辑（core-runtime 分片）。
- checkpointer 持久化实现见 checkpoint-storage 域。
- `langchain.agents.interrupt`（新位置，外部）。

## 8. 与相邻子系统交互

- 上游：节点/工具内调用 `interrupt()`；客户端通过 `Command(resume=...)` 驱动恢复。
- 本叶子 → 下游：`interrupt` 经 `get_config()` 读 scratchpad/send 通道；`Command` 作为节点/工具返回值被 Pregel 解释为状态更新、跳转与恢复；依赖 checkpointer 持久化。
- 依赖方向：本叶子依赖 `langgraph` 内部配置/常量/错误类型；被 prebuilt 与用户图广泛使用。

## 9. 语言专项适配口径

纯 Python 框架的**控制流反转（异常即控制流）**口径：`interrupt()` 用抛异常实现"暂停-恢复"，恢复时通过"节点从头重执行 + scratchpad 序号匹配 resume 值"的方式让作者写出"看起来线性、实际可暂停"的代码——这是框架把人机交互编织进同步控制流的能力缝。`Command` 是统一的返回值原语，同时承载状态更新、跳转、恢复三类意图，使工具/节点返回值能驱动图控制流。图类型以 sequence（中断-恢复生命周期）为主。

## 10. 图表清单与质量

| 图 | 文件 | 类型 | archify 质量档 |
|---|---|---|---|
| 中断-恢复时序 | `interrupt-command-sequence.html` | sequence | showcase |
| 中断-命令模块拓扑 | `interrupt-command-architecture.html` | architecture | standard（边标签与邻近组件重叠，去标签后通过；showcase 布局诊断未全过） |
| 节点执行中断-恢复状态机 | `interrupt-command-lifecycle.html` | lifecycle | standard（跨泳道回边走线经多轮调整后通过；showcase 仍有走线诊断） |

本次迭代新增 architecture 与 lifecycle 两图：architecture 刻画 `types.py` 中断原语、Pregel 运行时、scratchpad 与 checkpointer 的组件拓扑；lifecycle 刻画待调度→运行中→已中断→完成的单实体状态机（含 Command(resume) 回迁边）。JSON IR 源文件位于 `json/`。
