# LangGraph 架构分析文档（迭代改进版）

> 对 langgraph monorepo（commit `e539ac1`）的深度架构分析，按"系统级 → 域 → 叶子子系统"三级拆分，共 5 域 22 叶，产出设计文档级 MD 与 archify 交互式 HTML 图。所有输出物为简体中文。
>
> **本版本为迭代改进版**，相对[基线版本](../architecture/README.md)新增 15 张域级图、20 张叶子补图、1 张系统级生命周期图，详见 [improve-comparison.md](improve-comparison.md)。

## 系统级文档

| 文档 | 说明 |
|---|---|
| [系统级总览](system-overview.md) | 功能总览、解决的问题、系统边界、架构/时序/数据流/生命周期图说明 |
| [系统架构图](system-architecture.html) | 五层分层架构与依赖方向 |
| [核心执行时序图](system-sequence.html) | 一次 graph.invoke/stream 调用的完整生命周期 |
| [数据流图](system-dataflow.html) | 输入→编译→执行→持久化→输出流水线 |
| [执行生命周期图](system-lifecycle.html) | 图执行状态机（编译→恢复→超级步→流式输出→完成/中断） |
| [系统级 JSON IR](json/) | 四张系统图的 archify JSON 源文件 |

## 域与叶子索引

### 1. core-runtime（核心运行时）— [域总览](core-runtime/core-runtime.md) · [域架构图](core-runtime/core-runtime-architecture.html) · [域时序图](core-runtime/core-runtime-sequence.html) · [域数据流图](core-runtime/core-runtime-workflow.html)

libs/langgraph/langgraph，78 源文件 / 27951 行。框架核心：图构建、Pregel 执行引擎、流式输出、函数式 API。

| 叶子 | 文档 | 图 | 说明 |
|---|---|---|---|
| graph-state-api | [MD](core-runtime/graph-state-api/graph-state-api.md) | [架构](core-runtime/graph-state-api/graph-state-api-architecture.html) · [时序](core-runtime/graph-state-api/graph-state-api-sequence.html) | StateGraph/MessageGraph、Channels 状态通道 |
| pregel-engine | [MD](core-runtime/pregel-engine/pregel-engine.md) | [架构](core-runtime/pregel-engine/pregel-engine-architecture.html) · [生命周期](core-runtime/pregel-engine/pregel-engine-lifecycle.html) | Pregel BSP 超级步循环、并发调度 |
| pregel-io-checkpoint | [MD](core-runtime/pregel-io-checkpoint/pregel-io-checkpoint.md) | [架构](core-runtime/pregel-io-checkpoint/pregel-io-checkpoint-architecture.html) · [数据流](core-runtime/pregel-io-checkpoint/pregel-io-checkpoint-dataflow.html) | checkpoint 读写、输入输出合并、消息管理 |
| stream-framework | [MD](core-runtime/stream-framework/stream-framework.md) | [架构](core-runtime/stream-framework/stream-framework-architecture.html) · [数据流](core-runtime/stream-framework/stream-framework-dataflow.html) · [时序](core-runtime/stream-framework/stream-framework-sequence.html) | 多模式流、转换器、多路复用 |
| func-api | [MD](core-runtime/func-api/func-api.md) | [架构](core-runtime/func-api/func-api-architecture.html) · [时序](core-runtime/func-api/func-api-sequence.html) | create_agent、@entrypoint/@task |
| managed-values | [MD](core-runtime/managed-values/managed-values.md) | [架构](core-runtime/managed-values/managed-values-architecture.html) · [生命周期](core-runtime/managed-values/managed-values-lifecycle.html) | ManagedValue 抽象、IsLastStep |
| internal-foundation | [MD](core-runtime/internal-foundation/internal-foundation.md) | [架构](core-runtime/internal-foundation/internal-foundation-architecture.html) · [时序](core-runtime/internal-foundation/internal-foundation-sequence.html) | Runnable/serde/retry/config 基础设施 |
| remote-debug | [MD](core-runtime/remote-debug/remote-debug.md) | [架构](core-runtime/remote-debug/remote-debug-architecture.html) · [时序](core-runtime/remote-debug/remote-debug-sequence.html) | RemoteGraph 远端代理、调试、图绘制 |

### 2. checkpoint-storage（持久化存储）— [域总览](checkpoint-storage/checkpoint-storage.md) · [域架构图](checkpoint-storage/checkpoint-storage-architecture.html) · [域时序图](checkpoint-storage/checkpoint-storage-sequence.html) · [域数据流图](checkpoint-storage/checkpoint-storage-dataflow.html)

libs/checkpoint*，Checkpoint/Store/Cache 三套抽象 × Memory/Postgres/SQLite 多后端 + 一致性测试套件。

| 叶子 | 文档 | 图 | 说明 |
|---|---|---|---|
| checkpoint-base | [MD](checkpoint-storage/checkpoint-base/checkpoint-base.md) | [架构](checkpoint-storage/checkpoint-base/checkpoint-base-architecture.html) · [时序](checkpoint-storage/checkpoint-base/checkpoint-base-sequence.html) | BaseCheckpointSaver、MemorySaver、serde |
| store-cache-base | [MD](checkpoint-storage/store-cache-base/store-cache-base.md) | [架构](checkpoint-storage/store-cache-base/store-cache-base-architecture.html) · [时序](checkpoint-storage/store-cache-base/store-cache-base-sequence.html) | BaseStore、BaseCache、MemoryCache、RedisCache |
| checkpoint-postgres | [MD](checkpoint-storage/checkpoint-postgres/checkpoint-postgres.md) | [架构](checkpoint-storage/checkpoint-postgres/checkpoint-postgres-architecture.html) · [时序](checkpoint-storage/checkpoint-postgres/checkpoint-postgres-sequence.html) | AsyncPostgresSaver、PostgresStore |
| checkpoint-sqlite | [MD](checkpoint-storage/checkpoint-sqlite/checkpoint-sqlite.md) | [架构](checkpoint-storage/checkpoint-sqlite/checkpoint-sqlite-architecture.html) · [时序](checkpoint-storage/checkpoint-sqlite/checkpoint-sqlite-sequence.html) | SqliteSaver、SqliteStore、SqliteCache |
| checkpoint-conformance | [MD](checkpoint-storage/checkpoint-conformance/checkpoint-conformance.md) | [架构](checkpoint-storage/checkpoint-conformance/checkpoint-conformance-architecture.html) · [流程](checkpoint-storage/checkpoint-conformance/checkpoint-conformance-workflow.html) | 可移植一致性测试套件 |

### 3. prebuilt-agents（预构建智能体）— [域总览](prebuilt-agents/prebuilt-agents.md) · [域架构图](prebuilt-agents/prebuilt-agents-architecture.html) · [域时序图](prebuilt-agents/prebuilt-agents-sequence.html) · [域生命周期图](prebuilt-agents/prebuilt-agents-lifecycle.html)

libs/prebuilt，开箱即用的 ReAct Agent、工具执行节点、人机中断机制。

| 叶子 | 文档 | 图 | 说明 |
|---|---|---|---|
| chat-agent-executor | [MD](prebuilt-agents/chat-agent-executor/chat-agent-executor.md) | [生命周期](prebuilt-agents/chat-agent-executor/chat-agent-executor-lifecycle.html) · [时序](prebuilt-agents/chat-agent-executor/chat-agent-executor-sequence.html) | create_react_agent、ReAct 循环 |
| tool-node | [MD](prebuilt-agents/tool-node/tool-node.md) | [架构](prebuilt-agents/tool-node/tool-node-architecture.html) · [时序](prebuilt-agents/tool-node/tool-node-sequence.html) | ToolNode、InjectedState、工具校验 |
| interrupt-command | [MD](prebuilt-agents/interrupt-command/interrupt-command.md) | [架构](prebuilt-agents/interrupt-command/interrupt-command-architecture.html) · [时序](prebuilt-agents/interrupt-command/interrupt-command-sequence.html) · [生命周期](prebuilt-agents/interrupt-command/interrupt-command-lifecycle.html) | interrupt()、Command 人机交互 |

### 4. cli-tooling（CLI 工具链）— [域总览](cli-tooling/cli-tooling.md) · [域架构图](cli-tooling/cli-tooling-architecture.html) · [域时序图](cli-tooling/cli-tooling-sequence.html) · [域流程图](cli-tooling/cli-tooling-workflow.html)

libs/cli，命令行工具：本地开发、部署、Docker 打包。

| 叶子 | 文档 | 图 | 说明 |
|---|---|---|---|
| cli-core | [MD](cli-tooling/cli-core/cli-core.md) | [架构](cli-tooling/cli-core/cli-core-architecture.html) · [流程](cli-tooling/cli-core/cli-core-workflow.html) | CLI 入口、命令分发、配置加载 |
| cli-deploy-exec | [MD](cli-tooling/cli-deploy-exec/cli-deploy-exec.md) | [架构](cli-tooling/cli-deploy-exec/cli-deploy-exec-architecture.html) · [时序](cli-tooling/cli-deploy-exec/cli-deploy-exec-sequence.html) | deploy/exec/archive 命令 |
| docker-packaging | [MD](cli-tooling/docker-packaging/docker-packaging.md) | [架构](cli-tooling/docker-packaging/docker-packaging-architecture.html) · [流程](cli-tooling/docker-packaging/docker-packaging-workflow.html) | Docker 镜像构建、依赖追踪、uv 锁 |

### 5. sdk-python（Python SDK）— [域总览](sdk-python/sdk-python.md) · [域架构图](sdk-python/sdk-python-architecture.html) · [域时序图](sdk-python/sdk-python-sequence.html) · [域数据流图](sdk-python/sdk-python-dataflow.html)

libs/sdk-py，与 LangGraph Server API 交互的 Python 客户端，异步/同步双轨。

| 叶子 | 文档 | 图 | 说明 |
|---|---|---|---|
| sdk-client | [MD](sdk-python/sdk-client/sdk-client.md) | [架构](sdk-python/sdk-client/sdk-client-architecture.html) · [时序](sdk-python/sdk-client/sdk-client-sequence.html) | Client 主类、资源客户端、_async/_sync 双轨 |
| sdk-stream | [MD](sdk-python/sdk-stream/sdk-stream.md) | [架构](sdk-python/sdk-stream/sdk-stream-architecture.html) · [数据流](sdk-python/sdk-stream/sdk-stream-dataflow.html) · [时序](sdk-python/sdk-stream/sdk-stream-sequence.html) | 流式控制器、SSE 解码、多游标缓冲 |
| sdk-auth-encryption | [MD](sdk-python/sdk-auth-encryption/sdk-auth-encryption.md) | [架构](sdk-python/sdk-auth-encryption/sdk-auth-encryption-architecture.html) · [时序](sdk-python/sdk-auth-encryption/sdk-auth-encryption-sequence.html) | 认证、加密、HTTP 缓存、错误类型 |

## 统计

- 域：5 个
- 叶子子系统：22 个
- 设计文档级 MD：22 叶 + 5 域总览 + 1 系统总览 + 1 改进对比 + 本 README = 30 份
- 交互式 HTML 图：66 张（系统级 4 + 域级 15 + 叶子级 47）
- archify JSON IR：66 份
- 源码覆盖：约 214 个源文件 / 7.75 万行（已排除 tests/bench/examples）

### 图型分布

| 层级 | architecture | sequence | dataflow | workflow | lifecycle | 合计 |
|---|---|---|---|---|---|---|
| 系统级 | 1 | 1 | 1 | 0 | 1 | 4 |
| 域级 | 5 | 5 | 2 | 2 | 1 | 15 |
| 叶子级 | 22 | 14 | 3 | 3 | 5 | 47 |
| 合计 | 28 | 20 | 6 | 5 | 7 | 66 |

## 外部边界声明

以下组件不在本仓库源码内，在各叶子文档的"系统边界"小节中逐一标注：
- LLM 提供商（OpenAI/Anthropic 等）、LangSmith 可观测性平台、LangGraph Platform 远端运行时
- Postgres / SQLite / Redis 数据库与驱动
- `libs/sdk-js` 仅为 README 占位，JS/TS SDK 在外部仓库 `langchain-ai/langgraphjs`
