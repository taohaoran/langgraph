# core-runtime（核心运行时）域总览

> 本域包含以下叶子子系统；各叶子详情见对应文档。
> 源码基准：langgraph monorepo main 分支，commit `e539ac12`；主语言纯 Python（libs/langgraph/langgraph）。

## 1. 域职责

core-runtime 是 LangGraph 的执行内核（`libs/langgraph/langgraph`），负责把"图定义"编译成可执行的
状态机、按超级步（superstep）驱动节点执行、维护通道（channels）状态、在每一步之间通过 checkpoint
持久化，并向上游 prebuilt/sdk 暴露 `invoke`/`stream`/`ainvoke` 等入口。本域覆盖 9 个叶子：

- **Pregel 执行引擎**：图编译 → 超级步循环 → 任务执行（含并行、重试、超时）→ 状态读写的完整链路；
- **IO/Checkpoint 桥接**：状态读写与 checkpoint 落盘之间的字节边界；
- **图/状态 API**：`StateGraph`/`MessageGraph` 等图定义语言与状态模式；
- **通道原语**（本轮新增）：`channels/*` 九类通道承载状态变更语义；
- **托管值**：跨超级步缓存计算结果的 `ManagedValue` 机制；
- **函数 API**：`@entrypoint`/`@task` 装饰器把普通函数提升为图执行单元；
- **流框架**：`stream/*` 流式迭代器与消息复用（mux）机制；
- **远端与调试**：`runtime.py`/`remote.py`/`debug.py` 远程图与调试辅助；
- **内部基础设施**：`_internal/*`、`utils/*`、`config.py` 等支撑模块。

## 2. 叶子索引

| 叶子 | 文档 | 架构图 | 第二图 | 职责一句话 |
|------|------|--------|--------|-----------|
| pregel-engine | [pregel-engine.md](pregel-engine/pregel-engine.md) | [架构图](pregel-engine/pregel-engine-architecture.html) | [生命周期图](pregel-engine/pregel-engine-lifecycle.html) · [时序图](pregel-engine/pregel-engine-sequence.html) | 超级步执行循环与任务调度 |
| pregel-io-checkpoint | [pregel-io-checkpoint.md](pregel-io-checkpoint/pregel-io-checkpoint.md) | [架构图](pregel-io-checkpoint/pregel-io-checkpoint-architecture.html) | [数据流图](pregel-io-checkpoint/pregel-io-checkpoint-dataflow.html) | 状态读写与 checkpoint 字节桥接 |
| graph-state-api | [graph-state-api.md](graph-state-api/graph-state-api.md) | [架构图](graph-state-api/graph-state-api-architecture.html) | [时序图](graph-state-api/graph-state-api-sequence.html) | 图定义语言与状态模式 |
| channels（新增） | [channels.md](channels/channels.md) | [架构图](channels/channels-architecture.html) | [生命周期图](channels/channels-lifecycle.html) | 九类通道状态变更原语 |
| managed-values | [managed-values.md](managed-values/managed-values.md) | [架构图](managed-values/managed-values-architecture.html) | [生命周期图](managed-values/managed-values-lifecycle.html) · [时序图](managed-values/managed-values-sequence.html) | 跨超级步缓存托管值 |
| func-api | [func-api.md](func-api/func-api.md) | [架构图](func-api/func-api-architecture.html) | [生命周期图](func-api/func-api-lifecycle.html) · [时序图](func-api/func-api-sequence.html) | entrypoint/task 函数提升 |
| stream-framework | [stream-framework.md](stream-framework/stream-framework.md) | [架构图](stream-framework/stream-framework-architecture.html) | [数据流图](stream-framework/stream-framework-dataflow.html) | 流式迭代与消息复用 |
| remote-debug | [remote-debug.md](remote-debug/remote-debug.md) | [架构图](remote-debug/remote-debug-architecture.html) | [时序图](remote-debug/remote-debug-sequence.html) | 远程图与调试辅助 |
| internal-foundation | [internal-foundation.md](internal-foundation/internal-foundation.md) | [架构图](internal-foundation/internal-foundation-architecture.html) | [数据流图](internal-foundation/internal-foundation-dataflow.html) | 内部基础设施与工具 |

## 3. 域级机制细节

- **超级步状态机**：`pregel/main.py` 的 `Pregel.invoke/astream` 将图编译为内部管道，按
  `_algo.py:223` 等处的任务生成逻辑逐个超级步推进；每步先读 checkpoint、跑任务、再写 checkpoint，
  通道更新经 `_io.py` 归并。状态机隐藏在循环里——本域补了 pregel-engine-lifecycle 图。
- **通道语义族**：`channels/*` 提供 `LastValue`（覆盖写）、`Topic`（追加+去重游标）、
  `NamedBarrierValue`（条件合并）、`BinOp`（归约）等九类原语，`Send/Command` 通过 `graph/_node.py`
  的 `_get_channels` 注入读写。
- **能力缝归组（纯 Python 口径）**：抽象基类定契约（如 `Channel`、`ManagedValue`、`StreamProtocol`），
  实现方按语义注册（通道按类型、stream 按消息类型分发）；`func-api` 用装饰器（`@entrypoint`/`@task`）
  把函数提升为执行单元，属"装饰器与元类扩展点"能力缝。
- **双轨实现**：`astream`/`astate` 异步族与 `stream`/`state` 同步族共用同一套 `stream/_convert.py`
  适配器；`_runner` 区分 sync/async 执行器，取消经 `context` 传播。
- **外部边界**：checkpoint 持久化、LLM 调用、远端平台均不在本域源码内（见各叶子第 7 节）。

## 4. 域级图（均 standard 档，render 退出码 0）

![core-runtime 域架构图](core-runtime-architecture.html)

![core-runtime 域时序图](core-runtime-sequence.html)

![core-runtime 域数据流图](core-runtime-dataflow.html)

> 本域未生成 workflow 图：核心运行时无多角色分步流程语义（执行循环由 lifecycle 表达），与已有图信息重复，按资源节省原则省略。
