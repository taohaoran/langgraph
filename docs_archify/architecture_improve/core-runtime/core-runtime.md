# core-runtime 域总览

> 本页是 `core-runtime` 域（`libs/langgraph/langgraph` 核心运行时）的总览。
> 系统级总览与系统图由组织者统一产出（位于输出根），本页只汇总本域职责、叶子索引与跨叶机制。
>
> 源码基准：`langgraph` main 分支，commit `e539ac122f4126f6dd850581c1494948cf620e31`。

## 域职责

`core-runtime` 是 LangGraph 的执行内核，覆盖从"声明式建图"到"超级步执行"再到"流式输出"的完整闭环：

- **建图**：把 `TypedDict` 状态 schema 与节点/边编译成可执行图。
- **执行**：以 Bulk Synchronous Parallel（BSP / Pregel）超级步模型并发调度节点。
- **状态**：把状态拆成独立通道，按 reducer 聚合，支持 checkpoint 持久化与 Delta 回放。
- **流式**：把执行事件经多路复用器投影成多种流模式输出。
- **适配**：函数式 API、托管值、远端代理、图绘制、调试映射。
- **基础设施**：Runnable 包装、配置合并、错误族、序列化白名单、队列/超时/重试。

## 叶子索引

| 叶子 | 职责 | 文档 |
| --- | --- | --- |
| graph-state-api | StateGraph/MessageGraph 构建 API、分支/节点注册、Channels 状态通道族 | [graph-state-api.md](graph-state-api/graph-state-api.md) |
| pregel-engine | Pregel 超级步执行引擎（BSP 调度、并发执行、重试、校验） | [pregel-engine.md](pregel-engine/pregel-engine.md) |
| pregel-io-checkpoint | 输入输出映射、ChannelRead/Write 桥、checkpoint 构建与水化、消息/工具流回调 | [pregel-io-checkpoint.md](pregel-io-checkpoint/pregel-io-checkpoint.md) |
| stream-framework | v3 流式基础设施：事件协议、StreamMux 多路复用、流转换器、StreamChannel | [stream-framework.md](stream-framework/stream-framework.md) |
| func-api | `@entrypoint`/`@task` 函数式 API 及其到 Pregel 的桥接 | [func-api.md](func-api/func-api.md) |
| managed-values | `ManagedValue` 抽象与 `IsLastStep`/`RemainingSteps` 运行期托管值 | [managed-values.md](managed-values/managed-values.md) |
| internal-foundation | `_internal/` 与根级基础设施（Runnable/配置/常量/错误/serde/队列/超时/重试/重放/Runtime） | [internal-foundation.md](internal-foundation/internal-foundation.md) |
| remote-debug | `RemoteGraph` 远端代理、`PregelProtocol`、debug 映射、`draw_graph` 图绘制、v3 线协议适配 | [remote-debug.md](remote-debug/remote-debug.md) |

## 域级机制要点

### Pregel 执行模型（BSP）

一次 `invoke` 是多轮**超级步（super-step）**循环（见 [pregel-engine](pregel-engine/pregel-engine.md)）：

1. **计划** `prepare_next_tasks`：消费 `TASKS` Topic（PUSH/Send）+ 由 `updated_channels × trigger_to_nodes` 反查被触发的 PULL 节点。
2. **执行** `PregelRunner.tick`：并发跑选定节点，执行阶段各节点**互不可见**彼此写入。
3. **更新** `apply_writes`：步末按通道聚合写入、`channel.update(vals)`、 bumped 版本。
4. **持久化** `_put_checkpoint`：保存快照后回到计划阶段，直至无待触发通道 → `done`。

退出条件：无任务 / 超 `recursion_limit`（`out_of_steps`）/ `drain` 请求 / `interrupt_before|after` / 节点失败路由到 error_handler。

### 状态通道

状态不存于字典，而是拆成独立 `BaseChannel`（见 [graph-state-api](graph-state-api/graph-state-api.md)）：

- `LastValue`（每步至多一值）/`BinaryOperatorAggregate`（reducer 聚合）/`Topic`（PubSub）/`NamedBarrierValue`（多路 join）/`Ephemeral`/`Untracked`/`DeltaChannel`。
- `Annotated[T, reducer]` 在编译期被 `_get_channels` 静态分派到通道实现；无 reducer 字段退化为 `LastValue`。
- 通道的 `checkpoint()`/`from_checkpoint()` 契约被 [pregel-io-checkpoint](pregel-io-checkpoint/pregel-io-checkpoint.md) 消费；`DeltaChannel` 缺值时回放祖先写入重建。

### 流式输出

执行事件经 [stream-framework](stream-framework/stream-framework.md) 投影：

- v2 `StreamPart` → `ProtocolEvent` → `StreamMux` 按注册顺序跑 transformer 流水线 → 各模式 `StreamChannel` → 用户迭代。
- 内置 transformer：values/updates/messages/custom/debug/tasks/checkpoints/lifecycle/subgraph。
- 背压由调用者驱动 pump（每次游标前进至多 1 事件）；根 mux 分配单调 `seq` 保证全序。

### 桥接与适配

- [func-api](func-api/func-api.md)：`@entrypoint` 把函数编译成单节点 `Pregel`（START→函数→END/PREVIOUS）；`@task` 产出可并行 future。
- [managed-values](managed-values/managed-values.md)：运行期由 `PregelScratchpad` 现场派生、不持久化的注入值。
- [remote-debug](remote-debug/remote-debug.md)：`RemoteGraph` 实现同一 `PregelProtocol`，把本地调用翻译为 HTTP/SSE 到 LangGraph Server。

### 横切基础设施

[internal-foundation](internal-foundation/internal-foundation.md) 被全域复用：`RunnableCallable`（函数签名→参数注入）、`merge_configs`、`GraphBubbleUp` 错误族、`default_retry_on`、`AsyncQueue/SyncQueue`、serde 白名单、`ReplayState` 时间旅行。

## 域级图

本次迭代在域级补 3 张图，覆盖"组件拓扑 + 调用时序 + 数据管道"三种语义：

| 图 | 文件 | 类型 | 说明 |
| --- | --- | --- | --- |
| 域组件分层架构 | [core-runtime-architecture.html](core-runtime-architecture.html) | architecture | 8 个叶子模块按"建图/执行/输出/支撑"分层 + 外部边界（CheckpointSaver/langchain_core/LangGraph Server） |
| 一次 invoke 端到端时序 | [core-runtime-sequence.html](core-runtime-sequence.html) | sequence | 用户→StateGraph.compile→PregelLoop→节点执行→CheckpointSaver→StreamMux→返回 |
| invoke 数据管道 | [core-runtime-workflow.html](core-runtime-workflow.html) | dataflow | 用户输入→编译→超级步执行→持久化→流式输出的端到端数据流 |

- JSON IR 源：`json/core-runtime-architecture.json`、`json/core-runtime-sequence.json`、`json/core-runtime-workflow.json`。
- 三张均因 showcase 布局校验（跨层边标签间距/侧边路由穿越节点/纵向边标签压节点）未全过，按规范降 standard 渲染（均成功，约 805KB 自包含）。

## 图表质量档位汇总

本域 8 叶 + 域级共产出 14 张图（叶子：architecture 7 + lifecycle 2 + dataflow 1 + sequence 6；域级：architecture 1 + sequence 1 + dataflow 1），全部渲染成功；受 archify showcase 布局校验（边标签间距、侧边路由穿越节点、同列纵向边）严格约束，**全部按规范降为 standard 档**，降档原因已在各叶子第 10 节与域级图小节如实披露。
