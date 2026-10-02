# store-cache-base（store-cache-base）

> 本文是 `checkpoint-storage` 域下的叶子子系统文档。本叶子是 **键值存储与缓存的抽象层及内存实现**：
> `BaseStore`（跨线程共享的键值/语义搜索存储）及其 `AsyncBatchedBaseStore` 异步批处理装饰、
> `InMemoryStore` 参考实现；`BaseCache` 抽象与 `InMemoryCache`/`RedisCache` 参考实现。
> Postgres/SQLite 的 Store/Cache 后端在各自叶子复用本层契约。
>
> 源码基准：`langgraph-checkpoint`（libs/checkpoint），commit `e539ac12`。

## 1. 功能清单

| 能力 | 说明 | 源码路径 |
|---|---|---|
| Store 抽象基类 | `BaseStore(ABC)`：抽象 `batch`/`abatch`；便利方法 `get`/`search`/`put`/`delete`/`list_namespaces` 及异步版，内部构造 Op 调 batch | `store/base/__init__.py:708`、`:733`、`:745`、`:756`、`:779`、`:856`、`:937`、`:946` |
| 操作值对象 | `Item`/`SearchItem`、`GetOp`/`PutOp`/`SearchOp`/`ListNamespacesOp`、`MatchCondition`、`InvalidNamespaceError` | `store/base/__init__.py:51`、`:118`、`:157`、`:431`、`:203`、`:333`、`:368`、`:541` |
| TTL/索引配置 | `TTLConfig`、`IndexConfig`；`supports_ttl` 标志，子类显式开启 | `store/base/__init__.py:545`、`:578`、`:727` |
| 异步批处理装饰 | `AsyncBatchedBaseStore`：把并发 `aget`/`asearch`/`aput` 合并进后台任务批量 `abatch`，带取消韧性 | `store/base/batch.py:58`、`:82`、`:103`、`:131` |
| 嵌入适配 | `EmbeddingsLambda`：把 embed 函数包装成 `Embeddings` 接口 | `store/base/embed.py:109` |
| 内存 Store 实现 | `InMemoryStore(BaseStore)`：dict 存储 + 向量搜索；`batch`/`abatch`、`_batch_search`、namespace 列表 | `store/memory/__init__.py:136`、`:206`、`:221`、`:302`、`:460` |
| Cache 抽象基类 | `BaseCache(ABC, Generic[ValueT])`：抽象 `get`/`aget`/`set`/`aset`/`clear`/`aclear`；`FullKey=(Namespace,str)`；默认 serde `JsonPlusSerializer(pickle_fallback=False)` | `cache/base/__init__.py:17` |
| 内存 Cache | `InMemoryCache`：RLock 保护的双层 dict，按 TTL 过期；异步委托同步 | `cache/memory/__init__.py:10`、`:16`、`:30`、`:45` |
| Redis Cache | `RedisCache`：包 redis 客户端，prefix 拼 key，mget/pipeline 批量，不可用/损坏静默降级 | `cache/redis/__init__.py:11`、`:38`、`:76`、`:100` |

## 2. 核心类型与接口清单

| 类型 / 函数 | 位置 | 职责 |
|---|---|---|
| `BaseStore(ABC)` | `store/base/__init__.py:708` | Store 契约。唯一抽象方法是 `batch`/`abatch`；其余便利方法默认实现把单操作包成 Op 列表调 batch。 |
| `Item` / `SearchItem` | `store/base/__init__.py:51` / `:118` | 存储条目（namespace/key/value/metadata/创建时间）；SearchItem 额外带相关性 score。 |
| `GetOp`/`PutOp`/`SearchOp`/`ListNamespacesOp` | `store/base/__init__.py:157`/`:431`/`:203`/`:368` | 可批处理的操作值对象；`Op` 是它们的联合。 |
| `AsyncBatchedBaseStore` | `store/base/batch.py:58` | 抽象装饰基类：子类实现 `batch`/`abatch`；它把高频异步读写在后台任务里合并成一次 abatch。 |
| `InMemoryStore` | `store/memory/__init__.py:136` | 内存参考实现，支持向量搜索与 namespace 列表。 |
| `BaseCache[ValueT]` | `cache/base/__init__.py:17` | Cache 契约；`FullKey=(Namespace, key)`；注入 serde。 |
| `InMemoryCache` | `cache/memory/__init__.py:10` | RLock 内存缓存，过期时间戳判断。 |
| `RedisCache` | `cache/redis/__init__.py:11` | Redis 缓存，`_make_key`/`_parse_key` 编解码 namespace，mget/pipeline。 |

## 3. 关键调用链

**Store 便利读链 `get`（`store/base/__init__.py:756`）→ batch：**
1. 调用方 `store.get(namespace, key)` 构造单个 `GetOp`，调抽象 `batch([op])`（`:756-`）。
2. `InMemoryStore.batch`（`store/memory/__init__.py:206`）把 ops 按类型分发：GET 直接查内存 dict，PUT 写 dict，SEARCH 走 `_batch_search`（`:302`）。
3. 同步 list/namespace 经 `_handle_list_namespaces`（`:460`）按前缀/深度过滤。

**异步批处理 `AsyncBatchedBaseStore.aget`（`batch.py:82`）：**
1. 并发 `aget` 调用被 `_ensure_task`（`:77`）收拢进同一后台 asyncio task；多个请求合并为一个 `abatch`。
2. 任务取消时优雅恢复（对应测试 `test_async_batch_store_resilience`）；结果按 Op 顺序返回。

**Cache 读链 `RedisCache.get`（`cache/redis/__init__.py:38`）：**
1. 把 `FullKey` 列表经 `_make_key` 拼成 `prefix:ns:key` 的 Redis key（`:44-48`、`:18-21`）。
2. `self.redis.mget(redis_keys)`；抛异常（Redis 不可用）返回空 dict（`:50-54`）。
3. 逐结果按 `encoding:data` 拆前缀，`serde.loads_typed` 还原；损坏条目 `continue` 跳过（`:56-66`）。

**Cache 写链 `RedisCache.set`（`cache/redis/__init__.py:76`）：**
1. 开 `redis.pipeline()`；逐条目 `serde.dumps_typed(value)` 拼成 `encoding:data`；有 ttl 用 `setex`，否则 `set`（`:82-93`）。
2. `pipe.execute()`；异常静默吞掉（Redis 不可用不影响主流程）（`:95-98`）。

## 4. 配置项

| flag / option | 默认 / 行为 | 位置 |
|---|---|---|
| `supports_ttl` | `False`；子类显式置 True 才支持 TTL | `store/base/__init__.py:727` |
| `ttl_config` | `TTLConfig` TypedDict；控制过期清扫 | `store/base/__init__.py:545`、`:728` |
| `index`（InMemoryStore 构造） | `IndexConfig | None`；不配则禁用语义搜索 | `store/memory/__init__.py:183` |
| `serde`（BaseCache 构造） | 默认 `JsonPlusSerializer(pickle_fallback=False)` | `cache/base/__init__.py:24`、`:27` |
| `prefix`（RedisCache） | 默认 `"langgraph:cache:"`；所有 key 前缀 | `cache/redis/__init__.py:16`、`:40` |
| `refresh_ttl` | `get` 参数；命中时是否刷新 TTL | `store/base/__init__.py:761` |

## 5. 错误与重试语义

- **namespace 校验**：`_validate_namespace`（`store/base/__init__.py:1263`）拒绝空/非法 namespace，抛 `InvalidNamespaceError`（`:541`）。
- **Redis 不可用降级**：`RedisCache.get`/`set`/`clear` 对异常 `try/except` 静默降级——读返回空 dict、写/清理静默跳过（`cache/redis/__init__.py:50-54`、`:95-98`、`:118-122`）。缓存是性能优化，不可用不应阻断主流程。
- **损坏数据容错**：Redis 值 `split(b":",1)` 失败时 `continue` 跳过损坏条目（`:61-65`）。
- **缓存过期**：`InMemoryCache.get` 比对 `now < expiry`，过期即删（`cache/memory/__init__.py:27-30`）。
- **无重试**：本地缓存/内存存储无网络 IO；Redis 失败即降级，不重试。

## 6. 并发细节

- **RLock**：`InMemoryCache._lock = threading.RLock()` 保护双层 dict 的 get/set/clear（`cache/memory/__init__.py:16`、`:19`）。
- **异步批处理**：`AsyncBatchedBaseStore` 用后台 asyncio task 收拢并发 `a*` 调用为一次 `abatch`，减少往返；`__del__` 清理任务（`batch.py:63`、`:70`、`:77`）。
- **同步/异步双轨**：`BaseStore`/`BaseCache` 同时给同步 `batch`/`get` 与异步 `abatch`/`aget`；内存实现的异步方法直接委托同步（无 IO），Redis 客户端本身可同步或异步。
- **pipeline**：`RedisCache.set` 用 `redis.pipeline()` 批量执行，减往返（`cache/redis/__init__.py:82`）。
- **取消传播**：`AsyncBatchedBaseStore` 显式处理 asyncio task 取消，不残留脏合并。

## 7. 系统边界

**In-Scope（本仓库源码内）**
- `libs/checkpoint/langgraph/store/base/`、`store/memory/`、`cache/base/`、`cache/memory/`、`cache/redis/`。

**Out-of-Scope（不在本仓库源码内）**
- `redis-py` Redis 客户端（`RedisCache` 接收外部注入的 client，本层不构造连接）。
- Postgres/SQLite 的 Store/Cache 后端（见 postgres/sqlite 叶子）。
- 序列化器（serde 叶子）、Embeddings 模型外部服务。

## 8. 与相邻子系统交互

- **上游 → 本叶子**：prebuilt/langgraph 经 `BaseStore`/`BaseCache` 契约存取跨线程记忆与缓存；`FullKey` 由调用方指定 namespace+key。
- **本叶子 → 下游**：Cache 值经 `self.serde`（serde 叶子）编解码；RedisCache 经注入的 redis 客户端访问 Redis 服务端。
- **本叶子 → 外部**：Redis 缓存访问外部 Redis；向量搜索经 `EmbeddingsLambda` 调外部嵌入模型。
- **相邻**：Postgres/SQLite Store 后端复用 `BaseStore` 的 Op 值对象与查询构建；本层定义契约，后端落地。

## 9. 语言专项适配口径

- **能力缝归组**：`BaseStore`/`BaseCache` 两个抽象基类定义"键值存储"与"缓存"两条能力缝；`AsyncBatchedBaseStore` 是装饰器式能力缝——给任意 store 加异步批合并。
- **注册表/分派**：`batch(ops)` 按 Op 类型（Get/Put/Search/ListNamespaces）分派处理——操作值对象联合 + 类型分派。
- **双轨实现**：同步 `batch`/`get` + 异步 `abatch`/`aget`；内存实现异步委托同步，Redis 客户端可同步或异步。
- **可选依赖/懒加载**：`redis-py` 不在本层构造，由调用方注入 client——缓存后端可插拔；`EmbeddingsLambda` 包装函数为接口，避免强依赖具体嵌入库。
- **图型侧重**：architecture 表达"Store/Cache 双契约族 + 内存实现 + Redis 外部"；sequence 表达 cache 命中读链。无数据 ETL 管道与单实体状态机，不补 dataflow/lifecycle。

## 10. 图表清单与质量

| 图 | 文件 | 类型 | archify 质量档 |
|---|---|---|---|
| Store/Cache 架构图 | `store-cache-base-architecture.html` | architecture | standard |
| cache 命中读链时序图 | `store-cache-base-sequence.html` | sequence | showcase |

降档披露：架构图落 **standard**（原因：升级 archify 的 `composition/desktop-readability` 检查对 9 组件+9 连线密度敏感）。时序图一次过 **showcase**。两图 render 退出码均 0、HTML 均 >700KB。

JSON IR 源文件位于 `json/` 目录。

省略说明：本叶子不补 dataflow/lifecycle 图——store/cache 是键值 CRUD，无 ETL 管道；缓存条目无单对象状态机（过期判断已在 MD 第 5 节表达）。
