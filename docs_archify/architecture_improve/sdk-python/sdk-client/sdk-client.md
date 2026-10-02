# sdk-client（顶层客户端工厂与双轨客户端）

> 本文是 `sdk-python` 域下的叶子子系统文档，覆盖顶层客户端工厂 `get_client`/
> `get_sync_client`、`LangGraphClient`/`SyncLangGraphClient` 组装、类型化错误体系、
> 服务端缓存薄封装、`ServerRuntime`。资源端点的 CRUD 与流式运行分别见
> `sdk-resources`；HTTP 底层封装见 `sdk-http`；流式解码见 `sdk-stream`；认证/加密见
> `sdk-auth-encryption`。
>
> 源码基准：`libs/sdk-py/langgraph_sdk/`，git HEAD `e539ac12`（2026-09-09）。

## 1. 功能清单

| 能力 | 说明 | 源码路径 |
|---|---|---|
| 异步客户端工厂 | 按 url/api_key/headers/timeout 构造 `LangGraphClient` | `_async/client.py:29`（`get_client`） |
| 同步客户端工厂 | 同步版 `get_sync_client` | `_sync/client.py` |
| 顶层客户端组装 | 组合 HttpClient 与 5 个资源子客户端 | `_async/client.py:143`（`LangGraphClient`） |
| 进程内连接 | url=None 时经 ASGI transport 直连同进程 app | `_async/client.py:110-126` |
| 类型化错误 | 状态码→具体异常类映射 | `errors.py:190`（`_map_status_error`） |
| 服务端缓存 | cache_get/cache_set/swr 薄封装 | `cache.py:59/74/93` |
| 运行时上下文 | ServerRuntime（access_context/user/store/context） | `runtime.py:36/148/179` |
| 聚合导出 | client.py re-export 全部对外符号 | `client.py:33`（`__all__`） |

## 2. 核心类型与接口清单

| 类型 / 函数 | 位置 | 职责 |
|---|---|---|
| `get_client(*, url, api_key, headers, timeout)` | `_async/client.py:29` | 异步客户端工厂 |
| `LangGraphClient` | `_async/client.py:143` | 顶层客户端，持有 http + 5 子客户端，支持 `async with`/`aclose` |
| `LangGraphError` | `errors.py:13` | SDK 错误基类 |
| `APIError` | `errors.py:17` | 同时继承 `httpx.HTTPStatusError` 与 `LangGraphError` |
| `APIStatusError` | `errors.py:82` | 带 response/status_code/request_id 的状态错误基类 |
| `BadRequestError`/`AuthenticationError`/.../`RateLimitError`/`InternalServerError` | `errors.py:108-137` | 按状态码的具体异常 |
| `_map_status_error` | `errors.py:190` | 状态码→异常类路由 |
| `_araise_for_status_typed`/`_raise_for_status_typed` | `errors.py:213/224` | 异步/同步抛错入口 |
| `ServerRuntime`（TypeAlias） | `runtime.py:179` | `_ExecutionRuntime | _ReadRuntime` 联合类型 |
| `_ServerRuntimeBase` | `runtime.py:36` | access_context/user/store 基类 |
| `swr` | `cache.py:93` | stale-while-revalidate 缓存加载 |

## 3. 关键调用链

**链一：`get_client` 构造（`_async/client.py:29`）**

1. `url is None` 时默认 `url="http://api"`；若环境 `__LANGGRAPH_DEFER_LOOPBACK_TRANSPORT=true`，用 `get_asgi_transport()(app=None, root_path="/noauth")` 并把 transport 追加到 `_registered_transports`（`_async/client.py:110-114`）。
2. 否则尝试 `from langgraph_api.server import app` 构造进程内 ASGI transport；失败则 defer 注册（`_async/client.py:116-126`）。
3. 远端 url 时 `transport = httpx.AsyncHTTPTransport(retries=5)`（`_async/client.py:128-129`）。
4. 构造 `httpx.AsyncClient(base_url, transport, timeout=connect=5/read=300/write=300/pool=5, headers=_get_headers(api_key, headers))`（`_async/client.py:130-139`）。
5. 返回 `LangGraphClient(client)`，其 `__init__` 组装 HttpClient 与 5 个子客户端（`_async/client.py:154-160`）。

**链二：类型化错误抛出（`errors.py:213`）**

1. `_araise_for_status_typed(r)`：`status_code < 400` 直接返回（`errors.py:214-215`）。
2. `await _adecode_error_body(r)` 用 orjson 解码错误体（`errors.py:216`、`errors.py:158-171`）。
3. `_map_status_error(r, body)` 按状态码选异常类（`errors.py:217`、`errors.py:190-210`）。
4. 非 3.11 版本打日志后 `raise err`（`errors.py:219-221`）。

## 4. 配置项

| flag / option | 默认 / 行为 | 位置 |
|---|---|---|
| `url` | None→进程内 ASGI；否则远端 base_url | `_async/client.py:31/110` |
| `api_key` | NOT_PROVIDED 时按 `LANGGRAPH_API_KEY`→`LANGSMITH_API_KEY`→`LANGCHAIN_API_KEY` 加载；None 跳过 | `_async/client.py:47-54` |
| `timeout` | connect=5/read=300/write=300/pool=5 | `_async/client.py:133-137` |
| transport retries | 远端 `httpx.AsyncHTTPTransport(retries=5)` | `_async/client.py:129` |
| `__LANGGRAPH_DEFER_LOOPBACK_TRANSPORT` | "true" 时 defer 注册进程内 transport | `_async/client.py:112` |
| swr `fresh_for`/`max_age` | fresh_for 默认 0；max_age 默认 1 天 | `cache.py:137-140` |

## 5. 错误与重试语义

- **传输层重试**：远端 transport `retries=5`（httpx 连接级重试，`_async/client.py:129`）。
- **状态码错误**：统一经 `_map_status_error` 抛类型化异常（400/401/403/404/409/422/429/5xx）。
- **错误体解码**：orjson 失败回退 `data.decode()`，再失败返回 None（`errors.py:165-171`）。
- **缓存不可用**：非服务端环境 `cache_get/cache_set/swr` 抛 `RuntimeError`（`cache.py:66-70/85-89/132-136`）。
- **无用户**：`ensure_user()` 在无认证用户时抛 `PermissionError`（`runtime.py:140-145`）。

## 6. 并发细节

- `LangGraphClient` 支持 `async with` 上下文管理，退出时 `aclose()` 关闭底层 httpx client（`_async/client.py:162-178`）。
- 进程内 transport 经 `_registered_transports` 列表 defer 注册，待 app 初始化后解析（`_async/client.py:114/126`）。
- swr stale 命中时后台 best-effort 刷新（`cache.py:128-130`）。

## 7. 系统边界

**In-Scope（本仓库源码内）**
- 客户端工厂、顶层组装、错误映射、缓存薄封装、运行时上下文。

**Out-of-Scope（不在本仓库源码内）**
- **LangGraph API/Server**：远端 REST/SSE 服务与 `langgraph_api.server.app` 进程内 app，不在本仓库源码内。
- **`langgraph_api.cache`**：实际缓存后端由 Agent Server 提供，SDK 仅延迟绑定。
- **httpx/或json**：第三方 HTTP 与 JSON 库。
- 资源端点 CRUD 见 `sdk-resources`；流式解码见 `sdk-stream`。

## 8. 与相邻子系统交互

- 应用代码 → `get_client`（本叶子）→ `LangGraphClient` → `sdk-resources` 子客户端 → `sdk-http` HttpClient → LangGraph API（外部）。
- HttpClient → `errors.py`（本叶子）做状态码映射。
- 本叶子 `cache.py` → `langgraph_api.cache`（外部，服务端）。
- 本叶子 `runtime.py` → `auth.types.BaseUser`（`sdk-auth-encryption`）。

## 9. 语言专项适配口径（纯 Python）

- **双轨实现模式**：`_async/` 与 `_sync/` 镜像实现同一客户端契约（`LangGraphClient` vs `SyncLangGraphClient`），是本域典型双轨；同步版为异步版的线程/阻塞等价物。
- **可选依赖懒加载**：`cache.py` 用 `try/except ImportError` 延迟绑定 `langgraph_api.cache`，缺失时 `_cache_get=None`，调用时才抛 RuntimeError（`cache.py:18-35`）；进程内 app 同样 `try from langgraph_api.server import app`（`_async/client.py:117`）。
- **配置驱动分派**：url 是否为 None、api_key 是否为 NOT_PROVIDED、环境变量，共同决定 transport 与 header 路径。
- **类型 alias 兼容**：`ServerRuntime` 用 `TypeAliasType`（3.12+，否则 typing_extensions）做联合类型窄化（`runtime.py:12-15/179`）。

## 10. 图表清单与质量

| 图 | 文件 | 类型 | archify 质量档 |
|---|---|---|---|
| 组件架构图 | `sdk-client-architecture.html` | architecture | showcase |
| 请求链时序 | `sdk-client-sequence.html` | sequence | showcase |

- JSON IR 源文件位于 `json/` 目录。
- 省略 dataflow：本叶子是“客户端组装 + 请求-响应”，无独立数据管道，请求流已由 sequence 表达。
- 省略 workflow/lifecycle：无多角色泳道分支，也无单实体状态机（流式读取循环见 `sdk-stream`）。
