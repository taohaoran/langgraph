# checkpoint-base：检查点抽象层与内存实现（checkpoint-base）

> 本文是 `checkpoint-storage` 域下的叶子子系统文档。域级总览见 `../checkpoint-storage.md`。
> 本文只展开检查点（checkpoint）的**抽象契约层、内存参考实现与序列化器**，不展开
> Postgres / SQLite 后端（见 `../checkpoint-postgres/`、`../checkpoint-sqlite/`）与一致性测试
> （见 `../checkpoint-conformance/`）。
>
> 源码基准：`langgraph-checkpoint`（Python），目录 `libs/checkpoint/langgraph/`。

## 1. 功能清单

| 能力 | 说明 | 源码路径 |
| --- | --- | --- |
| 检查点读写抽象契约 | 定义 `BaseCheckpointSaver` 的同步/异步双套方法：取、列、存、写中间结果、删线程、按 run 删、拷线程、裁剪 | `checkpoint/base/__init__.py:177` |
| 检查点数据结构 | `Checkpoint`（通道值/版本/节点所见版本）、`CheckpointMetadata`（来源/步数/父链/run）、`CheckpointTuple`、`DeltaChannelHistory` | `checkpoint/base/__init__.py:93`、`:39`、`:140`、`:150` |
| 增量通道（DeltaChannel）祖先链回放 | 默认实现沿 `parent_config` 走父链累积 `pending_writes`，遇非空 `channel_values` 即止 | `checkpoint/base/__init__.py:583`、`:652` |
| 内存检查点实现 | `InMemorySaver`（别名 `MemorySaver`），三层 `defaultdict` 存储 + blob 分离存储 | `checkpoint/memory/__init__.py:33` |
| 版本号生成 | 默认整型自增；`InMemorySaver` 覆写为 `"001.000...".000...` 字符串 | `checkpoint/base/__init__.py:693`、`checkpoint/memory/__init__.py:613` |
| 单调 ID 生成 | `uuid6`（UUIDv6），时间有序、利于数据库聚簇排序 | `checkpoint/base/id.py:79` |
| 类型化序列化协议 | `SerializerProtocol`（`dumps_typed`/`loads_typed`）、旧版兼容适配器 | `checkpoint/serde/base.py:15`、`:29` |
| ormsgpack 主序列化器 | `JsonPlusSerializer`，ext 码 0~7 编排 pydantic/namedtuple/dataclass/numpy 等 | `checkpoint/serde/jsonplus.py:82` |
| 反序列化安全白名单 | `SAFE_MSGPACK_TYPES`/`SAFE_MSGPACK_METHODS`、strict 模式、allowlist 合并 | `checkpoint/serde/_msgpack.py:21` |
| 加密序列化包装 | `EncryptedSerializer` + `CipherProtocol`，可外接 AES | `checkpoint/serde/encrypted.py:8`、`checkpoint/serde/base.py:51` |
| 序列化事件钩子 | 注册/触发 allowlist 告警事件 | `checkpoint/serde/event_hooks.py:26` |
| 特殊写入索引映射 | `WRITES_IDX_MAP`（ERROR/SCHEDULED/INTERRUPT/RESUME 负索引） | `checkpoint/base/__init__.py:796` |

## 2. 核心类型与接口清单

| 类型 / 函数 | 位置 | 职责 |
| --- | --- | --- |
| `BaseCheckpointSaver[V]` | `checkpoint/base/__init__.py:177` | 抽象基类；`serde` 类属性默认 `JsonPlusSerializer()`，构造时经 `maybe_add_typed_methods` 包装 |
| `Checkpoint` (TypedDict) | `checkpoint/base/__init__.py:93` | `v`/`id`/`ts`/`channel_values`/`channel_versions`/`versions_seen`/`updated_channels` |
| `CheckpointMetadata` (total=False) | `checkpoint/base/__init__.py:39` | `source`(input/loop/update/fork)、`step`、`parents`、`run_id`、delta 计数 |
| `CheckpointTuple` (NamedTuple) | `checkpoint/base/__init__.py:140` | `config`+`checkpoint`+`metadata`+`parent_config`+`pending_writes` |
| `InMemorySaver` | `checkpoint/memory/__init__.py:33` | 继承 `BaseCheckpointSaver[str]`，同时实现同步/异步上下文管理器 |
| `SerializerProtocol` (Protocol) | `checkpoint/serde/base.py:15` | 能力缝：类型化序列化契约（消费方接口，实现方可替换） |
| `JsonPlusSerializer` | `checkpoint/serde/jsonplus.py:82` | 默认序列化实现，ormsgpack 编码 + ext hook 反序列化 |
| `EncryptedSerializer` | `checkpoint/serde/encrypted.py:8` | 装饰式包装：先内层 serde 再加密，类型串记为 `"{typ}+{cipher}"` |
| `_DeltaSnapshot` (NamedTuple) | `checkpoint/serde/types.py:19` | DeltaChannel 快照 blob，ext 码 7 编解码 |
| `uuid6` | `checkpoint/base/id.py:79` | 时间有序 UUID，`clock_seq=step` 用作 checkpoint 序号 |
| `EmptyChannelError` | `checkpoint/base/__init__.py:746` | 通道首次取值前抛出 |

接口定义位置说明：`SerializerProtocol` 定义在**消费方**（`BaseCheckpointSaver` 持有它），
实现方（`JsonPlusSerializer`/`EncryptedSerializer`）向下对齐——典型依赖倒置。

## 3. 关键调用链

**写入一条检查点（`InMemorySaver.put`，`checkpoint/memory/__init__.py:421`）**

1. 从 `checkpoint` 中弹出 `channel_values`，对每个 `new_versions` 通道把值经
   `self.serde.dumps_typed(values[k])` 写入 `self.blobs[(thread, ns, channel, version)]`；
   未在 `values` 中的版本记为 `("empty", b"")`。
2. 把去掉 `channel_values` 的 checkpoint 本体与 `get_checkpoint_metadata(config, metadata)`
   分别 `dumps_typed`，连同父 checkpoint_id 存入
   `self.storage[thread][ns][checkpoint_id] = (ckpt_bytes, meta_bytes, parent_id)`。
3. 返回带新 `checkpoint_id` 的 `configurable`。

**读取（`InMemorySaver.get_tuple`，`checkpoint/memory/__init__.py:230`）**

1. 若 config 指定 `checkpoint_id` 直接取；否则 `max(checkpoints.keys())` 取最新。
2. 反序列化 checkpoint 本体，再按 `channel_versions` 逐版本从 `blobs` 经
   `_load_blobs` → `serde.loads_typed` 回填 `channel_values`（`"empty"` 跳过）。
3. 组装 `CheckpointTuple`，`parent_config` 由父 checkpoint_id 构造。

**中间写入（`put_writes`，`checkpoint/memory/__init__.py:467`）**

1. 内层键为 `(task_id, WRITES_IDX_MAP.get(channel, idx))`；正则写仅在尚未存在时写入。
2. 特殊通道（ERROR/INTERRUPT 等）用负索引，避免与正则写序号冲突。

**DeltaChannel 历史回放（默认实现，`checkpoint/base/__init__.py:583`）**

1. 从目标 checkpoint 的 `parent_config` 出发循环 `get_tuple`；
2. 对每个祖先的 `pending_writes` 逆序累积目标通道写入；
3. 一旦某祖先 `channel_values[ch]` 有值即作为 `seed` 并停止该通道回放，直至所有通道收敛或到根。

## 4. 配置项

| 配置 | 默认 / 行为 | 位置 |
| --- | --- | --- |
| `serde`（构造参数） | 默认 `JsonPlusSerializer()`；可注入加密/自定义序列化器 | `checkpoint/base/__init__.py:210` |
| `pickle_fallback` | 默认 `False`；ormsgpack 编码失败时是否回落 pickle | `checkpoint/serde/jsonplus.py:100` |
| `allowed_msgpack_modules` | 默认 permissive（全部允许并告警）；strict 时仅白名单 | `checkpoint/serde/jsonplus.py:102` |
| 环境变量 `LANGGRAPH_STRICT_MSGPACK` | 非 `true/1/yes` 即关闭 strict；开启后仅 `SAFE_MSGPACK_TYPES` 可反序列化 | `checkpoint/serde/_msgpack.py:12` |
| 环境变量 `LANGGRAPH_AES_KEY` | AES 密钥，长度须 16/24/32 字节（EAX 模式） | `checkpoint/serde/encrypted.py:54` |
| `prune(strategy=)` | 默认 `"keep_latest"`，另有 `"delete"` | `checkpoint/base/__init__.py:375` |
| `LANGGRAPH_DELTA_MAX_SUPERSTEPS_SINCE_SNAPSHOT` | delta 快照系统级上限，默认 5000 | `checkpoint/base/__init__.py:81`（注释） |

## 5. 错误与重试语义

- 本叶子不涉及网络/外部存储，无重试与退避逻辑；异步方法在 `InMemorySaver` 中**直接同步调用**
  （`aget_tuple`/`aput` 等仅 `return self.xxx(...)`，`checkpoint/memory/__init__.py:523` 起），
  即内存实现不做线程池卸载。
- 反序列化失败按类型分级：白名单外类型在 permissive 模式**告警但放行**（返回原始元组数据），
  strict 模式**拒绝并返回原始数据**（`jsonplus.py:633` 起各 ext 分支 `except Exception: return None`）。
- `EmptyChannelError`（`checkpoint/base/__init__.py:746`）在 `create_checkpoint` 中被捕获并跳过
  未更新通道（`base/__init__.py:850`）。
- `InvalidModuleError`（`jsonplus.py:842`）在未配置 allowlist 时给出明确解锁指引。

## 6. 并发细节

- `InMemorySaver` 用 `defaultdict`/`dict` 作为存储，**无显式锁**；其线程安全性依赖调用方
  （Pregel 单循环）。`factory` 参数允许注入带上下文管理的 `defaultdict`。
- `uuid6` 用模块级 `_last_v6_timestamp` 保证单调（`id.py:90`），存在跨线程竞争但有时间戳
  递增兜底。
- `event_hooks` 用 `threading.Lock` 保护监听器列表，触发时先快照元组再逐个调用，
  单监听器异常被隔离并记日志（`event_hooks.py:41`）。
- 无 `context.Context` 传递（Python 库）；异步/同步为同一存储的两套入口，非并发流水线。

## 7. 系统边界

**In-Scope（本仓库源码内）**

- `BaseCheckpointSaver` 抽象契约与全部数据结构
- `InMemorySaver` 参考内存实现与 `PersistentDict` 辅助类
- 序列化协议与 `JsonPlusSerializer`/`EncryptedSerializer`/安全白名单/事件钩子
- `uuid6` 单调 ID

**Out-of-Scope（不在本仓库源码内）**

- Postgres / SQLite / Redis 后端实现：见 `checkpoint-postgres`、`checkpoint-sqlite`、`cache/redis`
- Pregel 核心循环（`langgraph` 主包）——本叶子只被其调用
- `ormsgpack`、`pycryptodome`、`numpy`、`langchain_core` 第三方依赖
- 可移植一致性测试：见 `checkpoint-conformance`

## 8. 与相邻子系统交互

- 上游（调用方）：`langgraph` 核心 Pregel 在每个 superstep 后调用 `put`/`put_writes`，
  恢复时调用 `get_tuple`/`aget_tuple`；传入 `RunnableConfig`（含 `thread_id`/`checkpoint_ns`/`checkpoint_id`）。
- 下游（依赖）：所有后端实现（Postgres/SQLite）继承 `BaseCheckpointSaver`，复用本叶子的
  `serde`、`WRITES_IDX_MAP`、`Checkpoint` 结构与 `uuid6`。
- 横向：`store`/`cache` 抽象（见 `../store-cache-base/`）与本叶子并行，均依赖
  `JsonPlusSerializer`（`jsonplus.py:504` 直接序列化 `store.base.Item`）。

## 9. 语言专项适配口径

纯 Python 框架库，按「能力缝 + 注册表/工厂 + 多后端矩阵」口径分析：

1. **能力缝归组**：`BaseCheckpointSaver` 与 `SerializerProtocol` 是两个抽象基类/Protocol 契约层，
   实现方（InMemory/Postgres/SQLite、JsonPlus/Encrypted）向下对齐，接口定义在消费方=依赖倒置。
2. **配置驱动**：运行时序列化行为由 `serde` 注入对象决定（普通/加密/严格白名单），同一存储
   经不同 serde 切换安全/加密后端。
3. **多后端矩阵**：同一 `BaseCheckpointSaver` 契约由 memory/postgres/sqlite 三套实现，
   一致性由 `checkpoint-conformance` 套件（见对应叶子）强制。
4. **图类型侧重**：本叶子以 architecture 图表达「抽象契约层 → 多实现 + serde 策略注入」；
   核心调用链（put/get/delta 回放）已在第 3 节文字化，跨叶子的 checkpoint 读写生命周期
   时序图由后端叶子（postgres/sqlite）补充。

## 10. 图表清单与质量

| 图 | 文件 | 类型 | archify 质量档 |
| --- | --- | --- | --- |
| 抽象层与内存实现架构图 | `checkpoint-base-architecture.html` | architecture | showcase |
| 抽象契约 put/get/version 调用时序 | `checkpoint-base-sequence.html` | sequence | showcase |

JSON IR 源文件：`json/checkpoint-base-architecture.json`、`json/checkpoint-base-sequence.json`。
第二张 sequence 图补充基线 architecture 未表达的时序语义：Pregel 经模板方法调用 `put`/`get_tuple`，
`BaseCheckpointSaver` 委托 `InMemorySaver` 子类实现，期间经 `serde` 序列化/反序列化并由
`get_next_version`（`checkpoint/base/__init__.py:693`）完成单调版本推进。checkpoint 读写的后端 SQL
视角仍由 `checkpoint-postgres`/`checkpoint-sqlite` 叶子的 sequence 图承担，本图聚焦抽象契约层。
