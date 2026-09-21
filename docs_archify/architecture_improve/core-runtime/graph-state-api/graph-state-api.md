# 图状态构建 API 与状态通道（graph-state-api）

> 本文是 `core-runtime` 域下的叶子子系统文档。域级总览见 `../core-runtime.md`。
> 本文只展开图构建（`graph/`）与状态通道（`channels/`）的职责边界，不重复展开 Pregel
> 超级步执行循环（见 `../pregel-engine/pregel-engine.md`）、输入输出与 checkpoint
> （见 `../pregel-io-checkpoint/pregel-io-checkpoint.md`）。
>
> 源码基准：`langgraph` main 分支，commit `e539ac122f4126f6dd850581c1494948cf620e31`。

## 1. 功能清单

本叶子是 LangGraph 的"图定义层"：开发者用声明式 API 描述节点与边，编译产物是一个
继承自 `Pregel` 的可执行图。状态不直接存于字典，而是拆成独立的"通道（channel）"。

| 能力 | 说明 | 源码路径 |
| --- | --- | --- |
| `StateGraph` 构建器 | 图的 builder 类，持有 nodes/edges/branches/channels/managed 注册表，不可直接执行，需 `.compile()` | `graph/state.py:131`（`StateGraph`） |
| `add_node` | 注册节点；自动推断节点名、从返回类型 `Command[Literal[...]]` 推断出口 ends；支持 retry/cache/error_handler/timeout/trace_policy | `graph/state.py:667`（实现）、`graph/_node.py:90`（`StateNodeSpec`） |
| `add_edge` / `add_sequence` | 注册静态边；多起点进入同一终点时生成 join 通道；`add_sequence` 链式加节点加边 | `graph/state.py:928`、`graph/state.py:1032` |
| `add_conditional_edges` | 注册条件分支；把 path 可调用对象包成 `BranchSpec`，运行期由 writer 写入 `branch:to:` 控制通道 | `graph/state.py:982`、`graph/_branch.py:83`（`BranchSpec`） |
| `set_entry_point` / `set_finish_point` | `START`/`END` 保留节点的便捷别名 | `graph/state.py:1079`、`graph/state.py:1116` |
| `validate` / `compile` | 编译前校验（必有入口、目标存在），编译为 `CompiledStateGraph`，注入 serde 白名单、应用节点默认策略、attach 节点/边/分支 | `graph/state.py:1129`、`graph/state.py:1177` |
| `CompiledStateGraph` | 继承 `Pregel`；把 builder 的图元转成 `PregelNode`（triggers+writers）与控制通道 | `graph/state.py:1404` |
| schema→通道解析 | 从 `TypedDict`/`BaseModel`/dataclass 的 `Annotated[T, reducer]` 注解解析出每个字段对应的通道实例；无 reducer 字段退化为 `LastValue` | `graph/state.py:1815`（`_get_channels`）、`graph/state.py:1850`（`_get_channel`） |
| 消息 reducer `add_messages` | 按 id 合并消息列表，支持 `RemoveMessage` 删除、`REMOVE_ALL_MESSAGES` 清空、`langchain-openai` 格式 | `graph/message.py:61` |
| `MessagesState` / `MessageGraph` | 预置"消息列表"状态与已废弃的单列表消息图 | `graph/message.py:372`、`graph/message.py:316` |
| UI 消息推送 | `push_ui_message`/`delete_ui_message` + `ui_message_reducer`，向 `ui` 状态键与流式 writer 同步推送 UI 事件 | `graph/ui.py:61`、`graph/ui.py:165` |
| 通道基类 `BaseChannel` | 定义 get/update/checkpoint/from_checkpoint/consume/finish 生命周期契约 | `channels/base.py:19` |
| 值型通道族 | `LastValue`/`AnyValue`/`EphemeralValue`/`UntrackedValue`/`BinaryOperatorAggregate`/`DeltaChannel` | `channels/last_value.py:20`、`channels/any_value.py:15`、`channels/ephemeral_value.py:15`、`channels/untracked_value.py:15`、`channels/binop.py:65`、`channels/delta.py:25` |
| 同步屏障通道 | `NamedBarrierValue`/`NamedBarrierValueAfterFinish`：等待全部具名来源到达才可用 | `channels/named_barrier_value.py:13` |
| PubSub 通道 `Topic` | 每步收集写入值为列表，`accumulate=False` 时步末清空（广播语义） | `channels/topic.py:23` |

## 2. 核心类型与接口清单

| 类型 / 函数 | 位置 | 职责 |
| --- | --- | --- |
| `StateGraph` | `graph/state.py:131` | builder，泛型 `[StateT, ContextT, InputT, OutputT]` |
| `CompiledStateGraph` | `graph/state.py:1404` | 继承 `Pregel`，持有 `builder`、`schema_to_mapper`，实现 `attach_node/attach_edge/attach_branch/_migrate_checkpoint` |
| `StateNodeSpec` | `graph/_node.py:90` | 节点 spec：runnable、input_schema、retry/cache/timeout/trace_policy、is_error_handler、ends、defer |
| `BranchSpec` | `graph/_branch.py:83` | 分支 spec：path Runnable、ends 映射表、input_schema；`run()` 把路由逻辑包成 `ChannelWrite` writer |
| `_NodeDefaults` | `graph/state.py:102` | `set_node_defaults` 设置的全图默认策略，编译时下发到各节点 spec |
| `BaseChannel` | `channels/base.py:19` | 通道抽象基类（Generic[Value, Update, Checkpoint]），定义 `get/update/checkpoint/from_checkpoint/consume/finish` |
| `BinaryOperatorAggregate` | `channels/binop.py:65` | reducer 通道；`update` 把 `operator(acc, v)` 逐值折叠；识别 `Overwrite` 三种序列化形态 |
| `LastValue` / `LastValueAfterFinish` | `channels/last_value.py:20`、`:81` | 每步至多一值；后者仅在 `finish()` 后可用、`consume()` 后清空 |
| `NamedBarrierValue(AfterFinish)` | `channels/named_barrier_value.py:13`、`:84` | 多路 join 屏障；`seen == names` 才 `get()` 成功 |
| `Topic` | `channels/topic.py:23` | PubSub；`_flatten` 拍平 list 写入，`accumulate=False` 步末清空 |
| `DeltaChannel` | `channels/delta.py:25` | beta；checkpoint 只存哨兵，靠回放祖先写入经批式 reducer 重建状态 |
| `add_messages` | `graph/message.py:61` | 消息合并 reducer（按 id upsert、删除墓碑） |

## 3. 关键调用链

**链一：构建与编译（声明式 → 可执行）**

1. 开发者 `StateGraph(State)`，`add_node("A", fn)`、`add_edge(START, "A")`（`state.py:667/928`）。
2. `add_node` 用 `get_type_hints` 推断首个参数注解为节点 `input_schema`；若返回类型为 `Command[Literal["x","y"]]`，把 `"x"/"y"` 提取为该节点的静态出口 `ends`（`state.py:836-856`）。
3. `_add_schema`（`state.py:343`）对 state/input/output schema 调 `_get_channels`（`state.py:1815`）：遍历 `get_type_hints`，对每个 `Annotated[T, meta]` 字段，先查是否托管值（`_is_field_managed_value`），再查是否携带通道实例或通道类（`_is_field_channel`），最后查最后一个 meta 是否是二元 callable（`_is_field_binop`，即 reducer）；都不匹配则退化为 `LastValue(T)`（`state.py:1871`）。
4. `.compile()`（`state.py:1177`）：`validate()` 校验图拓扑 → 计算 output/stream 通道 → 应用 `_node_defaults`（error_handler/retry/cache/timeout，逐节点优先）→ 构造 `CompiledStateGraph`，注入 `START: EphemeralValue(input_schema)` → `attach_node(START, None)` 及各业务节点 → `attach_edge`/`attach_branch` → `validate()`。
5. `attach_node`（`state.py:1444`）把节点包成 `PregelNode`：`channels`=输入状态键，`writers`=[`ChannelWrite` 写入状态更新 + `_control_branch` 写入 goto/Send]，并为该节点建一个 `branch:to:<key>` 触发通道（defer 节点用 `LastValueAfterFinish`，否则 `EphemeralValue(guard=False)`）。
6. `attach_edge`（`state.py:1551`）：单边向起点节点的 writers 追加写 `branch:to:<end>`；多起点 join 时建 `NamedBarrierValue`（或 defer 用 `...AfterFinish`）通道，终点节点 triggers 订阅它，各起点写入自己的名字。

**链二：条件分支路由**

1. `add_conditional_edges` 把 path 包成 `BranchSpec.from_path`（`_branch.py:89`），从 `Literal` 返回注解或 `path_map` 推断 destinations。
2. 运行期 `BranchSpec.run`（`_branch.py:122`）返回一个 `RunnableCallable` writer：先读（可选 `reader` 读状态并与 dict 输入合并），调 `path.invoke` 得到目的地序列，经 `writer(destinations, False)` 生成写 `branch:to:<dest>` 的 `ChannelWriteEntry`，由 `_finish` 直接 `do_write` 或返回 `ChannelWrite`。

**链三：通道更新语义**

- 每超级步末，Pregel 把本步所有写入按通道聚合后调 `channel.update(values)`（`channels/base.py:90`）。`BinaryOperatorAggregate.update`（`binop.py:123`）：首值直存，其后逐值 `self.operator(self.value, v)`；遇 `Overwrite` 哨兵值整体替换且每步至多一个（多了抛 `InvalidUpdateError`，`binop.py:134`）。
- `Topic.update`（`topic.py:77`）：非累积模式先清空旧值再 extend；空写入返回是否发生过变化。

## 4. 配置项

| 配置项 | 默认 / 行为 | 位置 |
| --- | --- | --- |
| `state_schema` | 图状态 schema（必填）；input/output 缺省继承它 | `state.py:216` |
| `context_schema` | 运行期不可变上下文（替代已废弃 `config_schema`） | `state.py:219` |
| `set_node_defaults(retry_policy/cache_policy/error_handler/timeout)` | 全图节点默认策略；per-node 优先；不被子图继承 | `state.py:272` |
| `add_node(defer=...)` | 节点延迟到运行将结束时才执行（配合 `LastValueAfterFinish`/`NamedBarrierValueAfterFinish`） | `state.py:672` |
| `compile(checkpointer/store/cache/interrupt_before/interrupt_after/debug/transformers)` | 注入 checkpoint 持久化、长时记忆 store、节点结果缓存、前后中断、自定义流转换器 | `state.py:1177` |
| `DeltaChannel(snapshot_frequency=1000)` | 每 N 次更新写一次全量快照 blob；另有系统级 `DELTA_MAX_SUPERSTEPS_SINCE_SNAPSHOT`（默认 5000）限制回放深度 | `channels/delta.py:74` |
| `EphemeralValue(guard=True)` / `UntrackedValue(guard=True)` | guard 为真时每步多值即报错 | `ephemeral_value.py:62`、`untracked_value.py:59` |

## 5. 错误与重试语义

- **并发写冲突**：`LastValue.update` 每步收到多于一个写入值即抛 `InvalidUpdateError`（`last_value.py:59`，错误码 `INVALID_CONCURRENT_GRAPH_UPDATE`），提示用户改用 `Annotated[key, reducer]`。`EphemeralValue/UntrackedValue(guard=True)` 同理。
- **Overwrite 冲突**：同一超级步收到两个 `Overwrite` 值抛 `InvalidUpdateError`（`binop.py:134`、`delta.py:167`）。
- **非法分支返回**：分支返回 `None`/`START` 抛 `ValueError`；向 `END` 发 `Send` 包抛 `InvalidUpdateError`（`_branch.py:207-210`）。
- **未知节点/边**：`validate` 对未知 source/target、缺少入口 `START` 抛 `ValueError`（`state.py:1138-1168`）。
- **节点返回值非法**：`attach_node._get_updates` 收到非 dict/Command 抛 `InvalidUpdateError`（`state.py:1488`，`INVALID_GRAPH_NODE_RETURN_VALUE`）。
- **重试**：本叶子只声明 `retry_policy`（`StateNodeSpec.retry_policy`），实际退避/重试执行在 Pregel runner（见 `../pregel-engine/`）。`ParentCommand`（子图向父图发 `Command.PARENT`）在 `_control_branch` 抛出（`state.py:1761`）。

## 6. 并发细节

- 本叶子是**编译期**组件，本身不启 goroutine/线程；并发语义由下游 Pregel 引擎实现。
- 通道是"每超级步一次批量更新"的单写者模型：`update(values)` 在步末被一次性调用，values 顺序任意（`base.py:90`），因此 reducer 必须可结合。
- 同步屏障 `NamedBarrierValue` 用 `seen` 集合计数多节点到达，`consume()` 在齐套后清零以支持下一超级步（`named_barrier_value.py:77`）——这是 Pregel 超级步间"栅栏"的通道侧实现。
- `copy()`/`from_checkpoint()` 用于在 checkpoint 恢复时克隆通道，`BinaryOperatorAggregate.copy` 直接复用 operator 与 value（`binop.py:109`）。
- `Topic` 非累积模式步末清空，天然实现"每步一次性广播"。

## 7. 系统边界

**In-Scope（本仓库源码内）**
- `graph/state.py`（StateGraph/CompiledStateGraph 构建与编译）
- `graph/_branch.py`、`graph/_node.py`（分支与节点 spec）
- `graph/message.py`（消息 reducer/MessageGraph/MessagesState）、`graph/ui.py`（UI 消息）
- `channels/` 全部 10 个通道实现与 `BaseChannel` 契约

**Out-of-Scope（不在本仓库源码内）**
- Pregel 超级步调度、任务并发执行、重试执行 → `pregel/`（`../pregel-engine/`）
- checkpoint 持久化读写（MemorySaver/SQLite/Postgres）→ `libs/checkpoint*/`（外部库，不在本包）
- LangChain `Runnable` 基类、`RunnableConfig`、`langchain_core.messages` → 第三方依赖，不在本仓库源码内
- `BaseStore` 长时记忆、`BaseCache` 节点缓存后端实现 → 外部库

## 8. 与相邻子系统交互

- **上游**：用户/预构建 API（`prebuilt`、`func`）→ 本叶子：用户调 `StateGraph(...).add_node(...).compile()`。
- **下游 → `pregel-engine`**：`compile()` 产物 `CompiledStateGraph` 继承 `Pregel`，把"图元"翻译成 `PregelNode(triggers, channels, writers)` 与通道字典；invoke/stream 进入 Pregel 超级步循环。
- **下游 → `pregel-io-checkpoint`**：通道的 `checkpoint()`/`from_checkpoint()` 契约被 checkpoint 层消费；`DeltaChannel` 依赖 saver 的 `get_delta_channel_history` 回放祖先写入。
- **`managed-values`**：`_add_schema` 识别 `ManagedValueSpec` 并与通道一并注入编译图（`state.py:366`）。
- **`internal-foundation`**：`coerce_to_runnable`、`coerce_timeout_policy`、`_serde.build_serde_allowlist`、`MISSING` 等均来自 `_internal/`。

## 9. 语言专项适配口径

本项目为**纯 Python 框架**，按纯 Python 专项清单分析：

1. **能力缝归组**：本叶子的能力缝是"schema 注解 → 通道实例"的声明式分派——`Annotated[T, reducer]` / `Annotated[T, ChannelClass()]` / 托管值装饰器在编译期被 `_get_channels` 静态分派到不同通道实现，是框架级静态多态。
2. **注册表与工厂**：通道无全局注册表，而是"字段即实例"——每个 `Annotated` 元数据直接携带通道实例或通道类；`_get_channel` 是解析工厂。`__root__` 单字段 schema 走特殊根通道路径。
3. **配置驱动分派**：同一 `add_messages` 经 `format="langchain-openai"` 切换输出格式（`message.py:236`）；`defer`/`guard`/`accumulate` 等布尔配置切换通道行为（`LastValueAfterFinish` vs `LastValue`）。
4. **双轨实现**：`DeltaChannel`（beta，批量 reducer + 回放重建）与 `BinaryOperatorAggregate`（逐值折叠）是同一名 reducer 能力的两条轨迹，由用户在 `Annotated` 中显式选择。
5. **可选依赖**：`langchain_core`、`pydantic` 为重依赖；`langchain-openai` 格式在缺包时降级告警（`message.py:376`）。

## 10. 图表清单与质量

| 图 | 文件 | 类型 | archify 质量档 |
| --- | --- | --- | --- |
| 图状态构建与通道架构 | `graph-state-api-architecture.html` | architecture | standard |
| compile() 编译调用链 | `graph-state-api-sequence.html` | sequence | standard |

- 降档说明：architecture 图受"边标签与节点重叠/垂直连线 side 方向"约束降 standard；本次新增的 sequence 图（add_node→add_edge→_validate→_get_channels→构造 PregelNode）因"自消息 0px 跨度/segment 超出画布"修复后仍未全过 showcase 标签间距，按规范降 standard 渲染（均成功，约 805KB 自包含）。
- JSON IR 源：`json/graph-state-api-architecture.json`、`json/graph-state-api-sequence.json`。
- 本次迭代新增 sequence 图，把"声明→编译→产物"的调用链从文字化升级为图；不另出 dataflow：编译是单线程直调链而非数据管道。
