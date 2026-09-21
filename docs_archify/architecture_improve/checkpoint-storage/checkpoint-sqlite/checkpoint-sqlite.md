# checkpoint-sqlite（SQLite 后端实现）

> 本文是 `checkpoint-storage` 域下的叶子子系统文档。域级总览见 `../checkpoint-storage.md`。
> 本文只展开 SQLite 后端的检查点保存（Saver）、键值/向量存储（Store）与缓存（Cache），
> 抽象契约见 `../checkpoint-base/checkpoint-base.md` 与 `../store-cache-base/store-cache-base.md`。
>
> 源码基准：`libs/checkpoint-sqlite`（8 源文件 / 3941 行）。

## 1. 功能清单

| 能力 | 说明 | 源码路径 |
|---|---|---|
| 同步检查点保存 | `SqliteSaver`，轻量同步用例（demo/小项目），单连接 + 锁 | `checkpoint/sqlite/__init__.py:45` |
| 异步检查点保存 | `AsyncSqliteSaver`，基于 `aiosqlite` | `checkpoint/sqlite/aio.py` |
| Delta 通道历史快路 | 两阶段分页遍历（stage1 取祖先 blob、stage2 per-channel `UNION ALL` 写历史） | `checkpoint/sqlite/__init__.py:503`、`_delta.py` |
| 键值/向量存储 | `SqliteStore`，向量检索用 `sqlite-vec` 扩展 | `store/sqlite/base.py` |
| 异步存储 | `AsyncSqliteStore` | `store/sqlite/aio.py` |
| SQLite 缓存 | `SqliteCache`，文件缓存，支持 TTL | `cache/sqlite/__init__.py:13` |

## 2. 核心类型与接口清单

| 类型 / 函数 | 位置 | 职责 |
|---|---|---|
| `SqliteSaver` | `checkpoint/sqlite/__init__.py:45` | 同步 Saver；`cursor()` 上下文管理器持锁 + 自动 commit（`:168`） |
| `AsyncSqliteSaver` | `checkpoint/sqlite/aio.py` | 异步 Saver，要求 `aiosqlite` 包 |
| `SqliteStore` | `store/sqlite/base.py` | `BaseStore` 的 SQLite 实现；`MIGRATIONS`/`VECTOR_MIGRATIONS` 列表即版本号 |
| `SqliteCache` | `cache/sqlite/__init__.py:13` | `BaseCache` 的文件实现，`(ns, key)` 主键 + `expiry` |
| `search_where` | `checkpoint/sqlite/utils.py` | 构造 `list()` 的 WHERE 子句（filter/before） |
| `_namespace_match` | `store/sqlite/base.py:104` | 注册为 SQLite 自定义函数 `_langgraph_namespace_match`，做段感知命名空间匹配 |
| `DELTA_STAGE1_SQL`/`build_delta_stage2_sql`/`step_walk_with_row` | `checkpoint/sqlite/_delta.py` | Delta 通道两阶段重建 SQL 构造与遍历步进 |

## 3. 关键调用链

**写入 `put`**：
1. `serde.dumps_typed(checkpoint)` 序列化整个 checkpoint 为单个 BLOB（SQLite 无 JSONB，不像 Postgres 那样拆分 inline/blob）。
2. `cursor()` 持 `threading.Lock` → `INSERT OR REPLACE INTO checkpoints` → commit（`__init__.py:424`）。
3. `put_writes`：全为特殊通道走 `INSERT OR REPLACE`，否则走 `INSERT OR IGNORE`，`executemany`（`:445`）。

**读取 `get_tuple`**：
1. 按 `checkpoint_id` 是否存在取指定或最新一条 `checkpoints`（`ORDER BY checkpoint_id DESC LIMIT 1`）。
2. 再查 `writes` 组装 `CheckpointTuple`，metadata 用 `json.loads`。

**Delta 通道历史（快路覆盖）**：
1. stage1：`DELTA_STAGE1_SQL` 按 `checkpoint_id < cursor` 分页取祖先 `(checkpoint_id, parent, type, blob)`，在 Python 内逐行 `step_walk_with_row` 判断每通道是否已 seed；全部 seed 或链尽则停。
2. stage2：`build_delta_stage2_sql` 按每通道链长生成 per-channel `UNION ALL` 读 `writes`，最后 `build_delta_channels_writes_history` 组装。
3. 与 Postgres 的差异：SQLite 无 JSONB，stage1 必须取回完整 checkpoint blob 在 Python 解析 `channel_values` 找 seed。

## 4. 配置项

| 配置 | 默认 / 行为 | 位置 |
|---|---|---|
| `conn` | `sqlite3.Connection`，`check_same_thread=False` 由实现加锁保证线程安全 | `__init__.py:72` |
| `PRAGMA journal_mode=WAL` | 启动时设置，改善并发与原子性 | `__init__.py:141`、`cache/sqlite/__init__.py:32` |
| `is_setup` | 幂等建表标志，首次 `cursor()` 时自动 `setup()` | `__init__.py:136` |
| `index`（SqliteIndexConfig） | 不传则关闭语义检索，向量走 `sqlite-vec` | `store/sqlite/base.py:90` |
| `ttl`（TTLConfig） | 配 TTL 后 `start_ttl_sweeper()` 后台线程清理 | `store/sqlite/base.py` |
| `get_next_version` | `{next_v:032}.{random:016}` 字符串版本 | `__init__.py:626` |

## 5. 错误与重试语义

- 同步 Saver 的所有异步方法（`aget_tuple`/`alist`/`aput`）直接 `raise NotImplementedError(_AIO_ERROR_MSG)`，提示改用 `AsyncSqliteSaver`（`__init__.py:585`）。
- 缓存读时惰性过期：命中行若 `expiry < now` 则 `DELETE` 该行并跳过，不抛错（`cache/sqlite/__init__.py:63`）。
- 迁移失败：`executescript` 抛错即向上传播，不自动重试。
- 命名空间匹配：`_namespace_match` 用 `re.search` 实现，未知 match_type 记 warning。

## 6. 并发细节

- `SqliteSaver.lock`（`threading.Lock`，`__init__.py:95`）：`cursor()` 全程持锁，`check_same_thread=False` 下由锁保证线程安全；文档明确"不扩展到多线程高并发"。
- `SqliteCache._lock`（`threading.RLock`，`cache/sqlite/__init__.py:30`）：`with self._lock, self._conn` 串行化访问。
- 异步方法（`aget`/`aset`/`aclear`）用 `asyncio.to_thread` 桥接同步实现（`cache/sqlite/__init__.py:74`）。
- `list()` 遍历用主游标 + 另开 `wcur` 子游标查 writes，避免单游标结果集被覆盖（`__init__.py:342`）。

## 7. 系统边界

**In-Scope（本仓库源码内）**
- `checkpoint/sqlite/`：同步/异步 Saver、Delta 两阶段遍历、搜索 WHERE 构造。
- `store/sqlite/`：`SqliteStore`/`AsyncSqliteStore`、迁移、命名空间匹配自定义函数。
- `cache/sqlite/`：`SqliteCache`。

**Out-of-Scope（不在本仓库源码内）**
- Python 标准库 `sqlite3` 与第三方 `aiosqlite`、`sqlite-vec` 扩展。
- 实际 SQLite 数据库文件与磁盘。
- LangChain `Embeddings` 接口及具体嵌入模型。
- 抽象契约本身见 `checkpoint-base` 叶子。

## 8. 与相邻子系统交互

- 上游：Pregel 经 `BaseCheckpointSaver` 接口；应用代码经 `BaseStore`/`BaseCache` 接口。
- 本叶子 → 下游：所有 SQL 经 `sqlite3`/`aiosqlite` 驱动 → SQLite 数据库；向量检索经 `sqlite-vec` 扩展与 LangChain `Embeddings`。
- 依赖方向：本叶子依赖 `libs/checkpoint` 的抽象与 `serde`，不反向依赖 core。

## 9. 语言专项适配口径

纯 Python 框架**多后端矩阵**口径：`SqliteSaver`/`AsyncSqliteSaver`/`SqliteStore`/`SqliteCache` 是 `checkpoint-base` 与 `store-cache-base` 抽象基类在 SQLite 后端的具体实现，与 Postgres、内存实现同一契约。与 Postgres 后端的关键差异：SQLite 无 JSONB，因此 Delta 通道重建必须取回完整 checkpoint blob 在 Python 内解析（Postgres 走 JSONB 键查找分页）；同步/异步非桥接而是双实现（`aiosqlite`）。图类型以 architecture（组件/连接边界）为主。

## 10. 图表清单与质量

| 图 | 文件 | 类型 | archify 质量档 |
|---|---|---|---|
| SQLite 后端结构 | `checkpoint-sqlite-architecture.html` | architecture | standard（showcase 布局校验有轻微告警，按规则降档） |
| SQLite 事务读写时序 | `checkpoint-sqlite-sequence.html` | sequence | showcase |

JSON IR 源文件：`json/checkpoint-sqlite-architecture.json`、`json/checkpoint-sqlite-sequence.json`。
第二张 sequence 图补充基线 architecture 未表达的时序语义：`SqliteSaver.put`（`checkpoint/sqlite/__init__.py:387`）
经 `cursor(transaction=True)`（`checkpoint/sqlite/__init__.py:169`）开启事务，写入 checkpoints/blob/writes 后在
游标关闭时 `commit()`（`checkpoint/sqlite/__init__.py:188`）；读取 `get_tuple`（`checkpoint/sqlite/__init__.py:191`）
走 `cursor(transaction=False)` 只读路径。与 `checkpoint-postgres` sequence 的差异在于本图突出 SQLite 单连接
+ 游标上下文管理的事务边界，而非 Postgres 的连接池。
