# sdk-stream（SSE 流式解码与订阅）

> 本文是 `sdk-python` 域下的叶子子系统文档，覆盖从 httpx 字节流到消费事件的三级解码
> 管道：行切分（`BytesLineDecoder`）、SSE 事件解析（`SSEDecoder`）、通道投影解码
> （Data/Messages/ToolCalls/Subgraphs/Extensions）与订阅控制器（`StreamController`）。
> 端点调用见 `sdk-resources`；HTTP 底层见 `sdk-http`。
>
> 源码基准：`libs/sdk-py/langgraph_sdk/sse.py`、`stream/`，git HEAD `e539ac12`（2026-09-09）。

## 1. 功能清单

| 能力 | 说明 | 源码路径 |
|---|---|---|
| 增量行切分 | 按 `\n \r \r\n` 切分字节流，缓冲不完整行 | `sse.py:17`（`BytesLineDecoder`） |
| SSE 事件解析 | event/data/id/retry 字段→`StreamPart` | `sse.py:78`（`SSEDecoder`） |
| 异步/同步行迭代 | `aiter_lines_raw`/`iter_lines_raw` | `sse.py:142/151` |
| 流式控制器 | 订阅、事件 ID 去重、关闭 | `stream/controller.py:96`（`StreamController`） |
| 已见事件 ID | `_SeenEventIds` 去重 | `stream/controller.py:33` |
| 数据解码器 | Data/Messages/ToolCalls/Subgraphs/Extensions | `stream/decoders.py:106/139/199/257/334` |
| 通道校验 | `validate_interleave_channels` | `stream/decoders.py:34` |

## 2. 核心类型与接口清单

| 类型 / 函数 | 位置 | 职责 |
|---|---|---|
| `BytesLineDecoder` | `sse.py:17` | 增量行切分器 |
| `SSEDecoder` | `sse.py:78` | SSE 协议字段解析，空行触发事件下发 |
| `StreamPart` | `schema.py` | 事件对象（event/data/id） |
| `StreamController` | `stream/controller.py:96` | 订阅与流编排 |
| `_SeenEventIds` | `stream/controller.py:33` | 事件 ID 去重 |
| `DataDecoder`/`MessagesDecoder`/`ToolCallsDecoder`/`SubgraphsDecoder`/`ExtensionsDecoder` | `stream/decoders.py:106-334` | 按通道投影解码 |
| `Decoder(Protocol)` | `stream/decoders.py:102` | 解码器契约 |

## 3. 关键调用链

**链一：SSE 解码管道（`sse.py:142`）**

1. `aiter_lines_raw(response)`：`async for chunk in response.aiter_bytes()`，经 `BytesLineDecoder.decode(chunk)` 切行（`sse.py:142-146`）。
2. `BytesLineDecoder.decode` 缓冲未以换行结尾的尾段，下次拼回（`sse.py:49-64`）。
3. 每行喂给 `SSEDecoder.decode(line)`：`event:/data:/id:/retry:` 字段累积；空行时用 `orjson.loads(self._data)` 生成 `StreamPart` 并复位（`sse.py:91-114`）。
4. 结尾 `flush()` 吐出残留缓冲（`sse.py:68-75/147-148`）。

**链二：通道投影（`stream/decoders.py:106`）**

1. 每个 `StreamPart` 按命名空间/路由键分发到对应解码器。
2. `DataDecoder`/`MessagesDecoder` 等把原始事件投影成消费层对象；`_message_route_key` 计算路由键（`stream/decoders.py:64`）。
3. `_SeenEventIds` 对事件 ID 去重，支撑断线续传。

## 4. 配置项

| flag / option | 默认 / 行为 | 位置 |
|---|---|---|
| interleave channels | `validate_interleave_channels` 校验 | `stream/decoders.py:34` |
| SSE retry 字段 | 解析但由上层决定重连 | `sse.py:133-135` |
| 事件 ID | 空行后不复位 last_event_id（SSE 规范） | `sse.py:109` |

## 5. 错误与重试语义

- **JSON 解析**：data 字段用 orjson 解析，失败由调用方处理。
- **断线续传**：`last_event_id` 与 `_SeenEventIds` 支撑重连后去重续传。
- **忽略字段**：未知 SSE 字段忽略，注释行（`:` 开头）忽略（`sse.py:116-117/137`）。
- 解码器本身不重试；传输层重试在 `sdk-http`/`sdk-client`。

## 6. 并发细节

- 异步/同步双轨：`aiter_lines_raw`（async）与 `iter_lines_raw`（sync）镜像实现。
- `StreamController` 管理 `_Subscription` 与关闭（`stream/controller.py:65/80`）。
- 增量解码器是有状态对象（buffer/trailing_cr/_event/_data），不可跨流共享。

## 7. 系统边界

**In-Scope（本仓库源码内）**
- 字节→行→事件→投影解码管道、订阅控制器。

**Out-of-Scope（不在本仓库源码内）**
- **httpx**：底层 HTTP 传输，不在本仓库源码内。
- **LangGraph API**：SSE 事件的产生方（服务端），不在本仓库源码内。
- `aiter_bytes` 的 HTTP 响应来自 `sdk-http`。

## 8. 与相邻子系统交互

- `sdk-http`/`sdk-resources` → 本叶子消费 httpx 响应字节流。
- 本叶子 → 应用消费者（async/async iterator）。
- 本叶子依赖 `schema.StreamPart`（sdk-client schema）。

## 9. 语言专项适配口径（纯 Python）

- **双轨实现模式**：`aiter_lines_raw`/`iter_lines_raw`、`_async/stream.py`/`_sync/stream.py` 镜像实现。
- **隐藏状态机在循环里**：`BytesLineDecoder`/`SSEDecoder` 是有状态增量解码器，状态机（缓冲/字段累积）隐藏在 `decode()` 循环里——按语言专项口径显式评估，其状态变迁（累积→空行下发→复位）已由 dataflow 管道表达，故不另画 lifecycle。
- **协议契约**：`Decoder(Protocol)` 定义解码器契约，多解码器实现同一接口。

## 10. 图表清单与质量

| 图 | 文件 | 类型 | archify 质量档 |
|---|---|---|---|
| 组件架构图 | `sdk-stream-architecture.html` | architecture | showcase |
| SSE 解码数据流 | `sdk-stream-dataflow.html` | dataflow | showcase |

- JSON IR 源文件位于 `json/` 目录。
- 省略 sequence：解码是单向管道，已由 dataflow 表达；跨方请求时序见 `sdk-resources`。
- 省略 workflow/lifecycle：无多角色泳道；增量解码器的状态机隐藏在循环里，已在第 9 节说明，未单画。
