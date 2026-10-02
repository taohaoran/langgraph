# 图构建与状态 API（graph-state-api）

> 本文是 `core-runtime` 域下的叶子子系统文档。域级总览见 `../core-runtime.md`。
> 本文只展开「用户如何用声明式 API 把一张图描述出来并编译成可执行对象」这一层：
> `StateGraph`/`MessageGraph` 构建器、节点与分支规格、`Send`/`Command` 控制原语。
> 编译产物 `CompiledStateGraph` 的实际执行（超级步调度）见 `../pregel-engine/pregel-engine.md`；
> 状态键如何落到通道原语见 `../channels/channels.md`；检查点读写见 `../pregel-io-checkpoint/pregel-io-checkpoint.md`。
>
> 源码基准：`langgraph` 1.2.11，commit `e539ac12`，目录 `libs/langgraph/langgraph/graph/` 与顶层 `types.py`。

![图构建与编译层架构图](graph-state-api-architecture.html)

## 1. 功能清单

本叶子负责**图的声明式构建与编译**，不负责执行。用户用 `StateGraph(state_schema)` 拿到一个构建器，
`add_node`/`add_edge`/`add_conditional_edges` 描述拓扑，最后 `compile()` 得到继承自 `Pregel` 的可执行图。

| 能力 | 说明 | 源码路径 |
|---|---|---|
| 状态图构建器 | `StateGraph`：以 `state_schema`（TypedDict/dataclass/pydantic）为骨架收集节点、边、分支；标注了 reducer 的字段自动成聚合通道 | `graph/state.py:131` |
| 节点注册 | `add_node(key, fn, ...)`：把任意可调用/`Runnable` 包成 `StateNodeSpec`，可配 `retry_policy/cache_policy/timeout` | `graph/state.py:376` |
| 静态边 | `add_edge(start, end)`：单起点串行；多起点元组表示"等全部完成再触发"（waiting edge） | `graph/state.py:928` |
| 条件边 | `add_conditional_edges(start, path_fn, mapping)`：按 `path_fn(state)` 返回值动态选下一节点 | `graph/state.py:982` |
| 入口/出口 | `set_entry_point` / `set_finish_point`：分别等价到 `START`/`END` 的便捷方法 | `graph/state.py:1079` / `:1116` |
| 编译 | `compile(checkpointer, store, interrupt_before/after, ...)`：校验拓扑、派生通道、组装为 `CompiledStateGraph` | `graph/state.py:1177` |
| 可执行图 | `CompiledStateGraph(Pregel)`：继承 Pregel，对外暴露 `invoke/stream/astream` | `graph/state.py:1404` |
| 消息图 | `MessageGraph(StateGraph)`：预置 `MessagesState`（消息列表自动追加）的便捷子类 | `graph/message.py:316` |
| 动态派发原语 | `Send(node, arg)`：在条件边里运行时扇出到同一节点的多个并行实例（map-reduce） | `types.py:704` |
| 命令原语 | `Command(update=, goto=, resume=)`：节点返回值同时表达"写状态 + 跳转 + 中断恢复" | `types.py:799` |
| 节点规格 | `StateNodeSpec` dataclass：节点 runnable、metadata、retry/cache/timeout 策略、错误处理器 | `graph/_node.py:99` |
| 分支规格 | `BranchSpec(path, ends, input_schema)`：条件边的可调用路径与映射 | `graph/_branch.py:74` |

对外暴露点：`graph/__init__.py` 导出 `StateGraph/MessageGraph/START/END` 等；`types.py` 导出 `Send/Command/interrupt/RetryPolicy/CachePolicy`。

## 2. 核心类型与接口清单

| 类型 / 函数 | 位置 | 职责 |
|---|---|---|
| `StateGraph` | `graph/state.py:131` | 构建器类；持有 `nodes/edges/waiting_edges/branches/channels/schemas/managed` 等中间态 |
| `StateNodeSpec` | `graph/_node.py:99` | 节点执行规格；`ends` 字段定义节点完成后的默认跳转，`defer`/`timeout` 控制执行策略 |
| `StateNode`（TypeAlias） | `graph/_node.py:90` | 节点可调用签名联合：纯 state、带 config、带 writer、带 store、带 runtime 等 9 种重载 + `Runnable` |
| `BranchSpec` | `graph/_branch.py:74` | 条件边规格；`from_path` 工厂从 path 可调用推断其入参类型注解 |
| `CompiledStateGraph` | `graph/state.py:1404` | `Pregel` 子类；`attach_node` 把节点返回的 dict/`Command` 翻译成 `(channel, value)` 写入 |
| `Send` | `types.py:704` | 运行时派发信封：`node`（目标节点名）+ `arg`（该实例的独立入参状态） |
| `Command` | `types.py:799` | 控制原语；`graph=Command.PARENT` 表示向上层子图发指令 |
| `_get_channels(schema, ...)` | `graph/state.py:1815` | 编译期：为 state_schema 每个字段选定通道实现 |
| `_is_field_channel(typ)` | `graph/state.py:1876` | 字段类型若是 `BaseChannel` 实例则直接采用 |
| `_is_field_binop(typ)` | `graph/state.py:1904` | 字段带 `Annotated[T, reducer]` 则派生 `BinaryOperatorAggregate` |

扩展点：节点函数签名由 `StateNode` 联合类型开放（用户可选择是否接收 config/writer/store/runtime）；
条件边 path 可同步可异步、可为 `Runnable`；子图通过 `Command(PARENT)` 与父图通信。

## 3. 关键调用链

**调用链一：从构建器到可执行图（编译链）。**

1. 用户 `builder.add_node("a", fn)` → `graph/state.py:376` 把 fn 包成 `StateNodeSpec` 存入 `self.nodes`。
2. 用户 `builder.add_edge("a", "b")` → `graph/state.py:928` 把 `("a","b")` 加入 `self.edges`；多起点则进 `waiting_edges`。
3. 用户 `builder.compile(checkpointer=...)` → `graph/state.py:1177`：先 `ensure_valid_checkpointer`，
   再为严格 msgpack 模式构建 serde 白名单，随后校验拓扑并派生通道。
4. 通道派生经 `_get_channels`（`state.py:1815`）→ 对每个字段调 `_is_field_channel`（`:1876`）或
   `_is_field_binop`（`:1904`），得到 `channels: dict[str, BaseChannel]`。
5. 最终构造 `CompiledStateGraph`（`state.py:1404`），其 `__init__` 调 `super().__init__(**kwargs)`
   即 `Pregel` 构造——运行权正式移交 Pregel 引擎。

![StateGraph 编译链时序](graph-state-api-sequence.html)

**调用链二：节点返回值如何变成状态写入与跳转。**

1. 节点跑完返回一个 dict 或 `Command`。
2. `CompiledStateGraph.attach_node`（`state.py:1441` 附近）的 `_get_updates`：dict 取其中落在
   `output_keys`（= 全部通道键 + managed 键）的项；`Command` 走 `_update_as_tuples()`（`types.py:831`）
   展平；list/tuple 中混有 `Command` 的逐项展开；`Command.PARENT` 被跳过（留给上层处理）。
3. 解析出的 `(channel, value)` 交给 Pregel 的 `ChannelWrite` 写入通道，`goto` 决定下一超步触发哪些节点。

**调用链三：条件边的运行时动态派发。**

1. `add_conditional_edges(start, path_fn, mapping)` → `graph/_branch.py:74` 的 `BranchSpec` 包装 path_fn 为 `Runnable`。
2. 超步内 `start` 节点完成后，Pregel 调用 path_fn(state)；返回值经 mapping 翻译成目标节点名，
   若返回 `Send` 则按其 `arg` 扇出一个独立节点实例（map-reduce）。

## 4. 配置项

| 配置 / 参数 | 默认 / 行为 | 位置 |
|---|---|---|
| `compile(checkpointer=)` | `None`：子图可继承父图 checkpointer；`False`：强制不用；传入 saver 则启用"短期记忆" | `state.py:1179` |
| `compile(store=)` | `None`：无长期存储；传入 `BaseStore` 供节点跨线程存取 | `state.py:1181` |
| `interrupt_before/after` | `[]`：在指定节点名前/后中断，配合 `Command(resume=)` 恢复 | `state.py:1182` |
| `debug` | `False`：开启后打印每超步执行细节 | `state.py:1184` |
| `add_node(retry_policy=)` | `None`：节点失败不重试；`RetryPolicy`/序列见 `types.py:418` | `state.py:376` |
| `add_node(cache_policy=)` | `None`：`CachePolicy`（`types.py:521`）对节点结果做缓存 | `_node.py:104` |
| `context_schema` | 运行时不可变上下文（替代已废弃的 `config_schema`） | `state.py:131` |

## 5. 错误与重试语义

- **拓扑校验错误**：`add_edge` 中 `END` 不能作起点、`START` 不能作终点（`state.py:945`/`:946`）；
  非 StateGraph 图一个起点已有出边时再连会报 "Already found path"（`state.py:957`）——提示用户改用带
  reducer 的 Annotated 键实现多写。
- **编译后修改**：已 `compile` 后再 `add_edge` 仅 `logger.warning`，不影响已编译图（`state.py:936`）。
- **节点级重试**：失败重试不由本层实现，而是把 `retry_policy` 透传给 `StateNodeSpec`，
  由 pregel-engine 在执行节点时按策略退避重试。
- **中断恢复**：`interrupt()`（`types.py:854` 附近）在节点内抛 `GraphInterrupt`，本层只把值透传给客户端；
  恢复靠 `Command(resume=)` 在下一次调用时回填。
- 本层不直接吞异常：所有用户 path_fn/节点异常原样冒泡到 Pregel 执行期处理。

## 6. 并发细节

- 本层是**纯同步构建期**代码：`add_node/add_edge/compile` 都在调用线程内串行执行，无协程、无锁。
- "并发"语义在本层通过**声明**表达：多起点 `add_edge([a,b], c)`（waiting edge）表示"等 a、b 都完成"；
  `Send` 列表表示"下一超步并行跑同节点的多个实例"；真正的并行调度由 pregel-engine 实现。
- `BranchSpec.from_path` 用 `get_type_hints`/`signature` 反射推断 path 入参类型（`_branch.py:34`），
  纯只读操作，无副作用。
- 节点签名联合（`_node.py:90`）在运行期由 pregel-engine 按参数 inspect 决定传不传 writer/store/runtime。

## 7. 系统边界

**In-Scope（本仓库源码内）**
- `graph/state.py`：`StateGraph` 构建器、拓扑校验、通道派生、`CompiledStateGraph`
- `graph/_node.py`：节点签名协议与 `StateNodeSpec`
- `graph/_branch.py`：条件边 `BranchSpec` 与 path 推断
- `graph/message.py`：`MessageGraph`/`MessagesState`
- `types.py`：`Send`/`Command`/`interrupt`/`RetryPolicy`/`CachePolicy`

**Out-of-Scope（不在本仓库源码内）**
- 超级步调度、并发执行、节点真正调用：Pregel 引擎（`pregel/main.py`），见 pregel-engine 叶子
- 状态键的存储语义（覆盖/追加/栅栏）：channels 原语族
- 检查点 blob 的序列化与持久化：checkpoint 后端（`libs/checkpoint-*`）
- 长期存储 `BaseStore` 的具体实现：store 后端库
- 节点函数本身、用户状态 schema：用户代码

## 8. 与相邻子系统交互

- **用户代码 → 本叶子**：声明式 API（add_node/add_edge/compile），对象协议。
- **本叶子 → channels**：编译期 `_get_channels` 把 state_schema 字段实例化为 `BaseChannel` 子类字典，
  交给 Pregel。
- **本叶子 → pregel-engine**：`CompiledStateGraph` 直接继承 `Pregel`，compile 的产物即 Pregel 构造参数；
  运行期 `invoke/stream` 全部在 Pregel 内完成。
- **本叶子 → pregel-io-checkpoint**：`compile(checkpointer=)` 把 saver 透传给 Pregel；
  节点返回的 `Command(resume=)` 经 `attach_node` 翻译后驱动中断/恢复。
- **types.py → 全运行时**：`Send/Command` 是跨层控制原语，条件边返回、节点返回、Pregel 派发都识别它们。

## 9. 语言专项适配口径（纯 Python）

按 `language-python.md` 清单落地：

1. **能力缝归组**：本叶子是典型"构建器 + 规格对象"能力缝——`StateGraph`（收集者）、`StateNodeSpec`/
   `BranchSpec`（值对象规格）、`CompiledStateGraph`（产物）。节点函数本身是开放扩展点。
2. **注册表与工厂**：无字符串注册表；通道派生是**类型注解驱动的静态分派**（`_is_field_channel` /
   `_is_field_binop`，`state.py:1876`/`:1904`），在 compile 期一次性完成。
3. **配置驱动分派**：节点执行签名（是否收 config/writer/store/runtime）由可调用对象的参数签名静态决定，
   而非显式开关——这是纯 Python 框架的运行时多态。
4. **双轨实现**：`StateGraph`（通用 state schema）与 `MessageGraph`（预置消息 reducer）是同一构建能力的
   便捷双轨；静态边 vs waiting edge vs 条件边 vs `Send`/`Command` 是控制流的多轨表达。
5. **隐藏状态机在循环里**：构建器自身无状态机；但编译产物在 Pregel 超步循环里的状态流转见
   pregel-engine 的 lifecycle 图。本叶子只补"编译链"这一次性流程的 sequence 图。
6. **外部边界**：本层不计算、不持久化，只做拓扑描述与组装；真正执行在 Pregel、真正存储在 channels 与
   checkpointer，均标注"不在本叶子源码内"。

## 10. 图表清单与质量

| 图 | 文件 | 类型 | archify 质量档 |
|---|---|---|---|
| 图构建与编译层架构图 | `graph-state-api-architecture.html` | architecture | showcase（一次通过） |
| StateGraph 编译链时序 | `graph-state-api-sequence.html` | sequence | showcase（首跑因首消息 y=150 <160、参与者标签"CompiledStateGraph"过宽被 layout/constraint 拒；下移 y、标签改为"可执行图/通道派生"、加高 viewBox 至 820 后通过） |

JSON IR 源文件位于 `json/`（`graph-state-api-architecture.json`、`graph-state-api-sequence.json`）。

**省略说明**：本叶子未生成 lifecycle 图——本层是同步构建器，自身无"单实体多状态迁移"（编译产物的超步
状态机归 pregel-engine）；未生成 dataflow 图——构建期无"源→变换→目的地"的数据管道血缘。按资源节省原则省略。
