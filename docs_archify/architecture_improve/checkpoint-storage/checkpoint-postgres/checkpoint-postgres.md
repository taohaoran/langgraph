# checkpoint-postgres（Postgres 后端实现）

> 本文是 `checkpoint-storage` 域下的叶子子系统文档。域级总览见 `../checkpoint-storage.md`。
> 本文只展开 Postgres 后端的检查点存储（Saver）与键值/向量存储（Store），
> 抽象契约见 `../checkpoint-base/checkpoint-base.md` 与 `../store-cache-base/store-cache-base.md`。
>
> 源码基准：`libs/checkpoint-postgres`（9 源文件 / 5046 行）。

## 1. 功能清单

| 能力 | 说明 | 源码路径 |
|---|---|---|
| 同步检查点保存 | `PostgresSaver`，实现 `BaseCheckpointSaver` 同步 `put`/`get_tuple`/`list` 等 | `checkpoint/postgres/__init__.py:40` |
| 异步检查点保存 | `AsyncPostgresSaver`，异步方法桥接同步实现 | `checkpoint/postgres/aio.py:40` |
| 共享 SQL 与迁移 | `BasePostgresSaver` 持有全部 SQL 常量与 `MIGRATIONS`，双实现共用 | `checkpoint/postgres/base.py:344` |
| Shallow 变体 | `ShallowPostgresSaver`/`AsyncShallowPostgresSaver`，每 `(thread_id, ns)` 单快照的精简 schema | `checkpoint/postgres/shallow.py` |
| 键值/向量存储 | `PostgresStore`，实现 `BaseStore.batch`，pgvector 相似度检索 | `store/postgres/base.py:667` |
| 异步存储 | `AsyncPostgresStore`，`abatch` 用 `run_in_executor` 桥接同步 `batch` | `store/postgres/aio.py` |
| 连接管理 | `get_connection` 统一单连接/连接池取连接 | `checkpoint/postgres/_internal.py:14` |

## 2. 核心类型与接口清单

| 类型 / 函数 | 位置 | 职责 |
|---|---|---|
| `BasePostgresSaver` | `base.py:344` | 同步/异步共享基类；`get_next_version` 字符串版本（`:613`）、`_search_where`（`:624`）、全部 SQL 常量 |
| `PostgresSaver` | `__init__.py:40` | 同步 Saver，提供 `from_conn_string` 上下文管理器 |
| `AsyncPostgresSaver` | `aio.py:40` | 异步 Saver，同步方法用 `run_coroutine_threadsafe` 桥接 |
| `PostgresStore` | `store/postgres/base.py:667` | `BaseStore` 的 Postgres 实现，含 TTL sweeper 后台线程 |
| `BasePostgresStore` | `store/postgres/base.py:236` | Store 的同步 SQL 构造基类（GET/PUT/SEARCH/LIST 四类查询构造） |
| `PostgresIndexConfig` | `store/postgres/base.py:220` | pgvector 索引配置（HNSW/IVFFlat/flat、distance_type） |
| `get_connection` | `_internal.py:14` | 单连接直用、`ConnectionPool` 借还连接 |
| `PoolConfig` | `store/postgres/base.py:151` | 连接池参数（min_size/max_size/kwargs） |

## 3. 关键调用链

**写入检查点 `put`（与 sequence 图对应）**：
1. Pregel 调用 `put(config, checkpoint, metadata, new_versions)`（`base.py` 的 `BasePostgresSaver.put`）。
2. 拆分 `channel_values`：`None/str/int/float/bool` 等 primitive 内联进 `checkpoints.checkpoint` JSONB；`_DeltaSnapshot` 与其它类型经 `serde.dumps_typed` 写入 `checkpoint_blobs`（`_DeltaSnapshot` 在 `channel_values` 留 `True` 标记）。
3. `UPSERT_CHECKPOINT_BLOBS_SQL`（`ON CONFLICT DO NOTHING`）批量写 blob；`UPSERT_CHECKPOINTS_SQL`（`ON CONFLICT DO UPDATE`）写主记录；特殊通道走 `UPSERT_CHECKPOINT_WRITES_SQL`，普通走 `INSERT ... DO NOTHING`。
4. 返回 `next_config`（新 `checkpoint_id`）。

**读取检查点 `get_tuple`**：
1. 执行 `SELECT_SQL`（子查询拼接 `channel_versions`→`blobs` 与 `pending_writes`）。
2. 对每行 `serde.loads_typed` 回填通道值与待写列表，组装 `CheckpointTuple` 返回。

**DeltaChannel 两阶段重建**：`_build_delta_stage1_sql`（K 个通道并行 JSONB 键查找分页，页大小 `_DELTA_PAGE_SIZE=1024`，`base.py:23`）→ `_build_delta_stage2_sql`（每通道 `UNION ALL` 写+blob）；遍历在 blob 存在（`type<>'empty'`）或 inline 值存在时终止。

**Store 批量 `batch`**：`_group_ops` 按 `GetOp/SearchOp/ListNamespacesOp/PutOp` 分组，分别构造查询后在同一 pipeline 游标执行（`store/postgres/base.py:982`）。

## 4. 配置项

| 配置 | 默认 / 行为 | 位置 |
|---|---|---|
| `index`（PostgresIndexConfig） | 不传则关闭语义检索；含 `dims`/`embed`/`fields`/`ann_index_config`/`distance_type` | `store/postgres/base.py:220` |
| `ttl`（TTLConfig） | 配 TTL 后须显式 `start_ttl_sweeper()` | `store/postgres/base.py:764` |
| `pool_config` | `min_size=1`、`max_size=None`，自动 `autocommit=True`/`prepare_threshold=0`/`dict_row` | `store/postgres/base.py:806` |
| `pipeline` | `Capabilities().has_pipeline()` 探测；pipeline 模式仅配单连接 | `__init__.py` / `base.py` |
| `sweep_interval_minutes` | 未传时默认 5 分钟 | `store/postgres/base.py:875` |
| `setup()` | 按 `checkpoint_migrations`/`store_migrations`/`vector_migrations` 表版本幂等续跑迁移 | `base.py` / `store/postgres/base.py:1115` |

## 5. 错误与重试语义

- 迁移失败：`setup()` 中任一迁移 `cur.execute` 抛错即 `logger.error` 后 `raise`，不自动重试（`store/postgres/base.py:1145`）。
- 向量索引参数净化：`_get_index_params` 仅放行已知 `kind` 与整型参数（`m/ef_construction/nlist/lists`），非法值抛 `ValueError`，防 SQL 注入（`store/postgres/base.py:1237`）。
- 异步禁止在主事件循环同步调用：`AsyncPostgresSaver` 的同步方法桥接时检测运行中的事件循环，直接同步调用会报错（`aio.py`）。
- TTL sweeper 单轮异常：`logger.exception` 记录后继续下一周期，不终止线程（`store/postgres/base.py:891`）。

## 6. 并发细节

- `PostgresStore.lock`（`threading.Lock`，`store/postgres/base.py:771`）：pipeline 模式下同一时刻只允许一个游标使用连接，加锁串行化。
- pipeline 模式下连接可被多线程/协程共用，但一次一个游标；非 pipeline 且支持 pipeline 能力时仍加锁 + `conn.pipeline()`（`_cursor`，`:941`）。
- TTL sweeper 守护线程 `ttl-sweeper`（`:899`），通过 `threading.Event` 唤醒/停止，`stop_ttl_sweeper` 可 join；`__del__` 兜底 `stop_ttl_sweeper(timeout=0.1)`（`:936`）。
- `AsyncPostgresSaver` 同步方法桥接用 `run_coroutine_threadsafe`，异步方法直走 `AsyncConnection`；`AsyncPostgresStore.abatch` 用 `loop.run_in_executor(None, self.batch, ops)`（`:1112`）。

## 7. 系统边界

**In-Scope（本仓库源码内）**
- `checkpoint/postgres/`：同步/异步 Saver、共享 SQL、连接管理、Shallow 变体。
- `store/postgres/`：`PostgresStore`/`BasePostgresStore`/`AsyncPostgresStore`、SQL 构造、TTL sweeper、pgvector 索引配置。

**Out-of-Scope（不在本仓库源码内）**
- `psycopg` / `psycopg-pool` 驱动与连接池实现。
- PostgreSQL 数据库本身与 pgvector 扩展（HNSW/IVFFlat 索引、向量距离算子）。
- LangChain `Embeddings` 接口及其具体嵌入模型实现。
- 检查点抽象契约本身见 `checkpoint-base` 叶子，不在本叶子重复。

## 8. 与相邻子系统交互

- 上游：Pregel（core-runtime）经 `BaseCheckpointSaver` 接口调用 `put`/`get_tuple`/`list`；应用代码经 `BaseStore` 接口调用 `batch`/`search`/`put`。
- 本叶子 → 下游：所有 SQL 经 `get_connection` → `psycopg` 驱动 → Postgres 数据库；向量检索经 LangChain `Embeddings` 把文本嵌入为向量后再写 `store_vectors`。
- 依赖方向：本叶子依赖 `libs/checkpoint` 的 `BaseCheckpointSaver`/`BaseStore` 抽象与 `serde`，不反向依赖 core Pregel。

## 9. 语言专项适配口径

纯 Python 框架的**能力缝 + 多后端矩阵**口径：`PostgresSaver`/`AsyncPostgresSaver`/`PostgresStore` 是 `checkpoint-base` 抽象基类体系（`BaseCheckpointSaver`/`BaseStore`）在 Postgres 后端的具体实现，与 `MemorySaver`/`InMemoryStore`、SQLite 后端实现同一契约——即"同一抽象接口 × 多后端实现矩阵"。本叶子额外承担同步/异步双轨实现（`run_coroutine_threadsafe`/`run_in_executor` 桥接）与 SQL schema/迁移自管理（`MIGRATIONS` 列表位置即版本号）。图类型以 architecture（组件/连接边界）+ sequence（读写生命周期）为主。

## 10. 图表清单与质量

| 图 | 文件 | 类型 | archify 质量档 |
|---|---|---|---|
| 后端结构与连接管理 | `checkpoint-postgres-architecture.html` | architecture | standard（showcase 布局校验：跨边与标签间距，迭代后仍有轻微告警，按规则降档） |
| 检查点写入/读取生命周期 | `checkpoint-postgres-sequence.html` | sequence | standard（showcase 时间轴 y 上限告警，按规则降档） |

JSON IR 源文件位于 `json/`。

本叶子已合规（architecture + sequence 共 2 图），不补第三张图：Postgres 后端无独立状态机（无单一实体
多状态迁移语义）、无独立数据管道（读写管道已由 sequence 图表达）、无多角色泳道流程（连接池/迁移为
初始化步骤而非业务流程），新增 lifecycle/dataflow/workflow 图均与已有图信息重复，按资源节省原则省略。
