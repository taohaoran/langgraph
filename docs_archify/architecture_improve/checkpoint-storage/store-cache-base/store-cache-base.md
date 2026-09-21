# store-cache-base：键值存储与缓存并行抽象（store-cache-base）

> 本文是 `checkpoint-storage` 域下的叶子子系统文档。域级总览见 `../checkpoint-storage.md`。
> 本文只展开 `BaseStore`（跨线程长期记忆）与 `BaseCache`（按键缓存）两条并行抽象及其
> 内存/Redis 参考实现，不展开 Postgres/SQLite 后端（见 `../checkpoint-postgres/`、
> `../checkpoint-sqlite/`）。
>
> 源码基准：`langgraph-checkpoint`（Python），目录 `libs/checkpoint/langgraph/store/` 与
> `libs/checkpoint/langgraph/cache/`。

## 1. 功能清单

| 能力 | 说明 | 源码路径 |
| --- | --- | --- |
| 长期记忆存储抽象 | `BaseStore`（ABC），仅 `batch`/`abatch` 两个抽象方法，其余便捷方法构造 Op 后委派 | `store/base/__init__.py:708` |
| 操作对象族 | `GetOp`/`SearchOp`/`PutOp`/`ListNamespacesOp`，`Op` 联合类型，`Result` 联合类型 | `store/base/__init__.py:157`、`:203`、`:431`、`:368`、`:537` |
| 存储条目模型 | `Item`（slots：value/key/namespace/时间戳）、`SearchItem`（追加 score） | `store/base/__init__.py:51`、`:118` |
| 索引与 TTL 配置 | `IndexConfig`（dims/embed/fields）、`TTLConfig`（refresh_on_read/omit_expired/default_ttl/sweep_interval） | `store/base/__init__.py:578`、`:545` |
| 命名空间校验 | 非空、标签为字符串、不含 `.`、根标签不为 `langgraph` | `store/base/__init__.py:1263` |
| 后台合批存储 | `AsyncBatchedBaseStore`：后台任务消费队列、同 tick 合批、按 (ns,key) 去重 | `store/base/batch.py:58` |
| 内存存储实现 | `InMemoryStore`：dict 存储 + 可选余弦相似度向量检索（numpy 可选） | `store/memory/__init__.py:136` |
| 嵌入函数适配 | `ensure_embeddings` 把同步/异步函数或供应商字符串包装为 LangChain `Embeddings` | `store/base/embed.py:34` |
| 文本路径抽取 | `get_text_at_path`/`tokenize_path`，支持 `[*]`/`{a,b}`/嵌套路径 | `store/base/embed.py:233`、`:336` |
| 缓存抽象 | `BaseCache[ValueT]`（ABC）：`get`/`set`/`clear` 同步+异步，TTL 元组 | `cache/base/__init__.py:15` |
| 内存缓存实现 | `InMemoryCache`：`RLock` 保护、读时惰性过期删除 | `cache/memory/__init__.py:11` |
| Redis 缓存实现 | `RedisCache`：MGET 批量读、pipeline+setex 写、keys 模式删，失败静默 | `cache/redis/__init__.py:10` |

## 2. 核心类型与接口清单

| 类型 / 函数 | 位置 | 职责 |
| --- | --- | --- |
| `BaseStore` (ABC) | `store/base/__init__.py:708` | 能力缝：唯一契约是 `batch`/`abatch`；`supports_ttl`/`ttl_config` 类属性 |
| `AsyncBatchedBaseStore` | `store/base/batch.py:58` | 持有事件循环、`asyncio.Queue` 与后台 task；同步方法经 `run_coroutine_threadsafe` 桥接 |
| `InMemoryStore` | `store/memory/__init__.py:136` | `_data` 与 `_vectors` 双层 dict；`_cosine_similarity` numpy 缺失时纯 Python 兜底 |
| `ensure_embeddings` / `EmbeddingsLambda` | `store/base/embed.py:34`/`:109` | 把任意可调用对象适配为 LangChain `Embeddings` |
| `BaseCache[ValueT]` (ABC) | `cache/base/__init__.py:15` | 能力缝：`FullKey=(namespace,key)` 维度批量 get/set/clear，持 `serde` |
| `InMemoryCache` / `RedisCache` | `cache/memory/__init__.py:11` / `cache/redis/__init__.py:10` | 同一 `BaseCache` 契约的内存与 Redis 后端 |
| `_dedupe_ops` | `store/base/batch.py:283` | 读 Op 去重、写 Op 同 key 后写覆盖前写，并把结果映射回原顺序 |
| `_run`（后台协程） | `store/base/batch.py:326` | 同 tick 合批 → `_dedupe_ops` → `abatch` → 回填 Future；store 用 weakref 避免泄漏 |

## 3. 关键调用链

**便捷读（`BaseStore.get`，`store/base/__init__.py:756`）**

1. 把 `(namespace, key, refresh_ttl)` 包成 `GetOp`。
2. 调 `self.batch([GetOp(...)])[0]` 返回首项；异步 `aget` 走 `abatch`。

**后台合批（`AsyncBatchedBaseStore`，`store/base/batch.py`）**

1. 异步方法把 `(future, op)` 放入 `_aqueue` 并 `await future`。
2. 后台 `_run` 协程取到首个 op 后，用 `get_nowait` 把同一 tick 内排队的 op 全部凑齐。
3. `_dedupe_ops` 去重（读按相等性去重、写按 (ns,key) 后写覆盖），一次性 `await abatch(dedupped)`。
4. 用 `listen` 索引把结果映射回各原始 future；任一 future 已取消则跳过。

**内存向量检索（`InMemoryStore.batch`，`store/memory/__init__.py:206`）**

1. `_prepare_ops` 分派 Get/Search/ListNamespaces/Put。
2. 对 SearchOp：`_filter_items` 按 namespace 前缀 + JSONB 风格 filter 过滤，有 query 则
   `ThreadPoolExecutor`（同步）或 `asyncio.gather`（异步）批量嵌入查询。
3. `_batch_search` 算余弦相似度、按分数排序、按 (ns,key) 做 max-pooling 去重、offset/limit 分页。
4. 对 PutOp：`_extract_texts` 按配置路径抽取文本 → `embed_documents` → 写 `_vectors`。

## 4. 配置项

| 配置 | 默认 / 行为 | 位置 |
| --- | --- | --- |
| `IndexConfig.dims` / `embed` / `fields` | 未配置则不支持向量检索；`fields=["$"]` 嵌整个 JSON | `store/base/__init__.py:578` |
| `PutOp.index` | `None` 用 store 默认、`False` 不索引、列表指定字段路径 | `store/base/__init__.py:490` |
| `PutOp.ttl`（分钟） | `NOT_PROVIDED` 时取 `TTLConfig.default_ttl`；`BaseStore` 默认不支持 TTL | `store/base/__init__.py:526`、`:920` |
| `TTLConfig.refresh_on_read` / `omit_expired` / `sweep_interval_minutes` | 默认刷新 TTL、不省略过期、不自动清扫 | `store/base/__init__.py:545` |
| `BaseCache.serde` | 默认 `JsonPlusSerializer(pickle_fallback=False)` | `cache/base/__init__.py:18` |
| `RedisCache.prefix` | 默认 `"langgraph:cache:"` | `cache/redis/__init__.py:18` |
| `SearchOp.filter` 操作符 | `$eq/$ne/$gt/$gte/$lt/$lte`，对齐 Postgres JSONB 语义 | `store/base/__init__.py:255` |

## 5. 错误与重试语义

- `BaseStore.put`/`aput` 对不支持 TTL 的实现显式抛 `NotImplementedError`（`store/base/__init__.py:920`）。
- 命名空间非法（空标签、含 `.`、根为 `langgraph`）抛 `InvalidNamespaceError`。
- `RedisCache` 采用**失败静默**：`mget` 异常返回 `{}`、`pipe.execute` 异常吞掉、`keys` 异常吞掉
  （`cache/redis/__init__.py:63`、`:106`、`:137`），即 Redis 不可用时缓存视为未命中，不阻断主流程。
- `InMemoryCache` 读时发现过期条目即惰性删除；无后台清扫线程。
- 无网络重试/退避；异步合批失败时同一批所有 future 统一 `set_exception`。

## 6. 并发细节

- `AsyncBatchedBaseStore` 在构造时捕获 `_loop = asyncio.get_running_loop()`，后台 task 跑在该循环；
  `_check_loop` 装饰器禁止在同一事件循环里调用同步方法（会死锁），提示改用异步接口
  （`store/base/batch.py:33`）。store 实例用 `weakref.ref` 传入后台协程，避免阻止 GC；
  `__del__` 取消后台 task。
- `InMemoryCache` 用 `threading.RLock` 保护 dict，可重入；`InMemoryStore` 无锁（单线程假设）。
- 嵌入查询在同步路径用 `ThreadPoolExecutor` 并发嵌入多个查询（`store/memory/__init__.py:277`），
  异步路径用 `asyncio.gather`。
- 无 `context` 传递；同步/异步为两套入口，合批发生在异步侧。

## 7. 系统边界

**In-Scope（本仓库源码内）**

- `BaseStore`/`AsyncBatchedBaseStore` 契约与合批去重机制
- `InMemoryStore` 内存实现与纯 Python 余弦相似度兜底
- `ensure_embeddings`/`EmbeddingsLambda`/路径抽取工具
- `BaseCache` 契约与 `InMemoryCache`/`RedisCache` 实现

**Out-of-Scope（不在本仓库源码内）**

- Postgres/SQLite 持久化 store 与 cache 实现：见 `checkpoint-postgres`、`checkpoint-sqlite`
- LangChain `Embeddings` 接口、OpenAI 等嵌入模型供应商 SDK
- `redis-py` 客户端与 Redis 服务器
- `numpy`（可选加速）

## 8. 与相邻子系统交互

- 上游：应用/Agent 经 `store.get/search/put` 存取跨线程长期记忆，经 `cache.get/set` 缓存 LLM/工具结果。
- 下游：`InMemoryStore` 经 `ensure_embeddings` 调用外部嵌入模型；`RedisCache` 经注入的 redis 客户端访问 Redis。
- 横向：`BaseCache.serde` 复用 checkpoint 包的 `JsonPlusSerializer`（`cache/base/__init__.py:8`）；
  `JsonPlusSerializer` 反向可序列化 `store.base.Item`（见 `checkpoint-base` 叶子）。

## 9. 语言专项适配口径

纯 Python 框架库，按「能力缝 + 注册表/工厂 + 多后端矩阵」口径：

1. **能力缝归组**：`BaseStore` 与 `BaseCache` 是两个 ABC 契约层；`BaseStore` 用「Op 命令对象 +
   单一 batch 入口」把多种操作统一为可合批/可去重的管道，是典型命令模式能力缝。
2. **配置驱动**：向量检索是否启用、嵌入哪些字段、TTL 行为均由构造时 `IndexConfig`/`TTLConfig`
   与每次 `put` 的 `index`/`ttl` 参数决定，同一实现经不同配置切换行为。
3. **多后端矩阵**：`BaseStore` 由 InMemory/Postgres/SQLite 实现，`BaseCache` 由 InMemory/Redis/
   SQLite 实现，契约一致性由 `checkpoint-conformance` 套件强制。
4. **双轨实现**：`_cosine_similarity` 有 numpy 高性能实现与纯 Python 兜底实现，运行时按是否安装
   numpy 自动选择（`store/memory/__init__.py:493`）。
5. **图类型**：以 architecture 图表达「双契约层 + 实现矩阵 + 合批机制」；合批/检索流程已在第 3 节
   文字化，跨叶子的后端存储读写时序由 postgres/sqlite 叶子补充。

## 10. 图表清单与质量

| 图 | 文件 | 类型 | archify 质量档 |
| --- | --- | --- | --- |
| 双抽象与实现矩阵架构图 | `store-cache-base-architecture.html` | architecture | showcase |
| Store/Cache 命中与回退调用时序 | `store-cache-base-sequence.html` | sequence | showcase |

JSON IR 源文件：`json/store-cache-base-architecture.json`、`json/store-cache-base-sequence.json`。
第二张 sequence 图补充基线 architecture 未表达的时序语义：`AsyncBatchedBaseStore`（`store/base/batch.py:58`）
在读取时先经 `BaseCache` 命中、未命中再入队 `abatch`（`store/base/__init__.py:745`）回退到底层实现；
写入时同 tick 的 `PutOp` 经合批去重后一次性 `_run`（`store/base/batch.py:326`）落库。后端持久化视角
的时序仍由 `checkpoint-postgres`/`checkpoint-sqlite` 叶子承担。
