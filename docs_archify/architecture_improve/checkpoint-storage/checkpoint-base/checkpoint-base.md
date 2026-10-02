# checkpoint-base（checkpoint-base）

> 本文是 `checkpoint-storage` 域下的叶子子系统文档。本叶子定义 LangGraph 持久化检查点的
> **抽象契约层**：`Checkpoint`/`CheckpointMetadata`/`CheckpointTuple` 等数据结构、
> `BaseCheckpointSaver` 抽象基类（同步/异步双轨接口）、版本与元数据工具函数，以及参考实现
> `InMemorySaver`。具体后端（Postgres / SQLite）实现见各自叶子，序列化机制见
> `checkpoint-serde` 叶子。
>
> 源码基准：`langgraph-checkpoint`（libs/checkpoint），commit `e539ac12`，版本 4.2.0。

## 1. 功能清单

| 能力 | 说明 | 源码路径 |
|---|---|---|
| 检查点数据结构 | `Checkpoint`（通道快照）、`CheckpointMetadata`（来源/步数/父链）、`CheckpointTuple`（检查点+元数据+父配置+pending writes）、`DeltaChannelHistory`（增量通道祖先链结果） | `base/__init__.py:93`、`:39`、`:140`、`:150` |
| 检查点保存抽象基类 | `BaseCheckpointSaver[V]`：定义 get/list/put/put_writes/delete/copy/prune 全套同步接口 + 同名 `a*` 异步接口；默认 `serde = JsonPlusSerializer()` | `base/__init__.py:177`、`:210` |
| 读路径（同步/异步） | `get`/`aget`（薄封装转 tuple）、`get_tuple`/`aget_tuple`（子类实现）、`list`/`alist`（带 filter/before/limit 迭代） | `base/__init__.py:228`、`:240`、`:254`、`:418`、`:430`、`:444` |
| 写路径（同步/异步） | `put`/`aput`（写检查点+新通道版本）、`put_writes`/`aput_writes`（写任务中间态 pending writes） | `base/__init__.py:278`、`:301`、`:469`、`:492` |
| 线程管理 | `delete_thread`/`adelete_thread`、`delete_for_runs`、`copy_thread`、`prune`（keep_latest/delete 策略），均带 `DeltaChannel` 安全警告 | `base/__init__.py:321`、`:332`、`:351`、`:375` |
| 增量通道历史回溯 | `get_delta_channel_history`/`aget_delta_channel_history`：沿 `parent_config` 父链逐祖先累积某通道 writes、直到找到 `channel_values` 种子（`_DeltaSnapshot`） | `base/__init__.py:583`、`:652` |
| 版本号生成 | `get_next_version`：默认整数 +1（基类）；`InMemorySaver` 覆写为 `32位整数.16位随机` 字符串版本 | `base/__init__.py:693`、`memory/__init__.py:613` |
| 元数据与工具函数 | `get_checkpoint_metadata`（剥离空字节、合并 config 元数据）、`EXCLUDED_METADATA_KEYS`、`WRITES_IDX_MAP`（特殊写入负索引）、`empty_checkpoint`/`create_checkpoint` | `base/__init__.py:758`、`:798`、`:796`、`:815`、`:830` |
| 单调递增 ID | `uuid6()`：UUIDv6 时间重排序，保证检查点 ID 单调递增、数据库索引局部性好 | `base/id.py:79` |
| 内存参考实现 | `InMemorySaver`：用 `defaultdict` 在内存中存 storage/writes/blobs 三张表；同步/异步双轨（异步直接委托同步）；实现上下文管理器协议 | `memory/__init__.py:33`、`:421`、`:523` |
| 持久化字典 | `PersistentDict`：pickle 后端、close/sync 时原子落盘（供调试/测试） | `memory/__init__.py:628` |

## 2. 核心类型与接口清单

| 类型 / 函数 | 位置 | 职责 |
|---|---|---|
| `Checkpoint` (TypedDict) | `base/__init__.py:93` | 某次超步的状态快照：`v`（格式版本，当前 1）、`id`（uuid6，单调递增）、`ts`、`channel_values`、`channel_versions`、`versions_seen`（每节点见过的通道版本）、`updated_channels` |
| `CheckpointMetadata` (TypedDict, total=False) | `base/__init__.py:39` | `source`(input/loop/update/fork)、`step`、`parents`、`run_id`、`counters_since_delta_snapshot`（beta 增量通道计数） |
| `CheckpointTuple` (NamedTuple) | `base/__init__.py:140` | 一次读取的完整结果：`config`+`checkpoint`+`metadata`+`parent_config`+`pending_writes` |
| `BaseCheckpointSaver[V]` | `base/__init__.py:177` | **契约核心**。`V ∈ {int,float,str}` 为通道版本类型。除 `get`/`aget`/`config_specs`/`get_next_version`/`with_allowlist` 有默认实现外，其余方法抛 `NotImplementedError`，由后端实现 |
| `SerializerProtocol`（聚合自 serde） | `serde/base.py`（见 serde 叶子） | 经 `__init__(serde=)` 注入；默认 `JsonPlusSerializer`，再经 `maybe_add_typed_methods` 包一层 typed dumps/loads |
| `PendingWrite` | `base/__init__.py:32` | `tuple[str, str, Any]` = (task_id, channel, value)，表示任务已产出但未落通道的中间写入 |
| `InMemorySaver` | `memory/__init__.py:33` | `BaseCheckpointSaver[str]` 参考实现；三张内存表 `storage`/`writes`/`blobs` |
| `uuid6()` | `base/id.py:79` | 生成时间有序 UUIDv6；`empty_checkpoint` 用 `clock_seq=-2`，`create_checkpoint` 用 `clock_seq=step` |
| `WRITES_IDX_MAP` | `base/__init__.py:796` | 特殊通道写入映射到负索引：`ERROR:-1, SCHEDULED:-2, INTERRUPT:-3, RESUME:-4`，避免与普通写入下标冲突 |

## 3. 关键调用链

**写链 `put`（`memory/__init__.py:421`）——Pregel 每超步结束调用：**
1. Pregel 引擎在编译图时拿到的 checkpointer 上调用 `saver.put(config, checkpoint, metadata, new_versions)`。
2. `put` 先 `checkpoint.copy()`，`pop("channel_values")` 取出通道值（`memory/__init__.py:442-445`）。
3. 对 `new_versions` 中每个通道，把值经 `self.serde.dumps_typed(values[k])` 序列化成 `(type, bytes)` 存入 `self.blobs[(thread, ns, channel, version)]`；值缺失则存占位 `("empty", b"")`（`memory/__init__.py:446-449`）。
4. 把去掉 channel_values 的 checkpoint 本体和元数据（经 `get_checkpoint_metadata` 净化）分别 `dumps_typed`，连同父 checkpoint_id 存入 `self.storage[thread][ns][checkpoint_id]`（`memory/__init__.py:450-458`）。
5. 返回带新 `checkpoint_id` 的 `configurable` 配置。

**读链 `get_tuple`（`memory/__init__.py:230`）——恢复/时间旅行调用：**
1. 按 `config.configurable.thread_id` / `checkpoint_ns` 取 ns 存储；若 config 指定 `checkpoint_id` 则精确取，否则 `max(checkpoints.keys())` 取最新（`memory/__init__.py:246-278`）。
2. 从 `self.writes[(thread,ns,cp_id)]` 取出该检查点的全部 pending writes，逐条 `serde.loads_typed(v)` 反序列化（`memory/__init__.py:249`、`:260-262`）。
3. checkpoint 本体 `serde.loads_typed` 后，用 `_load_blobs`（`memory/__init__.py:125`）按 `channel_versions` 逐通道从 `blobs` 表还原 `channel_values`。
4. 组装 `CheckpointTuple`（含 `parent_config` 指向上一 checkpoint）返回。

**增量通道回溯 `get_delta_channel_history`（基类默认实现 `base/__init__.py:583`）：**
1. 先 `get_tuple(config)` 定位目标检查点，从其 `parent_config` 开始沿父链游标走（`base/__init__.py:626-630`）。
2. 每到一个祖先 tuple：把其 `pending_writes` 倒序收集进对应通道；若该祖先 `channel_values` 已含目标通道，则记为 `seed` 并把该通道从 `remaining` 摘除（`base/__init__.py:634-642`）。
3. 直到所有通道都找到 seed 或走到根；返回 `{channel: {writes, seed?}}`。`InMemorySaver` 覆写为一次构造完整祖先链再逐通道累积（`memory/__init__.py:142`）。

## 4. 配置项

| flag / option | 默认 / 行为 | 位置 |
|---|---|---|
| `serde` 构造参数 | `None` → 用类属性 `JsonPlusSerializer()`；传入则覆盖；经 `maybe_add_typed_methods` 包装 | `base/__init__.py:212`、`:210`、`:217` |
| `config_specs` | 返回 `[]`（无子图配置项）；子类可覆写声明自己的 RunnableConfig 字段 | `base/__init__.py:219` |
| `thread_id`（运行期 config） | 必填，是存储/恢复检查点的主键；缺省时无法持久化/中断恢复/时间旅行 | `base/__init__.py:183-193` |
| `checkpoint_ns` | 默认 `""`，用于子图命名空间隔离 | `memory/__init__.py:245` |
| `prune(strategy=)` | `"keep_latest"`（每命名空间只留最新）/ `"delete"`（全删） | `base/__init__.py:379` |
| `LANGGRAPH_DELTA_MAX_SUPERSTEPS_SINCE_SNAPSHOT` | 增量通道快照的系统级上界，默认 5000 | `base/__init__.py:81` |

## 5. 错误与重试语义

- **未实现错误**：抽象方法 `get_tuple`/`list`/`put`/`put_writes`/删除/复制/裁剪等默认 `raise NotImplementedError`（`base/__init__.py:252` 等），由后端子类覆盖；`get`/`aget` 是对 `*_tuple` 的薄封装，不自行重试。
- **空通道**：`create_checkpoint` 遍历通道时捕获 `EmptyChannelError`（`base/__init__.py:848-851`），跳过尚未首次写入的通道，不抛给上层。
- **元数据净化**：`get_checkpoint_metadata` 剥离字符串中的 `\u0000` 空字节（`base/__init__.py:763`），避免写入某些数据库时的 C 字符串截断问题；config 顶层 `metadata`/`configurable` 中同名或 `EXCLUDED_METADATA_KEYS`/`__` 前缀键被忽略。
- **无重试/退避**：本层是内存/契约层，不涉及网络 IO，故无指数退避；后端（Postgres/SQLite）的重试与事务语义在各自叶子。
- **`with_allowlist` 降级**：若 serde 既非 `JsonPlusSerializer` 也非其加密包裹，仅 `logger.warning` 提示不强制 msgpack 白名单反序列化，不抛错（`base/__init__.py:738-742`）。

## 6. 并发细节

- **同步/异步双轨**：基类同时提供 `get`/`aget`、`put`/`aput` 等。`InMemorySaver` 的异步方法**直接 `return self.xxx(...)` 同步委托**（`memory/__init__.py:523-535`、`:559-577`），因为内存操作无阻塞 IO，无需线程池；这是双轨的"纯内存快路径"形态。
- **数据结构**：`storage`/`writes` 用 `defaultdict`，`blobs` 用普通 dict；`InMemorySaver` 本身**不加锁**，多线程并发写同一线程依赖 CPython GIL 与 dict 操作原子性，官方仅建议调试/测试用。
- **上下文管理**：实现 `AbstractContextManager` / `AbstractAsyncContextManager`，内部用 `ExitStack` 管理可注入的 `factory`（非 defaultdict 时需 `__enter__` 进入各表的上下文）（`memory/__init__.py:95-123`）。
- **取消传播**：本层不持有 asyncio task/队列；`alist` 是 async generator（`memory/__init__.py:537`），随调用方取消而停止 yield。
- **单调时钟**：`uuid6` 用模块级 `_last_v6_timestamp` 防止同毫秒乱序（`base/id.py:96-98`），是进程内的单调保护。

## 7. 系统边界

**In-Scope（本仓库源码内）**
- `libs/checkpoint/langgraph/checkpoint/base/`：数据结构、抽象契约、版本/元数据工具、uuid6。
- `libs/checkpoint/langgraph/checkpoint/memory/`：`InMemorySaver` 参考实现、`PersistentDict`。

**Out-of-Scope（不在本仓库源码内）**
- 实际持久化后端：`PostgresSaver`/`AsyncPostgresSaver`（见 checkpoint-postgres 叶子，依赖 psycopg 不在本仓库）、`SqliteSaver`（见 checkpoint-sqlite 叶子，依赖 sqlite3/aiosqlite 不在本仓库）。
- 序列化实现细节：`JsonPlusSerializer`/msgpack/加密序列化（见 checkpoint-serde 叶子）。
- Pregel 执行引擎如何调度 put/get（调用方，见 core-runtime 域）。
- Redis 缓存、Store 键值存储（见 store-cache-base 叶子）。

## 8. 与相邻子系统交互

- **上游（调用方）→ 本叶子**：Pregel 引擎（core-runtime 域）在编译时接收 `checkpointer`，每超步结束调 `put`/`put_writes`，恢复时调 `get_tuple`/`list`；`thread_id` 由运行时 `RunnableConfig.configurable` 传入。
- **本叶子 → 下游**：所有后端经 `self.serde`（`SerializerProtocol`，来自 checkpoint-serde 叶子）做对象↔`(type, bytes)` 编解码；`InMemorySaver` 不依赖外部进程。
- **本叶子 → 后端实现**：契约由 `BaseCheckpointSaver` 定义，postgres/sqlite 叶子继承并覆写 `get_tuple`/`put` 等以落地到数据库；`with_allowlist` 反向把 msgpack 白名单回灌进 serde。

## 9. 语言专项适配口径

本叶子是纯 Python 框架的典型**抽象基类契约层 + 参考实现**：
- **能力缝归组**：以 `BaseCheckpointSaver` 抽象基类定义"检查点存取"能力缝，方法分两类——有默认实现的模板方法（`get`→`get_tuple`、`aget`→`aget_tuple`，把"读单个"复用到"读 tuple"）与纯虚接口（`put`/`list` 等抛 `NotImplementedError`），后端只覆写后者。
- **双轨实现（Fast/Slow 式）**：同一能力同时给同步与异步两套方法（`put`/`aput`）。`InMemorySaver` 异步路径直接委托同步（无 IO，快路径）；postgres/sqlite 叶子则给出真正的 async 连接池实现——这是"同一契约多实现矩阵"在存储层的体现。
- **可选依赖/懒加载**：本叶子本身不重依赖 serde 之外的库；serde 内部对 msgpack/加密包的可选导入见 serde 叶子。
- **配置驱动**：通道版本类型 `V` 由子类泛型参数决定（`InMemorySaver[str]` 用字符串版本，基类默认 `int`），`get_next_version` 行为随 `V` 切换——是泛型+方法覆写的静态多态。
- **图型侧重**：architecture 表达"契约基类 + 内存实现 + serde 协作 + 外部后端"的分层；sequence 表达 put/get 的单次调用链。本叶子无独立多状态实体状态机（检查点状态变迁在 Pregel 循环里，归 core-runtime），故不补 lifecycle。

## 10. 图表清单与质量

| 图 | 文件 | 类型 | archify 质量档 |
|---|---|---|---|
| checkpoint-base 架构图 | `checkpoint-base-architecture.html` | architecture | standard |
| put/get 读写链时序图 | `checkpoint-base-sequence.html` | sequence | standard |

JSON IR 源文件位于 `json/` 目录。

降档披露：两图均落 **standard**。原因：升级后的 archify 新增 `composition/desktop-readability` 合成可读性检查，架构图主路径边 `pregel→base-saver` 在 1440 视口下触发该检查；已采取修复动作为缩短边标签为 `put/get`、加宽 `meta.viewBox` 至 `[1440,820]`。经两轮调整仍未过 showcase（该检查对整体构图密度敏感），按兜底策略接受 standard；两图 render 退出码均 0、HTML 均 >700KB。

省略说明：本叶子不补 dataflow 图——检查点读写是"对象经 serde 编码后落入内存表"的薄封装，数据流语义已被 sequence 调用链覆盖，无独立 ETL/管道结构；不补 lifecycle 图——检查点本身无多状态机（状态变迁在 Pregel 超步循环，归 core-runtime）。
