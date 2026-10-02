# LangGraph 系统架构总览

> 基于 LangGraph monorepo 源码深度分析产出（第二轮迭代分析，对比基线见 `improve-comparison.md`）。
> 源码基准：main 分支，commit `e539ac12`（2026-09-09）；纯 Python 主语言（sdk-js 仅 README 占位，cli 的 ts 文件为示例）。

## 1. 项目概述

LangGraph 是构建有状态、多智能体（multi-actor）应用的编排框架。仓库为 monorepo，各库位于 `libs/` 下，
本轮覆盖 5 个库：`langgraph`（核心）、`checkpoint`（检查点接口）、`prebuilt`（高级 Agent 构造器）、
`cli`（命令行）、`sdk-py`（Python SDK）；另有 `checkpoint-postgres`/`checkpoint-sqlite`（后端实现）、
`sdk-js`（JS SDK，仅 README 占位）。**架构范式**：纯 Python 框架库——"能力缝 + 注册表 + 插件体系"，
以抽象基类定义契约（Channel/Saver/Serializer/BaseStore）、以注册表/工厂路由实现（create_agent、
序列化器族、stream 消息分发）、以可插拔钩子链扩展（ManagedValue、工具注入）。

**版本与规模**：

| 库 | 版本 | 规模 |
|---|---|---|
| langgraph | 1.2.11 | 核心，9 叶（`libs/langgraph/langgraph`） |
| checkpoint | 4.2.0 | 检查点契约，6 叶（含后端实现库） |
| prebuilt | 1.1.0 | 3 叶（`libs/prebuilt`） |
| cli | 同仓版本 | 3 叶（`libs/cli/langgraph_cli`） |
| sdk-py | 同仓版本 | 5 叶（`libs/sdk-py/langgraph_sdk`） |
| 全仓 | — | ≈225 个 Python 文件 / 约 7.76 万行；requires-python ≥ 3.10 |

**依赖图**（pyproject.toml）：`checkpoint` → {`checkpoint-postgres`, `checkpoint-sqlite`, `prebuilt`, `langgraph`}；
`prebuilt` → `langgraph`；`sdk-py` → {`langgraph`, `cli`}。

## 2. 功能总览

| 领域 | 域 | 叶子 | 核心能力 |
|------|----|------|----------|
| 执行内核 | core-runtime（9 叶） | pregel-engine / pregel-io-checkpoint / graph-state-api / channels / managed-values / func-api / stream-framework / remote-debug / internal-foundation | 图编译、超级步循环、通道状态、流式执行、函数 API |
| 持久化 | checkpoint-storage（6 叶） | checkpoint-base / checkpoint-serde / checkpoint-conformance / checkpoint-postgres / checkpoint-sqlite / store-cache-base | checkpoint 契约、序列化协议、多后端、长期记忆 |
| 预构建 | prebuilt-agents（3 叶） | chat-agent-executor / tool-node / interrupt-command | ReAct 循环、工具执行、人机交互 |
| 工具链 | cli-tooling（3 叶） | cli-core / cli-deploy-exec / docker-packaging | 命令分发、部署编排、镜像构建 |
| 客户端 | sdk-python（5 叶） | sdk-client / sdk-resources / sdk-auth-encryption / sdk-stream / sdk-http | REST/SSE 客户端、资源端点、流式解码 |

## 3. 解决的问题

| 用户痛点 | 解法 |
|----------|------|
| 复杂 Agent 流程难编排 | StateGraph 图定义语言 + 通道原语，节点边声明式组装 |
| 长任务中断/恢复 | 每超级步 checkpoint 落盘，interrupt()/Command 人机交互 |
| 多智能体协同与状态共享 | 通道写入归并、Send/Command 动态路由、ManagedValue 跨步缓存 |
| 流式输出与部分进度 | stream/astream 流式迭代器 + 消息复用（mux）+ SSE 增量解码 |
| 持久化后端选择困难 | BaseCheckpointSaver 契约 + 内存/Postgres/SQLite 实现 + conformance 一致性套件 |
| 部署与调用链路割裂 | cli 一键构建部署 + sdk-py 同步/异步双轨客户端 |

## 4. 系统边界

**上边界（用户/第三方接入）**
- `langgraph` 公开 API：`StateGraph`/`MessageGraph`、`invoke`/`stream`、`@entrypoint`/`@task`、`interrupt`/`Command`
- `prebuilt` 的 `create_agent`/`create_react_agent`、`ToolNode`
- `sdk-py` 的 `Client`/`AsyncClient`；`cli` 的 `langgraph` 命令
- LLM 提供商（OpenAI/Anthropic 等）经标准消息协议接入——**不在本仓库源码内**

**下边界（基础设施）**
- PostgreSQL / SQLite 数据库与驱动、Redis（cache 后端）——**不在本仓库源码内**
- Docker 引擎（cli 构建镜像）、uv 解析器（依赖锁定）——**不在本仓库源码内**
- httpx / aiohttp 传输层（sdk-py）——**不在本仓库源码内**

**内边界（本仓库 vs 扩展/外部）**
- 本仓库 = 编排层：图执行、状态管理、持久化契约、SDK/CLI 封装
- 不承载模型推理：LLM 计算在外部提供商
- 扩展点：BaseCheckpointSaver / Serializer / BaseStore / BaseCache / Channel / ManagedValue 均可继承注册

**侧边界（多部署形态）**
- 单机内存（InMemorySaver）、Postgres/SQLite 持久化、远端 LangGraph Server 平台（sdk/cli 接入）
- `sdk-js` 为 JS 生态对等物（本轮仅 README 占位，不在分析范围）

**不做什么**
- 不实现模型推理/训练（外部 LLM）
- 不实现 RAG/向量检索（可由用户图节点组合）
- 不承载后端服务端实现（LangGraph Server 平台不在本仓库源码内）
- 不含 JS SDK 源码分析（占位）

## 5. 系统架构图说明

[系统架构图](system-architecture.html)（standard 档）按四层组织：

1. **接入层**：用户/应用、开发者 CLI（cli-tooling）、Python SDK（sdk-python）、预构建 Agent（prebuilt-agents）
2. **执行层**：core-runtime——图定义（graph-state-api）→ 编译（func-api）→ 超级步执行（pregel-engine）→
   通道状态（channels/managed-values）→ 流式输出（stream-framework）
3. **持久化层**：checkpoint-storage——契约（checkpoint-base）→ 序列化（checkpoint-serde）→
   多后端（postgres/sqlite/内存）→ 长期记忆（store-cache-base）
4. **外部依赖**：LLM 提供商、数据库/Redis、Docker、LangGraph 平台（均标注"不在本仓库源码内"）

关键交互：prebuilt → langgraph（图编译执行）；pregel-engine → checkpoint（每超级步读写）；
sdk/cli → 远端平台（REST/SSE/部署 API）。

## 6. 核心时序图说明

[系统时序图](system-sequence.html)（standard 档）描述一次带持久化的 agent 执行：

1. 用户调用 prebuilt Agent（ReAct 循环）
2. agent → LLM 推理，返回 tool_calls 或最终答案
3. 每超级步结束 pregel-engine → checkpoint 落盘（序列化 → 数据库）
4. 中断时 interrupt() 挂起，checkpoint 已持久化；Command 恢复后从 checkpoint 续跑
5. 最终结果经流式框架（stream/mux）返回用户

## 7. 系统数据流图说明

[系统数据流图](system-dataflow.html)（standard 档）按"状态写入 → 序列化 → 持久化 → 恢复 → 输出"管道组织：

1. **状态产生**：节点更新通道 → 图状态对象（channels 快照）
2. **序列化**：checkpoint-serde 将状态对象转字节（jsonplus 默认 / msgpack / 加密）
3. **持久化**：checkpoint tuple（版本+字节+元数据）写入后端数据库
4. **恢复**：按 config 读取最新版本 → 反序列化 → 重建通道状态（时间旅行/中断恢复）
5. **输出**：流式事件经 SDK SSE 管道解码输出到用户迭代器

## 8. 图表与语言适配口径

- **图表总览**：系统级 3 张 + 域级 15 张 + 叶子级 57 张 = **75 张**（HTML 均自包含交互式、>700KB、render 退出码 0）；
  全部 JSON IR 存于各层 `json/` 目录，含 `meta.output` 字段可重新渲染。
- **质量档位**：叶级 showcase 52/57、standard 5/57（降档原因见各叶第 10 节）；域级 showcase 3/15
  （cli-tooling 架构/工作流、sdk-python 架构）、standard 12/15；系统级 3/3 standard（组件多、跨层连接复杂为预期）。
- **语言适配口径（纯 Python）**：能力缝归组（抽象基类契约 + 注册表/工厂路由 + 装饰器扩展点）；
  双轨实现模式（sync/async 双轨、Fast/Slow 式多实现）；配置驱动静态分派（序列化协议、create_agent、
  SDK 端点按配置路由）；可选依赖与懒加载（try/except ImportError、is_*_available 检测）；
  隐藏状态机在循环里（超级步循环、ReAct 循环、SSE 增量解码器——均已显式补 lifecycle/dataflow 图）。
