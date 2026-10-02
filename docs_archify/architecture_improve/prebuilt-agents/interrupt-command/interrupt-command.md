# 人机中断与命令（interrupt-command）

> 本文是 `prebuilt-agents` 域下的叶子子系统文档。域级总览见 `../prebuilt-agents.md`（输出根统一产出）。
> 本文只展开 `interrupt()` 人机中断与 `Command(resume/goto/update)` 恢复原语；
> 不展开 Pregel 如何捕获 `GraphInterrupt` 并持久化（见 `../pregel-engine/pregel-engine.md`）。
>
> 源码基准：`langgraph` main 分支，commit `e539ac122f4126f6dd850581c1494948cf620e31`。

## 1. 功能清单

| 能力 | 说明 | 源码路径 |
| --- | --- | --- |
| `interrupt(value)` | 在节点内抛 `GraphInterrupt` 暂停图、把 value 暴露给调用方；恢复时返回 resume 值 | `types.py:851` |
| `Interrupt` | 中断信息值对象：`value`（暴露给客户端）+ `id`（恢复定位） | `types.py:575` |
| `Command[N]` | 更新状态 / 跳转节点 / 恢复中断的命令原语 | `types.py:799` |
| `Command.PARENT` | 指向最近父图的命令目标 | `types.py:848` |
| `Interrupt.from_ns` | 按 checkpoint 命名空间派生中断 id | `types.py:616` |
| `PregelTask.interrupts` | 任务上挂起的中断元组 | `types.py:644` |

## 2. 核心类型与接口清单

| 类型 / 函数 | 位置 | 职责 |
| --- | --- | --- |
| `interrupt(value)` | `types.py:851` | 节点内调用；首次抛异常、恢复时返回 resume 值 |
| `Interrupt` | `types.py:575` | 不可变中断记录；`id` 默认由 ns 哈希派生 |
| `Command[N]` | `types.py:799` | `graph/update/resume/goto` 四元命令；`_update_as_tuples` 把 update 规范化为 (通道, 值) |
| `GraphInterrupt` | `errors.py:102` | 中断控制异常（`GraphBubbleUp` 派生） |
| `PregelScratchpad` | `_internal/_scratchpad.py:9` | 提供 `interrupt_counter`/`resume`/`get_null_resume` |

## 3. 关键调用链

1. **首次中断**：节点调 `interrupt(question)`（`types.py:851`）→ 从 `conf[CONFIG_KEY_SCRATCHPAD]` 取 scratchpad，`idx = interrupt_counter()`（`:952-953`）。
2. **无 resume 值**：`scratchpad.resume` 为空、`get_null_resume` 也为空 → 抛 `GraphInterrupt((Interrupt.from_ns(value, ns),))`（`:967-974`），Pregel 顶层捕获并经 checkpointer 持久化暂停。
3. **客户端恢复**：调用方发 `Command(resume=answer)`；图从最近 checkpoint 恢复，节点从头重跑。
4. **重跑命中 resume**：重跑中再次执行到 `interrupt(question)`，此时 `scratchpad.resume` 已有值，`idx < len(resume)` → 发 `RESUME` 写入并 `return scratchpad.resume[idx]`（`:955-958`），不再抛异常，节点继续。
5. **多中断配对**：同一节点多个 `interrupt` 按 `interrupt_counter` 顺序与 resume 值按序配对（`:866-868` docstring）。

## 4. 配置项

| 配置项 | 默认 / 行为 | 位置 |
| --- | --- | --- |
| checkpointer | 必需——interrupt 依赖持久化暂停/恢复 | `types.py:870` docstring |
| `Command.resume` | 单值或 {interrupt_id: value} 映射 | `types.py:823` |
| `Command.goto` | 下一节点名 / Send / 序列 | `types.py:824` |
| `Command.update` | 状态更新 | `types.py:822` |
| `graph` | None 当前图 / `Command.PARENT` 父图 | `types.py:821`、`:848` |

## 5. 错误与重试语义

- `GraphInterrupt`（`errors.py:102`）不是失败而是控制流异常；节点重跑不触发 retry_policy。
- 恢复时若 resume 值与中断数不匹配，`assert` 失败（`types.py:962`）暴露编程错误。
- 未配 checkpointer 时无法持久化，interrupt 行为不成立（使用前置条件）。
- `Interrupt.id` 冲突时按 ns 哈希派生（`:612`）。

## 6. 并发细节

- `resume` 列表按 task 隔离（docstring：不跨任务共享，`:867-868`）。
- `interrupt_counter` 是 scratchpad 上的回调计数器，每次节点内 `interrupt()` 调用递增。
- 无线程原语；暂停/恢复由 Pregel 帧与 checkpointer 驱动。

## 7. 系统边界

**In-Scope（本仓库源码内）**
- `types.py` 中 `interrupt()`/`Interrupt`/`Command`；`errors.py` 中 `GraphInterrupt`/`NodeInterrupt`。

**Out-of-Scope（不在本仓库源码内）**
- Pregel 如何捕获 GraphInterrupt、写 checkpoint、从 checkpoint 恢复：见 pregel-engine 叶子。
- checkpointer 后端（SQLite/Postgres/内存）：见 checkpoint-storage 叶子。
- 客户端如何轮询/提交 resume：LangGraph Server 不在本仓库源码内。

## 8. 与相邻子系统交互

- 上游 → 本叶子：用户节点函数内调 `interrupt()`；调用方在暂停后构造 `Command(resume=...)`。
- 本叶子 → 下游：`GraphInterrupt` 冒泡到 pregel-engine 捕获；`Command.goto/update` 由 Pregel 解释为状态写与边跳转。
- 方向：`节点 → interrupt() → GraphInterrupt → pregel-engine(checkpoint 暂停) → 客户端 Command(resume) → 节点重跑 → interrupt 返回 resume 值`。

## 9. 语言专项适配口径

按纯 Python 框架口径：
- **控制流异常缝**：用 `GraphInterrupt`（`GraphBubbleUp` 派生）表达"可恢复暂停"这一非错误控制流，沿调用栈冒泡到引擎顶层，是解释器式框架的典型手法。
- **隐藏状态机在循环里**：含中断的运行有明确的 running→interrupted→resumed→completed 状态变迁（节点重跑即回迁），故补 lifecycle 图。
- **值对象 + 命令模式**：`Interrupt`（暴露给客户端的只读信息）与 `Command`（客户端发回的命令）是一对人机交互契约。
- 无注册表/代码生成。

## 10. 图表清单与质量

| 图 | 文件 | 类型 | archify 质量档 |
| --- | --- | --- | --- |
| 中断与恢复架构 | `interrupt-command-architecture.html` | architecture | showcase |
| interrupt/resume 时序 | `interrupt-command-sequence.html` | sequence | showcase |
| 含中断运行生命周期 | `interrupt-command-lifecycle.html` | lifecycle | showcase |

JSON IR 源文件位于 `json/` 目录。lifecycle 首版因跨 lane 的 drop/straight 边走线校验未过 showcase，移除显式 route 提示、加宽 viewBox 后自动布局通过（showcase）。本叶子未补 dataflow 图：中断恢复是请求-响应交互与状态机，无独立数据管道，按资源节省原则省略。
