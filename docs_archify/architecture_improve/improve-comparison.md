# 迭代改进对比说明

> 本文档列出 `docs_archify/architecture_improve/`（本轮迭代版）相对基线 `docs_archify/architecture/`（首次分析版）的主要改进点。基线目录保持只读，未做任何改动。

## 一、总体变化

| 指标 | 基线 | 迭代版 | 增量 |
|---|---|---|---|
| MD 文档 | 29 | 30 | +1（本对比文件） |
| HTML 图 | 30 | 66 | +36 |
| JSON IR | 30 | 66 | +36 |
| 总文件数 | 89 | 162 | +73 |
| 域级图 | 0 | 15 | +15（每域 ≥3） |
| 单图叶子数 | 17 | 0 | 全部补至 ≥2 图 |
| 系统级图 | 3 | 4 | +1（lifecycle） |

## 二、新增域级图（15 张，硬性缺口补齐）

基线 5 个域均无域级图，本轮每域补 3 张：

| 域 | architecture | sequence | 第三张 |
|---|---|---|---|
| core-runtime | core-runtime-architecture | core-runtime-sequence | core-runtime-workflow（dataflow 语义：端到端数据流） |
| checkpoint-storage | checkpoint-storage-architecture | checkpoint-storage-sequence | checkpoint-storage-dataflow（持久化数据管道） |
| prebuilt-agents | prebuilt-agents-architecture | prebuilt-agents-sequence | prebuilt-agents-lifecycle（agent 执行循环状态机） |
| cli-tooling | cli-tooling-architecture | cli-tooling-sequence | cli-tooling-workflow（部署流程多角色泳道） |
| sdk-python | sdk-python-architecture | sdk-python-sequence | sdk-python-dataflow（SSE 流式解码管道） |

各域总览 MD 均新增"域级图"小节引用上述 3 张图。

## 三、叶子补图（20 张，17 个单图叶子全部补至 ≥2）

### core-runtime 域（8 叶，补 8 张）

| 叶子 | 基线图 | 新增图 | 新增图语义 |
|---|---|---|---|
| graph-state-api | architecture | sequence | StateGraph.compile() 编译链 |
| pregel-io-checkpoint | architecture | dataflow | checkpoint 读写数据管道 |
| stream-framework | dataflow | architecture + sequence | 流式组件分层（必补）+ stream_mode 分发链 |
| func-api | architecture | sequence | @entrypoint/@task 调用链 |
| managed-values | architecture | lifecycle | ManagedValue 生命周期 |
| internal-foundation | architecture | sequence | Runnable.invoke 基础设施链 |
| remote-debug | architecture | sequence | RemoteGraph 远端调用链 |
| pregel-engine | architecture + lifecycle | — | 已合规，不补第三张（MD 第 10 节写明省略原因） |

### checkpoint-storage 域（5 叶，补 4 张）

| 叶子 | 基线图 | 新增图 | 新增图语义 |
|---|---|---|---|
| checkpoint-base | architecture | sequence | put/get/version 抽象契约调用时序 |
| store-cache-base | architecture | sequence | Store/Cache 命中与回退时序 |
| checkpoint-sqlite | architecture | sequence | SQLite 事务读写时序 |
| checkpoint-conformance | architecture | workflow | 一致性测试运行流程（多角色泳道） |
| checkpoint-postgres | architecture + sequence | — | 已合规，不补第三张 |

### prebuilt-agents 域（3 叶，补 3 张）

| 叶子 | 基线图 | 新增图 | 新增图语义 |
|---|---|---|---|
| tool-node | architecture | sequence | 工具调用执行链 |
| interrupt-command | sequence | architecture + lifecycle | 模块组件拓扑（必补）+ 中断状态机 |
| chat-agent-executor | lifecycle + sequence | — | 已合规，不补第三张 |

### cli-tooling 域（3 叶，补 2 张）

| 叶子 | 基线图 | 新增图 | 新增图语义 |
|---|---|---|---|
| cli-core | architecture | workflow | CLI 命令分发流程 |
| docker-packaging | architecture | workflow | Docker 镜像构建流程 |
| cli-deploy-exec | architecture + sequence | — | 已合规，不补第三张 |

### sdk-python 域（3 叶，补 3 张）

| 叶子 | 基线图 | 新增图 | 新增图语义 |
|---|---|---|---|
| sdk-stream | dataflow | architecture + sequence | 流式传输分层（必补）+ SSE 解码链 |
| sdk-auth-encryption | architecture | sequence | 认证加密调用链 |
| sdk-client | architecture + sequence | — | 已合规，不补第三张 |

## 四、系统级新增图（1 张）

- **system-lifecycle.html**：图执行生命周期状态机（已创建→恢复检查点→超级步执行→流式输出→完成/中断），与 system-sequence（多方消息交互）语义互补。

## 五、MD 文档更新

- **22 个叶子 MD**：第 10 节"图表清单与质量"全部更新，加入新增图的文件名、类型、质量档位；已合规叶子补写"不补第三张图"的省略原因（按 diagram-policy §6 模板）
- **5 个域总览 MD**：新增"域级图"小节，引用 3 张域级图并简要说明
- **README.md**：三层索引全面更新，每个域标题行加入 3 张域级图链接，每个叶子表格列从"架构图"扩展为"图"（含全部图型链接），统计与图型分布表更新
- **system-overview.md**：新增第七节"执行生命周期图"，第九节质量档位表加入 system-lifecycle，统计更新

## 六、质量档位

- 全部 66 张图 render 退出码 0、HTML >700KB（最小 797KB）
- 叶子级图目标 showcase，部分因布局约束（跨层连线多、标签间距、lifecycle 双 lane 路由）落 standard，降档原因均在各叶子 MD 第 10 节如实披露
- 域级与系统级图按规则允许 standard（组件多、跨层连接复杂为预期内），档位已在对应 MD 披露
- 链接校验：115 条相对链接，0 死链

## 七、未改动项

- 基线目录 `docs_archify/architecture/` 保持只读，未做任何改动
- 基线叶子 MD 的第 1–9 节文字（功能清单、核心类型、调用链、配置项、错误重试、并发细节、系统边界、相邻交互、语言专项）全部保留，仅更新第 10 节图表清单
- 基线 30 张图的 HTML 与 JSON 原样复制到对比目录
- 源码事实（行号、路径）未做编造性修改，仅在补图时引用真实源码位置
