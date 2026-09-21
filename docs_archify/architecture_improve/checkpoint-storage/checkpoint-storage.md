# checkpoint-storage 域（检查点与存储后端）

> 本域是 LangGraph 检查点/存储/缓存抽象层及其多后端实现。域内叶子均围绕
> "同一抽象契约 × 多后端实现矩阵"组织，并以一致性测试套件保证各后端可移植。

## 域职责

本域定义并实现 LangGraph 的状态持久化三件套：

- **Checkpoint（检查点）**：保存图每一步的完整状态（通道值、元数据、待写），用于断点续跑与时间旅行。
- **Store（键值/向量存储）**：跨线程的长期记忆与语义检索。
- **Cache（缓存）**：键值结果缓存。

域级设计是**契约驱动 + 多后端矩阵**：抽象基类（`BaseCheckpointSaver`/`BaseStore`/`BaseCache`）定义契约，内存/Postgres/SQLite 三后端各自实现同一契约，一致性测试套件（conformance）对各后端做能力验收。

## 叶子索引

| 叶子 | 职责 | 关键产物 |
|---|---|---|
| [checkpoint-base](checkpoint-base/checkpoint-base.md) | 检查点抽象契约、内存实现、序列化（serde） | `BaseCheckpointSaver`/`InMemorySaver`/`JsonPlusSerializer` |
| [store-cache-base](store-cache-base/store-cache-base.md) | Store 与 Cache 抽象契约、内存实现、批量合批、嵌入 | `BaseStore`/`BaseCache`/`AsyncBatchedBaseStore` |
| [checkpoint-postgres](checkpoint-postgres/checkpoint-postgres.md) | Postgres 后端（Saver + Store），JSONB/Delta 重建/pgvector | `PostgresSaver`/`AsyncPostgresSaver`/`PostgresStore` |
| [checkpoint-sqlite](checkpoint-sqlite/checkpoint-sqlite.md) | SQLite 后端（Saver + Store + Cache），单连接 + 锁 | `SqliteSaver`/`SqliteStore`/`SqliteCache` |
| [checkpoint-conformance](checkpoint-conformance/checkpoint-conformance.md) | 契约驱动的可移植一致性测试套件 | `@checkpointer_test`/`validate`/9 能力点 |

## 域级机制

**1. 检查点抽象层 → 多后端矩阵**
`checkpoint-base` 定义 `BaseCheckpointSaver`（同步 + 异步双套方法）与 `Checkpoint`/`CheckpointMetadata`/`CheckpointTuple` 数据结构；`serde` 子包提供 `JsonPlusSerializer`（ormsgpack）与可选 `EncryptedSerializer`。后端矩阵：内存（`InMemorySaver`，三层 dict）、Postgres（JSONB 主表 + blobs 表 + writes 表，DeltaChannel 两阶段 JSONB 重建）、SQLite（BLOB 整存，Delta 在 Python 内解析）。

**2. Store/Cache 并行抽象**
`BaseStore` 只抽象 `batch`/`abatch`（操作符族 Get/Search/Put/ListNamespaces），`AsyncBatchedBaseStore` 在同 tick 合批去重；`BaseCache` 抽象 get/set/clear。后端：`InMemoryStore`（纯 Python 余弦 + JSONB 风格比较）、`PostgresStore`（pgvector + TTL sweeper 线程）、`SqliteStore`（sqlite-vec）、`SqliteCache`（RLock + WAL）。

**3. 一致性测试套件的契约驱动设计**
conformance 把契约拆成 9 个能力点（5 必备 + 4 可选），用 `@checkpointer_test` 装饰器让后端自助注册工厂，`validate` 自动探测"是否覆写异步方法"来判断能力覆盖，按能力跑对应 spec 套件并出 FULL/BASE+PARTIAL/BASE/NONE 合规分级。这是多后端矩阵的验收侧。

## 域级图

本域整体补 3 张域级图，覆盖静态拓扑、调用时序与数据管道三种语义：

| 图 | 文件 | 类型 | archify 质量档 |
|---|---|---|---|
| 域内组件静态拓扑（抽象契约 × 多后端矩阵） | `checkpoint-storage-architecture.html` | architecture | standard（组件多、跨层连接复杂，按规则降档） |
| checkpoint put/get 核心时序 | `checkpoint-storage-sequence.html` | sequence | standard（时间轴 y 上限告警，按规则降档） |
| checkpoint 持久化数据管道 | `checkpoint-storage-dataflow.html` | dataflow | standard（阶段多、视图宽度告警，按规则降档） |

JSON IR 源文件：`json/checkpoint-storage-architecture.json`、`json/checkpoint-storage-sequence.json`、
`json/checkpoint-storage-dataflow.json`。三张图分别表达：域内"抽象契约层 → 后端矩阵 → 外部数据库"的
静态分层与边界；从 Pregel 调用经 `serde`、具体后端到数据库再回到 `CheckpointTuple` 的单次读写时序；
以及 checkpoint 状态经序列化、三表拆分、落库到读取回填的完整数据管道。

## 与相邻域交互

- 上游：core-runtime（Pregel）经 `BaseCheckpointSaver`/`BaseStore` 接口调用；prebuilt-agents 把 checkpointer/store 注入编译图。
- 下游：外部数据库（Postgres/SQLite/Redis）与 LangChain 嵌入接口一律标注"不在本仓库源码内"。
