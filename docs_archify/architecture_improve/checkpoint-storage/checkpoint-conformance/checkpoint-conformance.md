# checkpoint-conformance（可移植一致性测试套件）

> 本文是 `checkpoint-storage` 域下的叶子子系统文档。域级总览见 `../checkpoint-storage.md`。
> 本文只展开"契约驱动的可移植测试套件"本身——它不实现存储，而是定义各后端必须通过的能力契约。
> 被测试的抽象见 `../checkpoint-base/checkpoint-base.md`。
>
> 源码基准：`libs/checkpoint-conformance`（17 源文件 / 3364 行）。

## 1. 功能清单

| 能力 | 说明 | 源码路径 |
|---|---|---|
| 能力枚举 | 9 项能力（5 BASE + 4 EXTENDED），每项映射到异步方法名 | `capabilities.py:15` |
| 能力自动探测 | 按"是否覆写基类默认异步方法"判断后端支持哪些能力 | `capabilities.py:74` |
| 工厂注册 | `@checkpointer_test` 装饰器把后端工厂登记进模块级注册表 | `initializer.py:59` |
| 校验编排器 | `validate()` 逐能力建全新实例、跑对应 runner、汇总报告 | `validate.py:45` |
| 9 个 spec 测试套件 | put / put_writes / get_tuple / list / delete_thread / delete_for_runs / copy_thread / prune / delta_channel_history | `spec/test_*.py` |
| 报告与分级 | `CapabilityReport`，合规分级 FULL / BASE+PARTIAL / BASE / NONE | `report.py:104` |
| 进度回调 | dot / verbose / quiet 三种输出模式 | `report.py:24` |
| 测试夹具 | `generate_checkpoint`/`generate_config`/`generate_metadata` 工厂 | `test_utils.py:19` |

## 2. 核心类型与接口清单

| 类型 / 函数 | 位置 | 职责 |
|---|---|---|
| `Capability` | `capabilities.py:15` | 9 值枚举：PUT/PUT_WRITES/GET_TUPLE/LIST/DELETE_THREAD（BASE）+ DELETE_FOR_RUNS/COPY_THREAD/PRUNE/DELTA_CHANNEL_HISTORY（EXTENDED） |
| `BASE_CAPABILITIES` / `EXTENDED_CAPABILITIES` | `capabilities.py:30/:41` | 必备 vs 可选集合 |
| `DetectedCapabilities.from_instance` | `capabilities.py:74` | 用 `_is_overridden` 比对实例方法与 `BaseCheckpointSaver` 默认 |
| `checkpointer_test` | `initializer.py:59` | 装饰器，注册 `RegisteredCheckpointer`，支持 `skip_capabilities` 与 `lifespan` |
| `RegisteredCheckpointer` | `initializer.py:23` | 持 `factory`（异步生成器）+ `lifespan`；`create()` 每次产出全新实例 |
| `validate` | `validate.py:45` | 唯一入口；`_RUNNERS` 字典把能力映射到 runner 函数 |
| `CapabilityReport` | `report.py:104` | 聚合结果；`conformance_level()` 输出合规等级 |

## 3. 关键调用链

**一次 `validate` 运行**：
1. 调用方用 `@checkpointer_test(name=...)` 注册后端工厂（如 `InMemorySaver`、`PostgresSaver`），可选 `lifespan`（建/拆库）与 `skip_capabilities`。
2. `validate(registered)` 进入 `registered.enter_lifespan()`（整个校验周期一次）。
3. 逐 `Capability`：用 `registered.create()` 建**全新实例** → `DetectedCapabilities.from_instance(saver)` 探测该能力是否被覆写。
4. 未实现的能力记 `tests_skipped=1`；已实现的查 `_RUNNERS` 调对应 `run_*_tests(saver, on_test_result=...)`，收集 passed/failed/failures。
5. 汇总进 `CapabilityReport.results[cap]`，最后 `conformance_level()` 定级：所有已实现能力通过=FULL；仅 BASE 通过=BASE+PARTIAL/BASE；无 BASE 通过=NONE。

## 4. 配置项

| 配置 | 默认 / 行为 | 位置 |
|---|---|---|
| `skip_capabilities` | 注册时显式跳过某些能力（即便已实现） | `initializer.py:29` |
| `lifespan` | 异步生成器，整个校验周期一次（如建/删数据库） | `initializer.py:30` |
| `validate(capabilities=...)` | 只跑指定能力名集合；不传则跑全部 | `validate.py:48` |
| `progress` | `ProgressCallbacks.default()/verbose()/quiet()` | `report.py:38` |

## 5. 错误与重试语义

- 测试失败不抛异常中断整套：runner 返回 `(passed, failed, failures)`，失败项记入 `CapabilityResult.failures`，继续下一能力。
- 未实现能力不算失败，记 `tests_skipped`，不影响 `passed_all()`（只看 `detected=True` 的能力）。
- `skip_capabilities` 与自动探测"未实现"都会标记为跳过，二者区别仅在语义。

## 6. 并发细节

- 本套件为测试代码，无生产并发模型；但强调**每个能力套件用全新实例**（`async with registered.create() as saver`），避免套件间状态污染（`validate.py:78`）。
- 异步生成器工厂用 `__anext__` 两段式驱动，`finally` 中再 `__anext__()` 触发清理，吞掉 `StopAsyncIteration`（`initializer.py:39`）。

## 7. 系统边界

**In-Scope（本仓库源码内）**
- `conformance/`：能力枚举、注册、编排、报告、9 个 spec 套件、夹具。

**Out-of-Scope（不在本仓库源码内）**
- 被测试的各后端实现（Memory/Postgres/SQLite）——它们是"被测对象"，由其它库提供。
- pytest 等测试运行器本身。
- `BaseCheckpointSaver` 契约定义见 `checkpoint-base` 叶子。

## 8. 与相邻子系统交互

- 上游：各后端库（checkpoint-postgres / checkpoint-sqlite / checkpoint 的 InMemorySaver）导入本套件，用 `@checkpointer_test` 注册自己的工厂并调用 `validate`。
- 本叶子 → 下游：仅依赖 `checkpoint-base` 的抽象类型与 `test_utils` 夹具，不反向依赖任何后端。
- 交互方向：后端 → 注册 → 本套件编排 → 报告；本套件对后端无生产期依赖。

## 9. 语言专项适配口径

纯 Python 框架的**契约驱动 + 注册表**口径：本套件是"能力缝"的另一面——`BaseCheckpointSaver` 定义契约，本套件把契约拆成 9 个可独立探测的能力点，用装饰器注册表让各后端自助注册、自动探测、按能力分级出报告。这是"同一抽象接口 × 多后端实现矩阵"的**验收侧**：抽象层在 `checkpoint-base`，多后端在 postgres/sqlite/memory，本套件是它们共同的可移植性保证。图类型以 architecture（注册→探测→编排→报告的组件流）为主。

## 10. 图表清单与质量

| 图 | 文件 | 类型 | archify 质量档 |
|---|---|---|---|
| 一致性测试套件结构 | `checkpoint-conformance-architecture.html` | architecture | showcase |
| 一致性测试运行流程 | `checkpoint-conformance-workflow.html` | workflow | showcase |

JSON IR 源文件：`json/checkpoint-conformance-architecture.json`、`json/checkpoint-conformance-workflow.json`。
第二张 workflow 图补充基线 architecture 未表达的流程语义：按"后端工厂 / 套件编排 / 报告"三泳道展开
`validate`（`validate.py:45`）的运行流程——后端经 `@checkpointer_test`（`initializer.py:59`）注册工厂，
套件逐能力探测、运行 9 个 spec 用例，最后由 `CapabilityReport`（`report.py:104`）汇总并按
`conformance_level()`（`report.py:129`）定级 FULL / BASE+PARTIAL / BASE / NONE。本套件无业务时序/数据流，
故不补 sequence/dataflow 图。
