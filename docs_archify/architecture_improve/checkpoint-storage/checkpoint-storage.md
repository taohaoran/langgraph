# checkpoint-storage（检查点与存储）域总览

> 本域包含以下叶子子系统；各叶子详情见对应文档。
> 源码基准：langgraph monorepo main 分支，commit `e539ac12`；主语言纯 Python（libs/checkpoint、libs/checkpoint-postgres、libs/checkpoint-sqlite）。

## 1. 域职责

checkpoint-storage 域承载 LangGraph 的持久化层（`libs/checkpoint*`）：定义 checkpoint 存取契约、
序列化协议、多后端实现（内存/Postgres/SQLite）与长期记忆（store/cache），并提供可移植一致性测试套件。
覆盖 6 个叶子：

- **checkpoint-base**：`BaseCheckpointSaver` 契约与 `InMemorySaver` 参考实现；
- **序列化器族**（本轮新增）：`serde/*` 的 jsonplus/msgpack/加密序列化协议；
- **一致性套件**：`tests/` 下可移植到任意后端的 conformance 测试；
- **Postgres 后端**：checkpoint/store 的 async/sync 双轨 Postgres 实现；
- **SQLite 后端**：checkpoint/store/cache 的 SQLite 实现；
- **store/cache 基座**：长期记忆（store）与缓存（cache）的基类与内存/Redis 实现。

## 2. 叶子索引

| 叶子 | 文档 | 架构图 | 第二图 | 职责一句话 |
|------|------|--------|--------|-----------|
| checkpoint-base | [checkpoint-base.md](checkpoint-base/checkpoint-base.md) | [架构图](checkpoint-base/checkpoint-base-architecture.html) | [时序图](checkpoint-base/checkpoint-base-sequence.html) | Saver 契约与内存实现 |
| checkpoint-serde（新增） | [checkpoint-serde.md](checkpoint-serde/checkpoint-serde.md) | [架构图](checkpoint-serde/checkpoint-serde-architecture.html) | [数据流图](checkpoint-serde/checkpoint-serde-dataflow.html) | 序列化协议族 |
| checkpoint-conformance | [checkpoint-conformance.md](checkpoint-conformance/checkpoint-conformance.md) | [架构图](checkpoint-conformance/checkpoint-conformance-architecture.html) | [工作流图](checkpoint-conformance/checkpoint-conformance-workflow.html) | 可移植一致性测试 |
| checkpoint-postgres | [checkpoint-postgres.md](checkpoint-postgres/checkpoint-postgres.md) | [架构图](checkpoint-postgres/checkpoint-postgres-architecture.html) | [时序图](checkpoint-postgres/checkpoint-postgres-sequence.html) | Postgres 双轨后端 |
| checkpoint-sqlite | [checkpoint-sqlite.md](checkpoint-sqlite/checkpoint-sqlite.md) | [架构图](checkpoint-sqlite/checkpoint-sqlite-architecture.html) | [时序图](checkpoint-sqlite/checkpoint-sqlite-sequence.html) | SQLite 后端 |
| store-cache-base | [store-cache-base.md](store-cache-base/store-cache-base.md) | [架构图](store-cache-base/store-cache-base-architecture.html) | [时序图](store-cache-base/store-cache-base-sequence.html) | 长期记忆与缓存基座 |

## 3. 域级机制细节

- **契约分层**：`BaseCheckpointSaver`（抽象基类定契约）→ 内存/Postgres/SQLite 实现方；
  `Serializer` 接口（`serde/base.py`）与 jsonplus/msgpack/加密实现；store/cache 同样以
  `BaseStore`/`BaseCache` 基类约束实现方——典型"能力缝 + 注册表"结构。
- **配置驱动静态分派**：序列化协议按 checkpoint 构造时注入（`put` 时用哪个 serializer 由构造参数决定），
  Postgres/SQLite 的 async/sync 双轨（`aio`/`base` 两套实现）运行时选择——纯 Python 双轨实现模式。
- **一致性保证**：conformance 套件把同一组语义测试跑在任意后端上（内存为参照），保证各后端
  `put/get_tuple` 行为一致；`_delta.py` 支持增量写入（pending writes）。
- **外部边界**：PostgreSQL/SQLite 数据库与驱动、Redis、加密依赖（cryptography）均不在本仓库源码内。

## 4. 域级图（均 standard 档，render 退出码 0）

![checkpoint-storage 域架构图](checkpoint-storage-architecture.html)

![checkpoint-storage 域时序图](checkpoint-storage-sequence.html)

![checkpoint-storage 域数据流图](checkpoint-storage-dataflow.html)

> 本域未生成 workflow 图：持久化层无多角色分步流程语义（写入/恢复由数据流与时序图表达），按资源节省原则省略。
