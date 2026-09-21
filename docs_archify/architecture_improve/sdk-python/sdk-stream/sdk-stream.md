# v3 线程流式控制器与管道（sdk-stream）

> 本文是 `sdk-python` 域下的叶子子系统文档。域级总览见 `../sdk-python.md`。
> 本文展开 v3 以线程为中心的流式协议：共享 SSE/WS 连接、订阅扇出、去重重连、
> 投影解码器；底层 HTTP/SSE 基础解码见 `../sdk-client/sdk-client.md` 与
> `../sdk-auth-encryption/sdk-auth-encryption.md`。
>
> 源码基准：`libs/sdk-py/langgraph_sdk/stream/`（controller.py、decoders.py、
> subscription.py、sync_controller.py、multi_cursor_buffer.py、transport/）。

## 1. 功能清单

| 能力 | 说明 | 源码路径 |
| --- | --- | --- |
| 订阅注册/注销 | 按 channels/namespaces/depth 建订阅，分配 id 与有界队列 | `stream/controller.py:144/182/193` |
| 共享流扇出 | 单消费者 `_fanout` 把一条 SSE 分发给多个订阅队列 | `stream/controller.py:229` |
| 事件去重 | 有界 LRU `_SeenEventIds` 按 event_id 去重服务端重放 | `stream/controller.py:33` |
| 流轮转（rotate） | 订阅 fit 扩宽时 open-new-before-close-old | `stream/controller.py:319` |
| 断连重连 | 按游标 seq 写 `since`，指数退避+jitter，最多 5 次 | `stream/controller.py:284/276` |
| 游标跟踪 | `observe_applied_through_seq` 推进重连游标 | `stream/controller.py:369/380` |
| 通道推断/匹配 | `infer_channel` + `namespace_matches` 前缀匹配 | `stream/subscription.py:68/39` |
| 过滤器合并/覆盖 | `compute_union_filter` / `filter_covers` | `stream/subscription.py:107/161` |
| 投影解码器 | Data/Messages/ToolCalls/Subgraphs/Extensions 五种状态机 | `stream/decoders.py` |
| 传输抽象 | `AsyncProtocolTransport` / `SyncProtocolTransport` 协议 | `stream/transport/base.py:33/45` |
| SSE 传输 | `POST /threads/{id}/stream/events` + 命令 `POST .../commands` | `stream/transport/http.py:33` |
| WebSocket 传输 | HTTP 命令 + WS 事件流 | `stream/transport/ws.py:25` |
| 多游标缓冲 | append-only log，每迭代器独立游标 | `stream/multi_cursor_buffer.py:18` |
| 同步对应物 | `SyncStreamController` 阻塞式同构 | `stream/sync_controller.py:62` |

## 2. 核心类型与接口清单

| 类型 / 函数 | 位置 | 职责 |
| --- | --- | --- |
| `StreamController` | `controller.py:96` | 异步扇出控制器：订阅表 + 共享流 + 去重 + 轮转 + 重连 |
| `_Subscription` / `_SeenEventIds` | `controller.py:65/33` | 订阅记录（id+params+queue）；有界 LRU 去重集 |
| `EventStreamHandle` | `transport/base.py:15` | `events` 迭代器 + `ready`/`done` future + `close()` |
| `AsyncProtocolTransport` | `transport/base.py:33` | Protocol 契约：`send_command`/`open_event_stream`/`close` |
| `ProtocolSseTransport` | `transport/http.py:33` | SSE 实现，绑定单个 thread_id |
| `ProtocolWebSocketTransport` | `transport/ws.py:25` | WS 实现，命令走 HTTP、事件走 WS |
| `MessagesDecoder`/`ToolCallsDecoder`/`SubgraphsDecoder`/`DataDecoder`/`ExtensionsDecoder` | `decoders.py:139/199/257/106/334` | 每通道 event→item 状态机 |
| `MultiCursorBuffer` | `multi_cursor_buffer.py:18` | 无界 append-only 日志 + 每迭代器游标 |
| `matches_subscription`/`compute_union_filter`/`filter_covers` | `subscription.py:88/107/161` | 匹配/合并/覆盖纯函数 |
| `SyncStreamController` | `sync_controller.py:62` | 同步阻塞版控制器 |

## 3. 关键调用链

**链一：subscribe → 扇出**（`controller.py:201`）
1. `subscribe(channels, namespaces, depth)` → `_subscription_iter` 建 `SubscribeParams`。
2. `_register_subscription` 分配 id 与 `asyncio.Queue(maxsize=1024)`。
3. `_reconcile_stream`：现有流已覆盖则直接复用；否则 `compute_union_filter` 算并集，`transport.open_event_stream` 开新流，旧流 `_close_after` 延迟关闭。
4. `_ensure_fanout_running` 起 `_fanout` 任务。
5. 订阅方 `await sub.queue.get()` 阻塞取事件，收到 `None` 哨兵即结束。

**链二：fanout 分发**（`controller.py:229`）
1. `async for event in self._dedup_iter(shared.events)`：`_dedup_iter` 按 event_id 经 LRU 去重并推进游标。
2. 遍历所有订阅，`matches_subscription(event, sub.params)` 命中则 `put_nowait` 入队。
3. 共享流因非取消错误结束时，`_reconnect_shared_stream` 用当前最新 filter + `since=cursor` 重连，失败则给所有队列投 `None` 结束。

**链三：SSE 传输泵**（`transport/http.py:116`）
1. `open_event_stream` 起 `pump` 任务：`POST .../stream/events`，`raise_for_status` 后 `ready.set_result`。
2. `BytesLineDecoder.decode(chunk)` → `SSEDecoder.decode(line)` → data 为 dict 时入 `queue`。
3. `close()` 置 `cancel_event`、投 `None` 哨兵、取消 pump 任务。

## 4. 配置项

| 配置 | 默认 / 行为 | 位置 |
| --- | --- | --- |
| `max_queue_size` | 每订阅队列上限 1024 | `controller.py:119`、`transport/http.py:49` |
| `seen_event_ids_max` | 去重 LRU 上限 10000 | `controller.py:120` |
| `max_reconnect_attempts` | 5 次 | `controller.py:121` |
| `reconnect_backoff_base/cap` | 0.1s 起、2.0s 封顶的指数退避 + 25% jitter | `controller.py:122-123/276` |
| WS ping_interval/timeout | 20s/20s | `transport/ws.py:38-39` |
| 受保护通道 | `lifecycle/tools/input` 不得用于 interleave（fail-closed） | `decoders.py:31/34` |

## 5. 错误与重试语义

- 传输层 `pump` 把异常经 `ready`/`done` future 传出：headers 前错误 reject `ready`，运行中错误 resolve `done`。
- fanout 遇非取消异常视为 drop：`_reconnect_shared_stream` 最多 5 次，指数退避+ jitter；耗尽后给所有订阅队列投 `None` 干净结束。
- 重连用请求体 `since`（非 `Last-Event-ID` 头），服务端按 seq 过滤重放，客户端 LRU 再去重叠合段。
- `MultiCursorBuffer` 关闭后 push 仍接受（新游标可见完整历史），无驱逐策略（生命周期归持有方）。

## 6. 并发细节

- 单 `_fanout` 任务是共享流唯一消费者（fan-out 到多队列）；每订阅一个 `asyncio.Queue(maxsize=1024)`，`put_nowait` 非阻塞。
- 轮转时旧流关闭与新流吸收重放解耦（`_close_after` 延迟关闭）；`_rotation_close_tasks` 集合跟踪并在 `close()` 中 gather。
- `MultiCursorBuffer._wakeups` 为等待中的 `asyncio.Future` 集合，push/close 统一 `_wake_all`。
- 同步控制器（`sync_controller.py`）用阻塞迭代，无事件循环；`signal_paused` 协调暂停。

## 7. 系统边界

**In-Scope（本仓库源码内）**
- `stream/` 全部：控制器、订阅匹配、解码器、传输抽象与 SSE/WS 实现、多游标缓冲。

**Out-of-Scope（不在本仓库源码内）**
- LangGraph API 服务端（生产 `.../commands`、`.../stream/events` 的一方）；`langchain_protocol`（Event/SubscribeParams 类型）；`websockets` 第三方库。
- projection 迭代器与 `ChatModelStream` 的组装在 `_async/stream.py`（属 sdk-client 层调用方）。

## 8. 与相邻子系统交互

- 上游 sdk-client：`_async/stream.py` 的 `AsyncThreadStream` 构造 `StreamController` 与 transport，订阅投影。
- 本叶子 → sdk-client：复用 `_shared/utilities._quote_path_param`、顶层 `httpx.AsyncClient`；SSE 基础解码用 `sse.py`（sdk-auth-encryption 叶子）。
- 下游外部：`POST /threads/{id}/commands`（JSON in/out）与 `POST /threads/{id}/stream/events`（SSE）或 WebSocket。

## 9. 语言专项适配口径（纯 Python）

- **能力缝**：`AsyncProtocolTransport`/`SyncProtocolTransport` 是 Protocol（duck-type 契约），SSE 与 WS 两实现可互换；解码器是 `feed(event)->iter` 统一契约的策略对象族。
- **双轨实现**：`StreamController` ↔ `SyncStreamController` 同构镜像，传输有 async/sync 两份 handle（`EventStreamHandle` vs `SyncEventStreamHandle`）。
- **生产者-消费者**：单消费者 fan-out 到多有界队列；`MultiCursorBuffer` 是 append-only 多游标日志。
- **纯函数归并**：匹配/合并/覆盖（`subscription.py`）无状态，直接移植自 JS SDK（注释标注对应 `.ts` 文件）。

## 10. 图表清单与质量

| 图 | 文件 | 类型 | archify 质量档 |
| --- | --- | --- | --- |
| v3 流式事件管道 | `sdk-stream-dataflow.html` | dataflow | standard（dataflow 限 5 stage、流标签与节点重叠，经缩短副标题+`labelDy` 下压后通过） |
| 流式传输与解码分层 | `sdk-stream-architecture.html` | architecture | **showcase**（迭代新增；移除内部订阅表/去重子组件避免竖向连线方向校验失败，主路径横向走通后通过） |
| SSE 解码与扇出时序 | `sdk-stream-sequence.html` | sequence | standard（迭代新增；自消息 span 0px 校验失败，将内部步骤改为跨参与者消息后通过） |

JSON IR 源：`json/sdk-stream-dataflow.json`、`json/sdk-stream-architecture.json`、`json/sdk-stream-sequence.json`。
说明：本叶子以管道为主路径，已补 architecture（传输/解码/控制三层静态拓扑）与 sequence（SSE 解码链时序）。状态机（重连退避、轮转）已在卡片与 MD 第 5/6 节文字化表达，不另出 lifecycle 图。
