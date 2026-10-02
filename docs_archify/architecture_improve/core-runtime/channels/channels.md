# 状态通道原语族（channels）

> 本文是 `core-runtime` 域下的叶子子系统文档。域级总览见 `../core-runtime.md`（由组织者统一产出）。
> 本文只展开 LangGraph「状态通道（channel）」原语族的职责边界——它是 Pregel 引擎读写图状态的
> 最小存储单元；Pregel 超级步调度逻辑见 `../pregel-engine/pregel-engine.md`，检查点读写链路见
> `../pregel-io-checkpoint/pregel-io-checkpoint.md`，图构建时如何把 StateSchema 字段映射成通道见
> `../graph-state-api/graph-state-api.md`。
>
> 源码基准：`langgraph` 1.2.11，commit `e539ac12`（2026-09-09），目录 `libs/langgraph/langgraph/channels/`。

![channels 通道原语族架构图](channels-architecture.html)

## 1. 功能清单

通道（channel）是 LangGraph 对「图中一个状态键」的统一抽象：每个状态键在运行时被实例化为一个
`BaseChannel` 子类对象，负责本超步内对该键的写入聚合、读取、消费标记与序列化。Pregel 引擎
在每个超级步末尾对所有通道统一调用 `update(values)`，在需要节点输入时调用 `get()`。

| 能力 | 说明 | 源码路径 |
|---|---|---|
| 通道抽象基类 | 定义 `get/update/consume/finish/checkpoint/from_checkpoint` 六方法契约；空值约定 `MISSING`，空通道读抛 `EmptyChannelError` | `channels/base.py:19`（`BaseChannel`） |
| 每步单值覆盖 | `LastValue`：每超步最多收一个值，多写即报错；最常用的"普通状态键"通道 | `channels/last_value.py:20` |
| 收尾后可见 | `LastValueAfterFinish`：值只在 `finish()` 后才可读，读后清空 | `channels/last_value.py:81` |
| 任意值覆盖 | `AnyValue`：多写假定相等取最后值；空值更新会清空 | `channels/any_value.py:15` |
| 步内临时值 | `EphemeralValue`：仅保留上一超步写入，空值更新即清；`guard=True` 时每步限一值 | `channels/ephemeral_value.py:15` |
| 不落检查点 | `UntrackedValue`：运行时持有值但 `checkpoint()` 恒返回 `MISSING`，重启后丢失 | `channels/untracked_value.py:15` |
| 二元归约聚合 | `BinaryOperatorAggregate`：对当前值与每个新值调用归约算子（如 `operator.add`）；识别 `Overwrite` 哨兵重置 | `channels/binop.py:65` |
| 增量快照回放 | `DeltaChannel`（beta）：检查点只存哨兵，靠回放祖先写入经 reducer 重建；按 `snapshot_frequency` 与系统超步上限决定落快照 | `channels/delta.py:25` |
| 命名栅栏 | `NamedBarrierValue`：等待 `names` 集合全部到齐才可读，`consume()` 后清空；`...AfterFinish` 变体等 `finish()` 后放行 | `channels/named_barrier_value.py:13` / `:84` |
| 发布订阅主题 | `Topic`：把多个写入追加为列表；`accumulate=False` 每超步清空；支持 list 拍平 | `channels/topic.py:23` |

对外暴露点：`channels/__init__.py` 用 `__all__` 导出全部 10 个公共类，由 `graph/state.py` 在编译
`StateGraph` 时依据类型注解实例化。

## 2. 核心类型与接口清单

| 类型 / 函数 | 位置 | 职责 |
|---|---|---|
| `BaseChannel[Value, Update, Checkpoint]` | `channels/base.py:19` | 抽象基类；`ValueType/UpdateType` 抽象属性描述读写类型；`update()` 返回"是否本步被更新"布尔 |
| `BaseChannel.update(values)` | `channels/base.py:90` | 契约核心：Pregel 在每超步末尾对全部通道调用；空序列也会被调用（用于"空值即清空"语义） |
| `BaseChannel.consume()/finish()` | `channels/base.py:101` / `:112` | 默认 no-op 钩子；栅栏类与 AfterFinish 类覆写以实现"订阅任务跑完才放行" |
| `BaseChannel.checkpoint()/from_checkpoint()` | `channels/base.py:49` / `:61` | 序列化往返；`checkpoint()` 默认委托 `get()`，空通道返回 `MISSING` |
| `_get_overwrite(value)` | `channels/binop.py:29` | 识别三种 Overwrite 形态（dataclass、`{"__overwrite__":v}`、JSON 擦除形态），跨 JSON 边界保持重置语义 |
| `_strip_extras(t)` | `channels/binop.py:19` | 剥掉 `Annotated/Required/NotRequired`，并把抽象集合类型替换为 `list/set/dict` 以便实例化初值 |
| `DeltaChannel.replay_writes(writes)` | `channels/delta.py:139` | 从检查点回放祖先 `PendingWrite`；遇到 Overwrite 以其为新基座，之后批量喂给 reducer |

扩展点：用户自定义状态聚合行为有两条途径——在类型注解里写 `Annotated[T, reducer_fn]`（由
`graph/state.py:_is_field_binop` 自动转成 `BinaryOperatorAggregate`），或直接传入一个 `BaseChannel`
子类（由 `_is_field_channel` 识别）。

## 3. 关键调用链

**调用链一：一个超步末尾，Pregel 把本步所有节点写出按通道归集后下发。**

1. Pregel 引擎把同一状态键的多个写入聚成 `values: Sequence`，调用通道 `update(values)`。
2. 以 `LastValue` 为例：`channels/last_value.py:56` 的 `update` 在 `len(values) != 1` 时抛
   `InvalidUpdateError`（错误码 `INVALID_CONCURRENT_GRAPH_UPDATE`），否则取 `values[-1]` 覆盖。
3. 返回 `True/False` 告知引擎"通道是否发生了变化"——这是 Pregel 判断是否需要继续下一超步、
   是否要触发订阅节点的依据。

**调用链二：节点读取其依赖通道。**

1. 节点运行前，Pregel 调用通道 `get()` 取当前值喂给节点参数。
2. `channels/base.py:70` 的 `get()` 在通道从未写入时抛 `EmptyChannelError`；`is_available()`
   （`base.py:76`）是不抛异常的等价探测，栅栏类覆写为 `seen == names`（`named_barrier_value.py:75`）。
3. 节点跑完后，Pregel 对订阅了该通道的通道调用 `consume()`：`NamedBarrierValue.consume()`
   （`named_barrier_value.py:77`）在集齐后把 `seen` 清空，从而本超步只放行一次。

**调用链三：检查点序列化与恢复。**

1. 每超步落盘时，Checkpointer 调用各通道 `checkpoint()`：`UntrackedValue.checkpoint()`
   （`untracked_value.py:48`）恒返回 `MISSING`，故不进快照；`DeltaChannel.checkpoint()`
   （`delta.py:193`）也恒返回 `MISSING`，快照决策外置到 `create_checkpoint`。
2. 恢复时用 `from_checkpoint(blob)` 重建：`Topic.from_checkpoint`（`topic.py:66`）还兼容
   旧版 tuple 形态。
3. `DeltaChannel` 的 `from_checkpoint`（`delta.py:118`）遇到 `_DeltaSnapshot` 直接取值，
   遇到 `MISSING` 则置空并由调用方随后 `replay_writes()`（`delta.py:139`）回放祖先写入。

![单个通道在超步内的生命周期](channels-lifecycle.html)

## 4. 配置项

| 配置 / 参数 | 默认 / 行为 | 位置 |
|---|---|---|
| `guard`（EphemeralValue/UntrackedValue） | `True`：每超步只允许一个值，多写报错；`False`：取多值之一 | `ephemeral_value.py:17`、`untracked_value.py:17` |
| `accumulate`（Topic） | `False`：每超步开始清空再追加；`True`：跨超步累积 | `topic.py:30` |
| `operator`（BinaryOperatorAggregate） | 必填二元归约函数，如 `operator.add`；初值由 `typ()` 构造 | `binop.py:73` |
| `snapshot_frequency`（DeltaChannel） | `1000`：本通道每第 N 次更新写一个快照 blob；必须为正整数 | `delta.py:80` |
| 系统级 `DELTA_MAX_SUPERSTEPS_SINCE_SNAPSHOT` | 默认 `5000`：即使通道停止写入，最多 5000 超步强制落快照以限制回放深度 | `delta.py:67` 文档说明（常量在 pregel 侧） |

## 5. 错误与重试语义

- **并发多写冲突**：`LastValue.update`（`last_value.py:56`）、`EphemeralValue(guard=True)`
  （`ephemeral_value.py:63`）在一个超步收到多于一个值时抛 `InvalidUpdateError`，错误码
  `INVALID_CONCURRENT_GRAPH_UPDATE`——这是"同一步两个节点写同一键且未声明 reducer"的用户级错误，
  引擎不重试，直接冒泡给调用方。
- **栅栏收到非法名**：`NamedBarrierValue.update`（`named_barrier_value.py:56`）收到不在 `names`
  集合的值时抛 `InvalidUpdateError`。
- **Overwrite 重复**：`binop.py:132` 与 `delta.py:167` 限制每超步最多一个 Overwrite，重复即报错。
- **空通道读取**：`get()` 抛 `EmptyChannelError`（定义在 `langgraph/errors.py`），这是正常控制流
  而非异常——Pregel 用它判断"该通道尚无值、订阅节点本步不触发"。
- 通道层不实现任何重试/退避：它是纯内存归约原语，失败即抛，重试与容错完全由 pregel-engine 与
  checkpointer 负责。

## 6. 并发细节

- 通道本身是**同步、非线程安全的内存对象**：所有 `update/get/consume` 都在 Pregel 的单事件循环
  （asyncio 协作式调度）内按超步串行调用，不存在多线程并发写同一通道的情况。
- "并发"在 LangGraph 里是**逻辑并发**（同一超步内多个节点同时跑），其产出由引擎归集后在超步边界
  一次性串行 `update` 进通道——这正是 `LastValue` 能在 `update` 里直接检测"本步写了几个值"的前提。
- `DeltaChannel.replay_writes`（`delta.py:139`）用 `_copy.copy` 对 Overwrite 基座做防御性拷贝，
  避免回放与后续运行共享可变引用。
- 无锁、无队列、无取消传播需求；`asyncio.CancelledError` 由 pregel-engine 层捕获，通道不感知。

## 7. 系统边界

**In-Scope（本仓库源码内）**
- `channels/` 全部 10 个公共通道类 + `BaseChannel` 抽象基类
- Overwrite 哨兵识别、类型注解剥壳（`_strip_extras`）、Topic list 拍平（`_flatten`）等配套工具

**Out-of-Scope（不在本仓库源码内）**
- 通道被谁实例化、何时 `update/get`：Pregel 超级步引擎，见 `pregel-engine` 叶子（`pregel/main.py`）
- 通道 blob 如何持久化到 DB/文件：checkpoint 存储后端（`libs/checkpoint-*`，不在本叶子目录）
- StateSchema 类型注解如何映射到具体通道类：图编译期逻辑，见 `graph-state-api` 叶子
- reducer 函数本身（如用户传入的 `operator.add` 或自定义归约）：用户代码，不在本仓库

## 8. 与相邻子系统交互

- **graph-state-api → channels**：`StateGraph.compile()` 遍历 StateSchema 字段，经 `_is_field_channel`
  / `_is_field_binop` 把每个字段实例化为具体通道，建立 `channels: dict[str, BaseChannel]`。
- **pregel-engine → channels**：每个超步，引擎「节点执行完 → 归集写入 → 调 `channel.update()` →
  依返回值与 `is_available()` 决定下一步跑哪些节点 → 节点读 `channel.get()`」。
- **pregel-io-checkpoint → channels**：落盘点调 `checkpoint()` 取 blob 交给 checkpointer；恢复点调
  `from_checkpoint()` 重建，`DeltaChannel` 额外走 `replay_writes()`。
- **channels → 错误体系**：向上抛 `EmptyChannelError`（正常空值信号）与 `InvalidUpdateError`
  （用户配置错误），均定义于 `langgraph/errors.py`。

## 9. 语言专项适配口径（纯 Python）

按 `language-python.md` 清单落地：

1. **能力缝归组**：本叶子是典型的「抽象基类定义契约 + 多实现方」能力缝——`BaseChannel`（消费方，
   由 Pregel 引擎面向基类编程）与 10 个具体通道（实现方）。扩展点就是继承 `BaseChannel`。
2. **注册表与工厂**：通道实例化不是字符串注册表，而是**类型注解驱动的静态分派**——`graph/state.py`
   的 `_is_field_channel`（识别用户直接放进来的通道实例）与 `_is_field_binop`（识别
   `Annotated[T, reducer]`）在编译期把类型注解路由到 `BinaryOperatorAggregate` 或自定义通道。
3. **双轨实现**：存在"同一语义的轻/重两轨"——`LastValue`（普通）vs `LastValueAfterFinish`、
   `NamedBarrierValue` vs `NamedBarrierValueAfterFinish`；以及 `BinaryOperatorAggregate`（全量
   状态进快照）vs `DeltaChannel`（增量回放、省存储、beta）。
4. **隐藏状态机在循环里**：单个通道在 Pregel 超步循环中反复经历「空 → 写入 → 可读 → 消费 → 重置」，
   这是隐藏在引擎循环里的状态机，已用 lifecycle 图显式画出（见上图）。
5. **可选依赖**：`DeltaChannel` 依赖 `langgraph.checkpoint.base.PendingWrite` 与
   `langgraph.checkpoint.serde.types._DeltaSnapshot`，属同仓库内部依赖，非外部可选包。
6. **外部边界**：通道不承载任何"计算"，它是纯数据归约原语；真实业务计算在用户节点函数里，外部 DB
   持久化在 `checkpoint-*` 库——均标注"不在本仓库源码内"。

## 10. 图表清单与质量

| 图 | 文件 | 类型 | archify 质量档 |
|---|---|---|---|
| 通道原语族架构图 | `channels-architecture.html` | architecture | showcase（一次通过） |
| 单通道超步生命周期 | `channels-lifecycle.html` | lifecycle | showcase（首跑因 viewBox 高度不足被 layout/constraint 拒绝，加高至 620 后通过） |

JSON IR 源文件位于 `json/`（`channels-architecture.json`、`channels-lifecycle.json`）。

**省略说明**：本叶子未生成 sequence 图——通道是被引擎单向调用的被动原语，不存在"多个参与方按时间
互发消息"的交互语义（sequence 图的底层语义是多方消息时序），其调用先后已由 lifecycle 图与第 3 节
文字调用链表达；未生成 dataflow 图——通道内的值归约是原地覆写/追加，无"源→变换→目的地"的管道
血缘结构。按资源节省原则省略。
