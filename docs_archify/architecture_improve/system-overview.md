# LangGraph 系统级总览

> 本文档对 langgraph monorepo（commit `e539ac1`）进行系统级架构分析，覆盖 5 个域、22 个叶子子系统。所有结论均可追溯至源码。

## 一、功能总览

LangGraph 是一个用于构建**有状态、多角色智能体**的纯 Python 框架。其核心价值在于：以声明式图（StateGraph）定义智能体的控制流，以 Pregel BSP（Bulk Synchronous Parallel）模型驱动执行，以 checkpoint 机制保障可恢复性，并通过多模式流式输出实时反馈执行过程。

| 能力域 | 核心功能 | 代表模块 |
|---|---|---|
| 图构建与状态 | StateGraph/MessageGraph 声明式建图、Channels 状态通道、Topic 广播 | `langgraph/graph/`、`langgraph/channels/` |
| 执行引擎 | Pregel 超级步循环、并发节点调度、重试与校验 | `langgraph/pregel/main.py`、`_loop.py`、`_algo.py` |
| 流式输出 | values/updates/debug/messages/custom 多流模式、流转换器、多路复用 | `langgraph/stream/` |
| 持久化 | Checkpoint/Store/Cache 三套抽象 × Memory/Postgres/SQLite 多后端 | `libs/checkpoint*` |
| 预构建 Agent | ReAct Agent 工厂、ToolNode 工具执行、interrupt/Command 人机交互 | `libs/prebuilt/` |
| 函数式 API | `create_agent`、`@entrypoint`/`@task` 装饰器 | `langgraph/func/` |
| 客户端与工具链 | Python SDK（异步/同步双轨）、CLI 部署与 Docker 打包 | `libs/sdk-py/`、`libs/cli/` |
| 远端与调试 | RemoteGraph 远端代理、调试模式、图可视化绘制 | `langgraph/pregel/remote.py`、`debug.py` |

## 二、解决的问题

1. **智能体控制流的可组合性**：传统 LLM 应用将控制流硬编码在 Python 循环中，难以复用和可视化。LangGraph 用 StateGraph 将节点、边、条件分支声明式组合，编译为可执行的 Pregel 图。
2. **有状态执行与可恢复性**：长运行智能体需要持久化中间状态以支持崩溃恢复、人机交互中断和时间旅行调试。Checkpoint 抽象层在每个超级步结束时持久化完整状态，支持 Memory/Postgres/SQLite 多后端。
3. **并发与确定性**：Pregel BSP 模型将执行划分为超级步，每步内所有就绪节点并发执行、步间同步合并，避免了共享状态竞态，同时保证执行确定性。
4. **实时可观测性**：多模式流式输出（values/updates/debug/messages/custom）允许调用方按需获取完整状态、增量更新、执行轨迹或 token 级消息流。
5. **多后端部署矩阵**：同一套 Checkpoint/Store/Cache 抽象下，Memory 用于开发、Postgres 用于生产、SQLite 用于轻量部署，一致性测试套件（checkpoint-conformance）保障各后端行为一致。

## 三、系统边界

### 上边界（调用方）
- 用户应用通过 `graph.invoke()` / `graph.stream()` 或 Python SDK 调用图执行
- CLI 工具用于本地开发、部署到 LangGraph Platform、Docker 镜像打包
- 预构建 API（`create_react_agent`）提供开箱即用的 ReAct 智能体

### 下边界（依赖与外部系统）
- **LLM 提供商**（OpenAI、Anthropic 等）：不在本仓库源码内，通过 LangChain 或直接 HTTP 调用
- **LangSmith**：可观测性平台，不在本仓库源码内，通过回调上报追踪
- **LangGraph Platform**：远端部署运行时，不在本仓库源码内，SDK 通过 REST API 交互
- **Postgres / SQLite / Redis**：数据库与驱动不在本仓库源码内，checkpoint 后端仅提供 SQL 与连接管理
- **langgraphjs**：JS/TS SDK 已迁移至外部仓库 `langchain-ai/langgraphjs`，本仓库 `libs/sdk-js` 仅为 README 占位

### 内边界（仓库内库间依赖）
```
checkpoint ──→ checkpoint-postgres / checkpoint-sqlite ──→ prebuilt ──→ langgraph
sdk-py ──→ langgraph + cli
cli ──→ sdk-py
```
核心库 `langgraph` 不依赖 prebuilt/checkpoint 等上层库，依赖方向单向清晰。

### 侧边界（不做什么）
- 不提供 LLM 模型本身，仅编排模型调用
- 不提供向量数据库内核，Store 抽象支持嵌入检索但存储后端由外部提供
- 不提供 Web UI，图可视化仅生成 Mermaid/文本描述
- 不包含训练或微调能力
- `libs/sdk-js` 无实际源码，JS 客户端在外部仓库维护

## 四、系统架构图

![系统架构图](system-architecture.html)

架构分为五层：
1. **客户端层**：CLI 工具链与 Python SDK，是用户与框架交互的入口
2. **高层 API 层**：预构建 Agent API（ReAct/ToolNode/interrupt）与函数式 API（create_agent），编译为 Pregel 图
3. **核心运行时**：StateGraph 与 Channels（状态建模）、Pregel 执行引擎（BSP 超级步）、流式框架（多模式输出）
4. **持久化层**：Checkpoint/Store/Cache 抽象 + Memory/Postgres/SQLite 多后端实现
5. **内部基础设施**：Runnable 基类、序列化、重试、配置等横切关注点

## 五、核心执行时序图

![核心执行生命周期](system-sequence.html)

一次 `graph.invoke(input, config)` 调用经历三个阶段：
1. **初始化与恢复**：PregelLoop 初始化，从 CheckpointSaver 加载最新检查点（如有），恢复 Channels 状态
2. **超级步循环（BSP）**：每个 tick 计算就绪节点 → PregelRunner 并发执行 → 合并通道写入 → 持久化 checkpoint → 发射流事件 → 判断是否继续
3. **完成与返回**：无就绪节点时循环结束，返回最终状态或流迭代器

`interrupt()` 可在超级步间暂停执行，`Command` 恢复时可跳转节点或更新状态。

## 六、数据流图

![数据与请求流水线](system-dataflow.html)

数据流经五个阶段：输入与构建 → 图编译 → 执行循环 → 状态持久化 → 输出。Channels 是状态的唯一真相源，节点通过读写通道通信；Checkpoint 在每超级步结束时持久化完整状态，恢复时据此重建 Channels；流事件经 StreamTransform 转换后以 SSE 或迭代器形式输出。

## 七、执行生命周期图

![图执行生命周期](system-lifecycle.html)

图执行遵循明确的状态机：已创建（graph.compile）→ 恢复检查点（PregelLoop.init 加载最新 checkpoint）→ 超级步执行（BSP 循环，每步内计划→并发执行→更新通道→持久化）→ 流式输出（stream_mode 多路输出）→ 完成。在超级步间可由 `interrupt()` 触发中断等待，通过 `Command` 恢复时可跳转指定节点或更新状态；节点异常可重试或路由 error_handler。该状态机与系统时序图互补：时序图描述多方消息交互，生命周期图聚焦单一图执行实体自身的状态变迁。

## 八、语言适配口径

本项目为**纯 Python 框架 + CLI**（`libs/sdk-js` 仅占位），按纯 Python 专项清单分析：
- **能力缝归组**：BaseCheckpointSaver / BaseStore / BaseCache 抽象基类体系定义契约，多后端实现注册
- **注册表与工厂**：`create_react_agent` 工厂函数、Auto 风格字符串路由（预构建层）
- **可选依赖与懒加载**：psycopg/redis/httpx 等为可选依赖，缺失时优雅降级
- **配置驱动分派**：config 对象控制 checkpoint 后端、流模式、递归限制等运行时行为
- **双轨实现**：sdk-py 的 `_async/` 与 `_sync/` 同构双轨客户端；checkpoint 的同步/异步双套方法

## 九、图表质量档位

| 图 | 类型 | 档位 | 说明 |
|---|---|---|---|
| system-architecture | architecture | standard | 跨层连接多，showcase 布局校验未全过 |
| system-sequence | sequence | standard | 6 参与者 + 循环阶段，布局约束 |
| system-dataflow | dataflow | standard | 5 stage 上限，长跨边用 bottom-channel 路由 |
| system-lifecycle | lifecycle | standard | 双 lane 布局，中断态走 terminal 泳道 |

本轮迭代新增域级图 15 张、叶子补图 20 张、系统级 lifecycle 图 1 张。全部 66 张图 render 退出码 0、HTML >700KB。叶子级图中 showcase 与 standard 混合，降档原因均在各叶子 MD 第 10 节如实披露；域级与系统级图按规则允许 standard，档位已在对应 MD 披露。
