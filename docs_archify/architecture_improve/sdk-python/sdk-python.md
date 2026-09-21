# sdk-python 域总览

> 本域是 LangGraph 官方 Python SDK `langgraph-sdk`，源码位于 `libs/sdk-py/langgraph_sdk/`（45 个源文件 / 约 18.8k 行）。
> 域级定位：以异步/同步双轨客户端连接 LangGraph Server，封装五类资源、流式管道，并提供服务端扩展注册框架（认证/加密）与运行时工具。

## 域职责

- **客户端与资源**：顶层客户端工厂、异步/同步双轨、五类资源客户端（Assistants/Threads/Runs/Crons/Store）、HTTP 传输。
- **流式管道**：v3 线程流式协议的共享连接扇出、订阅去重重连、投影解码器、SSE/WebSocket 传输。
- **认证/加密/工具**：服务端授权与静态加密注册框架、类型化错误、SSE 解码、KV/SWR 缓存。

## 叶子索引

| 叶子 | 职责 | 设计文档 | 图 |
| --- | --- | --- | --- |
| sdk-client | 客户端主类、_async/_sync 双轨、资源客户端矩阵、HTTP 传输 | [sdk-client.md](sdk-client/sdk-client.md) | [architecture](sdk-client/sdk-client-architecture.html) · [sequence](sdk-client/sdk-client-sequence.html) |
| sdk-stream | 共享 SSE 扇出、订阅匹配、去重重连、投影解码器、SSE/WS 传输 | [sdk-stream.md](sdk-stream/sdk-stream.md) | [dataflow](sdk-stream/sdk-stream-dataflow.html) · [architecture](sdk-stream/sdk-stream-architecture.html) · [sequence](sdk-stream/sdk-stream-sequence.html) |
| sdk-auth-encryption | Auth/Encryption 服务端注册框架、类型化错误、SSE 解码、缓存 | [sdk-auth-encryption.md](sdk-auth-encryption/sdk-auth-encryption.md) | [architecture](sdk-auth-encryption/sdk-auth-encryption-architecture.html) · [sequence](sdk-auth-encryption/sdk-auth-encryption-sequence.html) |

## 域级图

| 图 | 文件 | 类型 | 说明 |
| --- | --- | --- | --- |
| 域组件拓扑 | [sdk-python-architecture.html](sdk-python-architecture.html) | architecture | 客户端双轨层 + 流式管道 + 服务端扩展三层 + 外部 LangGraph Server 边界 |
| 一次 SDK 调用核心时序 | [sdk-python-sequence.html](sdk-python-sequence.html) | sequence | 构造客户端→认证头→请求发送→SSE 流解码→扇出→结果返回 |
| SSE 流式数据解码管道 | [sdk-python-dataflow.html](sdk-python-dataflow.html) | dataflow | 请求构造→传输层→SSE 解码→控制器扇出→消费出口五阶段数据管道 |

JSON IR 源：`json/sdk-python-architecture.json`、`json/sdk-python-sequence.json`、`json/sdk-python-dataflow.json`。
说明：域级已覆盖 architecture（静态拓扑）+ sequence（调用时序）+ dataflow（数据管道）三种语义模型；无单一实体状态机（lifecycle）与多角色泳道流程（workflow），与已有图信息重复，按资源节省原则省略。

## 域级机制

- **异步/同步双轨**：`_async/` 与 `_sync/` 同名同构，方法一一对应（仅 async/await 差异）；无差异逻辑收敛到 `_shared/`（API Key 三段环境变量优先级、orjson 编解码、路径转义、重连同源校验）。
- **资源抽象**：每个资源一个类，构造时注入 `HttpClient`，方法拼路径/payload 后委托传输层；薄封装 + 统一错误包装。
- **流式管道**：单 `_fanout` 任务消费共享 SSE，按 `matches_subscription` 分发到有界队列；LRU(10000) 去重重放，按 `since` 游标 + 指数退避重连。
- **服务端扩展**：`Auth`/`Encryption` 经装饰器注册，由 `langgraph.json` 加载，服务端中间件在每请求/落库时调用。

## 外部边界

- LangGraph API 服务端、`langgraph_api`（loopback ASGI、`langgraph_api.cache`）、httpx/orjson/websockets、认证服务（OIDC/JWT）与具体加密算法——均不在本仓库源码内。
- `libs/sdk-js` 仅为 README 占位（迁移至外部 langchain-ai/langgraphjs），不在本仓库源码内、不拆叶。
