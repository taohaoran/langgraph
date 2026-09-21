# 内部基础设施（internal-foundation）

> 本文是 `core-runtime` 域下的叶子子系统文档。域级总览见 `../core-runtime.md`。
> 本文只展开被全框架复用的内部基础设施：Runnable 基类、配置合并、保留常量、错误类型、
> 序列化白名单、队列/超时/重试/重放、Runtime；不重复展开它们在 Pregel/图/流中的使用
> （分别见各相邻叶子）。
>
> 源码基准：`langgraph` main 分支，commit `e539ac122f4126f6dd850581c1494948cf620e31`。

## 1. 功能清单

| 能力 | 说明 | 源码路径 |
| --- | --- | --- |
| `RunnableCallable` | 包装任意同步/异步可调用对象为 `Runnable`；自动注入 config/store/writer/runtime 等协议参数 | `_internal/_runnable.py:299` |
| `RunnableSeq` | 把多个 Runnable 串成序列（`|` 组合符） | `_runnable.py:584` |
| `coerce_to_runnable` | 把普通函数/lambda 规范化为 `Runnable` | `_runnable.py:550` |
| `is_async_callable` / `is_async_generator` | 同步/异步判断工具 | `_runnable.py:528`、`:539` |
| `set_config_context` / `create_task_in_config_context` | contextvars 配置上下文注入与任务内传播 | `_runnable.py:127`、`:143` |
| `merge_configs` / `patch_config` / `ensure_config` | RunnableConfig 合并、打补丁、保证非空 | `_internal/_config.py:147`、`:194`、`:322` |
| `patch_configurable` / `merge_metadata` / `filter_to_user_tags` | configurable 子键合并、回调合并、标签过滤 | `_config.py:52`、`:87`、`:463` |
| 保留写入键/配置键 | `INPUT/INTERRUPT/RESUME/ERROR/TASKS/RETURN/PREVIOUS`、`CONFIG_KEY_SEND/READ/CALL/...` | `_internal/_constants.py:7-80` |
| 公共图常量 | `START/END/TAG_NOSTREAM/TAG_HIDDEN`；惰性重导出 Send/Interrupt | `constants.py:24-31` |
| 错误族 | `GraphBubbleUp` 基类 + `GraphInterrupt/GraphDrained/GraphRecursionError/InvalidUpdateError/NodeTimeoutError/NodeCancelledError/ParentCommand/NodeError` | `errors.py:50-241` |
| `ErrorCode` 枚举 | 错误码→排错文档链接 | `errors.py:34` |
| serde 白名单 | `build_serde_allowlist`/`apply_checkpointer_allowlist`；严格 msgpack 时限制可序列化类型 | `_internal/_serde.py:43` |
| `AsyncQueue` / `SyncQueue` / `Semaphore` | 带 `wait()` 的有界背压队列（不消费仅等待） | `_internal/_queue.py:12`、`:70`、`:44` |
| `coerce_timeout_policy` / `sync_timeout_unsupported` | 超时规范化；同步不可取消报错 | `_internal/_timeout.py:14`、`:21` |
| `default_retry_on` | 默认是否可重试（连接错误/5xx 可重试；ValueError 等不可） | `_internal/_retry.py:1` |
| `ReplayState` | 时间旅行重放：跟踪子图命名空间是否首次访问，决定加载重放点之前的 checkpoint | `_internal/_replay.py:14` |
| `Runtime` / `RunControl` / `ExecutionInfo` | 运行期对象：context/store/stream_writer/heartbeat/drain 请求 | `runtime.py:125`、`:79`、`:27` |
| `PregelScratchpad` | 节点执行期上下文（step/stop/计数器/resume） | `_internal/_scratchpad.py:9` |

## 2. 核心类型与接口清单

| 类型 / 函数 | 位置 | 职责 |
| --- | --- | --- |
| `RunnableCallable[I, O]` | `_runnable.py:299` | 函数→Runnable 适配器（本框架最基础的 Runnalbe 包装） |
| `RunnableSeq` | `_runnable.py:584` | 序列组合 |
| `GraphBubbleUp` | `errors.py:50` | 控制信号基类（中断/排空/父命令），不被当错误重试 |
| `NodeError`（frozen dataclass） | `errors.py:149` | 注入错误处理节点的上下文 |
| `Runtime[ContextT]` | `runtime.py:125` | 注入节点的运行期对象 |
| `ReplayState` | `_replay.py:14` | 重放会话状态 |
| `AsyncQueue`/`SyncQueue` | `_queue.py:12`/`:70` | 背压队列 |
| `PregelScratchpad` | `_scratchpad.py:9` | 节点执行期 scratchpad |

## 3. 关键调用链

**链一：函数如何变成可注入参数的 Runnable**

1. 用户节点函数 `fn(state, *, config, store, writer)` 经 `RunnableCallable(fn, afn)`（`_runnable.py:302`）包装。
2. 调用时 `RunnableCallable.invoke`（`_runnable.py:374`）按协议（`_RunnableWithConfigWriter` 等 Protocol，`:251-278`）检测函数签名，把 config/store/writer/runtime 作为关键字参数注入。
3. `coerce_to_runnable`（`:550`）对已是 Runnable 的直接复用、对普通函数走 `RunnableCallable`。

**链二：配置合并**

- `merge_configs(*configs)`（`_config.py:147`）按优先级合并多个 RunnableConfig（configurable/metadata/callbacks/tags）；`patch_config`（`:194`）在派生 config 上打补丁（如 checkpoint_ns 传播）。

**链三：时间旅行重放**

- `ReplayState.get_checkpoint(ns, checkpointer, config)`（`_replay.py:52`）：子图命名空间首次访问时，用 `checkpoint_id` 作 `before` 上界查最近 checkpoint（重放点之前）；后续同 ns（剥离 task-id 后缀）走普通 `get_tuple`。

## 4. 配置项

| 配置项 | 默认 / 行为 | 位置 |
| --- | --- | --- |
| `LANGGRAPH_DELTA_MAX_SUPERSTEPS_SINCE_SNAPSHOT` | 环境变量；DeltaChannel 无更新时最多回放超级步（默认 5000） | `_config.py:33` |
| `recursion_limit`（config） | 最大超级步；超出抛 `GraphRecursionError` | `errors.py:67` |
| `CONF` 命名空间 | 所有 `CONFIG_KEY_*` 挂在 `configurable` 下 | `_constants.py:32` |
| 严格 msgpack | `langgraph.checkpoint.serde._msgpack.STRICT_MSGPACK_ENABLED`；缺包时 `_serde` 降级为 False | `_serde.py:24` |

## 5. 错误与重试语义

- **`GraphBubbleUp` 体系**：`GraphInterrupt`/`GraphDrained`/`ParentCommand` 继承之，引擎按"控制信号"处理、不进重试/错误路由（见 pregel-engine）。
- **`NodeTimeoutError`**：故意**不**继承内置 `TimeoutError`（它是 `OSError` 子类），使默认 `RetryPolicy` 把它当可重试（`errors.py:190-194`）。`kind∈{idle,run}` 区分空闲/墙钟超时。
- **`NodeCancelledError`**：用户节点自己抛 `asyncio.CancelledError` 时由重试层转换，避免被引擎当静默拆除（`errors.py:168-187`）。
- **`default_retry_on`**（`_retry.py:1`）：`ConnectionError`/5xx 可重试；`ValueError/TypeError/.../OSError` 不可；其他默认可重试。
- **`NodeError`**：frozen dataclass，按参数类型注解注入错误处理节点。

## 6. 并发细节

- `AsyncQueue.wait()`（`_queue.py:17`）：空队列时仅等待不消费，复用 `asyncio.Queue` 的 getter 机制；`SyncQueue`（`:70`）用 deque + `Semaphore(0)` 实现 `wait()`。
- `Semaphore.wait`（`:47`）：阻塞到可获取但不获取（背压探测）。
- `set_config_context`（`_runnable.py:127`）用 contextvars Token 管理配置上下文，`create_task_in_config_context`（`:143`）把当前 context 复制到新 asyncio 任务。
- `Runtime`/`PregelScratchpad` 每任务一份，经 config 注入，跨线程经 `copy_context` 传播（见 pregel-engine）。
- `ReplayState` 单实例按引用共享给同一次父执行的所有派生 config。

## 7. 系统边界

**In-Scope（本仓库源码内）**
- `_internal/`（Runnable/配置/常量/serde/队列/超时/重试/重放/scratchpad/pydantic/future/fields）
- 根级 `errors.py`、`constants.py`、`runtime.py`、`config.py`、`typing.py`、`warnings.py`、`callbacks.py`、`types.py`（类型重导出）

**Out-of-Scope（不在本仓库源码内）**
- `langchain_core.runnables.Runnable` 基类、`RunnableConfig`、callback manager → 第三方依赖
- `langgraph.checkpoint.base`/`serde` → 外部库
- pydantic → 第三方依赖

## 8. 与相邻子系统交互

- **被全域依赖**：graph-state-api/pregel-engine/pregel-io-checkpoint/stream-framework/func-api/managed-values 均 import 本叶子的 `_runnable`/`_config`/`_constants`/`errors`/`_serde`/`_timeout`/`_queue`。
- **下游 → 外部**：serde 白名单套到外部 `BaseCheckpointSaver.with_allowlist`；队列/超时服务于执行引擎。

## 9. 语言专项适配口径

本项目为纯 Python 框架，按纯 Python 专项清单分析：

1. **能力缝归组**：`RunnableCallable` 的"协议检测+参数注入"是装饰器/运行时分发能力缝——按函数签名（有无 config/store/writer/runtime）决定注入哪些关键字参数，而非显式参数。
2. **可选依赖与懒加载**：`_serde` 用 try/except ImportError 降级 `STRICT_MSGPACK_ENABLED`；`default_retry_on` 在函数体内 `import httpx/requests`（延迟导入）；`constants.__getattr__` 惰性重导出 Send/Interrupt。
3. **配置面工程**：`merge_configs`/`ensure_config` 处理 RunnableConfig 合并；`Runtime` 是 dataclass 注入面。
4. **双轨实现**：`AsyncQueue`/`SyncQueue`、`merge_configs` 的同步/异步回调管理器成对出现。

## 10. 图表清单与质量

| 图 | 文件 | 类型 | archify 质量档 |
| --- | --- | --- | --- |
| 内部基础设施层 | `internal-foundation-architecture.html` | architecture | standard |
| Runnable.invoke 基础设施链 | `internal-foundation-sequence.html` | sequence | standard |

- 降档说明：architecture 图降 standard；本次新增的 sequence 图（invoke→merge_configs→serde 校验→retry/timeout 包裹→执行→返回序列化）因"自消息 0px 跨度/segment 超出画布"修复后仍未全过 showcase，按规范降 standard 渲染（均成功，约 805KB 自包含）。
- JSON IR 源：`json/internal-foundation-architecture.json`、`json/internal-foundation-sequence.json`。
- 本次迭代新增 sequence 图，把"横切基础设施如何串进一次 invoke"从文字化升级为图；不另出 dataflow：基础设施是调用链而非数据管道。
