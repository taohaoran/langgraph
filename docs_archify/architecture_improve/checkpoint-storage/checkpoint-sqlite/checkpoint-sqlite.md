# checkpoint-sqlite（checkpoint-sqlite）

> 本文是 `checkpoint-storage` 域下的叶子子系统文档。本叶子是检查点的 **SQLite 后端**：
> `SqliteSaver`/`AsyncSqliteSaver` 同步/异步双轨、`SqliteStore` 键值/向量存储、`SqliteCache`
> 缓存，以及增量 checkpoint 辅助。它把 `BaseCheckpointSaver` 契约落地到 SQLite（单文件/内存库）。
>
> 源码基准：`langgraph-checkpoint-sqlite`，commit `e539ac12`。

## 1. 功能清单

| 能力 | 说明 | 源码路径 |
|---|---|---|
| 同步检查点后端 | `SqliteSaver(BaseCheckpointSaver[str])`：`from_conn_string`、`setup()` 建表、`cursor()`、`get_tuple`/`list`/`put`/`put_writes`/`delete_thread`/增量历史 | `checkpoint/sqlite/__init__.py:45`、`:99`、`:129`、`:169`、`:191`、`:295`、`:387`、`:445`、`:484`、`:503` |
| 异步检查点后端 | `AsyncSqliteSaver`：基于 aiosqlite，`asetup`/`aget_tuple`/`alist`/`aput`/`aput_writes`/`adelete_thread`/异步增量历史 | `checkpoint/sqlite/aio.py:38`、`:305`、`:346`、`:430`、`:509`、`:561`、`:602`、`:622` |
| 增量 checkpoint 辅助 | `build_delta_stage2_sql`/`step_walk_with_row`/`build_delta_channels_writes_history`：祖先链走查与增量通道写历史 | `checkpoint/sqlite/_delta.py:42`、`:68`、`:127` |
| SQL 注入防护 | `_validate_filter_key`：过滤键仅允许字母数字/下划线/点/横线；`_metadata_predicate` 生成参数化 WHERE | `checkpoint/sqlite/utils.py:14`、`:33` |
| 键值存储后端 | `SqliteStore(BaseStore)`：`batch`/`abatch`、GET/PUT/SEARCH/list_namespaces、`sweep_ttl`/TTL sweeper 线程 | `store/sqlite/base.py:787`、`:1236`、`:1475`、`:1129`、`:1145` |
| 缓存后端 | `SqliteCache(BaseCache)`：`get`/`set`/`clear` 及异步版，按 FullKey 存取带 TTL | `cache/sqlite/__init__.py:13`、`:46`、`:78`、`:98` |
| 版本号 | `get_next_version` 覆写为字符串版本（与 InMemorySaver 同格式） | `checkpoint/sqlite/__init__.py:626` |

## 2. 核心类型与接口清单

| 类型 / 函数 | 位置 | 职责 |
|---|---|---|
| `SqliteSaver` | `checkpoint/sqlite/__init__.py:45` | 同步后端。`__init__` 持有 `conn: sqlite3.Connection`、`lock=threading.Lock()`。 |
| `AsyncSqliteSaver` | `checkpoint/sqlite/aio.py:38` | 异步后端，基于 aiosqlite；`_started` 守护连接初始化（`:731`）。 |
| `SqliteStore` | `store/sqlite/base.py:787` | SQLite 键值/向量存储，复用 BaseSqliteStore 的查询构建。 |
| `SqliteCache[ValueT]` | `cache/sqlite/__init__.py:13` | SQLite 缓存，实现 BaseCache 契约。 |
| `cursor(transaction=)` | `checkpoint/sqlite/__init__.py:169` | 游标上下文：`with self.lock` → `setup()` → `conn.cursor()` → 退出 commit/close。 |
| `_validate_filter_key` | `checkpoint/sqlite/utils.py:14` | 白名单校验过滤键，防 SQL 注入。 |

## 3. 关键调用链

**写链 `SqliteSaver.put`（`__init__.py:387`）：**
1. 取 `thread_id`/`checkpoint_ns`；`type_, serialized = self.serde.dumps_typed(checkpoint)`——整份 checkpoint 序列化成单个 `(type, bytes)`（`:420`）。
2. 元数据经 `get_checkpoint_metadata` 净化后 `json.dumps(ensure_ascii=False).encode("utf-8","ignore")`（`:421-423`）。
3. `with self.cursor() as cur`：`INSERT OR REPLACE INTO checkpoints(...) VALUES(?,?,?,?,?,?,?)` 写入 (thread, ns, cp_id, parent, type, checkpoint, metadata)（`:424-436`）。
4. 返回带新 checkpoint_id 的 config（`:437-442`）。

**游标获取 `cursor`（`__init__.py:169`）：**
1. `with self.lock` 串行化访问（SQLite 单连接写）；先 `self.setup()` 确保表存在（`:181-182`）。
2. `cur = self.conn.cursor()`，`yield cur`；finally 中若 `transaction` 则 `self.conn.commit()` 并 close（`:183-189`）。

**读链 `get_tuple`（`__init__.py:191`）**：按 config 指定/最新 checkpoint_id 查 checkpoints 表，`type` 列决定 `serde.loads_typed((type, checkpoint))` 还原 checkpoint，再 join checkpoint_writes 还原 pending_writes，组装 CheckpointTuple。

**与 Postgres 差异**：SQLite 把整份 checkpoint 当一个 typed blob 存（`type` 列 + `checkpoint` 列），不做 Postgres 那种"基础值进 JSONB、大对象进 blob 表"的拆分。

## 4. 配置项

| flag / option | 默认 / 行为 | 位置 |
|---|---|---|
| `conn` 构造参数 | `sqlite3.Connection`；`:memory:` 或文件路径 | `__init__.py:85` |
| `from_conn_string` | 同步上下文工厂；异步版 `AsyncSqliteSaver.from_conn_string` | `__init__.py:99`、`aio.py:133` |
| `cursor(transaction=)` | 默认 True，退出时 commit；False 不提交 | `__init__.py:176` |
| serde | 默认 `JsonPlusSerializer` | `__init__.py:24` |
| TTL sweeper | `SqliteStore.start_ttl_sweeper` 后台线程 | `store/sqlite/base.py:1145` |
| 过滤键白名单 | 仅 `[a-zA-Z0-9_.-]`，否则 `ValueError` | `utils.py:12`、`:19` |

## 5. 错误与重试语义

- **SQL 注入防护**：过滤键不匹配白名单抛 `ValueError`（`utils.py:19-23`）；WHERE 条件走参数化占位符，不拼接用户值。
- **建表幂等**：`setup()` 用 `CREATE TABLE IF NOT EXISTS`/迁移表，重复调用安全（`__init__.py:129`）。
- **事务提交**：`cursor(transaction=True)` 退出 commit；异常时连接回滚由 sqlite3 默认行为处理。
- **反序列化**：`type` 列决定 serde 分派；未知 type 抛 `NotImplementedError`（见 serde 叶子）。
- **无重试**：本地 SQLite 无网络 IO，无退避重试。

## 6. 并发细节

- **锁**：`self.lock = threading.Lock()` 保护单 `sqlite3.Connection` 的 cursor 访问（`__init__.py:181`）——SQLite 单连接写需串行。
- **同步/异步双轨**：`SqliteSaver` 用标准库 `sqlite3`；`AsyncSqliteSaver` 用 `aiosqlite`（异步连接）。两者共享 `_delta.py` 的 SQL 构建与 `utils.py` 的谓词生成。
- **TTL sweeper 线程**：`SqliteStore.start_ttl_sweeper` 后台线程；`stop_ttl_sweeper(timeout)` 优雅停；`__del__` 兜底（`store/sqlite/base.py:1145`、`:1203`、`:1231`）。
- **取消传播**：异步版在 aiosqlite 连接上跑，随 asyncio 取消传播。

## 7. 系统边界

**In-Scope（本仓库源码内）**
- `libs/checkpoint-sqlite/langgraph/checkpoint/sqlite/`、`store/sqlite/`、`cache/sqlite/`。

**Out-of-Scope（不在本仓库源码内）**
- SQLite 数据库文件/服务（本地文件或 `:memory:`）。
- `aiosqlite`：异步 SQLite 封装（PyPI 可选依赖，异步后端才用）。
- 序列化器（serde 叶子）、`BaseCheckpointSaver`/`BaseStore`/`BaseCache` 契约（checkpoint-base / store-cache-base 叶子）。

## 8. 与相邻子系统交互

- **上游 → 本叶子**：Pregel 引擎经 `BaseCheckpointSaver` 契约调用；`SqliteStore`/`SqliteCache` 经 `BaseStore`/`BaseCache` 契约被 prebuilt/langgraph 调用。
- **本叶子 → 下游**：经 `self.serde`（serde 叶子）把 checkpoint 编解码成 `(type, bytes)`；metadata 用标准库 `json`。
- **本叶子 → 外部**：同步走 `sqlite3` 标准库；异步走 `aiosqlite`。
- **相邻**：与 Postgres 后端实现同一契约但策略不同（整 blob vs JSONB 拆分）；`_delta.py` 与 Postgres 的 `base.py` 增量 SQL 对称。

## 9. 语言专项适配口径

- **双轨实现（Fast/Slow 式）**：同步 `sqlite3` + 异步 `aiosqlite` 两套后端，共享 SQL 构建与谓词工具——与 Postgres 叶子同构的双轨。
- **可选依赖懒加载**：`sqlite3` 是标准库（同步必装）；`aiosqlite` 是异步 extras 依赖——主包不强制。
- **安全/配置驱动**：`_validate_filter_key` 白名单 + 参数化查询是安全编程面；`transaction` 标志控制是否提交。
- **图型侧重**：architecture 表达"同步/异步双轨 + Store/Cache + sqlite3/aiosqlite 外部"；sequence 表达 put 的单事务写链。整 blob 写入无独立 ETL 管道，不补 dataflow；无单实体状态机，不补 lifecycle。

## 10. 图表清单与质量

| 图 | 文件 | 类型 | archify 质量档 |
|---|---|---|---|
| SQLite 后端架构图 | `checkpoint-sqlite-architecture.html` | architecture | standard |
| put 写链时序图 | `checkpoint-sqlite-sequence.html` | sequence | showcase |

降档披露：架构图落 **standard**（原因：升级 archify 的 `composition/desktop-readability` 检查对 9 组件+9 连线密度敏感）。时序图一次过 **showcase**。两图 render 退出码均 0、HTML 均 >700KB。

JSON IR 源文件位于 `json/` 目录。

省略说明：本叶子不补 dataflow/lifecycle 图——checkpoint 整 blob 写入已被 sequence 覆盖；无单对象状态机。
