# 函数式 API（func-api）

> 本文是 `core-runtime` 域下的叶子子系统文档。域级总览见 `../core-runtime.md`（输出根统一产出）。
> 本文只展开 `@entrypoint` / `@task` 装饰器如何把普通 Python 函数编译/桥接到 Pregel 图；
> 不重复展开 Pregel 单帧执行算法本身（见 `../pregel-engine/pregel-engine.md`）与 task 的
> call 实现如何注册进 config（同见 pregel-engine）。
>
> 源码基准：`langgraph` main 分支，commit `e539ac122f4126f6dd850581c1494948cf620e31`。

## 1. 功能清单

| 能力 | 说明 | 源码路径 |
| --- | --- | --- |
| `@task` 装饰器 | 把普通函数标记为可在图内异步调用的任务；支持 retry/cache/timeout/name 参数 | `func/__init__.py:110` |
| `_TaskFunction` | task 包装对象；`__call__` 立即返回 `SyncAsyncFuture` | `func/__init__.py:59`、`:86` |
| `@entrypoint` 装饰器 | 把普通函数编译成单节点 `Pregel` 图；挂 checkpointer/store/cache | `func/__init__.py:262`、`:516` |
| `entrypoint.final` | 数据类原语：解耦"返回给调用方的值"与"存入 checkpoint 的值" | `func/__init__.py:475` |
| `get_runnable_for_entrypoint` | 把用户函数包成 `RunnableCallable`（sync 走线程池），带缓存 | `pregel/_call.py:175` |
| `get_runnable_for_task` | 把 task 函数包成 `RunnableSeq(执行 → ChannelWrite(RETURN))` | `pregel/_call.py:200` |
| `_call_with_options` | task 调用入口：从运行 config 取 call 实现并派发 | `pregel/_call.py:276` |
| `SyncAsyncFuture` | 兼容同步/异步的 Future（可 await） | `pregel/_call.py:253` |
| 包导出 | 仅导出 `task`、`entrypoint` | `func/__init__.py:56` |

## 2. 核心类型与接口清单

| 类型 / 函数 | 位置 | 职责 |
| --- | --- | --- |
| `task(...)` | `func/__init__.py:110` | 带参装饰器工厂；返回 `decorator`，真正包装在 `:234` |
| `_TaskFunction[P,T]` | `func/__init__.py:59` | 持有原函数 + retry/cache/timeout；`__call__` 是扩展点 |
| `entrypoint[ContextT]` | `func/__init__.py:262` | 装饰器类（为支持 `.final` 属性）；`__init__` 收图级配置，`__call__` 编译图 |
| `entrypoint.final[R,S]` | `func/__init__.py:475` | `value`（返回）+ `save`（存 checkpoint）双值原语 |
| `RunnableCallable` | `_internal/_runnable.py` | langchain_core Runnable 基类的 langgraph 封装（sync/async 双轨） |
| `RunnableSeq` | `langgraph.constants` 体系 | 顺序链：执行函数 → 写 `RETURN` 通道 |

## 3. 关键调用链

1. **entrypoint 编译**：`@entrypoint(checkpointer=...)(func)` → `entrypoint.__call__`（`func/__init__.py:516`）。先拒绝生成器函数（`:526-528`），再用 `get_runnable_for_entrypoint(func)`（`:531`，实现在 `_call.py:175`）包成 Runnable。
2. **单节点图组装**：`func/__init__.py:576-609` 构造 `Pregel(nodes={func.__name__: PregelNode(bound=bound, triggers=[START], channels=START, writers=[ChannelWrite([END ← _pluck_return_value, PREVIOUS ← _pluck_save_value])])}, channels={START: EphemeralValue, END: LastValue, PREVIOUS: LastValue}, input_channels=START, output_channels=END)`。
3. **task 调用入图**：用户在 entrypoint 里调 `task(*args)` → `_TaskFunction.__call__`（`func/__init__.py:86`）→ `_call_with_options`（`_call.py:276`）。
4. **call 实现派发**：`_call.py:288-290` 取 `config = get_config()`，`impl = config[CONF][CONFIG_KEY_CALL]`，调 `impl(func, (args, kwargs), retry_policy=..., cache_policy=..., callbacks=..., timeout=...)` 返回 future。该 impl 由正在运行的 Pregel 帧注入。
5. **task 节点收尾**：`get_runnable_for_task`（`_call.py:232-234`）把执行结果经 `ChannelWrite([ChannelWriteEntry(RETURN)])` 写回图。

## 4. 配置项

| 配置项 | 默认 / 行为 | 位置 |
| --- | --- | --- |
| `checkpointer` | 提供后可持久化、跨次调用读 `previous` | `func/__init__.py:467` |
| `store` | 通用 KV 存储（可带语义索引） | `func/__init__.py:468` |
| `cache` / `cache_policy` | 任务结果缓存 | `func/__init__.py:469-470` |
| `retry_policy` | 任务失败重试策略序列 | `func/__init__.py:471` |
| `timeout` | 经 `coerce_timeout_policy` 归一；仅 async 函数支持 | `func/__init__.py:472`、`_call.py:285` |
| `context_schema` | run 级上下文 schema（旧名 `config_schema` 已弃用） | `func/__init__.py:473`、`:449` |
| `stream_mode` | entrypoint 内固定 `"updates"`、`stream_eager=True` | `func/__init__.py:532`、`:602` |

## 5. 错误与重试语义

- task 失败由 `retry_policy`（`RetryPolicy` 序列）决定重试与退避；`_call_with_options` 把它透传给 call 实现（`_call.py:293`）。
- sync 函数配 `timeout` 直接抛 `sync_timeout_unsupported`（`_call.py:285-287`，装饰期 `func/__init__.py:235` 也会预检）。
- `entrypoint.final` 返回注解类型不匹配（未参数化或参数个数≠2）时抛 `TypeError`（`func/__init__.py:565-571`）。
- 生成器/异步生成器函数作 entrypoint 抛 `NotImplementedError`（`func/__init__.py:527`）。

## 6. 并发细节

- sync 用户函数被包成 `RunnableCallable(func, afunc=run_in_executor(None, func), ...)`（`_call.py:185-194`），即同步任务在线程池执行，不阻塞事件循环。
- task 调用立即返回 `SyncAsyncFuture`（`_call.py:253`，继承 `concurrent.futures.Future`），真实执行由 Pregel 帧调度；future 可被 `await`（`__await__` 在 `:254`）。
- `CACHE: dict[(func, is_task), Runnable]`（`_call.py:245`）按函数对象缓存已包装的 Runnable，避免重复包装；仅当函数可被 `_lookup_module_and_qualname` 定位时才缓存。
- 取消/超时沿 Pregel 的 config/callback 体系传播，本叶子不另起线程。

## 7. 系统边界

**In-Scope（本仓库源码内）**
- `func/__init__.py`：`task`/`entrypoint` 装饰器与 `entrypoint.final`。
- `pregel/_call.py`：Runnable 包装、call 派发、future。

**Out-of-Scope（不在本仓库源码内）**
- Pregel 如何在帧内执行节点、如何把 call 实现注入 config：见 pregel-engine 叶子。
- checkpointer / store / cache 的具体实现：见 checkpoint-storage 与 pregel-io-checkpoint 叶子（Postgres/SQLite saver 不在本仓库运行库）。
- `RunnableCallable` 的完整同步/异步原语：`_internal/_runnable.py` 属 internal-foundation 叶子。
- 第三方 langchain_core Runnable 协议：为其上游依赖，不在本仓库源码内。

## 8. 与相邻子系统交互

- 上游 → 本叶子：用户 import `from langgraph.func import entrypoint, task`，用装饰器标注函数。
- 本叶子 → 下游：`entrypoint.__call__` 构造的 `Pregel` 对象交给 pregel-engine 执行；`task` 调用经 `config[CONF][CONFIG_KEY_CALL]` 反向回调正在运行的 Pregel。
- 方向：`用户代码 → func 装饰器 → Pregel（编译产物）→ pregel-engine（运行）→ call 实现回注 task future`。

## 9. 语言专项适配口径

按纯 Python 框架口径：
- **装饰器与元类扩展点**：`@entrypoint`/`@task` 是纯 Python 装饰器扩展点；`entrypoint` 被写成类而非函数，正是为了让 `entrypoint.final` 这个属性在 IDE/类型检查器里有自动补全（`func/__init__.py:260` 注释）。
- **注册表与缓存工厂**：`_call.py:245` 的 `CACHE` 是"函数对象 → Runnable"的包装注册表，import/首次调用时惰性填充。
- **双轨实现模式**：sync/async 双轨——同一函数同时包 sync 入口与 `run_in_executor` 的 afunc（`_call.py:180-194`），运行时按 `is_async_callable` 选轨。
- **配置驱动分派**：task 不直接执行函数，而是从 `config[CONF][CONFIG_KEY_CALL]` 取运行图注入的实现——同一 task 在普通上下文与 Pregel 上下文行为不同，是典型配置驱动静态分派。
- **隐藏状态机在循环里**：task future 的 created→scheduled→running→done/failed 状态机隐藏在 Pregel 帧循环中，故补 lifecycle 图。

## 10. 图表清单与质量

| 图 | 文件 | 类型 | archify 质量档 |
| --- | --- | --- | --- |
| 函数式 API 架构 | `func-api-architecture.html` | architecture | showcase |
| task 调用入图时序 | `func-api-sequence.html` | sequence | showcase |
| task future 生命周期 | `func-api-lifecycle.html` | lifecycle | showcase |

JSON IR 源文件位于 `json/` 目录。本叶子未补 dataflow / workflow 图：函数式 API 是"编译为图 + 调用派发"，无独立数据管道，也无多角色泳道审批流程，与已有图信息重复，按资源节省原则省略。
