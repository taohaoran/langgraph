# 远端代理、调试与图绘制（remote-debug）

> 本文是 `core-runtime` 域下的叶子子系统文档。域级总览见 `../core-runtime.md`。
> 本文只展开 `RemoteGraph` 远端执行代理、`PregelProtocol` 抽象、debug 流事件映射、
> 图绘制 `draw_graph` 与远端流适配；不重复展开本地 Pregel 执行
> （见 `../pregel-engine/pregel-engine.md`）。
>
> 源码基准：`langgraph` main 分支，commit `e539ac122f4126f6dd850581c1494948cf620e31`。

## 1. 功能清单

| 能力 | 说明 | 源码路径 |
| --- | --- | --- |
| `PregelProtocol` | 抽象接口：`invoke/stream/get_graph/get_state/update_state/get_state_history` 等；本地 Pregel 与 RemoteGraph 共同实现 | `pregel/protocol.py:25` |
| `StreamProtocol` | 流回调协议（`modes` + `__call__(StreamChunk)`） | `protocol.py:275` |
| `RemoteGraph` | 远端 LangGraph Server 的客户端代理；行为同本地图，可作为子图节点 | `pregel/remote.py:118` |
| 同步/异步客户端 | 由 `url/api_key/headers` 构造 `LangGraphClient`/`SyncLangGraphClient`，或注入自定义 client | `remote.py:132-179` |
| 远端状态读写 | `get_state/get_state_history/update_state/bulk_update_state`（含 a 版）映射到远端线程 API | `remote.py:436-671` |
| 远端流式 | `stream/astream/stream_events`（v1/v2/v3），`_get_drawable_nodes`/`_create_state_snapshot` 把远端响应转成本地结构 | `remote.py:724-1131`、`:322` |
| v3 线协议适配 | `_translate_command_input`：本地 `Command(resume=...)` 映射为 v3 wire `input`；拒绝 `goto/update` | `pregel/_remote_run_stream.py:21` |
| `RemoteException` | 远端错误包装 | `remote.py:112` |
| debug 任务映射 | `map_debug_tasks`/`map_debug_task_results`/`map_task_result_writes`：stream_mode=debug 的 task/task_result 事件 | `pregel/debug.py:41`、`:106`、`:83` |
| debug checkpoint 映射 | `map_debug_checkpoint`/`tasks_w_writes`：拼 `StateSnapshot`（config/values/next/tasks） | `debug.py:144`、`:209` |
| ANSI 着色 | `get_colored_text`/`get_bolded_text`/`COLOR_MAPPING` | `debug.py:282-302` |
| `draw_graph` | 从 Pregel 节点/通道/触发表构造 `langchain_core.graph.Graph`（可绘制图） | `pregel/_draw.py:42` |
| `Edge`/`TriggerEdge` | 边描述 NamedTuple | `_draw.py:29`、`:36` |

## 2. 核心类型与接口清单

| 类型 / 函数 | 位置 | 职责 |
| --- | --- | --- |
| `PregelProtocol[StateT, ContextT, InputT, OutputT]` | `protocol.py:25` | 本地图与远端代理统一抽象（继承 `Runnable`） |
| `RemoteGraph(PregelProtocol)` | `remote.py:118` | 远端执行代理；`assistant_id` 定位远端图 |
| `StreamProtocol` | `protocol.py:275` | 流回调契约 |
| `_translate_command_input` | `_remote_run_stream.py:21` | v3 线协议命令适配 |
| `map_debug_*` 族 | `debug.py` | debug 流事件映射纯函数 |
| `draw_graph` | `_draw.py:42` | 图可视化构造 |

## 3. 关键调用链

**链一：经 RemoteGraph 远端执行**

1. 用户 `RemoteGraph(assistant_id, url=...).invoke(input, config)`。
2. `RemoteGraph` 构造时按 `url/api_key` 经 `get_client/get_sync_client` 创建同步/异步客户端（`remote.py:173-179`）。
3. `invoke`（`remote.py:1162`）/`stream`（`:757`）把输入经 `_translate_command_input`（`_remote_run_stream.py:21`）适配为 v3 wire 格式，调 SDK client 的线程/运行 API。
4. SDK 经 HTTP/SSE 与 `LangGraph Server` 通信；远端 `StateSnapshot` 经 `_create_state_snapshot`（`remote.py:322`）转成本地结构。
5. `RemoteGraph` 实现 `PregelProtocol`，因此可直接 `add_node(remote_graph)` 作为子图。

**链二：本地 debug 流事件生成**

- Pregel 步末把任务与 pending writes 交 `map_debug_tasks`（`debug.py:41`）产 task 事件、`map_debug_task_results`（`:106`）产 task_result 事件、`map_debug_checkpoint`（`:144`）产 checkpoint 事件（经 `tasks_w_writes` 把 pending writes 折进每个 task 的 result/interrupts/error）。

**链三：图绘制**

- `draw_graph(config, nodes, specs, trigger_to_nodes, ...)`（`_draw.py:42`）遍历 PregelNode 与触发表，构造 `langchain_core.runnables.graph.Graph`，供 `draw_mermaid_png` 等渲染。

## 4. 配置项

| 配置项 | 默认 / 行为 | 位置 |
| --- | --- | --- |
| `url` / `api_key` / `headers` | 远端 API 地址与鉴权；api_key 缺省读 `LANGGRAPH_API_KEY/LANGSMITH_API_KEY/LANGCHAIN_API_KEY` | `remote.py:153-156` |
| `client` / `sync_client` | 注入自定义客户端，替代默认 | `remote.py:157-158` |
| `distributed_tracing` | 是否发送 LangSmith 分布式追踪头 | `remote.py:163` |
| `version="v1"/"v2"/"v3"` | 流式协议版本；v3 仅支持 `Command(resume=...)` | `_remote_run_stream.py:34-39` |
| `draw_graph(limit=250)` | 绘制节点上限 | `_draw.py:53` |

## 5. 错误与重试语义

- **未初始化客户端**：`_validate_client`/`_validate_sync_client` 在缺 client/url 时抛 `ValueError`（`remote.py:181-193`）。
- **v3 不支持的命令**：`Command.goto/update` 在 v3 流路径抛 `NotImplementedError`（`_remote_run_stream.py:36`）。
- **v3 不支持的 stream 模式**：`_reject_v3_unsupported` 对不兼容模式抛错（`remote.py:195`）。
- **远端错误**：远端 API 错误经 `RemoteException` 包装；本地不实现重试，由调用方/SDK 处理。
- debug 映射为纯函数，无错误路径。

## 6. 并发细节

- `RemoteGraph` 提供同步（`stream/invoke`）与异步（`astream/ainvoke`）双轨，分别用 `SyncLangGraphClient`/`LangGraphClient`。
- 远端流式经 SSE 解码（`langgraph_sdk.stream.decoders.DataDecoder`），事件逐块产出。
- `draw_graph`/debug 映射为同步纯函数，无并发。

## 7. 系统边界

**In-Scope（本仓库源码内）**
- `pregel/protocol.py`（抽象协议）、`remote.py`（RemoteGraph）、`_remote_run_stream.py`（v3 适配）、`debug.py`（debug 映射）、`_draw.py`（图绘制）

**Out-of-Scope（不在本仓库源码内）**
- `langgraph_sdk`（`LangGraphClient`/`SyncLangGraphClient`/SSE 解码器/ThreadStream）→ 外部库
- LangGraph Server 远端服务本身 → 外部部署
- `langchain_core.runnables.graph.Graph`（DrawableGraph）→ 第三方依赖
- LangSmith 追踪 → 外部服务

## 8. 与相邻子系统交互

- **上游**：用户把 `RemoteGraph` 当本地图使用，或作为子图节点。
- **下游 → 外部 SDK/Server**：所有远端调用经 `langgraph_sdk` 到 LangGraph Server。
- **复用 → pregel-engine / pregel-io-checkpoint**：`_draw.py` 复用 `prepare_next_tasks`/`apply_writes`/`channels_from_checkpoint`/`map_input` 等做静态图分析；`debug.py` 复用 `read_channels`。
- **下游 → protocol.py**：本地 `Pregel`（pregel-engine）与 `RemoteGraph` 共同实现 `PregelProtocol`，使二者可互换。

## 9. 语言专项适配口径

本项目为纯 Python 框架，按纯 Python 专项清单分析：

1. **能力缝归组**：`PregelProtocol` 是"本地/远端可互换"能力缝——同一抽象下，本地 Pregel 在进程内执行，`RemoteGraph` 把调用翻译为 HTTP/SSE，用户代码无感知。
2. **双轨实现**：同步/异步客户端与 `stream/astream`、`invoke/ainvoke` 成对；v1/v2/v3 流协议版本并存。
3. **可选依赖**：`langgraph_sdk` 为重依赖（RemoteGraph 专用），普通本地执行不需要它。
4. **配置驱动分派**：`version` 参数与 `goto/update` 支持矩阵决定 v3 线协议行为。

## 10. 图表清单与质量

| 图 | 文件 | 类型 | archify 质量档 |
| --- | --- | --- | --- |
| 远端代理与调试/绘制 | `remote-debug-architecture.html` | architecture | standard |
| RemoteGraph 远端调用链 | `remote-debug-sequence.html` | sequence | standard |

- 降档说明：architecture 图降 standard；本次新增的 sequence 图（客户端→RemoteGraph→HTTP/SSE→LangGraph Server→本地 Pregel 执行→SSE 回传→解码消费）因"自消息 0px 跨度/segment 超出画布"修复后仍未全过 showcase，按规范降 standard 渲染（均成功，约 805KB 自包含）。
- JSON IR 源：`json/remote-debug-architecture.json`、`json/remote-debug-sequence.json`。
- 本次迭代新增 sequence 图，把"本地→SDK→Server→本地消费"的远端调用链从文字化升级为图；不另出 dataflow：远端调用是请求-响应时序而非数据管道。
