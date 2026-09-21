# Pregel 执行引擎（pregel-engine）

> 本文是 `core-runtime` 域下的叶子子系统文档。域级总览见 `../core-runtime.md`。
> 本文只展开 Pregel 超级步（super-step）执行引擎本身：BSP 调度、并发执行、重试、校验；
> 不重复展开状态通道（见 `../graph-state-api/graph-state-api.md`）、checkpoint 读写与 IO
> （见 `../pregel-io-checkpoint/pregel-io-checkpoint.md`）、流式输出（见
> `../stream-framework/stream-framework.md`）。
>
> 源码基准：`langgraph` main 分支，commit `e539ac122f4126f6dd850581c1494948cf620e31`。

## 1. 功能清单

本叶子是 LangGraph 的**执行内核**，实现 Bulk Synchronous Parallel（BSP / Pregel）模型：
把"actor（节点）+ channel（通道）"组织成一轮轮超级步，每步分"计划 → 执行 → 更新"三阶段。

| 能力 | 说明 | 源码路径 |
| --- | --- | --- |
| `Pregel` 门面 | 实现 `Runnable`，对外暴露 `invoke/stream/astream`、`get_state`、`update_state` 等；持有 nodes/channels/checkpointer/store/cache | `pregel/main.py:450` |
| `NodeBuilder` | 直接（非经 StateGraph）构造 `PregelNode` 的 DSL：subscribe/read/do/write | `pregel/main.py:206` |
| `PregelLoop` | 超级步循环的状态持有与状态机：`tick()`（计划+中断判断）、`after_tick()`（应用写入+发值）、`__enter__/__exit__` | `pregel/_loop.py:158`（基类）、`:1469`（`SyncPregelLoop`）、`:1722`（`AsyncPregelLoop`） |
| `prepare_next_tasks` | 计划阶段：消费 `TASKS` Topic（PUSH/Send 任务）+ 由 `updated_channels × trigger_to_nodes` 反查被触发节点（PULL），逐个 `prepare_single_task` | `pregel/_algo.py:392` |
| `prepare_single_task` | 把一个节点实例化为可执行任务：读输入通道、缓存命中、构造 `PregelExecutableTask` | `pregel/_algo.py:524` |
| `apply_writes` | 更新阶段：按路径排序任务 → 更新 seen 版本 → 消费读过的通道 → 按通道聚合写入 → `channel.update(vals)` → 返回本步更新的通道集合 | `pregel/_algo.py:232` |
| `PregelRunner.tick` | 执行阶段：把任务提交到后台执行器并发跑，`FIRST_COMPLETED` 等待，单任务快路径、错误路由到 error_handler、超时取消、panic 聚合 | `pregel/_runner.py:176`（同步）、`:360`（异步 `atick`） |
| `run_with_retry` | 单任务重试：按 `RetryPolicy` 链匹配 `retry_on`、`max_attempts`、指数退避 `interval * backoff_factor^(attempts-1)`、`jitter` | `pregel/_retry.py:573` |
| `_TimedAttemptScope` | 节点级超时：`run_timeout` 墙钟上限 + `idle_timeout`（由进度信号刷新），`refresh_on=auto/heartbeat` | `pregel/_retry.py:128` |
| `_call` / `_acall` | 把用户节点函数包成 `Runnable`，注入 `CONFIG_KEY_CALL` 以支持子图/子任务递归调度 | `pregel/_call.py`、`_runner.py:700` |
| `BackgroundExecutor` / `AsyncBackgroundExecutor` | 后台任务上下文管理器：线程池/asyncio 任务，`max_concurrency` 信号量，退出时取消/等待/重抛首个异常 | `pregel/_executor.py:40`、`:122` |
| `validate_graph` | 执行前校验：保留名冲突、节点读的通道存在、输入/输出通道存在、interrupt 节点存在 | `pregel/_validate.py:13` |
| 任务结果缓存 | `match_cached_writes`：命中 `CachePolicy` 时跳过执行直接复用历史写入 | `pregel/_loop.py:728`、`_algo.py:524` |

## 2. 核心类型与接口清单

| 类型 / 函数 | 位置 | 职责 |
| --- | --- | --- |
| `Pregel` | `pregel/main.py:450` | 执行引擎门面，`PregelProtocol` + `Runnable` |
| `PregelLoop` | `pregel/_loop.py:158` | 循环状态机基类；`status` ∈ {done, interrupt_before, interrupt_after, out_of_steps, draining} |
| `SyncPregelLoop` / `AsyncPregelLoop` | `_loop.py:1469` / `:1722` | 同步/异步上下文管理；`__enter__` 加载 checkpoint、`__exit__` 收尾写快照 |
| `PregelRunner` | `pregel/_runner.py:135` | 并发任务执行器；`commit` 回调任务完成 |
| `FuturesDict` | `_runner.py:75` | Future→task 映射，挂 `weakref.WeakMethod(commit)` 完成回调、`threading.Event` 同步 |
| `prepare_next_tasks` / `apply_writes` | `_algo.py:392` / `:232` | BSP 纯函数：计划与更新 |
| `PregelExecutableTask` / `PregelTask` | `types.py`（重导出） | 可执行任务的数据结构：name/path/writes/triggers/runnable/config |
| `RetryPolicy` | `types.py`（重导出） | `max_attempts`、`interval`、`backoff_factor`、`jitter`、`retry_on` |
| `BackgroundExecutor` | `_executor.py:40` | 同步线程池提交协议 `Submit` |
| `Call` | `_algo.py:120` | 子图/任务调用描述（递归调度单元） |

## 3. 关键调用链

**链一：一次 `invoke` 的超级步循环（同步主路径）**

1. `Pregel.invoke`（`main.py:3819`）→ `stream(...)` → 构造 `SyncPregelLoop`，`__enter__` 从 checkpointer 加载初始 checkpoint（`_loop.py:1629`）。
2. `loop.tick()`（`_loop.py:599`）：若 `step > stop` 置 `out_of_steps` 退出；调 `prepare_next_tasks`（`_algo.py:392`）——先消费 `TASKS` Topic 里的 PUSH/Send 任务，再用 `updated_channels × trigger_to_nodes` 反查本步该跑的 PULL 节点，逐个 `prepare_single_task` 实例化。
3. 无任务 → `status="done"`；若 `interrupt_before` 命中 → 抛 `GraphInterrupt`（`_loop.py:667`）。
4. `PregelRunner.tick`（`_runner.py:176`）：单任务走快路径直接 `run_with_retry`；多任务经 `BackgroundExecutor.submit` 并发提交，`FIRST_COMPLETED` 等待每个任务完成即 `yield`（边跑边产流），任一失败且可路由 → 调度 error_handler。
5. 所有任务完成 → `loop.after_tick()`（`_loop.py:683`）：汇总 `tasks[*].writes` → `apply_writes`（`_algo.py:232`）按通道聚合调 `channel.update(vals)`， bumped 版本，返回 `updated_channels`；按 `values`/`updates` 模式发流。
6. `_put_checkpoint`（`_loop.py:1081`）写快照；回到 `tick()` 进入下一超级步，直到无待触发通道 → `done`。

**链二：单任务重试**

- `run_with_retry`（`_retry.py:573`）：捕获异常 → `_should_retry_on`（`_retry.py:841`）按 `RetryPolicy.retry_on`（异常类/元组/谓词）判断是否可重试 → 未达 `max_attempts` 则 `sleep(interval * backoff_factor^(attempts-1) ± jitter)` 后重试；超过则上抛。`GraphBubbleUp`（中断/取消/父命令）不重试。

**链三：错误处理路由**

- 节点抛异常且 `_should_route_to_error_handler`（`_runner.py:171`）为真且非 `GraphBubbleUp` → `schedule_error_handler` 构造错误处理任务并调度执行；处理节点本身的失败不再自我捕获（`_loop.py:751`）。

## 4. 配置项

| 配置项 | 默认 / 行为 | 位置 |
| --- | --- | --- |
| `stream_mode` | 默认 `"values"`；可取 values/updates/debug/messages/custom | `main.py:764` |
| `step_timeout` | 整个超级步（全部任务）的墙钟上限 | `main.py:771` |
| `max_concurrency`（config） | 异步后台执行器并发信号量上限 | `_executor.py:135` |
| `interrupt_before_nodes` / `interrupt_after_nodes` | 在指定节点前/后中断（抛 `GraphInterrupt`） | `main.py:768-769` |
| `retry_policy` | 全图默认重试策略；节点级可覆盖 | `main.py:776` |
| `cache_policy` / `cache` | 节点结果缓存策略与后端 | `main.py:775-777` |
| `debug` | 调试模式（详细任务/检查点流） | `main.py:772` |
| `defer`（节点） | 延迟到运行将结束时执行（见 graph-state-api） | `_node.py` |
| `step`/`stop`（循环内） | 当前步与最大步；`step > stop` → `out_of_steps` | `_loop.py:607` |

## 5. 错误与重试语义

- **重试**：仅 `RetryPolicy.retry_on` 匹配的异常才重试；指数退避 `interval * backoff_factor^(attempts-1)`，`jitter` 抖动；达到 `max_attempts` 上抛（`_retry.py:650-667`）。
- **`GraphBubbleUp`**：中断（`GraphInterrupt`）、取消（`NodeCancelledError`）、父图命令（`ParentCommand`）等控制信号**不**当作错误重试，直接冒泡（`_executor.py:81`、`_runner.py:225`）。
- **错误处理路由**：节点异常 → 若配了 `error_handler` 且非 BubbleUp，则调度处理节点，原异常记 `_handled_exception_ids` 不再重抛；处理节点失败直接失败整个运行（`_runner.py:176-358`）。
- **panic 聚合**：`_panic_or_proceed`（`_runner.py:650`）在所有任务收尾后，重抛首个未被处理的异常；超时未完成任务触发 `NodeTimeoutError`。
- **校验**：`validate_graph` 在构造时拒绝保留通道名、未声明的输入/输出/触发通道、不存在的 interrupt 节点（`_validate.py`）。
- **失败后状态**：异常时 checkpoint 仍会落盘（`_loop.__exit__`），使失败现场可从 checkpointer 恢复/重放。

## 6. 并发细节

- **同步模型**：`PregelRunner.tick` 是一个生成器（`yield` 在每个任务完成点让出控制权给流式层）；任务经 `BackgroundExecutor` 的线程池并发跑，`concurrent.futures.wait(..., FIRST_COMPLETED)` 边完成边产出（`_runner.py:282`）。
- **异步模型**：`AsyncBackgroundExecutor` 用 `asyncio` 任务 + 可选 `asyncio.Semaphore(max_concurrency)`；`run_coroutine_threadsafe` 跨线程调度（`_executor.py:142`）。
- **上下文传播**：`copy_context()` 把当前 contextvars（含 config/checkpoint_ns）复制到工作线程/任务（`_executor.py:64`、`:164`）。
- **可见性边界（BSP 核心）**：执行阶段各节点**互不可见**彼此写入，所有写入在 `after_tick → apply_writes` 步末一次性提交通道（`main.py:471` 文档明确）。
- **完成同步**：`FuturesDict` 用 `threading.Event` 等待所有 done 回调；`weakref.WeakMethod(commit)` 防止循环引用（`_runner.py:190`）。
- **取消语义**：`__cancel_on_exit__` 的任务在上下文退出时 `cancel()`；`next_tick` 用 `time.sleep(0)` 让出线程（`_executor.py:220`）。
- **确定性**：`apply_writes` 按 `task.path[:3]` 排序任务以保证写入应用顺序确定（`_algo.py:256`）。

## 7. 系统边界

**In-Scope（本仓库源码内）**
- `pregel/main.py`（Pregel 门面、状态读写）、`_loop.py`（超级步循环状态机）
- `_algo.py`（BSP 计划/更新纯函数）、`_runner.py`（并发调度）
- `_retry.py`（重试/超时）、`_executor.py`（后台并发池）、`_call.py`、`_validate.py`

**Out-of-Scope（不在本仓库源码内）**
- 通道的具体语义实现 → `channels/`（`../graph-state-api/`）
- checkpoint 持久化后端（Memory/SQLite/Postgres）→ `libs/checkpoint*/`，外部库
- 流式事件格式化/SSE → `stream/`（`../stream-framework/`）
- `langchain_core` Runnable/线程池 `get_executor_for_config` → 第三方依赖
- LLM/工具实际计算 → 由用户节点代码承载，不在本包

## 8. 与相邻子系统交互

- **上游 ← graph-state-api**：`CompiledStateGraph`（继承本叶子 `Pregel`）把节点/通道/边编译成 `PregelNode` + 通道字典后交给本引擎执行。
- **下游 → pregel-io-checkpoint**：循环经 `_put_checkpoint`/`get_state`/`update_state` 与 checkpoint 层交互；pending writes、pending interrupt 读写在 `_io.py`/`_read.py`/`_write.py`。
- **下游 → stream-framework**：`tick`/`after_tick` 中 `self._emit(...)` 把任务/检查点/值事件推入流多路复用器。
- **下游 → managed-values**：`prepare_next_tasks` 接收 `ManagedValueMapping` 作为节点可读值。
- **下游 → internal-foundation**：复用 `_runnable`、`_config`、`_timeout`、`_serde`、`_queue` 等基础设施。

## 9. 语言专项适配口径

本项目为纯 Python 框架，按纯 Python 专项清单分析：

1. **能力缝归组**：BSP 执行模型本身是"隐式状态机藏在循环里"——`PregelLoop.tick()`/`after_tick()` 的循环即超级步状态机，故必须补 lifecycle 图（已补）。
2. **注册表/工厂**：`trigger_to_nodes`（通道→被触发节点列表）是引擎内部的触发反查表，由 `_trigger_to_nodes`（`main.py:4175`）从 nodes 派生，避免每步全量扫描节点。
3. **双轨实现**：同步（`SyncPregelLoop`+线程池）与异步（`AsyncPregelLoop`+asyncio）是同一 BSP 语义的两条执行轨，由 `run_with_retry`/`run_with_async_retry` 双函数承载。
4. **配置驱动分派**：`CachePolicy` 命中与否、`RetryPolicy` 链首匹配、`max_concurrency` 信号量开关，均由运行期配置决定行为。
5. **可选依赖**：`langchain_core` 为重依赖；asyncio 取消语义按 Python ≥3.11 分支（`SUPPORTS_TASK_CANCELLING`，`_retry.py:56`）。

## 10. 图表清单与质量

| 图 | 文件 | 类型 | archify 质量档 |
| --- | --- | --- | --- |
| Pregel 执行引擎结构 | `pregel-engine-architecture.html` | architecture | standard |
| 超级步执行状态机 | `pregel-engine-lifecycle.html` | lifecycle | standard |

- 降档说明：两张图 showcase 布局校验均因"边标签与节点间距/侧边路由穿越节点"严格约束未全过，按规范降 standard 渲染（均成功，各约 784KB 自包含）。lifecycle 图的"保存 checkpoint 后回到 plan"回边因布局约束未画成显式回环，改由卡片说明（见卡片"超级步循环"）。
- JSON IR 源：`json/pregel-engine-architecture.json`、`json/pregel-engine-lifecycle.json`。
- 本叶子另出 lifecycle 图是硬性要求（执行循环即状态机）；不补第三张 sequence 图：本单元的超级步时序已由 lifecycle（状态机）+ architecture（组件拓扑）表达，且与域级 `core-runtime-sequence.html` 的"一次 invoke 端到端时序"信息重复，按资源节省原则省略（省略原因：sequence 的"多方消息按时间交互"语义已被 lifecycle 状态机覆盖，再画一张与域级时序图信息重复）。
