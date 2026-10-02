# checkpoint-conformance（checkpoint-conformance）

> 本文是 `checkpoint-storage` 域下的叶子子系统文档。本叶子是 `langgraph-checkpoint` 库自带的
> **测试与一致性验证套件**：覆盖 InMemorySaver、JsonPlus/加密序列化、BaseStore/InMemoryStore、
> RedisCache 的单元测试，以及对接外部可移植一致性套件 `langgraph.checkpoint.conformance`
> （以 `importorskip` 可选引入）的集成测试。它是各后端（Postgres/SQLite）自检 `BaseCheckpointSaver`
> 契约的事实标准。
>
> 源码基准：`langgraph-checkpoint`（libs/checkpoint/tests），commit `e539ac12`。

## 1. 功能清单

| 能力 | 说明 | 源码路径 |
|---|---|---|
| 内存检查点测试 | `TestMemorySaver`：combined metadata、search/asearch；`TestInMemorySaverDeltaChannel` 与 `TestBaseFallbackGetChannelWrites`：增量通道祖先链回溯、种子边界、并发任务隔离 | `tests/test_memory.py:35`、`:324`、`:424`、`:563` |
| 告警/白名单测试 | 未注册 msgpack 类型告警、allowlist 静默告警、strict 模式阻断、`with_allowlist` 代理隔离 | `tests/test_memory.py:222`、`:246`、`:272`、`:297` |
| 序列化往返测试 | `test_jsonplus.py`（1232 行）：pydantic v1/v2、dataclass、Enum、numpy、datetime、UUID、set/deque 等类型的 dumps/loads 往返 | `tests/test_jsonplus.py:47-103` |
| 加密序列化测试 | `test_encrypted.py`（444 行）：`EncryptedSerializer` 加解密往返、未加密透传、cipher name 拼接 | `tests/test_encrypted.py` |
| Store 接口测试 | `MockAsyncBatchedStore` 验证 `AsyncBatchedBaseStore` 韧性/取消/去重；`get_text_at_path`；namespace 列表/通配/分页/max_depth；向量插入/更新/搜索/分页/并发 | `tests/test_store.py:26`、`:38`、`:244`、`:601`、`:733` |
| Redis 缓存测试 | `TestRedisCache`：set/get、批量、TTL、命名空间隔离、clear by namespace、空操作、异步、Redis 不可用降级、损坏数据、大对象 | `tests/test_redis_cache.py:12` |
| 可移植一致性集成 | `test_delta_channel_conformance`：用 `@checkpointer_test` 装饰器把 `InMemorySaver` 包成 fixture，调外部 `validate(...)` 校验 `delta_channel_history` 能力 | `tests/test_conformance_delta.py:14-37` |
| 测试嵌入工具 | `CharacterEmbeddings`：确定性字符频次+随机投影假 embedding，供 store 向量测试复用 | `tests/embed_test_utils.py:11` |

## 2. 核心类型与接口清单

| 类型 / 函数 | 位置 | 职责 |
|---|---|---|
| `@checkpointer_test` 装饰器 | `langgraph.checkpoint.conformance.initializer`（外部可选包） | 把一个异步 fixture 生成器包装成一致性套件可驱动的 checkpointer 工厂 |
| `validate(checkpointer, capabilities=...)` | `langgraph.checkpoint.conformance`（外部可选包） | 入口：对 checkpointer 跑指定能力集的一致性用例，返回 `report.results[cap] -> {passed, failures}` |
| `pytest.importorskip("langgraph.checkpoint.conformance")` | `tests/test_conformance_delta.py:7` | 可选依赖守卫：未装该包则整文件 skip，而非报错 |
| `MockAsyncBatchedStore` | `tests/test_store.py:26` | 继承 `AsyncBatchedBaseStore`，内部委托 `InMemoryStore`，用于测批量合并/取消韧性 |
| `CharacterEmbeddings` | `tests/embed_test_utils.py:11` | 确定性假 embedding（固定 seed），避免测试依赖外部向量模型 |
| `TestRedisCache.setup` fixture | `tests/test_redis_cache.py:13` | 连 `localhost:6379`，`ping` 失败 `pytest.skip`；每用例前后 `flushdb` 清理 |

## 3. 关键调用链

**可移植一致性集成（`test_conformance_delta.py:14`）：**
1. 模块级 `pytest.importorskip("langgraph.checkpoint.conformance", reason=...)`——包未安装时整个测试文件 skip（`:7-10`）。
2. 测试函数内延迟 import `validate` 与 `checkpointer_test`（注释明确：必须在 importorskip 之后 import，否则安全（`:16-20`）。
3. 用 `@checkpointer_test(name="InMemorySaver")` 装饰 `async def mem_saver()` 生成器（`yield InMemorySaver()`）（`:24-26`）。
4. 调 `await validate(mem_saver, capabilities={"delta_channel_history"})` 拿到 report；遍历 `report.results`，任一 `passed is False` 则 `pytest.fail` 列出 failures（`:28-37`）。

**Store 批量韧性测试（`test_store.py:38`）：**
1. `MockAsyncBatchedStore` 内部持有 `InMemoryStore`，`batch`/`abatch` 直接委托（`test_store.py:26-35`）。
2. 测试注入 doc 后断言 `AsyncBatchedBaseStore` 在 task 取消时能优雅恢复、不泄漏、不重复合并（`:38-72`）。

**Redis 缓存测试夹具（`test_redis_cache.py:13`）：**
1. `setup` 连 `redis.Redis(host=localhost, port=6379, db=0)`；`ping` 抛 `ConnectionError` 时 `pytest.skip`（`:16-22`）。
2. 构造 `RedisCache(client, prefix="test:cache:")`，`flushdb` 清理；`teardown_method` 再 `flushdb`（`:24-33`）。
3. 后续用例覆盖正常路径与 `test_redis_unavailable_*`（连接失败降级）、`test_corrupted_data_handling`（损坏字节容错）。

## 4. 配置项

| flag / option | 默认 / 行为 | 位置 |
|---|---|---|
| `langgraph.checkpoint.conformance` | 可选独立包；未装时 `test_conformance_delta.py` 整体 skip | `test_conformance_delta.py:7` |
| Redis 连接 | `localhost:6379` db0；不可用则 `pytest.skip` | `test_redis_cache.py:16-22` |
| `CharacterEmbeddings(dims=500, seed=42)` | 确定性投影维度与随机种子 | `embed_test_utils.py:14` |
| `capabilities={"delta_channel_history"}` | 本次一致性校验开启的能力集（可扩） | `test_conformance_delta.py:30-32` |
| `LANGGRAPH_STRICT_MSGPACK` | 影响 test_memory 白名单/strict 用例的预期行为 | `test_memory.py:272` |

## 5. 错误与重试语义

- **可选依赖缺失**：`importorskip` 优雅 skip（非失败）；Redis 不可用 `pytest.skip`——测试套件在无外部服务环境下仍可跑通其余用例。
- **一致性失败即失败**：`validate` 返回的任一 capability `passed is False` 时，测试函数 `pytest.fail` 并打印 failures 明细（`test_conformance_delta.py:34-37`），不静默放过。
- **损坏数据容错**：`test_corrupted_data_handling` 验证 Redis 缓存遇到损坏字节时不崩溃、返回未命中而非抛异常（`test_redis_cache.py:268`）。
- **取消韧性**：`test_async_batch_store_resilience` / `test_async_batch_store_handles_cancellation` 断言批量存储在 asyncio task 取消后可恢复、无脏数据（`test_store.py:38`、`:207`）。
- **无网络重试**：测试本身不重试；被观测的实现层（Redis 不可用）由用例显式断言降级路径。

## 6. 并发细节

- **asyncio 用例**：大量 `async def test_*` 用 `@pytest.mark.asyncio` 跑，覆盖 `a*` 异步后端路径（search/asearch、async batch、async vector search）。
- **并发安全用例**：`test_async_batched_vector_search_concurrent`（`test_store.py:811`）、`test_async_fallback_concurrent_tasks_do_not_interfere`（`test_memory.py:519`）——断言多并发任务访问同一 store/saver 不串数据。
- **夹具隔离**：Redis 每用例 `flushdb`；内存 saver 用例各自新建实例；无跨用例共享可变状态。
- **锁/取消传播**：测试重点验证实现层在 `asyncio.CancelledError` 下的行为（批量合并任务取消不残留），而非测试自身加锁。

## 7. 系统边界

**In-Scope（本仓库源码内）**
- `libs/checkpoint/tests/`：上述 7 个测试文件与嵌入工具。

**Out-of-Scope（不在本仓库源码内）**
- `langgraph-checkpoint-conformance`：可移植一致性套件本体（`validate`/`checkpointer_test`），是独立 PyPI 包，本仓库仅 `importorskip` 集成。
- Redis 服务端（localhost:6379）、Postgres/SQLite 测试数据库——外部进程。
- 被测实现本体：InMemorySaver（checkpoint-base 叶子）、JsonPlusSerializer（serde 叶子）、BaseStore/RedisCache（store-cache-base 叶子）。

## 8. 与相邻子系统交互

- **本叶子 → 上游实现**：测试直接 import 并驱动 `InMemorySaver`、`JsonPlusSerializer`、`EncryptedSerializer`、`InMemoryStore`、`RedisCache`——是 checkpoint-storage 域各叶子的验收方。
- **本叶子 → 外部一致性包**：`test_conformance_delta.py` 把 `InMemorySaver` 作为 checkpointer 工厂交给 `validate`，由外部包驱动跨后端同一份用例（Postgres/SQLite 后端在各自 tests/ 里用同样的 `validate` 入口）。
- **本叶子 → 外部服务**：Redis 用例连本地 Redis；向量用例用 `CharacterEmbeddings` 替代真实 embedding 模型。

## 9. 语言专项适配口径

- **可选依赖懒加载**：`pytest.importorskip("langgraph.checkpoint.conformance")` 是典型的"缺失即 skip 而非 import 即崩"——把可移植一致性套件做成可选 extras，主测试套件不强依赖它。
- **测试夹具即工厂**：`@checkpointer_test` 把生成器 fixture 包装成 checkpointer 工厂；`CharacterEmbeddings` 是测试侧的 fake 实现（实现 `Embeddings` Protocol），替代外部真实向量模型——测试替身模式。
- **双轨覆盖**：每个 store/saver 行为都同时有同步（`test_*`）与异步（`test_a*`/`async def test_*`）用例，与被测实现的同步/异步双轨对应。
- **图型侧重**：architecture 表达"测试文件 + 夹具 + 外部一致性包 + 外部服务"的组件关系；workflow 表达"夹具 setup → 驱动后端 → 断言"的用例执行流程（带泳道）。无数据流管道与单实体状态机，故不补 dataflow/lifecycle。

## 10. 图表清单与质量

| 图 | 文件 | 类型 | archify 质量档 |
|---|---|---|---|
| 测试套件组件架构图 | `checkpoint-conformance-architecture.html` | architecture | standard |
| 用例执行流程图 | `checkpoint-conformance-workflow.html` | workflow | showcase |

降档披露：架构图落 **standard**（原因：升级 archify 的 `composition/desktop-readability` 检查对 10 组件+9 连线密度敏感）。工作流图一次过 **showcase**：升级后 workflow schema 变更——需 `schema_version:2`、节点带 `col`(0-5) 与组件类型（无 start/end 节点）、并省略 `viewBox` 让编译器自动量画布；已按诊断逐轮修正（列间距/节点宽度/viewBox）。两图 render 退出码均 0、HTML 均 >700KB。

JSON IR 源文件位于 `json/` 目录。

省略说明：本叶子不补 dataflow/lifecycle 图——测试套件无数据 ETL 管道、被测实体无单对象状态机；测试执行流程已由 workflow 泳道图表达。
