# LangGraph 系统架构文档（第二轮迭代分析）

> 基于 LangGraph monorepo 源码（纯 Python 主语言，commit `e539ac12`，langgraph 1.2.11 / checkpoint 4.2.0 /
> prebuilt 1.1.0，≈225 py 文件 / 7.76 万行）深度分析产出，覆盖系统级、5 个域、26 个叶子子系统的功能、
> 问题域、系统边界、架构图、时序图、数据流图、工作流图与生命周期图。所有图表由 archify 渲染为
> 自包含交互式 HTML（共 75 张，render 退出码全 0）。
>
> **迭代说明**：本目录为第二轮迭代分析产物（对比目录）。第一轮基线位于
> [`docs_archify/architecture/`](../architecture/)（只读未改动），两轮差异见
> [improve-comparison.md](improve-comparison.md)。

## 文档导航

### 系统级

| 文档 | 说明 | 图表 |
|------|------|------|
| [system-overview.md](system-overview.md) | 功能总览、解决的问题、系统边界、核心代码映射 | [系统架构图](system-architecture.html) · [时序图](system-sequence.html) · [数据流图](system-dataflow.html) |

### core-runtime（core-runtime/）

| 子系统 | 文档 | 架构图 | 第二/三图 |
|--------|------|--------|----------|
| 域总览 | [core-runtime.md](core-runtime/core-runtime.md) | [域架构图](core-runtime/core-runtime-architecture.html) | [域时序图](core-runtime/core-runtime-sequence.html) · [域数据流图](core-runtime/core-runtime-dataflow.html) |
| pregel-engine | [pregel-engine.md](core-runtime/pregel-engine/pregel-engine.md) | [架构图](core-runtime/pregel-engine/pregel-engine-architecture.html) | [生命周期图](core-runtime/pregel-engine/pregel-engine-lifecycle.html) · [时序图](core-runtime/pregel-engine/pregel-engine-sequence.html) |
| pregel-io-checkpoint | [pregel-io-checkpoint.md](core-runtime/pregel-io-checkpoint/pregel-io-checkpoint.md) | [架构图](core-runtime/pregel-io-checkpoint/pregel-io-checkpoint-architecture.html) | [数据流图](core-runtime/pregel-io-checkpoint/pregel-io-checkpoint-dataflow.html) |
| graph-state-api | [graph-state-api.md](core-runtime/graph-state-api/graph-state-api.md) | [架构图](core-runtime/graph-state-api/graph-state-api-architecture.html) | [时序图](core-runtime/graph-state-api/graph-state-api-sequence.html) |
| channels（新增） | [channels.md](core-runtime/channels/channels.md) | [架构图](core-runtime/channels/channels-architecture.html) | [生命周期图](core-runtime/channels/channels-lifecycle.html) |
| managed-values | [managed-values.md](core-runtime/managed-values/managed-values.md) | [架构图](core-runtime/managed-values/managed-values-architecture.html) | [生命周期图](core-runtime/managed-values/managed-values-lifecycle.html) · [时序图](core-runtime/managed-values/managed-values-sequence.html) |
| func-api | [func-api.md](core-runtime/func-api/func-api.md) | [架构图](core-runtime/func-api/func-api-architecture.html) | [生命周期图](core-runtime/func-api/func-api-lifecycle.html) · [时序图](core-runtime/func-api/func-api-sequence.html) |
| stream-framework | [stream-framework.md](core-runtime/stream-framework/stream-framework.md) | [架构图](core-runtime/stream-framework/stream-framework-architecture.html) | [数据流图](core-runtime/stream-framework/stream-framework-dataflow.html) |
| remote-debug | [remote-debug.md](core-runtime/remote-debug/remote-debug.md) | [架构图](core-runtime/remote-debug/remote-debug-architecture.html) | [时序图](core-runtime/remote-debug/remote-debug-sequence.html) |
| internal-foundation | [internal-foundation.md](core-runtime/internal-foundation/internal-foundation.md) | [架构图](core-runtime/internal-foundation/internal-foundation-architecture.html) | [数据流图](core-runtime/internal-foundation/internal-foundation-dataflow.html) |

### checkpoint-storage（checkpoint-storage/）

| 子系统 | 文档 | 架构图 | 第二图 |
|--------|------|--------|--------|
| 域总览 | [checkpoint-storage.md](checkpoint-storage/checkpoint-storage.md) | [域架构图](checkpoint-storage/checkpoint-storage-architecture.html) | [域时序图](checkpoint-storage/checkpoint-storage-sequence.html) · [域数据流图](checkpoint-storage/checkpoint-storage-dataflow.html) |
| checkpoint-base | [checkpoint-base.md](checkpoint-storage/checkpoint-base/checkpoint-base.md) | [架构图](checkpoint-storage/checkpoint-base/checkpoint-base-architecture.html) | [时序图](checkpoint-storage/checkpoint-base/checkpoint-base-sequence.html) |
| checkpoint-serde（新增） | [checkpoint-serde.md](checkpoint-storage/checkpoint-serde/checkpoint-serde.md) | [架构图](checkpoint-storage/checkpoint-serde/checkpoint-serde-architecture.html) | [数据流图](checkpoint-storage/checkpoint-serde/checkpoint-serde-dataflow.html) |
| checkpoint-conformance | [checkpoint-conformance.md](checkpoint-storage/checkpoint-conformance/checkpoint-conformance.md) | [架构图](checkpoint-storage/checkpoint-conformance/checkpoint-conformance-architecture.html) | [工作流图](checkpoint-storage/checkpoint-conformance/checkpoint-conformance-workflow.html) |
| checkpoint-postgres | [checkpoint-postgres.md](checkpoint-storage/checkpoint-postgres/checkpoint-postgres.md) | [架构图](checkpoint-storage/checkpoint-postgres/checkpoint-postgres-architecture.html) | [时序图](checkpoint-storage/checkpoint-postgres/checkpoint-postgres-sequence.html) |
| checkpoint-sqlite | [checkpoint-sqlite.md](checkpoint-storage/checkpoint-sqlite/checkpoint-sqlite.md) | [架构图](checkpoint-storage/checkpoint-sqlite/checkpoint-sqlite-architecture.html) | [时序图](checkpoint-storage/checkpoint-sqlite/checkpoint-sqlite-sequence.html) |
| store-cache-base | [store-cache-base.md](checkpoint-storage/store-cache-base/store-cache-base.md) | [架构图](checkpoint-storage/store-cache-base/store-cache-base-architecture.html) | [时序图](checkpoint-storage/store-cache-base/store-cache-base-sequence.html) |

### prebuilt-agents（prebuilt-agents/）

| 子系统 | 文档 | 架构图 | 第二/三图 |
|--------|------|--------|----------|
| 域总览 | [prebuilt-agents.md](prebuilt-agents/prebuilt-agents.md) | [域架构图](prebuilt-agents/prebuilt-agents-architecture.html) | [域时序图](prebuilt-agents/prebuilt-agents-sequence.html) · [域生命周期图](prebuilt-agents/prebuilt-agents-lifecycle.html) |
| chat-agent-executor | [chat-agent-executor.md](prebuilt-agents/chat-agent-executor/chat-agent-executor.md) | [架构图](prebuilt-agents/chat-agent-executor/chat-agent-executor-architecture.html) | [生命周期图](prebuilt-agents/chat-agent-executor/chat-agent-executor-lifecycle.html) · [时序图](prebuilt-agents/chat-agent-executor/chat-agent-executor-sequence.html) |
| tool-node | [tool-node.md](prebuilt-agents/tool-node/tool-node.md) | [架构图](prebuilt-agents/tool-node/tool-node-architecture.html) | [时序图](prebuilt-agents/tool-node/tool-node-sequence.html) |
| interrupt-command | [interrupt-command.md](prebuilt-agents/interrupt-command/interrupt-command.md) | [架构图](prebuilt-agents/interrupt-command/interrupt-command-architecture.html) | [生命周期图](prebuilt-agents/interrupt-command/interrupt-command-lifecycle.html) · [时序图](prebuilt-agents/interrupt-command/interrupt-command-sequence.html) |

### cli-tooling（cli-tooling/）

| 子系统 | 文档 | 架构图 | 第二图 |
|--------|------|--------|--------|
| 域总览 | [cli-tooling.md](cli-tooling/cli-tooling.md) | [域架构图](cli-tooling/cli-tooling-architecture.html) | [域时序图](cli-tooling/cli-tooling-sequence.html) · [域工作流图](cli-tooling/cli-tooling-workflow.html) |
| cli-core | [cli-core.md](cli-tooling/cli-core/cli-core.md) | [架构图](cli-tooling/cli-core/cli-core-architecture.html) | [工作流图](cli-tooling/cli-core/cli-core-workflow.html) |
| cli-deploy-exec | [cli-deploy-exec.md](cli-tooling/cli-deploy-exec/cli-deploy-exec.md) | [架构图](cli-tooling/cli-deploy-exec/cli-deploy-exec-architecture.html) | [时序图](cli-tooling/cli-deploy-exec/cli-deploy-exec-sequence.html) |
| docker-packaging | [docker-packaging.md](cli-tooling/docker-packaging/docker-packaging.md) | [架构图](cli-tooling/docker-packaging/docker-packaging-architecture.html) | [数据流图](cli-tooling/docker-packaging/docker-packaging-dataflow.html) |

### sdk-python（sdk-python/）

| 子系统 | 文档 | 架构图 | 第二图 |
|--------|------|--------|--------|
| 域总览 | [sdk-python.md](sdk-python/sdk-python.md) | [域架构图](sdk-python/sdk-python-architecture.html) | [域时序图](sdk-python/sdk-python-sequence.html) · [域数据流图](sdk-python/sdk-python-dataflow.html) |
| sdk-client | [sdk-client.md](sdk-python/sdk-client/sdk-client.md) | [架构图](sdk-python/sdk-client/sdk-client-architecture.html) | [时序图](sdk-python/sdk-client/sdk-client-sequence.html) |
| sdk-resources（新增） | [sdk-resources.md](sdk-python/sdk-resources/sdk-resources.md) | [架构图](sdk-python/sdk-resources/sdk-resources-architecture.html) | [时序图](sdk-python/sdk-resources/sdk-resources-sequence.html) |
| sdk-auth-encryption | [sdk-auth-encryption.md](sdk-python/sdk-auth-encryption/sdk-auth-encryption.md) | [架构图](sdk-python/sdk-auth-encryption/sdk-auth-encryption-architecture.html) | [时序图](sdk-python/sdk-auth-encryption/sdk-auth-encryption-sequence.html) |
| sdk-stream | [sdk-stream.md](sdk-python/sdk-stream/sdk-stream.md) | [架构图](sdk-python/sdk-stream/sdk-stream-architecture.html) | [数据流图](sdk-python/sdk-stream/sdk-stream-dataflow.html) |
| sdk-http（新增） | [sdk-http.md](sdk-python/sdk-http/sdk-http.md) | [架构图](sdk-python/sdk-http/sdk-http-architecture.html) | [时序图](sdk-python/sdk-http/sdk-http-sequence.html) |

## 产出统计

| 层级 | MD | HTML | JSON |
|------|-----|------|------|
| 系统级 | 2（system-overview、README）+ improve-comparison | 3 | 3 |
| core-runtime | 10（1 域 + 9 叶） | 25（3 域 + 22 叶） | 25 |
| checkpoint-storage | 7（1 域 + 6 叶） | 15（3 域 + 12 叶） | 15 |
| prebuilt-agents | 4（1 域 + 3 叶） | 12（3 域 + 9 叶） | 12 |
| cli-tooling | 4（1 域 + 3 叶） | 12（3 域 + 9 叶） | 12 |
| sdk-python | 6（1 域 + 5 叶） | 11（3 域 + 8 叶） | 11 |
| **合计** | **34** | **75** | **75** |

> 说明：叶子级 HTML 57 张中，部分叶子含第三张图（生命周期/时序）。HTML 与 JSON 严格一一对应，
> 全部 render 退出码 0、HTML >700KB（自包含交互式）。

## 覆盖范围与说明

- **覆盖**：5 域 26 叶（本轮新增 4 叶：channels、checkpoint-serde、sdk-resources、sdk-http）；
  系统级/域级每级 3 张图、叶子级每叶 ≥2 张图，配额全部满足。
- **质量档位**：叶级 showcase 52/57（其余 5 张 standard，降档原因见各叶第 10 节）；
  域级 showcase 3/15；系统级 standard 3/3（组件多、跨层连接复杂为预期）。
- **语言适配口径（纯 Python）**：能力缝 + 注册表 + 插件体系；双轨实现（sync/async）；
  配置驱动静态分派；可选依赖懒加载；隐藏状态机在循环里（均已显式补图）。
- **未覆盖**：`sdk-js`（仅 README 占位，无源码）；LangGraph Server 平台服务端（不在本仓库源码内）。
- **基线对比**：第一轮基线 `docs_archify/architecture/`（22 叶 30 图）保持只读；本轮改进点见
  [improve-comparison.md](improve-comparison.md)。
