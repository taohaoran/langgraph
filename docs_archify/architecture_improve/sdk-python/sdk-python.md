# sdk-python（Python SDK）域总览

> 本域包含以下叶子子系统；各叶子详情见对应文档。
> 源码基准：langgraph monorepo main 分支，commit `e539ac12`；主语言纯 Python（libs/sdk-py/langgraph_sdk）。

## 1. 域职责

sdk-python 域（`libs/sdk-py/langgraph_sdk`）是与 LangGraph Server REST/SSE API 交互的 Python 客户端：
`Client` 主类、资源端点（assistant/thread/run/cron/store）、流式控制器、HTTP 封装与认证加密。覆盖 5 个叶子：

- **sdk-client**：`Client` 主类与运行入口（`client.py`）；
- **sdk-resources**（本轮新增）：资源端点客户端（`assistants/threads/runs/cron/store`）；
- **sdk-auth-encryption**：认证与加密（`auth/*`、`encryption/*`）；
- **sdk-stream**：流式控制器与 SSE 解码（`stream/*`、`sse.py`）；
- **sdk-http**（本轮新增）：同步/异步 HTTP 封装（`http.py`、`_shared/*`）。

## 2. 叶子索引

| 叶子 | 文档 | 架构图 | 第二图 | 职责一句话 |
|------|------|--------|--------|-----------|
| sdk-client | [sdk-client.md](sdk-client/sdk-client.md) | [架构图](sdk-client/sdk-client-architecture.html) | [时序图](sdk-client/sdk-client-sequence.html) | Client 主类与入口 |
| sdk-resources（新增） | [sdk-resources.md](sdk-resources/sdk-resources.md) | [架构图](sdk-resources/sdk-resources-architecture.html) | [时序图](sdk-resources/sdk-resources-sequence.html) | 资源端点客户端 |
| sdk-auth-encryption | [sdk-auth-encryption.md](sdk-auth-encryption/sdk-auth-encryption.md) | [架构图](sdk-auth-encryption/sdk-auth-encryption-architecture.html) | [时序图](sdk-auth-encryption/sdk-auth-encryption-sequence.html) | 认证与加密 |
| sdk-stream | [sdk-stream.md](sdk-stream/sdk-stream.md) | [架构图](sdk-stream/sdk-stream-architecture.html) | [数据流图](sdk-stream/sdk-stream-dataflow.html) | 流式控制器与解码 |
| sdk-http（新增） | [sdk-http.md](sdk-http/sdk-http.md) | [架构图](sdk-http/sdk-http-architecture.html) | [时序图](sdk-http/sdk-http-sequence.html) | HTTP 封装与重试 |

## 3. 域级机制细节

- **双轨实现（纯 Python 口径）**：`_async/client.py` 与 `_sync/client.py` 两套 API 共享
  `runtime.py`/`http.py` 基座；`AsyncClient`/`Client` 双轨并行——SDK 的典型双轨模式。
- **注册表与工厂**：`_shared/utilities.py` 的 `_get_model` 等工厂按端点类型分派；`schema.py` 统一
  数据模型；错误类型注册见 `errors.py`。
- **流式管道**：SSE 原始流 → 增量解码器（json/sse/text 多格式）→ 流式控制器（多游标缓冲去重）→
  用户迭代器；解码器是"隐藏状态机在循环里"（逐块增量解析），已补 dataflow 图。
- **可选依赖与懒加载**：`try/except ImportError` 对可选项（如 httpx 版本差异）优雅降级；
  `cache.py` 提供解析缓存。
- **外部边界**：LangGraph Server（REST/SSE API）、LangSmith 远端、httpx/aiohttp 传输层均不在本域源码内。

## 4. 域级图（架构图 showcase 档，时序图/数据流图 standard 档，render 退出码全 0）

![sdk-python 域架构图](sdk-python-architecture.html)

![sdk-python 域时序图](sdk-python-sequence.html)

![sdk-python 域数据流图](sdk-python-dataflow.html)

> 本域未生成 workflow/lifecycle 图：客户端无多角色流程与单实体状态机语义（流式管道由 dataflow 表达），按资源节省原则省略。
