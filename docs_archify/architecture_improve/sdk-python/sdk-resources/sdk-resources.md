# sdk-resources（资源端点客户端组）

> 本文是 `sdk-python` 域下的叶子子系统文档（本轮新增叶子），覆盖 5 类资源端点客户端：
> assistants（图版本配置）、threads（多轮会话）、runs（单次调用）、crons（定时任务）、
> store（持久文档存储）。它们共享 `HttpClient`（见 `sdk-http`），每个方法映射一个
> REST/SSE 端点。顶层组装见 `sdk-client`；流式解码见 `sdk-stream`。
>
> 源码基准：`libs/sdk-py/langgraph_sdk/_async/` 与 `_sync/`，git HEAD `e539ac12`（2026-09-09）。

## 1. 功能清单

| 能力 | 说明 | 源码路径 |
|---|---|---|
| assistants CRUD | get/create/update/delete/search/get_graph/get_schemas/get_subgraphs/get_versions | `_async/assistants.py:29`（`AssistantsClient`） |
| threads CRUD | get/create/update/delete/search/get_state/update_state/get_history/join_stream | `_async/threads.py:30`（`ThreadsClient`） |
| runs 调用 | create/create_batch/wait/list/get/join/delete | `_async/runs.py:56`（`RunsClient`） |
| crons 定时 | create/create_for_thread/update/delete/search | `_async/cron.py:34`（`CronClient`） |
| store 文档 | get_item/delete_item/search_items/list_namespaces | `_async/store.py:18`（`StoreClient`） |
| 同步双轨 | `_sync/` 下同名同步客户端 | `_sync/assistants.py` 等 |

## 2. 核心类型与接口清单

| 类型 / 函数 | 位置 | 职责 |
|---|---|---|
| `AssistantsClient(http)` | `_async/assistants.py:43` | 持有 http，方法映射 `/assistants/...` |
| `ThreadsClient(http)` | `_async/threads.py:30` | 会话与状态读写 |
| `RunsClient(http)` | `_async/runs.py:56` | 单次图调用与等待 |
| `CronClient(http)` | `_async/cron.py:34` | 定时任务 |
| `StoreClient(http)` | `_async/store.py:18` | 键值/命名空间文档存储 |
| `runs.wait(...)` | `_async/runs.py:678` | 创建 run 并等待最终状态 |
| `request_reconnect` | `_async/http.py` | 断线重连的请求方法（wait 使用） |

## 3. 关键调用链

**链一：`assistants.get`（`_async/assistants.py:46`）**

1. 入参 `assistant_id` 经 `_quote_path_param` 转义（`_async/assistants.py:88`）。
2. `return await self.http.get(f"/assistants/{...}", headers=headers, params=params)`（`_async/assistants.py:87-91`）。
3. HttpClient 经 `sdk-http` 发出 GET，返回解析后的 `Assistant` dict。

**链二：`runs.wait`（`_async/runs.py:678`）**

1. `checkpoint_during` 非空时发 DeprecationWarning，引导改用 `durability`（`_async/runs.py:795-800`）。
2. 构造 payload dict（input/command/config/context/metadata/interrupt/multitask_strategy 等，`_async/runs.py:801-823`）。
3. endpoint 按 thread_id 是否为 None 选 `/threads/{id}/runs/wait` 或 `/runs/wait`（`_async/runs.py:824-828`）。
4. `await self.http.request_reconnect(endpoint, "POST", json={非 None 字段}, on_response=on_response)`（`_async/runs.py:835-842`）。
5. 若 `raise_error` 且响应含 `__error__`，抛 `Exception`（`_async/runs.py:843-851`）。
6. 返回最终状态 dict。

## 4. 配置项

| flag / option | 默认 / 行为 | 位置 |
|---|---|---|
| `multitask_strategy` | reject/interrupt/rollback/enqueue | `_async/runs.py:728-729` |
| `if_not_exists` | reject/create | `_async/runs.py:730-731` |
| `on_disconnect` | cancel/continue | `_async/runs.py:724-725` |
| `durability` | sync/async/exit（替代 checkpoint_during） | `_async/runs.py:738-741` |
| `raise_error` | 默认 True，`__error__` 转异常 | `_async/runs.py:843` |
| `on_run_created` | run 创建回调 | `_async/runs.py:649/832` |

## 5. 错误与重试语义

- **断线重连**：`wait` 走 `request_reconnect`，传输层断线后自动重连续传 SSE。
- **业务错误**：响应体 `__error__` 字段在 `raise_error=True` 时转 `Exception`（`_async/runs.py:843-851`）。
- **HTTP 状态错误**：由 `sdk-http`/`sdk-client` 的 `errors.py` 统一映射为类型化异常。
- **弃用警告**：`checkpoint_during` 触发 DeprecationWarning（`_async/runs.py:796`）。

## 6. 并发细节

- 资源客户端本身无独立线程；异步方法经 httpx async 发出请求。
- `on_run_created` 回调在响应到达时同步触发（`_async/runs.py:830-833`）。
- SSE 流式事件的消费与投影在 `sdk-stream` 叶子。

## 7. 系统边界

**In-Scope（本仓库源码内）**
- 5 个资源客户端的端点映射、payload 构造、双轨实现。

**Out-of-Scope（不在本仓库源码内）**
- **LangGraph API**：assistants/threads/runs/crons/store 的服务端实现，不在本仓库源码内。
- **HttpClient 底层**：见 `sdk-http`；SSE 解码见 `sdk-stream`。

## 8. 与相邻子系统交互

- `sdk-client` 的 `LangGraphClient` → 挂载本叶子 5 个资源客户端。
- 本叶子资源客户端 → `sdk-http` HttpClient 发请求。
- 本叶子 `runs.wait`/`stream` → `sdk-stream` 消费 SSE 事件流。
- 本叶子 → LangGraph API（外部）。

## 9. 语言专项适配口径（纯 Python）

- **双轨实现模式**：`_async/` 与 `_sync/` 同名客户端镜像实现，是本域最典型的双轨；同步版为阻塞等价物。
- **能力缝归组**：每个资源客户端是一个“能力缝”——把一类 REST 资源封装成方法集合，共享 HttpClient 契约。
- **配置驱动分派**：thread_id 是否为 None、multitask_strategy/if_not_exists/durability 等参数决定 endpoint 与服务端行为。
- **重载签名**：`runs.wait` 用 `@overload` 区分 thread_id 为 None/非 None 的返回类型（`_async/runs.py:625-676`）。

## 10. 图表清单与质量

| 图 | 文件 | 类型 | archify 质量档 |
|---|---|---|---|
| 组件架构图 | `sdk-resources-architecture.html` | architecture | showcase |
| runs.wait 时序 | `sdk-resources-sequence.html` | sequence | showcase |

- JSON IR 源文件位于 `json/` 目录。
- 省略 dataflow：资源端点是请求-响应，无独立数据管道；SSE 事件流的解码管道见 `sdk-stream`。
- 省略 workflow/lifecycle：无多角色泳道分支，也无单实体状态机（run 状态机在服务端）。
