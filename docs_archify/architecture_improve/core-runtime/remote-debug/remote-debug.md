# 远端代理与调试（remote-debug）

> 本文是 `core-runtime` 域下的叶子子系统文档。域级总览见 `../core-runtime.md`（输出根统一产出）。
> 本文只展开 `RemoteGraph` 远端图代理、运行期 `Runtime` 注入对象与 `debug.py` 调试映射；
> 不重复展开本地 Pregel 执行引擎（见 `../pregel-engine/pregel-engine.md`）。
>
> 源码基准：`langgraph` main 分支，commit `e539ac122f4126f6dd850581c1494948cf620e31`。

## 1. 功能清单

| 能力 | 说明 | 源码路径 |
| --- | --- | --- |
| `RemoteGraph` | 远端 LangGraph Server 的客户端代理，实现 `PregelProtocol`，可当本地图节点 | `pregel/remote.py:118` |
| 客户端装配 | 按 `url/api_key/headers` 构造默认 `LangGraphClient`/`SyncLangGraphClient`，或接受注入 | `pregel/remote.py:173-179` |
| `invoke/stream/astream` | 代理远端图执行，把远端 SSE 流转成本地流/结果 | `pregel/remote.py:1133`、`:724`、`:879` |
| 状态读写 | `get_state`/`get_state_history`/`update_state`/`bulk_update_state` 代理远端 checkpoint | `pregel/remote.py:436`、`:508`、`:602`、`:588` |
| 图结构绘制 | `get_graph`/`_get_drawable_nodes` 拉远端图结构并绘制 | `pregel/remote.py:256`、`:232` |
| `Runtime[ContextT]` | 注入节点的运行期工具包：context/store/stream_writer/previous/execution_info | `runtime.py:125` |
| `get_runtime` | 按 context_schema 取当前运行期 Runtime | `runtime.py:296` |
| `ExecutionInfo`/`ServerInfo` | 运行/服务信息不可变数据类 | `runtime.py:27`、`:61` |
| `map_debug_*` | 把 Pregel 任务/写入/检查点映射成调试流 payload | `pregel/debug.py:41`、`:83`、`:144` |

## 2. 核心类型与接口清单

| 类型 / 函数 | 位置 | 职责 |
| --- | --- | --- |
| `RemoteGraph` | `pregel/remote.py:118` | 远端图代理；持有 `assistant_id`、sync/async client、config |
| `PregelProtocol` | 协议基类 | RemoteGraph 实现的统一图协议（本地 Pregel 也实现它），使其可互换 |
| `RemoteException` | `pregel/remote.py:112` | 远端错误包装 |
| `Runtime[ContextT]` | `runtime.py:125` | 节点注入对象；`ToolRuntime`（prebuilt）是其子类 |
| `RunControl` | `runtime.py:79` | 运行控制句柄 |
| `map_debug_tasks` 等 | `pregel/debug.py:41` | 调试投影纯函数族 |

## 3. 关键调用链

1. **构造代理**：`RemoteGraph(assistant_id, url=..., api_key=...)`（`remote.py:132`）；若传了 `url` 则 `get_client`/`get_sync_client`（`:174`/`:178`）建默认客户端，否则用注入的 client。
2. **调用前校验**：`_validate_client`（`remote.py:181`）/`_validate_sync_client`（`:188`）在缺失客户端时抛 `ValueError`。
3. **远端执行**：`invoke`/`stream`（`remote.py:1133`/`:724`）经 `_get_stream_modes`（`:672`）解析流模式，调用 client 向远端 `POST /threads/runs`（SSE）。
4. **事件回流**：`_remote_run_stream.py` 消费远端 SSE，映射成本地 `StreamPart`/`StateSnapshot`（`_create_state_snapshot`，`remote.py:322`）。
5. **配置过滤**：`_sanitize_config`（`remote.py:407`）与 `rm_pregel_keys`（`debug.py:131`）在跨边界前剔除本地 Pregel 内部键。

## 4. 配置项

| 配置项 | 默认 / 行为 | 位置 |
| --- | --- | --- |
| `url` / `api_key` / `headers` | 远端接入；api_key 缺省读 `LANGGRAPH_API_KEY` 等环境变量 | `remote.py:137-155` |
| `client` / `sync_client` | 注入自定义 HTTP 客户端 | `remote.py:140-141` |
| `config` | 附加 `RunnableConfig` | `remote.py:142` |
| `distributed_tracing` | 是否下发 LangSmith 分布式追踪头 | `remote.py:144`、`:1297` `_merge_tracing_headers` |
| `name` | 作为子图节点时的人名 | `remote.py:143` |

## 5. 错误与重试语义

- 远端调用失败由 `LangGraphClient` 抛错，RemoteGraph 包成 `RemoteException`（`remote.py:112`）或透传。
- 客户端未初始化（sync/async 用错轨）抛 `ValueError`（`remote.py:183`/`:190`）。
- v3 流模式在远端不支持时 `_reject_v3_unsupported`（`remote.py:195`）显式拒绝。
- 本叶子不实现重试/退避——交给底层 HTTP SDK 与远端服务端；调试映射为纯函数，无失败路径。

## 6. 并发细节

- sync/async 双客户端：`LangGraphClient`（async）与 `SyncLangGraphClient`（sync）成对持有；`stream` 走同步、`astream` 走异步。
- 远端 SSE 流式消费在 `_remote_run_stream.py` 内以迭代器拉取，不阻塞调用方迭代。
- `Runtime` 是 frozen dataclass（`runtime.py:124` `_DC_KWARGS`），不可变、线程安全地注入各节点。
- 无线程创建；并发模型由 HTTP SDK 与本地图帧调度决定。

## 7. 系统边界

**In-Scope（本仓库源码内）**
- `pregel/remote.py`：RemoteGraph 代理。
- `runtime.py`：Runtime/ExecutionInfo/ServerInfo/RunControl。
- `pregel/debug.py`：调试映射纯函数。
- `pregel/_remote_run_stream.py`：远端流消费。

**Out-of-Scope（不在本仓库源码内）**
- 远端 LangGraph Server 本体（执行图、持久化）：不在本仓库源码内。
- `langgraph_sdk` 的 `LangGraphClient`/`SyncLangGraphClient` HTTP 实现：为外部依赖。
- 本地 Pregel 引擎：见 pregel-engine 叶子。
- LangSmith 追踪后端：外部服务。

## 8. 与相邻子系统交互

- 上游 → 本叶子：用户把 `RemoteGraph` 当普通节点 `add_node` 进本地图；本地图对其调用 `PregelProtocol` 接口。
- 本叶子 → 下游：经 `LangGraphClient` 发起 HTTP/SSE 到远端 Server；`Runtime` 反向注入本地节点。
- 方向：`本地图节点 → RemoteGraph → LangGraphClient → 远端 Server（HTTP/SSE）→ 事件回流映射 → 本地流`。

## 9. 语言专项适配口径

按纯 Python 框架口径：
- **能力缝归组**：`PregelProtocol` 是契约缝——本地 `Pregel` 与 `RemoteGraph` 是同一契约的两个实现（本地执行 vs HTTP 代理），实现可互换，是典型双轨实现。
- **外部边界（编排层定位）**：RemoteGraph 是纯适配层，真正计算在远端 Server，HTTP SDK 为外部依赖，均标注"不在本仓库源码内"。
- **配置驱动分派**：sync/async 双轨由调用方法（`stream` vs `astream`）与已初始化的客户端决定。
- **纯函数映射**：`debug.py` 的 `map_debug_*` 是无状态投影纯函数，把内部任务结构翻译成外部调试协议 payload。
- 无注册表/代码生成/可选依赖懒加载。

## 10. 图表清单与质量

| 图 | 文件 | 类型 | archify 质量档 |
| --- | --- | --- | --- |
| 远端代理与调试架构 | `remote-debug-architecture.html` | architecture | showcase |
| RemoteGraph 远端调用时序 | `remote-debug-sequence.html` | sequence | showcase |

JSON IR 源文件位于 `json/` 目录。本叶子未补 dataflow / lifecycle 图：RemoteGraph 是请求-响应代理（时序图已表达），无独立数据管道，Runtime/调试映射也无单实体状态机，按资源节省原则省略。
