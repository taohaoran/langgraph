# SDK 客户端主类与资源客户端（sdk-client）

> 本文是 `sdk-python` 域下的叶子子系统文档。域级总览见 `../sdk-python.md`。
> 本文只展开顶层客户端工厂、异步/同步双轨实现与五类资源客户端；
> 流式管道见 `../sdk-stream/sdk-stream.md`，认证/加密/错误类型见
> `../sdk-auth-encryption/sdk-auth-encryption.md`。
>
> 源码基准：`libs/sdk-py/langgraph_sdk/`（`client.py`、`runtime.py`、`_async/`、`_sync/`、`_shared/`）。

## 1. 功能清单

| 能力 | 说明 | 源码路径 |
| --- | --- | --- |
| 异步客户端工厂 | `get_client(url, api_key, headers, timeout)` 建 `LangGraphClient` | `_async/client.py:29` |
| 同步客户端工厂 | `get_sync_client(...)` 建 `SyncLangGraphClient` | `_sync/client.py:20` |
| 顶层客户端组装 | 聚合 http + 五类资源子客户端，支持 async/sync 上下文管理 | `_async/client.py:143`、`_sync/client.py:89` |
| 进程内连接 | `url=None` 时经 ASGI transport 直连本机 langgraph_api（loopback） | `_async/client.py:110-126`、`_shared/utilities.py:244` |
| 资源客户端 | Assistants / Threads / Runs / Crons / Store | `_async/{assistants,threads,runs,cron,store}.py` |
| HTTP 传输封装 | get/post/put/patch/delete + stream + request_reconnect | `_async/http.py:26` `HttpClient` |
| API Key 解析 | 显式参数 → `LANGGRAPH_API_KEY` → `LANGSMITH_API_KEY` → `LANGCHAIN_API_KEY` | `_shared/utilities.py:26` `_get_api_key` |
| JSON 编解码 | orjson 经线程池序列化/反序列化 | `_async/http.py:290/306` |
| 运行时上下文 | `ServerRuntime`（access_context + user + store + context） | `runtime.py:36-183` |
| 路径参数转义 | `_quote_path_param` 防路径穿越 | `_shared/utilities.py:201` |
| 重连同源校验 | `_validate_reconnect_location` 防凭据外泄 | `_shared/utilities.py:167` |
| 包再导出 | 顶层 `client.py` 汇总 async/sync 全部符号 | `client.py:12-55` |

## 2. 核心类型与接口清单

| 类型 / 函数 | 位置 | 职责 |
| --- | --- | --- |
| `LangGraphClient` | `_async/client.py:143` | 异步顶层客户端，持有 `http` 与五子客户端 |
| `SyncLangGraphClient` | `_sync/client.py:89` | 同步对应物，方法一一对应 |
| `HttpClient` / `SyncHttpClient` | `_async/http.py:26` / `_sync/http.py` | 传输层：HTTP 方法 + 错误包装 + SSE 流 |
| `RunsClient` | `_async/runs.py:56` | 运行资源：create/create_batch/wait/list/get/cancel/join/delete |
| `AssistantsClient`/`ThreadsClient`/`CronClient`/`StoreClient` | `_async/*.py` | 其余四类资源客户端 |
| `ServerRuntime`（TypeAliasType） | `runtime.py:179` | `_ExecutionRuntime | _ReadRuntime` 联合，`.execution_runtime` 窄化 |
| `_get_headers` / `_get_api_key` | `_shared/utilities.py:51/26` | 认证头合成与环境变量解析 |
| `configure_loopback_transports` | `_shared/utilities.py:238` | 服务器启动后回填延迟注册的 ASGI transport |

## 3. 关键调用链

**链一：构造异步客户端**（`_async/client.py:29`）
1. `url=None` 时默认 `http://api`；尝试 `from langgraph_api.server import app` 建 ASGI transport，失败则注册延迟 transport（待 `configure_loopback_transports(app)` 回填）。
2. `url` 给定则用 `httpx.AsyncHTTPTransport(retries=5)`。
3. 合成 `httpx.AsyncClient(base_url, timeout=(connect=5,read=300,write=300,pool=5), headers=_get_headers(...))`。
4. 包进 `LangGraphClient`，其 `__init__` 把 http 注入五个资源客户端。

**链二：`runs.wait` 流式调用**（`_async/runs.py:625` → `_async/http.py:187`）
1. `RunsClient` 拼 payload（过滤 None），调 `self.http.stream(...)`。
2. `HttpClient.stream` 设 `Accept: text/event-stream`、`Cache-Control: no-store`，`client.stream(...)` 起流式响应。
3. 校验 Content-Type；`aiter_lines_raw` 逐行喂 `SSEDecoder.decode`，yield `(event, data)`。
4. 遇 `Location` 头则记录 `reconnect_path`；`httpx.HTTPError` 时置 `retry=True`，按 `Last-Event-ID` 最多重连 5 次。

**链三：普通 JSON 请求**（`_async/http.py:54`）
1. `_aencode_json` 在线程池里 `orjson.dumps`（带 `_orjson_default` 处理 pydantic/dict/set）。
2. `client.post` → `_araise_for_status_typed`（类型化错误）→ `_adecode_json` 在线程池 `orjson.loads`。

## 4. 配置项

| 配置 | 默认 / 行为 | 位置 |
| --- | --- | --- |
| `url` | `None` 走进程内 loopback；同步默认 `http://localhost:8123` | `_async/client.py:111`、`_sync/client.py:73` |
| `api_key` | 未提供时按 `LANGGRAPH_API_KEY→LANGSMITH_API_KEY→LANGCHAIN_API_KEY` 顺序加载；`None` 显式跳过 | `_shared/utilities.py:26-48` |
| `timeout` | connect=5, read=300, write=300, pool=5 | `_async/client.py:136` |
| httpx retries | 异步/同步 transport `retries=5` | `_async/client.py:129`、`_sync/client.py:75` |
| 保留头 | `x-api-key` 不允许用户覆盖 | `_shared/utilities.py:21` |
| `Accept`/`Cache-Control` | 流请求固定 `text/event-stream` / `no-store` | `_async/http.py:199-200` |

## 5. 错误与重试语义

- HTTP 错误经 `_araise_for_status_typed`（`errors.py`）转为类型化 SDK 异常；`request_reconnect` 对非终态在响应体上 `add_note`。
- SSE 流中途断连（`httpx.HTTPError`/`TransportError`）自动按 `Location` + `Last-Event-ID` 重连，最多 5 次，超限抛 `TransportError`。
- 传输层 `retries=5`（httpx 负责瞬时错误重试）。
- 跨源 `Location` 重连被 `_validate_reconnect_location` 拒绝（scheme+host+port 比对），防凭据泄漏。
- 进程内 ASGI transport 首次连接失败仅 debug 日志并延迟注册，不立即报错。

## 6. 并发细节

- 异步轨基于 `asyncio`：`orjson.dumps/loads` 经 `run_in_executor` 不阻塞事件循环；`client.stream` 为异步上下文管理器。
- 同步轨为阻塞式 httpx.Client，与异步轨方法同构但无 await。
- `_registered_transports` 为模块级列表，`configure_loopback_transports` 统一回填 app（服务器启动后调用）。
- 无锁竞争；每个顶层客户端持有独立 httpx.Client。

## 7. 系统边界

**In-Scope（本仓库源码内）**
- `client.py`/`runtime.py`/`_async/`/`_sync/`/`_shared/`：客户端工厂、双轨资源客户端、HTTP 封装。

**Out-of-Scope（不在本仓库源码内）**
- LangGraph API 远端服务、`langgraph_api.server`（进程内 loopback 目标）、httpx/orjson 第三方库——均外部。
- 流式控制器与解码器在 `stream/`（见 sdk-stream 叶子）；认证/加密在 `auth/`、`encryption/`（见 sdk-auth-encryption）。

## 8. 与相邻子系统交互

- 上游：应用代码 → `get_client`/`get_sync_client` → 顶层客户端。
- 本叶子 → sdk-stream：`HttpClient.stream` 使用 `sse.py` 的 `SSEDecoder`/`aiter_lines_raw`；高级流式封装在 `stream/controller.py`（见 sdk-stream）。
- 本叶子 → sdk-auth-encryption：`errors.py` 的 `_araise_for_status_typed`、`schema.py` 的 `StreamPart`/类型。
- 下游外部：httpx → LangGraph API / ASGI transport → 本机服务器。

## 9. 语言专项适配口径（纯 Python）

- **双轨实现模式（核心能力缝）**：`_async/` 与 `_sync/` 是同构镜像——同名类、同名方法、同步/异步差异仅 `async/await`；`_shared/` 收敛无差异逻辑（API Key、JSON、路径转义、时间区），是典型"同一能力两套实现 + 共享层"。
- **资源抽象**：每个资源一个类，构造时注入 `HttpClient`，方法只拼路径/payload 后委托 `http.get/post/...`；薄封装 + 统一传输。
- **配置驱动**：`api_key` 三段环境变量优先级、`url=None` 自动进程内连接，是运行时分派。
- **可选依赖与懒加载**：`langgraph_api.server`/`asgi_transport` 延迟导入并 fallback 到 `httpx.ASGITransport`；`TYPE_CHECKING` 保护 `BaseStore` 类型导入。

## 10. 图表清单与质量

| 图 | 文件 | 类型 | archify 质量档 |
| --- | --- | --- | --- |
| 异步/同步双轨与资源矩阵 | `sdk-client-architecture.html` | architecture | standard（shared→http 连线沿 region 边框、外部边界贴边，去标签并扩画布后通过） |
| 一次 runs.wait 调用链 | `sdk-client-sequence.html` | sequence | standard（同通道消息间距重排后通过） |

JSON IR 源：`json/sdk-client-architecture.json`、`json/sdk-client-sequence.json`。
说明：本叶子未生成第三张图（dataflow/lifecycle/workflow）：客户端主路径为"构造→HTTP 委托→资源调用"的请求-响应链，已由 architecture（组件拓扑）与 sequence（调用时序）完整覆盖；无独立数据管道、无单一实体状态机、无多角色泳道流程，与已有图信息重复，按资源节省原则省略。
