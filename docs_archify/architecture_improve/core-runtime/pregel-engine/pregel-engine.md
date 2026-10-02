# Pregel BSP 超级步执行引擎（pregel-engine）

> 本文是 `core-runtime` 域下的叶子子系统文档，也是 LangGraph 的运行时心脏。域级总览见
> `../core-runtime.md`。本文展开 `Pregel` 运行时：BSP（Bulk Synchronous Parallel）超级步循环、
> 并发任务执行、invoke/stream 入口。图如何被编译成 Pregel 见 `../graph-state-api/graph-state-api.md`；
> 通道如何读写/序列化见 `../channels/channels.md` 与 `../pregel-io-checkpoint/pregel-io-checkpoint.md`。
>
> 源码基准：`langgraph` 1.2.11，commit `e539ac12`，目录 `libs/langgraph/langgraph/pregel/`
> （`main.py` 4364 行、`_loop.py` 1988 行、`_runner.py` 941 行、`_algo.py` 1460 行）。

![Pregel 引擎核心架构图](pregel-engine-architecture.html)

## 1. 功能清单

Pregel 把「actor（节点）+ channel（通道）」模型组织成多个超级步（superstep）执行：每步分
**Plan（规划哪些节点本步跑）→ Execute（并发跑完，写对本步不可见）→ Update（超步末统一把写入
归约进通道）**，重复直到没有节点被选中或达到步数上限。

| 能力 | 说明 | 源码路径 |
|---|---|---|
| 运行时入口 | `Pregel`：实现 `PregelProtocol`，对外提供 `invoke/stream/astream/ainvoke/stream_events` | `pregel/main.py:450`（类）、`:3836`（invoke） |
| 同步入口 | `invoke`：内部委托 `stream`，收集 values 模式输出与中断 | `pregel/main.py:3836` |
| 超步循环 | `PregelLoop`：基类；`SyncPregelLoop`/`AsyncPregelLoop` 分别实现同步/异步上下文 | `pregel/_loop.py:158` / `:1469` / `:1722` |
| 单超步规划 | `PregelLoop.tick`：步数上限检查→`prepare_next_tasks`→无任务则 done→interrupt_before 检查→返回是否继续 | `pregel/_loop.py:599` |
| 超步收尾 | `PregelLoop.after_tick`：`apply_writes` 归约→emit values→清 pending writes→`_put_checkpoint` | `pregel/_loop.py:683` |
| 任务规划 | `prepare_next_tasks`：依据更新的通道选出本步应触发的节点 | `pregel/_algo.py:349` |
| 写入归约 | `apply_writes`：把本步所有任务的待写应用到通道、推进版本、算出 updated_channels | `pregel/_algo.py:232` |
| 并发执行 | `PregelRunner.tick`：把任务提交到 executor、用 FuturesDict 收集、commit 写入、错误路由 | `pregel/_runner.py:176`（类 `:135`） |
| 线程/协程池 | `BackgroundExecutor`（同步线程池）/ `AsyncBackgroundExecutor`（asyncio 信号量门控） | `pregel/_executor.py:40` / `:122` |
| 节点重试 | `run_with_retry`：按 `RetryPolicy` 对节点执行退避重试 | `pregel/_retry.py:573` |
| 节点包装 | `_call`：把用户函数包成带 ChannelWrite 的 Runnable，注入 CONFIG_KEY_CALL | `pregel/_call.py` |
| 中断检测 | `should_interrupt`：按 interrupt_before/after 命中任务抛 `GraphInterrupt` | `pregel/_algo.py:155` |

## 2. 核心类型与接口清单

| 类型 / 函数 | 位置 | 职责 |
|---|---|---|
| `Pregel` | `pregel/main.py:450` | 运行时对象；持有 nodes/channels/managed/checkpointer/store 等编译态配置，实现 Runnable 接口 |
| `PregelLoop` | `pregel/_loop.py:158` | 一次 run 的状态机：step、checkpoint、tasks、updated_channels、status（done/out_of_steps/interrupt_before/draining） |
| `SyncPregelLoop` / `AsyncPregelLoop` | `_loop.py:1469` / `:1722` | 分别实现 `__enter__`/`__aenter__` 上下文管理，包装同步/异步执行 |
| `PregelRunner` | `_runner.py:135` | 并发执行器；`submit`/`put_writes` 以 weakref 注入，`node_error_handler_map` 把失败路由到错误处理节点 |
| `FuturesDict` | `_runner.py:75` | dict 子类：future → task，完成回调触发 `commit`，带停止事件 |
| `BackgroundExecutor` / `AsyncBackgroundExecutor` | `_executor.py:40` / `:122` | 任务提交抽象；异步版用 semaphore + `gated` 协程控制并发 |
| `Call` | `_algo.py:120` | 可调用任务的包装类型 |
| `prepare_next_tasks` | `_algo.py:349` | 任务规划核心：通道版本对比 → 选中订阅了更新通道的节点 |
| `apply_writes` | `_algo.py:232` | 写入归约核心 |

扩展点：错误处理节点（`node_error_handler_map`/`schedule_error_handler`，`_runner.py:147`）、
运行控制 `RunControl`（cooperative drain）、`store` 长期记忆、自定义 `RetryPolicy`。

## 3. 关键调用链

**调用链一：`invoke` 一次完整 run。**

1. 用户调 `Pregel.invoke(input, config)`（`main.py:3836`），内部 `for chunk in self.stream(...)`
   逐块收集输出。
2. `stream` 构造 `SyncPregelLoop`（`_loop.py:1469`）并进入上下文（`__enter__`，`_loop.py:1629`）。
3. 循环每超步调 `PregelLoop.tick`（`_loop.py:599`）：步数超限置 `out_of_steps`；否则
   `prepare_next_tasks`（`_algo.py:349`）算出本步任务；无任务置 `done`；命中 interrupt_before 抛
   `GraphInterrupt`；有任务则返回 True 继续。
4. 任务交给 `PregelRunner.tick`（`_runner.py:176`）：通过 `BackgroundExecutor.submit` 并发执行，
   每个 future 完成回调 `commit` 把写入投递（`_runner.py:75`）。
5. 全部任务完成后调 `after_tick`（`_loop.py:683`）：`apply_writes`（`_algo.py:232`）归约通道、
   emit values、清 pending writes、`_put_checkpoint`（`_loop.py:1081`）落盘。
6. 回到第 3 步直到 done，循环退出，`stream` 产出最终 chunk，`invoke` 返回。

![Pregel invoke 调用链时序](pregel-engine-sequence.html)

**调用链二：节点执行与错误路由。**

1. `PregelRunner.tick` 单任务快路径直接 `run_with_retry(t, retry_policy, ...)`（`_runner.py:196`，
   重试实现在 `_retry.py:573`）。
2. 抛异常时 `self.commit(t, exc)`（`_runner.py:203`）；若不是 `GraphBubbleUp` 且该节点配了错误
   处理器，则经 `schedule_error_handler` 调度错误处理任务（`_runner.py:208`），异常 id 记入
   `_handled_exception_ids` 防止重复上抛。

**调用链三：超步状态机（隐藏在循环里）。**

`PregelLoop.status` 在 `out_of_steps/done/draining/interrupt_before` 之间迁移，由 `tick` 的返回值与
各种前置检查驱动；这是典型的"状态机藏在循环里"，已用 lifecycle 图显式画出。

![Pregel 超级步执行状态机](pregel-engine-lifecycle.html)

## 4. 配置项

| 配置 / 参数 | 默认 / 行为 | 位置 |
|---|---|---|
| `durability` | `"async"`：检查点随下一超步异步落盘；`"sync"`：下一超步前同步落盘；`"exit"`：仅退出时落 | `main.py:3847` |
| `stream_mode` | `"values"`：每超步输出最终状态；另有 updates/debug/messages 等 | `main.py:3840` |
| `interrupt_before/after` | `()`：在指定节点名前/后抛中断 | `main.py:3844`/`:3845` |
| `step` 上限 | `self.stop`：超步达到即置 `out_of_steps` | `_loop.py:604` |
| `retry_policy` | 节点级重试策略，透传 `run_with_retry` | `_runner.py:182` |
| `control` | `RunControl`：协作式 drain，`drain_requested` 时本轮结束 | `_loop.py:655` |
| `checkpointer` | 编译期注入；`None` 则不落检查点 | `main.py` Pregel 构造 |

## 5. 错误与重试语义

- **节点失败重试**：`run_with_retry`（`_retry.py:573`）按 `RetryPolicy`（限定词/退避）重试，
  重试次数耗尽才上抛。
- **错误处理节点路由**：未被 `GraphBubbleUp` 包裹的异常，若节点在 `node_error_handler_map` 中，
  不直接终止 run，而是调度对应错误处理任务（`_runner.py:204-213`）；已处理异常 id 记入集合避免重复上抛。
- **中断**：`should_interrupt`（`_algo.py:155`）命中即抛 `GraphInterrupt`，作为正常控制流把控制权
  交还客户端，下次 `Command(resume=)` 恢复。
- **超步上限**：`step > stop` 时 status=`out_of_steps`，正常退出而非异常（`_loop.py:604`）。
- **drain**：`control.drain_requested` 时 status=`draining`，本轮规划后不再继续（`_loop.py:655`）。
- **BubbleUp**：`GraphBubbleUp` 是引擎内部信号（中断/返回等），不被当作普通节点错误。

## 6. 并发细节

- **同步路径**：`BackgroundExecutor`（`_executor.py:40`）封装线程池；`PregelRunner.tick` 用
  `concurrent.futures.Future` + `FuturesDict`（`_runner.py:75`）收集本步任务，完成回调 `commit`。
- **异步路径**：`AsyncBackgroundExecutor`（`_executor.py:122`）用 `asyncio.Semaphore` 经 `gated`
  协程（`_executor.py:214`）门控并发度。
- **BSP 可见性**：执行期各节点的写入对同超步其他节点不可见，统一在 `after_tick→apply_writes`
  （`_loop.py:683`）归约——这是"逻辑并发、物理归约"的关键。
- **weakref 解耦**：`submit`/`put_writes` 以 `weakref.ref` 注入 runner（`_runner.py:140`），避免
  循环引用阻碍 run 对象回收。
- **取消传播**：`asyncio.CancelledError`/future cancel 经 `_should_stop_others`（`_runner.py:616`）
  触发停止事件，取消同超步其余任务。

## 7. 系统边界

**In-Scope（本仓库源码内）**
- `pregel/main.py`：`Pregel` 运行时与入口
- `pregel/_loop.py`：超级步循环与同步/异步上下文
- `pregel/_runner.py`：并发执行与错误路由
- `pregel/_algo.py`：任务规划与写入归约
- `pregel/_executor.py`：线程/协程池抽象
- `pregel/_call.py`/`_retry.py`：节点包装与重试

**Out-of-Scope（不在本仓库源码内）**
- 通道存储语义：channels 原语族
- 检查点 blob 读写与持久化：`BaseCheckpointSaver` 实现（`libs/checkpoint-*`）
- 图编译（StateGraph→Pregel）：graph-state-api 叶子
- 长期记忆 `BaseStore`：store 后端库
- 用户节点函数本身：用户代码

## 8. 与相邻子系统交互

- **graph-state-api → 本叶子**：`CompiledStateGraph` 继承 `Pregel`，编译产物即 Pregel 构造参数
  （nodes/channels/managed/checkpointer）。
- **本叶子 → channels**：`prepare_next_tasks`/`apply_writes` 经通道版本判断触发与归约；
  `apply_writes` 调通道 `update`。
- **本叶子 → pregel-io-checkpoint**：`_put_checkpoint` 调 `create_checkpoint`；启动水化调
  `channels_from_checkpoint`；节点写出经 `ChannelWrite`/`CONFIG_KEY_SEND`。
- **本叶子 → checkpointer**：以 `weakref` 持有 saver，超步末 `put` blob，恢复时 `get`。
- **用户 → 本叶子**：`invoke/stream/astream/ainvoke` 是对外唯一入口。

## 9. 语言专项适配口径（纯 Python）

按 `language-python.md` 清单落地：

1. **隐藏状态机在循环里（本叶子最典型）**：BSP 超级步是藏在 `PregelLoop.tick/after_tick` 循环里的
   状态机（status: done/out_of_steps/draining/interrupt_before），已用 lifecycle 图显式画出，
   而非默认省略。
2. **双轨实现**：同步 `SyncPregelLoop`/`BackgroundExecutor` 与异步 `AsyncPregelLoop`/
   `AsyncBackgroundExecutor` 成对存在，同一套状态机两套执行底座。
3. **回调/闭包能力缝**：`CONFIG_KEY_READ`/`CONFIG_KEY_SEND`/`CONFIG_KEY_CALL` 由引擎注入 config，
   节点与 IO 层面向闭包编程；runner 用 weakref 解耦 submit/put_writes。
4. **并发模型**：纯 Python 内靠 `concurrent.futures`（同步）与 `asyncio` semaphore（异步）实现
   逻辑并发；无 GIL 外的原生并行——CPU 密集节点仍受 GIL 约束（属运行时事实，非本仓库实现）。
5. **外部边界**：本引擎是编排核心，但不持久化（checkpointer）、不存储语义（channels）、不定义图
   拓扑（graph-state-api）；这些相邻叶子/库均标注"不在本叶子源码内"。

## 10. 图表清单与质量

| 图 | 文件 | 类型 | archify 质量档 |
|---|---|---|---|
| Pregel 引擎核心架构图 | `pregel-engine-architecture.html` | architecture | showcase（首跑因"prepare/apply_writes"标签在 loop→algo 间距不足被 composition/label-gap 拒；标签缩短为"规划/归约"后通过） |
| Pregel 超级步执行状态机 | `pregel-engine-lifecycle.html` | lifecycle | showcase（一次通过） |
| Pregel invoke 调用链时序 | `pregel-engine-sequence.html` | sequence | showcase（首跑因末消息 y=620 超出可读时间轴且画布高度不足被 layout/constraint 拒；末消息上移至 y=590、viewBox 加高至 760 后通过） |

JSON IR 源文件位于 `json/`（`pregel-engine-architecture.json`、`pregel-engine-lifecycle.json`、
`pregel-engine-sequence.json`）。

**省略说明**：本叶子未生成 dataflow 图——引擎内的数据（待写/通道值）流向已由架构图与 sequence
图表达，无独立的"源→变换→目的地"管道血缘值得单画；未生成 workflow 图——BSP 流程已被 lifecycle
状态机与 architecture 图完整覆盖，workflow 泳道语义与它们重复。按资源节省原则省略。
