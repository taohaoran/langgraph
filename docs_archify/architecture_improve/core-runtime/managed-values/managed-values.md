# 托管值 ManagedValue（managed-values）

> 本文是 `core-runtime` 域下的叶子子系统文档。域级总览见 `../core-runtime.md`（输出根统一产出）。
> 本文只展开运行期"托管值"抽象基类与内置实现（`IsLastStep`/`RemainingSteps`）；不重复展开
> Pregel 如何在执行循环里调用 `get()`（见 `../pregel-engine/pregel-engine.md`）与 state schema
> 如何识别托管值字段（见 `../graph-state-api/graph-state-api.md`）。
>
> 源码基准：`langgraph` main 分支，commit `e539ac122f4126f6dd850581c1494948cf620e31`。

## 1. 功能清单

| 能力 | 说明 | 源码路径 |
| --- | --- | --- |
| `ManagedValue`（ABC） | 托管值抽象基类：唯一静态方法 `get(scratchpad) -> V`；值由引擎从运行上下文派生，不落盘 | `managed/base.py:18` |
| `ManagedValueSpec` | `type[ManagedValue]` 类型别名；schema 注解里携带此类即被识别为托管值 | `managed/base.py:24` |
| `is_managed_value` | TypeGuard：判断注解值是否为 `ManagedValue` 子类 | `managed/base.py:27` |
| `ManagedValueMapping` | `dict[str, ManagedValueSpec]`：字段名 → 托管值类映射 | `managed/base.py:31` |
| `IsLastStepManager` / `IsLastStep` | `Annotated[bool, IsLastStep]`；`step == stop - 1` 时为真 | `managed/is_last_step.py:9`、`:15` |
| `RemainingStepsManager` / `RemainingSteps` | `Annotated[int, RemainingSteps]`；`stop - step` 剩余步数 | `managed/is_last_step.py:18`、`:24` |
| 包导出 | 顶层 `managed/__init__.py` 仅导出 `IsLastStep`、`RemainingSteps` | `managed/__init__.py:1` |

## 2. 核心类型与接口清单

| 类型 / 函数 | 位置 | 职责 |
| --- | --- | --- |
| `ManagedValue[V]` | `managed/base.py:18` | 抽象基类；`@staticmethod @abstractmethod get(scratchpad: PregelScratchpad) -> V`。这是本叶子唯一扩展点：用户可继承它自定义托管值 |
| `ManagedValueSpec` | `managed/base.py:24` | 类型别名（`type[ManagedValue]`），供 channels/specs 联合类型标注 |
| `is_managed_value` | `managed/base.py:27` | 用 `inspect.isclass` + `issubclass` 做运行期类型守卫 |
| `IsLastStepManager` | `managed/is_last_step.py:9` | 内置实现：返回是否最后一步 |
| `RemainingStepsManager` | `managed/is_last_step.py:18` | 内置实现：返回剩余步数 |
| `PregelScratchpad` | `_internal/_scratchpad.py:9` | 运行期上下文数据类：`step`/`stop`、call/interrupt/subgraph 计数器、`resume` 列表（由 Pregel 每步构造，作为 `get()` 入参） |

## 3. 关键调用链

1. **编译期识别**：用户在 state schema 写 `Annotated[bool, IsLastStep]`。`graph/state.py:206` 把带托管值注解的字段收集进 `managed: dict[str, ManagedValueSpec]`，随 channels 一起交给 Pregel。
2. **规格拆分**：`pregel/_checkpoint.py:248-252` 遍历 `specs`，`isinstance(v, BaseChannel)` 为真归入 `channel_specs`，否则归入 `managed_specs`（即 `ManagedValueMapping`），随后 `:277` 把两者作为元组返回。
3. **运行期注入（节点输入构造）**：`pregel/_algo.py:222-223` 在读完普通通道值后，`values.update({k: managed[k].get(scratchpad) for k in managed_keys})` 把托管值合并进节点输入 dict。
4. **单通道进程注入**：`pregel/_algo.py:1369-1370` 对 `proc.channels` 列表里不在普通 channels 中的字段，直接 `val[chan] = managed[chan].get(scratchpad)`。
5. **取值语义**：`IsLastStepManager.get` 返回 `scratchpad.step == scratchpad.stop - 1`（`is_last_step.py:11-12`）；`RemainingStepsManager.get` 返回 `scratchpad.stop - scratchpad.step`（`is_last_step.py:20-21`）。

## 4. 配置项

| 配置项 | 默认 / 行为 | 位置 |
| --- | --- | --- |
| `Annotated[bool, IsLastStep]` | 声明即启用；节点在最后一步收到 `True` | `managed/is_last_step.py:15` |
| `Annotated[int, RemainingSteps]` | 声明即启用；节点收到剩余步数 | `managed/is_last_step.py:24` |
| `step` / `stop` | 由 Pregel 运行参数决定（`recursion_limit`、`stop`），托管值只读这两个量 | `_internal/_scratchpad.py:9-10` |

本叶子无独立 flag；行为完全由"是否在 schema 中声明对应 Annotated 字段"决定。

## 5. 错误与重试语义

- 托管值本身不产生可重试错误：`get()` 是纯函数，只读 `scratchpad.step/stop`，无 IO、无异常路径。
- 若用户自定义 `ManagedValue.get` 抛异常，异常会沿 `pregel/_algo.py:223` 的节点输入构造点向上冒泡，终止当前步；Pregel 层的重试/回退由 checkpoint 与任务重试机制负责（见 pregel-engine 叶子），本叶子不参与。
- 托管值不写入 checkpoint，因此不存在"反序列化失败"路径。

## 6. 并发细节

- 托管值是无状态静态方法，不持有锁、不创建线程/协程；`PregelScratchpad` 是每步新建的临时数据类（`_internal/_scratchpad.py:9`）。
- 每个 superstep 独立构造 scratchpad 并对托管字段调用一次 `get()`，天然无跨步共享状态，无需同步。
- 子图场景下 `PregelScratchpad` 携带 `subgraph_counter` 等计数器回调（`_internal/_scratchpad.py:16`），由父图传递，托管值对其透明。

## 7. 系统边界

**In-Scope（本仓库源码内）**
- `managed/base.py`：托管值抽象契约与类型守卫。
- `managed/is_last_step.py`：两个内置托管值实现。
- 与 `_internal/_scratchpad.py` 的耦合点（`PregelScratchpad` 入参）。

**Out-of-Scope（不在本仓库源码内）**
- Pregel 执行循环如何调度节点、如何构造 scratchpad：见 pregel-engine 叶子。
- state schema 如何用 `Annotated` 注解定义聚合器/托管值：见 graph-state-api 叶子。
- checkpoint 持久化本身（托管值刻意不参与）：见 pregel-io-checkpoint 叶子。
- 第三方 LLM / 工具：不被本叶子引用。

## 8. 与相邻子系统交互

- 上游 → 本叶子：`graph-state-api`（`graph/state.py`）解析 `Annotated` 注解，把 `ManagedValue` 子类登记为 `managed_specs`。
- 本叶子 → 下游：`pregel-engine`（`_checkpoint.py` 拆分规格、`_algo.py` 调 `get()`）在每步把 `PregelScratchpad` 传入 `Spec.get()`，取回派生值塞进节点输入。
- 方向：`graph-state-api → managed(base) → pregel-engine（注入点）→ 用户节点`。

## 9. 语言专项适配口径

按纯 Python 框架口径：
- **能力缝归组**：本叶子是典型"抽象基类定义契约、实现方注册"的能力缝——`ManagedValue(ABC)` 是消费方（Pregel）依赖的契约，`IsLastStepManager`/`RemainingStepsManager` 是内置实现方，用户可继承扩展。
- **装饰器/注解扩展点**：注册机制是 `typing.Annotated` 第二参数——把托管值类挂在类型注解上，import 时无副作用，靠 `is_managed_value`（`isclass + issubclass`）在 schema 解析期识别。
- **配置驱动分派**：节点输入里哪些字段是"通道值"、哪些是"托管值"，由 `_checkpoint.py:249` 的 `isinstance(v, BaseChannel)` 判断静态分派，而非运行时参数。
- **隐藏状态机在循环里**：托管值本身无跨步状态，其"生命周期"隐藏在每个 superstep 循环内（派生→注入→丢弃），故补 lifecycle 图显式表达。
- 无代码生成、无外部绑定、无可选依赖。

## 10. 图表清单与质量

| 图 | 文件 | 类型 | archify 质量档 |
| --- | --- | --- | --- |
| 托管值架构图 | `managed-values-architecture.html` | architecture | showcase |
| 注入调用时序 | `managed-values-sequence.html` | sequence | showcase |
| 单步生命周期 | `managed-values-lifecycle.html` | lifecycle | showcase |

JSON IR 源文件位于 `json/` 目录。本叶子未补 dataflow / workflow 图：托管值无数据管道语义（值即时派生、不流经队列），也无多角色泳道审批流程，与已有图信息重复，按资源节省原则省略。
