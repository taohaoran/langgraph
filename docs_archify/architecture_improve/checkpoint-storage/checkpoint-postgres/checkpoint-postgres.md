# checkpoint-postgres（checkpoint-postgres）

> 本文是 `checkpoint-storage` 域下的叶子子系统文档。本叶子是检查点的 **Postgres 生产后端**：
> `PostgresSaver`/`AsyncPostgresSaver` 同步/异步双轨、`ShallowPostgresSaver` 浅模式、
> `PostgresStore` 键值存储与向量检索。它把 `BaseCheckpointSaver` 契约落地到 Postgres
> （checkpoints / checkpoint_blobs / checkpoint_writes / checkpoint_migrations 四张表）。
>
> 源码基准：`langgraph-checkpoint-postgres`，commit `e539ac12`。

## 1. 功能清单

| 能力 | 说明 | 源码路径 |
|---|---|---|
| 同步检查点后端 | `PostgresSaver`：`from_conn_string` 上下文工厂、`setup()` 建表迁移、`list`/`get_tuple`/`put`/`put_writes`/`delete_thread` | `checkpoint/postgres/__init__.py:40`、`:64`、`:85`、`:112`、`:192`、`:263`、`:347`、`:381` |
| 异步检查点后端 | `AsyncPostgresSaver`：`alist`/`aget_tuple`/`aput`/`aput_writes`/`adelete_thread`/异步 `_cursor`；同步方法转为在连接上跑 | `checkpoint/postgres/aio.py:40`、`:119`、`:181`、`:232`、`:307`、`:340`、`:364` |
| 共享基类 | `BasePostgresSaver(BaseCheckpointSaver[str])`：blob/write 编解码、增量通道两阶段 SQL 构建、`get_next_version` 字符串版本 | `checkpoint/postgres/base.py:344`、`:549`、`:574`、`:590`、`:613`、`:201`、`:293` |
| 浅模式后端 | `ShallowPostgresSaver`/`AsyncShallowPostgresSaver`：不存完整 channel_values，仅存元数据（低开销） | `checkpoint/postgres/shallow.py:169`、`:529` |
| 连接管理 | `_internal.get_connection`：单 `Connection` 直接用，`ConnectionPool` 借连接；`_cursor` 上下文管理 pipeline/事务 | `checkpoint/postgres/_internal.py:11`、`__init__.py:405` |
| 异步连接管理 | `_ainternal.get_connection`：异步版（`AsyncConnection`/`AsyncConnectionPool`） | `checkpoint/postgres/_ainternal.py:11` |
| 键值存储后端 | `PostgresStore(BaseStore)`：`batch`/`abatch`、GET/PUT/SEARCH/list_namespaces 查询构建、HNSW/IVFFlat 向量索引配置 | `store/postgres/base.py:667`、`:982`、`:1112`、`:236`、`:191`、`:201` |
| TTL 清扫 | `sweep_ttl`/`start_ttl_sweeper`/`stop_ttl_sweeper`：后台线程清扫过期 store 记录 | `store/postgres/base.py:834`、`:850`、`:908` |
| 建表迁移 | `MIGRATIONS` SQL：建 checkpoint_migrations/checkpoints/checkpoint_blobs/checkpoint_writes 四表+索引 | `checkpoint/postgres/base.py:44-90` |

## 2. 核心类型与接口清单

| 类型 / 函数 | 位置 | 职责 |
|---|---|---|
| `BasePostgresSaver[V]` | `checkpoint/postgres/base.py:344` | Postgres 后端共享逻辑：`_dump_blobs`/`_load_blobs`（blob 表读写）、`_dump_writes`/`_load_writes`、增量通道两阶段走查 SQL。 |
| `PostgresSaver` | `checkpoint/postgres/__init__.py:40` | 同步实现。`__init__(conn, pipe, serde)` 持有 `conn`、`pipe`、`lock=threading.Lock()`、`supports_pipeline=Capabilities().has_pipeline()`（`:55-61`）。 |
| `AsyncPostgresSaver` | `checkpoint/postgres/aio.py:40` | 异步实现，覆写全部 `a*` 方法；同步方法在 `_ainternal` 连接上跑。 |
| `ShallowPostgresSaver` | `checkpoint/postgres/shallow.py:169` | 浅模式：`put` 不落地 blob 全量值。 |
| `_internal.get_connection` | `checkpoint/postgres/_internal.py:11` | 把 `Connection | ConnectionPool` 统一成可用连接上下文。 |
| `PostgresStore` | `store/postgres/base.py:667` | Postgres 键值/向量存储；`batch` 路由 GET/PUT/SEARCH/list_namespaces 四类 Op。 |
| `PoolConfig`/`HNSWConfig`/`IVFFlatConfig` | `store/postgres/base.py:151`、`:191`、`:201` | 连接池与 ANN 索引配置 TypedDict。 |

## 3. 关键调用链

**写链 `PostgresSaver.put`（`__init__.py:263`）：**
1. 从 config 拆出 `thread_id`/`checkpoint_ns`/旧 `checkpoint_id`，构造带新 checkpoint_id 的 `next_config`（`:295-307`）。
2. 分流 channel_values：`_DeltaSnapshot` 替换为占位 `True` 并移入 blob_values；`None`/str/int/float/bool 内联进 JSONB；其余非基础类型移入 blob_values（`:311-319`）。
3. `with self._cursor(pipeline=True) as cur`（`:321`）：先 `executemany(UPSERT_CHECKPOINT_BLOBS_SQL, self._dump_blobs(...))` 写大对象（`:322-333`），再 `execute(UPSERT_CHECKPOINTS_SQL, (..., Jsonb(copy), Jsonb(metadata)))` 写检查点主行（`:334-344`）。

**游标获取 `_cursor`（`__init__.py:405`）：**
1. `with self.lock, _internal.get_connection(self.conn) as conn` 取连接（`:413`）。
2. 若已在 pipeline 模式：直接 `conn.cursor(binary=True, dict_row)`，结束 `self.pipe.sync()`（`:418-423`）。
3. 否则按 `pipeline` 参数：支持 pipeline 则开 `conn.pipeline()`，否则退到 `conn.transaction()`（`:424-439`）。

**写 pending writes `put_writes`（`__init__.py:347`）：**
1. 若所有写入 channel 都在 `WRITES_IDX_MAP`（特殊事件 ERROR/INTERRUPT 等）用 upsert SQL，否则用普通 insert SQL（`:363-367`）。
2. `executemany(query, self._dump_writes(thread, ns, cp_id, task_id, task_path, writes))`（`:368-379`）。

**读链 `get_tuple`（`__init__.py:192`）**：按 config 指定/最新 checkpoint_id join checkpoints 与 checkpoint_blobs，`_load_checkpoint_tuple` 把 JSONB 行 + blob 行组装成 `CheckpointTuple`（含 parent_config、pending_writes）。

## 4. 配置项

| flag / option | 默认 / 行为 | 位置 |
|---|---|---|
| `conn` 构造参数 | `Connection | ConnectionPool`；pool 模式下不能同时传 pipe | `__init__.py:45-56` |
| `pipe` | `Pipeline | None`；单连接 pipeline 模式可多线程共享连接但同一时刻一个 cursor | `__init__.py:414-417` |
| `from_conn_string(pipeline=)` | 默认 False；True 时开 `conn.pipeline()` | `__init__.py:64-84` |
| `supports_pipeline` | `Capabilities().has_pipeline()`；不支持时退到事务上下文 | `__init__.py:60`、`:427` |
| 连接参数 | `autocommit=True, prepare_threshold=0, row_factory=dict_row` | `__init__.py:77` |
| ANN 索引 | `HNSWConfig`/`IVFFlatConfig` 可选配置向量索引 | `store/postgres/base.py:191` |
| TTL sweeper | `start_ttl_sweeper(...)` 后台线程周期清扫过期记录 | `store/postgres/base.py:850` |

## 5. 错误与重试语义

- **连接类型错误**：`get_connection` 遇未知连接类型抛 `TypeError`（`_internal.py:20`）。
- **pool+pipe 冲突**：构造时传 `ConnectionPool` 又传 `pipe` 抛 `ValueError`（`__init__.py:56-58`）。
- **pipeline 降级**：服务器不支持 pipeline 时自动退到 `conn.transaction()`（`__init__.py:433-439`），不报错。
- **事务边界**：`put`/`put_writes` 在 `_cursor(pipeline=True)` 内批量执行，异常随连接回滚；`autocommit=True` 下 pipeline sync 后生效。
- **序列化失败**：blob 经 `self.serde.dumps_typed` 失败会上抛（见 serde 叶子）；本层不吞异常。
- **无应用层重试**：连接级故障不自动重试；由调用方/连接池处理。

## 6. 并发细节

- **锁**：`self.lock = threading.Lock()`，所有 `_cursor` 串行化对非 pipeline 连接的访问（`__init__.py:55`、`:413`）。pipeline 模式下连接可多线程共享但同一时刻仅一个 cursor。
- **同步/异步双轨**：`PostgresSaver`（psycopg 同步 Connection）与 `AsyncPostgresSaver`（AsyncConnection）是两套独立实现；共享 `BasePostgresSaver` 的 SQL 构建与 blob/write 编解码。
- **批量**：`executemany` + pipeline 减少往返；`_cursor(pipeline=True)` 结束 `pipe.sync()`。
- **TTL sweeper 线程**：`PostgresStore.start_ttl_sweeper` 启动后台线程周期 `sweep_ttl`；`stop_ttl_sweeper(timeout)` 优雅停止；`__del__` 兜底（`store/postgres/base.py:850`、`:908`、`:936`）。
- **取消传播**：异步版在 `AsyncConnection` 上跑，随 asyncio 取消传播；无自研队列。

## 7. 系统边界

**In-Scope（本仓库源码内）**
- `libs/checkpoint-postgres/langgraph/checkpoint/postgres/`：saver 实现。
- `libs/checkpoint-postgres/langgraph/store/postgres/`：PostgresStore。

**Out-of-Scope（不在本仓库源码内）**
- Postgres 数据库服务端（不在本仓库）。
- `psycopg` / `psycopg_pool`：Postgres 驱动（本包硬依赖 PyPI 库，`__init__.py:22-25`）。
- 序列化器（serde 叶子）、`BaseCheckpointSaver` 契约（checkpoint-base 叶子）。
- 向量扩展（pgvector HNSW/IVFFlat）是 Postgres 扩展，不在本仓库。

## 8. 与相邻子系统交互

- **上游 → 本叶子**：Pregel 引擎经 `BaseCheckpointSaver` 契约调用 `put`/`get_tuple`/`alist`；`thread_id` 来自 RunnableConfig。
- **本叶子 → 下游**：通过 `self.serde`（serde 叶子）把 channel_values 编成 `(type, bytes)`；主行用 `Jsonb` 直接落 JSONB 列，大对象落 checkpoint_blobs 表。
- **本叶子 → 外部**：经 psycopg/psycopg_pool 驱动访问 Postgres；`setup()` 创建四表+索引+迁移记录。
- **相邻**：`ShallowPostgresSaver` 复用 `BasePostgresSaver` 但简化 put；`PostgresStore` 是独立的键值/向量存储，复用同一连接管理。

## 9. 语言专项适配口径

- **双轨实现（Fast/Slow 式）**：同步 `PostgresSaver` + 异步 `AsyncPostgresSaver` 共享 `BasePostgresSaver` 的 SQL 构建与编解码——同一契约两套 IO 实现，是存储层双轨。
- **可选依赖/懒加载**：`psycopg`/`psycopg_pool` 是 `langgraph-checkpoint-postgres` 这个 **extras 包**的硬依赖；主 `langgraph-checkpoint` 包不 import psycopg——安装 extras 才解锁后端，符合"主包不强制重依赖"的懒加载口径。
- **配置驱动分派**：`supports_pipeline`（运行时探测服务器能力）决定走 pipeline 还是事务；浅/全模式由选哪个子类决定——配置/选类切换后端行为。
- **注册表/工厂**：`from_conn_string` 是类方法工厂（上下文管理器）；`_dump_writes` 按 channel 是否在 `WRITES_IDX_MAP` 字符串分派 upsert/insert SQL。
- **图型侧重**：architecture 表达"继承体系 + 四表 + psycopg 外部边界"；sequence 表达 put 的事务写链；JSONB 列与 blob 表的分流已在 sequence/MD 第 3 节表达，不另补 dataflow。

## 10. 图表清单与质量

| 图 | 文件 | 类型 | archify 质量档 |
|---|---|---|---|
| Postgres 后端架构图 | `checkpoint-postgres-architecture.html` | architecture | standard |
| put 事务写链时序图 | `checkpoint-postgres-sequence.html` | sequence | showcase |

降档披露：架构图落 **standard**（原因：升级 archify 的 `composition/desktop-readability` 检查对 10 组件+9 连线密度敏感）。时序图一次过 **showcase**。两图 render 退出码均 0、HTML 均 >700KB。

JSON IR 源文件位于 `json/` 目录。

省略说明：本叶子不补 dataflow/lifecycle 图——checkpoint 写入是"对象→serde→JsonB/blob 表"的薄管道，已被 sequence 写链覆盖；检查点无单对象状态机（状态变迁在 Pregel 循环）。
