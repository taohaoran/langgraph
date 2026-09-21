# 函数式 API（func-api）

> 本文是 `core-runtime` 域下的叶子子系统文档。域级总览见 `../core-runtime.md`。
> 本文只展开 `@entrypoint`/`@task` 装饰器及其到 Pregel 的桥接；不重复展开 Pregel 执行
> （见 `../pregel-engine/pregel-engine.md`）与通道语义（见 `../graph-state-api/graph-state-api.md`）。
>
> 源码基准：`langgraph` main 分支，commit `e539ac122f4126f6dd850581c1494948cf620e31`。

## 1. 功能清单

| 能力 | 说明 | 源码路径 |
| --- | --- | --- |
| `entrypoint` 装饰器类 | 把一个普通函数包装成可持久化的工作流入口；配置 checkpointer/store/cache/retry/timeout/context_schema | `func/__init__.py:262` |
| `@task` 装饰器 | 把普通函数包装成可并行子任务，调用产出 `SyncAsyncFuture` | `func/__init__.py:132` |
| `_TaskFunction` | 任务包装：持有 retry/cache/timeout 策略，`__call__` 调 `_call_with_options` 返回 future | `func/__init__.py:59` |
| 自动建图 | `entrypoint.__call__` 把入口函数转成单节点 `Pregel`：START→函数→END/PREVIOUS 三通道 | `func/__init__.py:576` |
| 注入参数 | 入口函数可声明 `config`/`previous`/`runtime` 参数，运行期自动注入 | `func/__init__.py:271-280` |
| `entrypoint.final[R, S]` | 分离"返回值"与"保存值"：返回 R、保存 S 到 PREVIOUS 通道 | `func/__init__.py:546-574` |
| 类型推断 | 从函数签名首个参数推断 input_type、从返回注解推断 output/save_type | `func/__init__.py:535-574` |
| 任务缓存清理 | `clear_cache/aclear_cache` 按 `CACHE_NS_WRITES + identifier` 清任务缓存 | `func/__init__.py:96-106` |
| serde 白名单 | 启用严格 msgpack 时按 input/output/save/context schema 构建 serde 白名单 | `func/__init__.py:610-619` |

## 2. 核心类型与接口清单

| 类型 / 函数 | 位置 | 职责 |
| --- | --- | --- |
| `entrypoint` | `func/__init__.py:262` | 装饰器工厂；`__call__` 把函数转 `Pregel` |
| `task` | `func/__init__.py:132` | 装饰器；返回 `_TaskFunction` |
| `_TaskFunction[P, T]` | `func/__init__.py:59` | 任务包装；`__call__`→`SyncAsyncFuture` |
| `_TaskFunction.clear_cache` | `func/__init__.py:96` | 清任务结果缓存 |
| `get_runnable_for_entrypoint` | `pregel/_call.py`（导入） | 把入口函数包成注入 `config/previous/runtime` 的 Runnable |
| `_call_with_options` | `pregel/_call.py`（导入） | 带 retry/cache/timeout 调度子任务，返回 future |

## 3. 关键调用链

**链一：`@entrypoint` 如何变成 Pregel**

1. 用户 `@entrypoint(checkpointer=...)(def myfn(x): ...)`。
2. `entrypoint.__call__(func)`（`__init__.py:516`）拒绝生成器函数；`get_runnable_for_entrypoint(func)` 把函数包成能注入 `config/previous/runtime` 的 Runnable `bound`。
3. 从签名首参拿 `input_type`，从返回注解解析 `output_type`/`save_type`（支持 `entrypoint.final[R,S]`，`:554-574`）。
4. 构造单节点 `Pregel`（`:576`）：节点 `func.__name__`，`triggers=[START]`、`channels=START`；writers 把返回值写 `END`（经 `_pluck_return_value`）、保存值写 `PREVIOUS`（经 `_pluck_save_value`）。通道为 `START: EphemeralValue(input_type)`、`END: LastValue(output_type)`、`PREVIOUS: LastValue(save_type)`；`input_channels=START`、`output_channels=END`、`stream_eager=True`。
5. 启用严格 msgpack 时构建 serde 白名单并套到 checkpointer（`:610`）。返回 `Pregel`，用户可 `.invoke/.stream`。

**链二：`@task` 子任务并行**

1. 用户在 entrypoint 内调 `add_one_task(n)` → `_TaskFunction.__call__`（`:86`）→ `_call_with_options(func, args, kwargs, retry_policy=..., cache_policy=..., timeout=...)`，返回 `SyncAsyncFuture`。
2. 多个任务同时调用即产出多个 future（并行）；`future.result()` 等待并取结果。子任务经 `CONFIG_KEY_CALL` 递归调度进同一 Pregel 实例。

## 4. 配置项

| 配置项 | 默认 / 行为 | 位置 |
| --- | --- | --- |
| `entrypoint(checkpointer=...)` | 启用跨运行持久化；提供后 `previous` 参数可用 | `func/__init__.py:295` |
| `entrypoint(store=...)` | 长时记忆 KV store 注入 `runtime.store` | `:297` |
| `entrypoint(cache=...)` / `cache_policy=...` | 任务结果缓存后端与策略 | `:299-302` |
| `entrypoint(retry_policy=...)` | 全图重试策略（per-task 可覆盖） | `:303` |
| `entrypoint(timeout=...)` | 每尝试墙钟/空闲超时；仅异步任务支持取消 | `:304` |
| `entrypoint(context_schema=...)` | 运行期上下文 schema（替代已废弃 `config_schema`） | `:300` |
| `@task(name=..., retry_policy=..., cache_policy=..., timeout=...)` | 任务级策略覆盖 | `:114-116` |

## 5. 错误与重试语义

- **生成器不支持**：入口是生成器/异步生成器抛 `NotImplementedError`（`__init__.py:527`）。
- **无参数**：入口函数无参数抛 `ValueError`（`:538`）。
- **返回注解错误**：用了未参数化的 `entrypoint.final` 或只给一个类型参数抛 `TypeError`（`:558-571`）。
- **任务失败**：`_TaskFunction.__call__` 经 `_call_with_options` 应用 `retry_policy`；退避重试在 `pregel-engine` 的 `run_with_retry`。
- **超时**：`TimeoutPolicy` 触发 `NodeTimeoutError`，由重试策略决定是否重试；同步任务不可安全取消（文档明确，`:307-309`）。
- **任务只能在 entrypoint/StateGraph 内调用**：否则 `CONFIG_KEY_CALL` 未配置时报错（见 `_call.py`）。

## 6. 并发细节

- `@task` 的核心并发语义是"调用即产出 future"：子任务经 `_call_with_options` 提交到 Pregel 后台执行器，多个 future 可并行执行，`.result()` 同步等待（`func/__init__.py:86-94`）。
- 入口函数本身在单节点 Pregel 中执行；子任务递归调度通过 `CONFIG_KEY_CALL` 桥接，复用主引擎的并发池。
- `previous` 参数依赖 checkpointer：同一 thread_id 上一次的返回值/保存值在 `__enter__` 时水化注入。
- 异步入口/任务需要 Python ≥3.11（文档明确 `:146`）。

## 7. 系统边界

**In-Scope（本仓库源码内）**
- `func/__init__.py`（`entrypoint`/`task`/`_TaskFunction`）

**Out-of-Scope（不在本仓库源码内）**
- 子任务调度 `_call_with_options`/`SyncAsyncFuture`/`get_runnable_for_entrypoint` → `pregel/_call.py`（`../pregel-engine/`）
- Pregel 执行 → `pregel/`
- `BaseCheckpointSaver`/`BaseStore`/`BaseCache` 后端 → 外部库
- `langchain_core` Runnable → 第三方依赖

## 8. 与相邻子系统交互

- **上游**：用户 → 本叶子装饰器。
- **下游 → pregel-engine**：`entrypoint.__call__` 直接构造 `Pregel`（继承其执行引擎）；`@task` 经 `pregel/_call.py` 调度子任务。
- **下游 → graph-state-api**：复用 `EphemeralValue`/`LastValue` 通道与 `START/END` 常量。
- **下游 → internal-foundation**：复用 `_serde`/`_timeout`/`_runnable`/`_typing`。
- **外部**：checkpointer/store/cache 后端由外部库实现。

## 9. 语言专项适配口径

本项目为纯 Python 框架，按纯 Python 专项清单分析：

1. **能力缝归组**：`@entrypoint`/`@task` 装饰器是"声明式编排"能力缝——用户写普通函数，装饰器把它编译成 Pregel 图，是"函数→图"的静态分派。
2. **注册表/工厂**：无全局注册表；`entrypoint.__call__` 是"函数签名→单节点 Pregel"的工厂。
3. **配置驱动分派**：`checkpointer/store/cache/retry/timeout` 构造参数决定运行期行为；`entrypoint.final[R,S]` 注解决定返回/保存双通道。
4. **双轨实现**：同步/异步函数经 `is_async_callable`（`_internal/_runnable.py`）自动适配。
5. **可选依赖**：`langgraph.checkpoint.base`/`store.base`/`cache.base` 为重依赖但仅在配置对应参数时使用。

## 10. 图表清单与质量

| 图 | 文件 | 类型 | archify 质量档 |
| --- | --- | --- | --- |
| 函数式 API 与 Pregel 桥接 | `func-api-architecture.html` | architecture | standard |
| @entrypoint/@task 调用链 | `func-api-sequence.html` | sequence | standard |

- 降档说明：architecture 图因交叉边穿越节点降 standard；本次新增的 sequence 图（装饰→自动建图→Pregel.invoke→@task 子任务）因"segment 超出画布/返回消息超出时间轴"压缩布局后仍未全过 showcase，按规范降 standard 渲染（均成功，约 805KB 自包含）。
- JSON IR 源：`json/func-api-architecture.json`、`json/func-api-sequence.json`。
- 本次迭代新增 sequence 图，把"装饰→建图→invoke→@task 并发"的调用链从文字化升级为图；不另出 dataflow：函数式 API 是调用链而非数据管道。
