# sdk-http（HTTP 请求封装与共享工具层）

> 本文是 `sdk-python` 域下的叶子子系统文档（本轮新增叶子），覆盖 `HttpClient`
> （异步/同步双轨的 get/post/put/patch/delete/request_reconnect/stream）与
> `_shared/` 共享类型与工具（API key 解析、headers 合并、JSON 编解码、路径参数转义、
> ASGI loopback transport）。它是所有资源客户端（`sdk-resources`）的底层 HTTP 层。
>
> 源码基准：`libs/sdk-py/langgraph_sdk/_async/http.py`、`_sync/http.py`、`_shared/`，
> git HEAD `e539ac12`（2026-09-09）。

## 1. 功能清单

| 能力 | 说明 | 源码路径 |
|---|---|---|
| GET/POST/PUT/PATCH/DELETE | 标准 REST 封装，统一 raise_for_status + JSON 解码 | `_async/http.py:39/54/79/100/121` |
| Location 重连 | `request_reconnect` 跟随 Location 头，最多 5 次 | `_async/http.py:138` |
| SSE 流式 | `stream` 走 text/event-stream，自动重连 | `_async/http.py:187` |
| API key 解析 | 按优先级从环境变量加载 | `_shared/utilities.py:26`（`_get_api_key`） |
| headers 合并 | 合并 User-Agent/x-api-key，禁止保留头覆盖 | `_shared/utilities.py:51`（`_get_headers`） |
| JSON 编解码 | orjson + pydantic model_dump 兜底 | `_shared/utilities.py:72`（`_orjson_default`） |
| 路径参数转义 | `_quote_path_param` | `_shared/utilities.py:201` |
| loopback transport | ASGI 进程内 transport 注册 | `_shared/utilities.py:238/244` |

## 2. 核心类型与接口清单

| 类型 / 函数 | 位置 | 职责 |
|---|---|---|
| `HttpClient(client)` | `_async/http.py:26` | 持有 httpx.AsyncClient，提供 REST 方法 |
| `request_reconnect` | `_async/http.py:138` | 跟随 Location 头的重连请求 |
| `stream` | `_async/http.py:187` | SSE 流式迭代器 |
| `_get_api_key` | `_shared/utilities.py:26` | API key 环境变量优先级解析 |
| `_get_headers` | `_shared/utilities.py:51` | headers 合并与保留头校验 |
| `_aencode_json`/`_adecode_json` | `_async/http.py` 导入 | orjson 编解码 |
| `get_asgi_transport` | `_shared/utilities.py:244` | 进程内 ASGI transport 工厂 |

## 3. 关键调用链

**链一：`post` 请求（`_async/http.py:54`）**

1. `await _aencode_json(json)` 得 `(request_headers, content)`（`_async/http.py:65`）。
2. 合并运行时 headers（`_async/http.py:69-70`）。
3. `r = await self.client.post(path, headers, content, params)`（`_async/http.py:71-73`）。
4. `on_response(r)` 回调（`_async/http.py:74-75`）。
5. `await _araise_for_status_typed(r)` 状态码映射抛错（`_async/http.py:76`）。
6. `return await _adecode_json(r)`（`_async/http.py:77`）。

**链二：`request_reconnect`（`_async/http.py:138`）**

1. `async with self.client.stream(...)` 打开流（`_async/http.py:153-155`）。
2. `r.raise_for_status()`，失败读 body 并 `add_note` 后重抛（`_async/http.py:158-166`）。
3. 取 `location` 头；无 Location 或 `reconnect_limit<=0` 则解码返回（`_async/http.py:167-169`）。
4. `_validate_reconnect_location` 校验后，解码失败时警告并 `r.aclose()`，递归以 GET 重连，`reconnect_limit-1`（`_async/http.py:170-185`）。

## 4. 配置项

| flag / option | 默认 / 行为 | 位置 |
|---|---|---|
| `reconnect_limit` | 默认 5 次 | `_async/http.py:147/184` |
| API key 环境变量 | 显式参数 > LANGGRAPH_API_KEY > LANGSMITH_API_KEY > LANGCHAIN_API_KEY | `_shared/utilities.py:40-48` |
| User-Agent | `langgraph-sdk-py/{version}` | `_shared/utilities.py:62` |
| 保留头 | RESERVED_HEADERS 不可被用户覆盖 | `_shared/utilities.py:57-59` |
| stream Accept | `text/event-stream`、`Cache-Control: no-store` | `_async/http.py:199-200` |

## 5. 错误与重试语义

- **状态码错误**：统一经 `_araise_for_status_typed` 映射为类型化异常（见 `sdk-client` errors.py）。
- **重连**：`request_reconnect` 跟随 Location 头，最多 5 次；解码失败触发重连（`_async/http.py:171-185`）。
- **stream 重连**：`stream` 维护 `last_event_id`/`reconnect_path`，断线续传（`_async/http.py:211-216`）。
- **保留头冲突**：用户设置保留头抛 `ValueError`（`_shared/utilities.py:57-59`）。

## 6. 并发细节

- 异步/同步双轨：`_async/http.py`（AsyncClient）与 `_sync/http.py`（Client）镜像实现。
- `stream` 用 `async with self.client.stream(...)` 管理响应生命周期（`_async/http.py:153`）。
- 共享无状态工具函数（纯函数），无锁。

## 7. 系统边界

**In-Scope（本仓库源码内）**
- HttpClient 双轨封装、共享工具函数。

**Out-of-Scope（不在本仓库源码内）**
- **httpx**：底层 HTTP 传输库，不在本仓库源码内。
- **LangGraph API**：远端服务，不在本仓库源码内。
- **orjson**：JSON 库，不在本仓库源码内。
- 状态码异常类定义见 `sdk-client` errors.py；SSE 解码见 `sdk-stream`。

## 8. 与相邻子系统交互

- `sdk-resources` 资源客户端 → 本叶子 HttpClient。
- 本叶子 → `sdk-client` errors.py（状态码映射）。
- 本叶子 → `sdk-stream`（stream 产出的字节流经 SSEDecoder 解码）。
- 本叶子 → httpx（外部）→ LangGraph API（外部）。

## 9. 语言专项适配口径（纯 Python）

- **双轨实现模式**：`_async/http.py`/`_sync/http.py` 是本域双轨的根——所有资源客户端都建立在这对双轨之上。
- **可选依赖懒加载**：进程内 ASGI transport 与 `langgraph_api.server` 的延迟绑定在 `sdk-client`；本叶子只提供 `get_asgi_transport` 工厂。
- **配置驱动分派**：API key 优先级、保留头校验、stream 重连参数由调用方/环境驱动。
- **共享层**：`_shared/` 是异步/同步双轨共用的纯函数工具层，避免双份实现。

## 10. 图表清单与质量

| 图 | 文件 | 类型 | archify 质量档 |
|---|---|---|---|
| 组件架构图 | `sdk-http-architecture.html` | architecture | showcase |
| 请求生命周期时序 | `sdk-http-sequence.html` | sequence | showcase |

- JSON IR 源文件位于 `json/` 目录。
- 省略 dataflow：HTTP 请求是请求-响应，无独立数据管道；SSE 字节流的解码管道见 `sdk-stream`。
- 省略 workflow/lifecycle：无多角色泳道分支，也无单实体状态机。
