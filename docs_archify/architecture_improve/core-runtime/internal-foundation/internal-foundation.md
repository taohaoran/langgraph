# 内部基础设施（internal-foundation）

> 本文是 `core-runtime` 域下的叶子子系统文档。域级总览见 `../core-runtime.md`（输出根统一产出）。
> 本文只展开 `_internal/*`、`utils/*` 与顶层 `config/errors/constants/callbacks` 等被全库复用的
> 横切设施；不展开它们被 Pregel 如何使用（见 `../pregel-engine/pregel-engine.md`）。
>
> 源码基准：`langgraph` main 分支，commit `e539ac122f4126f6dd850581c1494948cf620e31`。

## 1. 功能清单

| 能力 | 说明 | 源码路径 |
| --- | --- | --- |
| `merge_configs` / `patch_config` | 多源 `RunnableConfig` 合并与就地打补丁 | `_internal/_config.py:147`、`:194` |
| `patch_configurable` / `_merge_callbacks` | `CONF` 子键补丁、回调聚合 | `_internal/_config.py:52`、`:119` |
| `get_callback_manager_for_config` | 由 config 构造回调管理器 | `_internal/_config.py:236` |
| `RunnableCallable` | sync/async 双轨可调用 Runnable，包装普通函数 | `_internal/_runnable.py:299`（`invoke:374`、`ainvoke:452`） |
| `RunnableSeq` | 顺序链 Runnable（执行 → ChannelWrite） | `_internal/_runnable.py:584` |
| `is_async_callable` / `coerce_to_runnable` | 协程判定 / 函数转 Runnable | `_internal/_runnable.py:528`、`:550` |
| `set_config_context` | 配置上下文线程局部传递 | `_internal/_runnable.py:127` |
| `chain_future` / `run_coroutine_threadsafe` | sync↔async future 桥接 | `_internal/_future.py:127`、`:189` |
| `AsyncQueue` / `SyncQueue` / `Semaphore` | 队列与信号量并发原语 | `_internal/_queue.py:12`、`:70`、`:44` |
| `_serde` | 严格 msgpack 序列化 allowlist 构建 | `_internal/_serde.py` |
| `GraphBubbleUp` 异常族 | 图内部控制流异常基类（Interrupt/ParentCommand/Drained） | `errors.py:50` |
| `ErrorCode` | 机器可读错误码枚举 | `errors.py:34` |
| 回调集成 | langchain callbacks 桥接 | `callbacks.py` |
| 其他小设施 | `_cache/_constants/_fields/_pydantic/_replay/_retry/_timeout/_typing/_scratchpad` | `_internal/` |

## 2. 核心类型与接口清单

| 类型 / 函数 | 位置 | 职责 |
| --- | --- | --- |
| `RunnableCallable` | `_internal/_runnable.py:299` | 本库最常用的节点包装基类；同时持 sync 与 async 入口 |
| `RunnableSeq` | `_internal/_runnable.py:584` | 顺序组合多个 Runnable |
| `AsyncQueue` / `SyncQueue` | `_internal/_queue.py:12`/`:70` | 事件/任务在生产者消费者间传递的队列 |
| `GraphBubbleUp` | `errors.py:50` | 非错误控制流异常基类；`GraphInterrupt`/`ParentCommand`/`GraphDrained` 继承之 |
| `NodeError` / `NodeTimeoutError` / `NodeCancelledError` | `errors.py:149`/`:190`/`:168` | 节点级失败包装 |
| `PregelScratchpad` | `_internal/_scratchpad.py:9` | 运行期上下文数据类（见 managed-values 叶子） |

## 3. 关键调用链

1. **配置合并**：上游传入多个 config → `merge_configs(*configs)`（`_config.py:147`）逐键合并，`CONF` 子键递归合并（`:175`），callbacks 经 `_merge_callbacks`（`:119`）聚合。
2. **打补丁**：`patch_config`（`_config.py:194`）把新 `configurable` 写进 `config[CONF]`（`:232`）；`patch_configurable`（`:52`）支持空 config 起始。
3. **节点包装**：用户函数 → `RunnableCallable(sync, async, name, ...)`（`_runnable.py:299`）；`invoke`（`:374`）/`ainvoke`（`:452`）按函数是否协程选轨，sync 经 `run_in_executor`。
4. **future 桥接**：跨事件循环传递结果用 `chain_future`（`_future.py:82`）把源 future 状态拷到目标；`run_coroutine_threadsafe`（`:189`）在线程里跑协程。
5. **异常控制流**：`GraphInterrupt`（`errors.py:102`）等人机中断不是错误，而是 `GraphBubbleUp` 派生的控制信号，沿调用栈冒泡到 Pregel 顶层捕获。

## 4. 配置项

| 配置项 | 默认 / 行为 | 位置 |
| --- | --- | --- |
| `config[CONF]` | 内部运行期配置命名空间 | `_internal/_constants.py`（`CONF` 键） |
| `CONFIG_KEY_CHECKPOINT_ID/MAP/NS` | checkpoint 相关内部键 | `_config.py:25-27` 导入 |
| `recursion_limit` | 最大 superstep 数 | 经 config 透传 |
| `callbacks` | 运行期回调列表 | `_config.py` 合并 |
| `CACHE_NS_WRITES` / `PREVIOUS` | 缓存与上一次值通道键 | `_internal/_constants.py` |

## 5. 错误与重试语义

- `GraphBubbleUp`（`errors.py:50`）是控制流异常，Pregel 顶层捕获后按类型分支（中断/取消/ draining），不视为节点失败。
- 节点真正失败包装为 `NodeError`（`errors.py:149`）；超时 → `NodeTimeoutError`（`:190`）；取消 → `NodeCancelledError`（`:168`）。
- `GraphRecursionError`（`:67`）超过 `recursion_limit` 抛出。
- 重试策略在 `_internal/_retry.py`（薄封装）与各 Runnable 的 retry 配置，本叶子只提供基元。

## 6. 并发细节

- `AsyncQueue`（`_queue.py:12`）基于 `asyncio.Queue`；`SyncQueue`（`:70`）提供线程侧等价物；`Semaphore`（`:44`）限并发。
- `chain_future`（`_future.py:82`）用 done 回调把一个 future 的结果/异常拷给另一个，跨线程/跨事件循环桥接，无需忙等。
- `set_config_context`（`_runnable.py:127`）用 contextvars 在线程/任务间传递当前 config；`create_task_in_config_context`（`:143`）派生任务时继承上下文。
- 锁/临界区：队列内部用 asyncio 原语；本叶子不暴露业务锁。

## 7. 系统边界

**In-Scope（本仓库源码内）**
- `_internal/` 全部 15 个模块、`utils/`、顶层 `config.py`/`errors.py`/`constants.py`/`callbacks.py`/`warnings.py`/`typing.py`/`version.py`。

**Out-of-Scope（不在本仓库源码内）**
- `langchain_core.runnables` 的 Runnable 基类与 callback manager 完整实现：为外部依赖。
- checkpoint saver、store、cache 的具体后端：见 checkpoint-storage 叶子。
- Pregel 如何调度这些原语：见 pregel-engine 叶子。

## 8. 与相邻子系统交互

- 上游 → 本叶子：所有图/节点/func-api/stream 都 import `_internal` 的配置、Runnable、队列、异常。
- 本叶子 → 下游：被 pregel-engine、graph-state-api、func-api、stream-framework、remote-debug 广泛复用；不反向依赖业务叶子。
- 方向：`本叶子（横切设施）→ 被各运行期叶子 import`；config 由用户/编译期经 merge_configs 流向节点执行。

## 9. 语言专项适配口径

按纯 Python 框架口径：
- **能力缝归组**：`RunnableCallable`/`RunnableSeq` 是"可运行对象"契约缝；`GraphBubbleUp` 异常族是控制流缝。
- **双轨实现**：sync/async 双轨贯穿——RunnableCallable 同时持 sync/async 入口、`AsyncQueue`/`SyncQueue` 成对、`chain_future` 跨轨桥接。
- **配置面工程**：`RunnableConfig` 的 `CONF` 子键体系 + `merge_configs`/`patch_config` 合并/补丁原语，是框架级配置面。
- **contextvars 上下文传播**：`set_config_context` 用 contextvars 在并发任务间传配置，是 Python 框架的标准并发上下文模式。
- **控制流异常**：用异常（`GraphBubbleUp` 派生）表达中断/取消等非错误控制流，沿栈冒泡，是解释器式框架的常见手法。

## 10. 图表清单与质量

| 图 | 文件 | 类型 | archify 质量档 |
| --- | --- | --- | --- |
| 内部基础设施架构 | `internal-foundation-architecture.html` | architecture | showcase |
| 运行配置合并数据流 | `internal-foundation-dataflow.html` | dataflow | standard |

JSON IR 源文件位于 `json/` 目录。**dataflow 降档披露**：`patch_config` 到 `CONF`/回调管理器两条并行边的标签反复与目标节点重叠（layout 校验 3 轮：先 labelDx 错开、再 labelDy 下移均未过 showcase），最终按"短标签 + labelDy"勉强通过渲染，自动回退 standard；两条边的语义（写 CONF、聚合回调）已在卡片与本文第 3 节说明。本叶子未补 sequence / lifecycle 图：基础设施是静态工具集，无单次多方时序交互，也无单实体状态机，按资源节省原则省略。
