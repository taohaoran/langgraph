# 认证授权、静态加密与运行时工具（sdk-auth-encryption）

> 本文是 `sdk-python` 域下的叶子子系统文档。域级总览见 `../sdk-python.md`。
> 本文展开服务端扩展注册框架（Auth / Encryption）与 SDK 运行时工具
> （类型化错误、SSE 解码、KV/SWR 缓存）；客户端主类与资源见
> `../sdk-client/sdk-client.md`，流式管道见 `../sdk-stream/sdk-stream.md`。
>
> 源码基准：`libs/sdk-py/langgraph_sdk/`（auth/、encryption/、errors.py、
> sse.py、cache.py、schema.py）。

## 1. 功能清单

| 能力 | 说明 | 源码路径 |
| --- | --- | --- |
| 认证处理器注册 | `@auth.authenticate` 注册校验凭据的异步函数 | `auth/__init__.py:225` |
| 授权处理器注册 | `@auth.on.*` 按 resource+action 注册授权规则 | `auth/__init__.py:732/821` |
| 授权匹配 | 精确 > resource 通配 > 全局，命中即调用 | `auth/__init__.py:96-106` |
| store 命名空间改写 | `@auth.on.store` 把 namespace 前加用户身份 | `auth/__init__.py:592` |
| 加密处理器注册 | `@encryption.encrypt/decrypt.blob|.json` | `encryption/__init__.py:80/147` |
| 加密上下文推导 | `@encryption.context` 从认证用户推导上下文 | `encryption/__init__.py:394` |
| 类型化 HTTP 错误 | 400/401/403/404/409/422/429/5xx 映射 | `errors.py:190` |
| SSE 增量解码 | BytesLineDecoder 分通用换行 + SSEDecoder | `sse.py:17/78` |
| 异步/同步行迭代 | `aiter_lines_raw`/`iter_lines_raw` | `sse.py:142/151` |
| KV 缓存 | `cache_get`/`cache_set`（服务端薄封装） | `cache.py:59/74` |
| SWR 缓存 | stale-while-revalidate 语义 + `SWRResult` | `cache.py:93` |

## 2. 核心类型与接口清单

| 类型 / 函数 | 位置 | 职责 |
| --- | --- | --- |
| `Auth` | `auth/__init__.py:13` | 认证授权注册中心：`_handlers[(r,a)]` + `_global_handlers` |
| `_On` / `_ResourceOn` / `_ThreadsOn` / `_AssistantsOn` / `_CronsOn` / `_StoreOn` | `auth/__init__.py:732/350/512/488/548/592` | 资源/动作级装饰器入口 |
| `AuthContext` / `MinimalUserDict` / `FilterType` | `auth/types.py` | 授权上下文与用户/过滤类型 |
| `Encryption` | `encryption/__init__.py:208` | 静态加解密注册中心（beta） |
| `_EncryptDecorators` / `_DecryptDecorators` | `encryption/__init__.py:70/137` | blob/json 加解密装饰器 |
| `APIError` 层级 | `errors.py:17/62/82/96/108-137` | 类型化异常树，继承 httpx.HTTPStatusError |
| `_map_status_error` / `_araise_for_status_typed` | `errors.py:190/213` | 状态码→异常映射；异步/同步两版 |
| `BytesLineDecoder` / `SSEDecoder` | `sse.py:17/78` | 增量行切分与 SSE 事件解析 |
| `swr` / `SWRResult` | `cache.py:93/37` | stale-while-revalidate 缓存 |
| `StreamPart` | `schema.py`（经 sse.py 引入） | SSE 解析产物类型 |

## 3. 关键调用链

**链一：授权处理器注册与匹配**（`auth/__init__.py`）
1. 用户在 `auth.py` 实例化 `Auth()`，用 `@auth.authenticate` 注册认证函数（仅允许一个，重复抛 `ValueError`）。
2. 用 `@auth.on.threads.create` 等注册动作处理器；`_register_handler` 把 `(resource, action)` 存入 `_handlers`，`None` 归一为 `"*"`。
3. 服务端中间件每请求先跑认证，再按"精确 > resource 通配 > 全局"选最具体处理器调用；返回 None/True 放行、False 拒 403、dict 作查询过滤。

**链二：错误类型化**（`errors.py:213`）
1. `_araise_for_status_typed` 在状态码 ≥400 时读 body（orjson，失败回落字符串）。
2. `_map_status_error` 按状态码实例化对应子类（401→AuthenticationError 等），从 body 提取 `code/param/type`，带 `x-request-id`。

**链三：SSE 解码**（`sse.py:142`）
1. `aiter_lines_raw` 对 `aiter_bytes()` 逐 chunk 调 `BytesLineDecoder.decode`（处理 `\n\r\r\n` 通用换行与跨 chunk 残留）。
2. 每行喂 `SSEDecoder.decode`：累计 event/data/id/retry，遇空行 flush 出 `StreamPart`。

## 4. 配置项

| 配置 | 默认 / 行为 | 位置 |
| --- | --- | --- |
| langgraph.json `auth.path` | 指向 `./auth.py:my_auth`，服务端加载 | `auth/__init__.py:35-38` |
| langgraph.json `encryption.path` | 指向加密实例 | `encryption/__init__.py:233-236` |
| `cache_set` ttl | 上限 1 天；None/0 默认 1 天 | `cache.py:80-81` |
| `swr` fresh_for/max_age | fresh_for 默认 0（每次后台重验）；max_age 默认 1 天 | `cache.py:137-140` |
| 加密 beta 警告 | 构造 `Encryption()` 一次 `LangGraphBetaWarning` | `encryption/__init__.py:29` |

## 5. 错误与重试语义

- 认证/授权处理器必须 `async` 且含 `ctx`、`value` 参数，否则注册时 `ValueError`；全局处理器仅允许一个。
- 加密处理器重复注册抛 `DuplicateHandlerError`；签名必须恰好 `(ctx, data)` 两参，否则 `TypeError`。
- 错误解码对空 body / 非 JSON / 非 dict 均兜底返回 None，不掩盖原始请求错。
- `cache_get/set/swr` 在非服务端（无 `langgraph_api`）直接抛 `RuntimeError`，提示需 Agent Server 0.7.29+。

## 6. 并发细节

- 注册阶段无锁（单线程模块加载期完成）。
- `swr` 的 stale 分支"立即返回旧值 + 后台 best-effort 刷新"，后台任务生命周期由服务端 `langgraph_api.cache` 管理。
- `SSEDecoder`/`BytesLineDecoder` 为有状态迭代器，按流实例化，非线程安全。

## 7. 系统边界

**In-Scope（本仓库源码内）**
- `auth/`、`encryption/`：扩展注册框架与类型；`errors.py`/`sse.py`/`cache.py`：运行时工具。

**Out-of-Scope（不在本仓库源码内）**
- `langgraph_api`（服务端中间件实际调用处理器、`langgraph_api.cache` 实现）；Fernet/具体加密算法由用户自实现；认证服务（OIDC/JWT 校验）由用户在 `@authenticate` 中接入。
- 客户端 API Key 头组装在 `_shared/utilities._get_headers`（见 sdk-client）。

## 8. 与相邻子系统交互

- 上游：用户 `auth.py`/`encryption.py`（经 langgraph.json）→ `Auth`/`Encryption`。
- 本叶子 → sdk-client：`errors._araise_for_status_typed` 被 `_async/http.py` 调用；`sse.py` 被 `http.py` 与 stream transport 复用。
- 下游外部：`langgraph-api` 服务端中间件在每请求/落库时调用注册的处理器；`cache.py` 代理 `langgraph_api.cache`。

## 9. 语言专项适配口径（纯 Python）

- **可插拔钩子链/装饰器注册**：`Auth.on.*` 与 `Encryption.encrypt/decrypt.*` 是装饰器驱动的插件注册点，框架定义契约（ctx/value 或 ctx/data），用户实现。
- **可选依赖与懒加载**：`langgraph_api.cache` 与 `langgraph_api` ASGI transport 均 try-import，缺失时给清晰 `RuntimeError`/fallback，不顶层硬依赖。
- **配置驱动静态分派**：授权匹配顺序、加密是否启用均由"是否注册处理器 + langgraph.json"决定，而非运行时参数。
- **注意口径**：本库 `auth/` 是**服务端授权注册框架**，不是客户端 OAuth 流程；客户端仅做 API Key 头注入（见 sdk-client）。

## 10. 图表清单与质量

| 图 | 文件 | 类型 | archify 质量档 |
| --- | --- | --- | --- |
| 服务端扩展注册与运行时工具 | `sdk-auth-encryption-architecture.html` | architecture | **showcase**（长连线标签缩短后通过） |
| 服务端认证与加解密调用链 | `sdk-auth-encryption-sequence.html` | sequence | standard（迭代新增；5 参与者 12 消息，授权匹配/加解密约束在卡片文字化后通过） |

JSON IR 源：`json/sdk-auth-encryption-architecture.json`、`json/sdk-auth-encryption-sequence.json`。
说明：本叶子以组件/注册关系为主路径，已补 sequence（认证授权→加密写→解密读三阶段时序）。授权匹配顺序、SWR 语义在卡片与 MD 第 3/4 节文字化表达，不另出 dataflow 图。
