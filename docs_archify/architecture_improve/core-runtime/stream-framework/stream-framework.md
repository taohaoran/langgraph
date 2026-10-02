# 流式框架（stream-framework）

> 本文是 `core-runtime` 域下的叶子子系统文档。域级总览见 `../core-runtime.md`（输出根统一产出）。
> 本文只展开 `stream/` 包如何把 Pregel 原始事件投影成用户可迭代的多模式流；
> 不重复展开 Pregel 如何产生 StreamPart（见 `../pregel-engine/pregel-engine.md`）。
>
> 源码基准：`langgraph` main 分支，commit `e539ac122f4126f6dd850581c1494948cf620e31`。

## 1. 功能清单

| 能力 | 说明 | 源码路径 |
| --- | --- | --- |
| `StreamTransformer`（ABC） | 流式投影扩展点：`init()` 返回投影 dict，`process(event)->bool` 观察事件 | `stream/_types.py:44` |
| `ProtocolEvent` | 统一事件信封 TypedDict（`type/method/params.namespace/params.data`） | `stream/_types.py:28` |
| `convert_to_protocol_event` | 把 v2 `StreamPart` 转成 `ProtocolEvent`（补 namespace/timestamp） | `stream/_convert.py:14` |
| `StreamMux` | 事件多路复用器：按注册序把事件喂给 transformer 链，维护主事件日志 | `stream/_mux.py:26` |
| `StreamChannel[T]` | 类型化投影通道：被 transformer 写入、被用户迭代（支持 sync/async、tee） | `stream/stream_channel.py:14` |
| `GraphRunStream` / `AsyncGraphRunStream` | 用户面对的运行流对象：`__iter__`/`__aiter__` 拉取主日志 | `stream/run_stream.py:57`、`:357` |
| `SubgraphRunStream` / `AsyncSubgraphRunStream` | 子图版本运行流（带 namespace mixin） | `stream/run_stream.py:659`、`:702` |
| 内置 transformers | `UpdatesTransformer`/`MessagesTransformer`/`LifecycleTransformer`/`SubgraphTransformer`/`CustomTransformer`/`TasksTransformer`/`CheckpointsTransformer`/`DebugTransformer`/`ValuesTransformer` | `stream/transformers.py` |
| 包导出 | `stream/__init__.py` 暴露上述公开类型 | `stream/__init__.py:19` |

## 2. 核心类型与接口清单

| 类型 / 函数 | 位置 | 职责 |
| --- | --- | --- |
| `StreamTransformer` | `stream/_types.py:44` | 抽象基类；类属性 `requires_async`/`supports_sync`/`required_stream_modes`/`before_builtins`；钩子 `init/process/aprocess/finalize/fail` |
| `StreamMux` | `stream/_mux.py:26` | 持有 transformer 列表、`extensions`（投影合并 dict）、主事件队列；`push()`/`close()`/`fail()` |
| `StreamChannel[T]` | `stream/stream_channel.py:14` | 泛型投影通道；`push()` 写、`__iter__`/`__aiter__` 读、`tee()/atee()` 分支 |
| `GraphRunStream` | `stream/run_stream.py:57` | 同步运行流；`__iter__`（`:254`）驱动 pump 直到完成 |
| `LifecycleTransformer` | `stream/transformers.py:608` | 跟踪任务开始/终态，产出 per-task 生命周期投影 |
| `SubgraphTransformer` | `stream/transformers.py:670` | 跟踪子图状态（`SubgraphStatus`） |
| `CustomTransformer` | `stream/transformers.py:85` | 用户自定义投影的基类 |

## 3. 关键调用链

1. **事件进入**：Pregel 产出 v2 `StreamPart`，`convert_to_protocol_event`（`_convert.py:14`）补 `namespace`/`timestamp`/`interrupts` 得到 `ProtocolEvent`。
2. **mux 分发**：`StreamMux.push(event)`（`_mux.py:269`）按注册序遍历 `self._transformers`，逐个调 `transformer.process(event)`（`:289-291`）；任一返回 `False` 则 `keep=False`，事件不进主日志但已观察的 transformer 副作用保留。
3. **入日志**：若 `keep`，根 mux 给事件分配递增 `seq`（`:293-295`）后 `self._events.push(event)`（`:296`）。
4. **投影写入**：transformer 在 `process` 内把派生数据 `push` 进自己 `init()` 时声明的 `StreamChannel`；mux 在 `_register`（`:227`）时把这些 channel 绑定并合入 `extensions`（`:260`）。
5. **用户消费**：`GraphRunStream.__iter__`（`run_stream.py:254`）/`__aiter__`（`:626`）驱动 pump 拉取主日志；用户也可直接 `async for` 某个投影 `StreamChannel`。

## 4. 配置项

| 配置项 | 默认 / 行为 | 位置 |
| --- | --- | --- |
| `transformers=[...]` | 编译图时注册自定义 transformer 链 | `stream/__init__.py:4`（docstring） |
| `stream_events(version="v3")` | 驱动 transformer 管道的流式入口 | `stream/__init__.py:5` |
| `before_builtins` | True 时排在内置 transformer 之前（用于内容改写如 PII 脱敏） | `_types.py` 类属性 |
| `required_stream_modes` | 各 transformer 声明所需 stream mode，mux 取并集决定向图请求哪些 mode | `_types.py` 类属性 |
| `StreamChannel(maxlen=...)` | 投影通道缓冲长度 | `stream_channel.py:49` |

## 5. 错误与重试语义

- 任一 transformer 的 `process` 抛错，mux 经 `fail(err)`（`_mux.py:326`）把错误传播到所有投影 channel 与主日志，终止本次流。
- `transformer_requires_async` 判断某 transformer 需 async 但在同步 `stream()` 下注册时，`_register` 直接 `raise RuntimeError`（`_mux.py:234-239`）。
- `init()` 未返回 dict 抛 `TypeError`（`_mux.py:241-245`）；投影 key 与已有冲突抛 `ValueError`（`:252-256`）。
- 无自动重试；流式是单次单向管道，失败即终态。

## 6. 并发细节

- sync/async 双轨：同步路径 `GraphRunStream.__iter__` + pump（`run_stream.py:24` `_drive_until_done`）；异步路径 `AsyncGraphRunStream.__aiter__`。
- `StreamChannel` 用队列缓冲生产-消费；`tee()/atee()`（`stream_channel.py:245`/`:288`）把单通道扇出为多个迭代器。
- 异步 transformer 可 `self.schedule(coro)` 调度解耦后台任务，mux 在关闭时 `_collect_scheduled_tasks`（`_mux.py:451`）收敛。
- 子图 mux（`_make_child`，`_mux.py:193`）共享事件对象但不分配 `seq`，避免重复信封。

## 7. 系统边界

**In-Scope（本仓库源码内）**
- `stream/` 全部模块：事件协议、mux、transformer 体系、StreamChannel、运行流。

**Out-of-Scope（不在本仓库源码内）**
- Pregel 如何产生 v2 StreamPart、如何调度帧：见 pregel-engine 叶子。
- checkpointer / store：见 checkpoint-storage 叶子。
- 终端渲染 / 前端展示：不在本仓库源码内。
- `langchain_core.runnables` 事件协议上游：为其依赖。

## 8. 与相邻子系统交互

- 上游 → 本叶子：pregel-engine 每帧产出 `StreamPart`，经 `_convert` 变成 `ProtocolEvent`。
- 本叶子 → 下游：transformer 把事件投影成 `StreamChannel`，最终由 `GraphRunStream` 交付用户；`extensions` 投影也挂在运行流对象上供属性访问。
- 方向：`Pregel → convert → StreamMux.push → transformer 链 → StreamChannel → GraphRunStream → 用户`。

## 9. 语言专项适配口径

按纯 Python 框架口径：
- **能力缝归组**：`StreamTransformer(ABC)` 是典型契约缝——内置 transformer（Updates/Messages/Lifecycle/Subgraph）与用户 `CustomTransformer` 都是实现方，mux 是宿主。
- **注册表与插件**：`transformers=[...]` 在编译期注入；mux 在 `_register` 时调 `init()` 拿投影 dict、按 `_projection_owners` 记录归属并检测 key 冲突，是运行时注册表。
- **可插拔钩子链**：`process/aprocess/finalize/afinalize/fail` 是钩子链；`before_builtins` 控制链内顺序，是内容改写类 transformer（PII 脱敏）的挂载点。
- **双轨实现**：sync `process`/async `aprocess` 双轨，`supports_sync` 标记可跨轨；`requires_async` 在同步流下硬拒绝。
- **数据流管道**：事件单向流动、transformer 依次观察、投影即数据汇，天然是 dataflow 语义，故补 dataflow 图。

## 10. 图表清单与质量

| 图 | 文件 | 类型 | archify 质量档 |
| --- | --- | --- | --- |
| 流式框架架构 | `stream-framework-architecture.html` | architecture | showcase |
| 流式事件管道数据流 | `stream-framework-dataflow.html` | dataflow | showcase |

JSON IR 源文件位于 `json/` 目录。dataflow 初版把三个内置 transformer 画成 mux 的三路扇出，因 `process`/`生命周期` 标签反复与相邻节点重叠（layout 校验 3 轮未过），按"删边+细节写文字"策略合并为单个 `transformers 链`节点，三类 transformer 的分工见第 1 节表。本叶子未补 sequence / lifecycle 图：事件分发的时序已由 dataflow 主路径表达，transformer 无独立单实体状态机，按资源节省原则省略。
