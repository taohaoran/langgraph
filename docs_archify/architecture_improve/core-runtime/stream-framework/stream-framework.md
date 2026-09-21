# 流式输出框架（stream-framework）

> 本文是 `core-runtime` 域下的叶子子系统文档。域级总览见 `../core-runtime.md`。
> 本文只展开 v3 流式基础设施：事件协议、多路复用器、流转换器、单消费者队列与运行流迭代器；
> 不重复展开 Pregel 如何产生原始流（见 `../pregel-engine/pregel-engine.md`）。
>
> 源码基准：`langgraph` main 分支，commit `e539ac122f4126f6dd850581c1494948cf620e31`。

## 1. 功能清单

| 能力 | 说明 | 源码路径 |
| --- | --- | --- |
| `ProtocolEvent` | 统一事件信封（`method`=流模式、`params.namespace/timestamp/data/interrupts`、根 mux 分配的单调 `seq`） | `stream/_types.py:28` |
| `StreamTransformer`（ABC） | 流转换扩展点：`init()` 返回投影字典、`process/aprocess` 观察事件、`finalize/afinalize`/`fail/afail` 收尾、`schedule()` 调度异步任务 | `stream/_types.py:44` |
| `StreamMux` | 中央事件分发器：按注册顺序跑 transformer 流水线、合并投影为 `extensions`、自动接线命名 `StreamChannel`、`_make_child` 为子图套小 mux | `stream/_mux.py:26` |
| `StreamChannel` | 单消费者可排空队列；命名时 push 自动转发为 `ProtocolEvent`；调用者驱动 pump 背压；`tee/atee` 扇出 | `stream/stream_channel.py:14` |
| `convert_to_protocol_event` | 把 v2 `StreamPart(type, ns, data, interrupts)` 转成 `ProtocolEvent` | `stream/_convert.py:10` |
| 内置转换器 `ValuesTransformer` | values 模式投影（`required_stream_modes=("values",)`） | `stream/transformers.py:28` |
| `UpdatesTransformer` | updates 模式投影 | `transformers.py:120` |
| `MessagesTransformer` | messages 模式投影（ eagerly 快照文本字段） | `transformers.py:155` |
| `CustomTransformer` | custom 模式投影 | `transformers.py:85` |
| `LifecycleTransformer` / `SubgraphTransformer` | tasks 事件→生命周期状态；子图发现并 `_make_child` 套小 mux | `transformers.py:608` / `:670` |
| `CheckpointsTransformer` / `DebugTransformer` / `TasksTransformer` | checkpoints/debug/tasks 模式投影 | `transformers.py:927` / `:966` / `:1002` |
| `GraphRunStream` / `AsyncGraphRunStream` | 运行流迭代器：`__iter__/__aiter__` 驱动 pump、`extensions` 取投影、`interleaved()` 合并多路、`output/interrupted/abort` | `stream/run_stream.py:57` / `:357` |
| `SubgraphRunStream` / `AsyncSubgraphRunStream` | 子图运行流（从父 mux 借 pump） | `run_stream.py:659` / `:702` |

## 2. 核心类型与接口清单

| 类型 / 函数 | 位置 | 职责 |
| --- | --- | --- |
| `StreamTransformer` | `_types.py:44` | 抽象基类；`before_builtins`/`required_stream_modes`/`requires_async`/`_native` 类属性 |
| `StreamMux` | `_mux.py:26` | 事件总线；`push/apush` 分发、`close/fail` 收尾、`extensions` 合并投影 |
| `StreamChannel[T]` | `stream_channel.py:14` | 单消费者队列；`_bind`/`_bind_mux`/`_wire` 内部接线 |
| `TransformerFactory` | `_mux.py:15` | `(scope) -> StreamTransformer`，子 mux 用 |
| `ProtocolEvent` | `_types.py:28` | 事件信封 TypedDict |
| `GraphRunStream` | `run_stream.py:57` | 同步运行流门面 |

## 3. 关键调用链

**链一：一个事件从 Pregel 到用户迭代**

1. Pregel 每产出一个 v2 `StreamPart(type, ns, data, interrupts)` → `convert_to_protocol_event`（`_convert.py:10`）包成 `ProtocolEvent`。
2. `StreamMux.push(event)`（`_mux.py:269`）按注册顺序调每个 transformer 的 `process(event)`；任一返回 False 则事件被抑制出主日志（已看到的 transformer 副作用保留）。
3. 事件保留 → 根 mux 分配单调 `seq` → 追加到主事件日志 `self._events`（`_mux.py:293`）。
4. 各 transformer 把派生数据 push 到自己的 `StreamChannel`；命名 channel 经 `_wire` 自动转发为另一事件（绕过 transformer 流水线防递归，`_mux.py:496`）。
5. 用户 `for event in run_stream`（`run_stream.py:254`）→ `_pump_next`（`:160`）驱动 pump 向 Pregel 要下一批事件 → 消费主日志。`run_stream.values` 等属性经 `__getattr__`（`:95`）返回对应 `StreamChannel`。

**链二：子图流**

- `SubgraphTransformer`（`transformers.py:670`）发现子图 handle 时调 `mux._make_child(scope)`（`_mux.py:193`）按 factories 克隆一条 transformer 流水线；子 mux 不分配 `seq`，继承父 pump，转发事件对象共享。

**链三：异步流水线**

- `apush`（`_mux.py:351`）await 每个 transformer 的 `aprocess`（慢速 aprocess 串行化流水线，保证后续 transformer 看到异步结果）；`schedule(coro)` 调度解耦异步任务，`aclose` 统一 join、`afail` 取消。

## 4. 配置项

| 配置项 | 默认 / 行为 | 位置 |
| --- | --- | --- |
| `stream_mode` | 请求的流模式集合；`required_stream_modes` 并集决定 v3 run 向 Pregel 要哪些模式 | `_types.py:114` |
| `transformers=[...]`（编译时） | 用户自定义 `StreamTransformer` 工厂；`before_builtins=True` 的先注册（PII 脱敏等内容改写） | `_mux.py:65` |
| `StreamChannel(name=...)` | 命名→自动转发为 `custom:<name>`；`_native` transformer 的 channel 直接用名 | `_mux.py:486` |
| `schedule(on_error="log"/"raise")` | 调度任务失败：记录日志 vs 上抛进 fail 路径 | `_types.py:238` |
| `is_async=True` | mux 异步派发；要求 async 的 transformer 不能在同步 `stream()` 下注册 | `_mux.py:234` |

## 5. 错误与重试语义

- **异步 transformer 误用同步**：`transformer_requires_async` 检测到 `aprocess/afinalize/afail` 被覆写且未声明 `supports_sync`，注册到同步 mux 时抛 `RuntimeError`（`_mux.py:234`）。
- **投影键冲突**：两个 transformer 返回同名投影键抛 `ValueError` 并标注归属（`_mux.py:246`）。
- **`init()` 非 dict**：抛 `TypeError`（`_mux.py:241`）。
- **收尾容错**：`close()` 中某个 transformer `finalize` 失败不阻塞其余收尾，首个异常在清理后重抛（`_mux.py:312-324`）；`fail()` 中异常被忽略以保证清理完成。
- **调度任务错误**：`schedule(on_error="log")` 包裹任务捕获并记录（`_types.py:286`）；`on_error="raise"` 在 `aclose` join 时上抛，转 fail 路径。
- 本叶子不含重试；流取消由 `abort()`/`afail` 处理。

## 6. 并发细节

- **调用者驱动 pump（背压核心）**：`StreamChannel` 无界内存靠"每次游标前进至多产一事件"约束（`stream_channel.py:38`）；`_request_more/_arequest_more` 回调在每次消费时向 Pregel 要更多。
- **单消费者**：`StreamChannel` 只允许一个订阅者，第二次 `__iter__` 抛错；扇出用 `tee/atee`（`stream_channel.py:26`）。
- **同步/异步双轨**：`GraphRunStream`+线程 pump vs `AsyncGraphRunStream`+asyncio pump；异步 `apush` 串行 await transformer，`schedule()` 解耦并发。
- **seq 全序**：仅根 mux 在事件入主日志前分配 `seq`（`_mux.py:293`），转发事件先入、原始事件后入，保证单调；子 mux 不分配、共享信封对象避免拷贝。
- **ContextVar**：`_tool_call_writer`（见 `pregel-io-checkpoint`）在工具回调中桥接 writer。

## 7. 系统边界

**In-Scope（本仓库源码内）**
- `stream/_types.py`（协议事件 + transformer ABC）、`_mux.py`（多路复用器）、`stream_channel.py`
- `run_stream.py`（运行流迭代器）、`transformers.py`（内置转换器族）、`_convert.py`

**Out-of-Scope（不在本仓库源码内）**
- Pregel 原始 `StreamPart` 的产生 → `pregel/`（`../pregel-engine/`）
- SSE/Web 传输、LangChain `astream_events` v1 适配 → 第三方/服务层
- LLM token 事件源 → LangChain callback（外部依赖）

## 8. 与相邻子系统交互

- **上游 ← pregel-engine / pregel-io-checkpoint**：Pregel 超级步产出 `StreamPart`；`StreamMessagesHandler`/`StreamToolCallHandler`（`pregel/_messages.py`、`_tools.py`）把 LLM/工具事件推入 mux。
- **下游 → 用户**：`GraphRunStream.__iter__`/`interleaved()` 把投影通道暴露给用户消费。
- **下游 → pregel-engine**：`bind_pump/bind_apump` 把 pump 回调接到 mux 各投影，实现消费驱动生产。

## 9. 语言专项适配口径

本项目为纯 Python 框架，按纯 Python 专项清单分析：

1. **能力缝归组**：`StreamTransformer` ABC + `init()→投影字典` + `process()` 观察事件，是典型"观察者/投影"能力缝——transformer 是可插拔观察者，mux 是事件总线。
2. **注册表/工厂**：`TransformerFactory=(scope)→transformer` 工厂 + `_make_child` 克隆到子图作用域；`required_stream_modes` 类属性声明依赖，mux 据此向 Pregel 汇总所需模式。
3. **插件体系**：用户 `transformers=[...]` 注入自定义 transformer；`before_builtins` 提供内容改写优先级钩子（PII 脱敏先于消息快照）。
4. **双轨实现**：同步 `push/close` 与异步 `apush/aclose/afail` 双轨；`StreamChannel` 同步/异步迭代协议按 `_bind(is_async)` 二选一。
5. **配置驱动分派**：`stream_mode` 集合决定挂接哪些内置 transformer；`_native` 决定投影键直接作为 run 属性暴露。

## 10. 图表清单与质量

| 图 | 文件 | 类型 | archify 质量档 |
| --- | --- | --- | --- |
| 流式事件管道 | `stream-framework-dataflow.html` | dataflow | standard |
| 流式框架组件分层 | `stream-framework-architecture.html` | architecture | standard |
| stream_mode 分发链时序 | `stream-framework-sequence.html` | sequence | standard |

- 降档说明：dataflow 图受"同列纵向边穿越节点/边标签间距"约束降 standard；本次新增 architecture 图因"chan→user 长边穿越中间组件"改用 orthogonal-v 路由后降 standard；新增 sequence 图因"segment 超出画布/消息超出时间轴"压缩 y 后仍未全过 showcase，按规范降 standard 渲染（三张均成功，约 805KB 自包含）。
- JSON IR 源：`json/stream-framework-dataflow.json`、`json/stream-framework-architecture.json`、`json/stream-framework-sequence.json`。
- 本次迭代补齐 architecture（组件分层）与 sequence（stream_mode 分发链：values/updates/debug 路由），与既有 dataflow（事件管道）形成"管道+拓扑+时序"三图互补。
