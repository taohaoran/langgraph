# 相对基线的改进对比（第二轮迭代分析）

> 本文件列出对比目录 `docs_archify/architecture_improve/` 相对基线目录
> `docs_archify/architecture/`（第一轮，只读未改动）的主要改进点。
> 基线：5 域 22 叶、30 张图、README/system-overview 齐全；本轮：5 域 26 叶、75 张图。

## 1. 新增叶子（4 个，基线缺失）

| 叶子 | 域 | 覆盖内容 |
|------|----|----------|
| channels | core-runtime | `channels/*` 九类通道状态变更原语（base/any_value/binop/delta/ephemeral_value/last_value/named_barrier_value/topic/untracked_value）——基线的状态读写分析缺少通道语义层 |
| checkpoint-serde | checkpoint-storage | `serde/*` 序列化协议族（base/jsonplus/_msgpack/encrypted/event_hooks/types）——基线的持久化分析未覆盖序列化协议实现 |
| sdk-resources | sdk-python | `_async/_sync` 资源端点（assistants/threads/runs/cron/store）——基线只分析 Client 主类，未拆资源端点 |
| sdk-http | sdk-python | `http.py` 同步/异步 HTTP 封装与 `_shared/*`——基线的 HTTP 层与客户端主类混在一起 |

## 2. 叶子级图补齐（≥2 张硬性达标）

基线部分叶子图数不足 2 张（配额缺口）；本轮 26 个叶子**全部 ≥2 张**，其中：
- pregel-engine / chat-agent-executor / interrupt-command / managed-values / func-api 各 3 张
  （architecture + 生命周期/时序补足）；
- 隐藏状态机显式补图：超级步循环（pregel-engine-lifecycle）、ReAct 循环（chat-agent-executor-lifecycle +
  域级 prebuilt-agents-lifecycle）、通道超步生命周期（channels-lifecycle）、函数任务生命周期（func-api-lifecycle）、
  中断状态机（interrupt-command-lifecycle）。

## 3. 域级/系统级图（基线缺域级图，本轮补足）

- 基线 30 张图全部为叶子级；本轮新增 **15 张域级图**（5 域 × 3 张）与 **3 张系统级图**，
  形成系统级 → 域 → 叶子三层完整图体系；
- 域级图选型差异化：core-runtime（architecture/sequence/dataflow）、checkpoint-storage（architecture/sequence/dataflow）、
  prebuilt-agents（architecture/sequence/**lifecycle**——ReAct 状态机）、cli-tooling（architecture/sequence/**workflow**——部署泳道）、
  sdk-python（architecture/sequence/dataflow——SSE 管道）。

## 4. 质量档位变化

| 项 | 基线 | 本轮 |
|----|------|------|
| 总图数 | 30（全叶级） | 75（57 叶 + 15 域 + 3 系统） |
| 叶级 showcase | 未披露 | 52/57 showcase（5 张 standard，降档原因均已在各叶第 10 节披露） |
| 域级/系统级 | 无 | 15 域级（3 showcase + 12 standard）+ 3 系统级（standard，组件多跨层复杂为预期） |
| 渲染可维护性 | 基线 JSON 缺 `meta.output` 字段，无法用当前 CLI 重新渲染 | 全部 75 份 JSON 含 `meta.output`，可随时重渲染（archify CLI 已升级） |

降档明细（叶级 standard 5 张）：internal-foundation-dataflow、checkpoint-base-architecture、
checkpoint-serde-architecture、checkpoint-conformance-architecture、checkpoint-postgres-architecture、
checkpoint-sqlite-architecture、store-cache-base-architecture（6 张 checkpoint-storage 架构图同因
`composition/desktop-readability` 密度检查，详见各叶第 10 节）。

## 5. 修正与加强的事实

- **版本基准更新**：基于 `e539ac12`（2026-09-09）重新核对版本（langgraph 1.2.11 / checkpoint 4.2.0 /
  prebuilt 1.1.0）与依赖图（checkpoint → {postgres, sqlite, prebuilt, langgraph}；prebuilt → langgraph；
  sdk-py → {langgraph, cli}）；
- **workflow schema 更新适配**：archify CLI 升级后 workflow 使用 schema_version 1 + lanes（含 exception
  失败泳道）+ nodes(lane,col,type) + edges(route/role) + mainPath——本轮全部 workflow 图按新 schema 渲染
  （cli-core-workflow、checkpoint-conformance-workflow、cli-tooling-workflow）；
- **纯 Python 语言专项落实**：能力缝归组、双轨实现、配置驱动静态分派、可选依赖懒加载、隐藏状态机在循环里
  五类口径在每叶第 9 节落写并系统级披露；基线未系统化该口径；
- **叶子职责边界细化**：pregel-io-checkpoint 与 checkpoint-storage 域的职责划分、sdk 客户端与资源端点拆分，
  均在域总览第 3 节"域级机制细节"中显式说明。

## 6. 验证记录（对比目录）

- `check-links.py` 全目录 0 死链；
- 75 张 HTML 全部 render 退出码 0、>700KB；75 份 JSON 与 HTML 一一对应；
- 26 份叶子 MD 全部 10 小节、简体中文、每叶 ≥3 处真实 `文件:函数:行号` 引用；
- 基线目录 `docs_archify/architecture/` 全程只读，未改动任何文件。
