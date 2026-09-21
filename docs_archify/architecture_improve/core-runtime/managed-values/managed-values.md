# 托管值 ManagedValue（managed-values）

> 本文是 `core-runtime` 域下的叶子子系统文档。域级总览见 `../core-runtime.md`。
> 本文只展开运行期"托管值"抽象与内置实现；不重复展开 Pregel 如何把托管值注入节点
> （见 `../pregel-engine/pregel-engine.md`）与 schema 如何识别托管值字段
> （见 `../graph-state-api/graph-state-api.md`）。
>
> 源码基准：`langgraph` main 分支，commit `e539ac122f4126f6dd850581c1494948cf620e31`。

## 1. 功能清单

| 能力 | 说明 | 源码路径 |
| --- | --- | --- |
| `ManagedValue`（ABC） | 托管值抽象：唯一静态方法 `get(scratchpad) -> V`；值由引擎派生、不持久化 | `managed/base.py:18` |
| `ManagedValueSpec` | `type[ManagedValue]` 的别名；schema 注解中携带此类即注册 | `managed/base.py:24` |
| `is_managed_value` | TypeGuard：判断对象是否是 `ManagedValue` 子类 | `managed/base.py:27` |
| `ManagedValueMapping` | `dict[str, ManagedValueSpec]`：键→托管值类映射 | `managed/base.py:31` |
| `IsLastStepManager` / `IsLastStep` | `Annotated[bool, IsLastStep]`；`step == stop - 1` 时为真 | `managed/is_last_step.py:9`、`:15` |
| `RemainingStepsManager` / `RemainingSteps` | `Annotated[int, RemainingSteps]`；`stop - step` 剩余步数 | `managed/is_last_step.py:18`、`:24` |

## 2. 核心类型与接口清单

| 类型 / 函数 | 位置 | 职责 |
| --- | --- | --- |
| `ManagedValue[V]` | `managed/base.py:18` | 抽象基类；`@staticmethod @abstractmethod get(scratchpad)` |
| `ManagedValueSpec` | `managed/base.py:24` | 类型别名（`type[ManagedValue]`） |
| `IsLastStepManager` | `managed/is_last_step.py:9` | 实现"是否最后一步" |
| `RemainingStepsManager` | `managed/is_last_step.py:18` | 实现"剩余步数" |
| `PregelScratchpad` | `_internal/_scratchpad.py:9` | 运行期上下文：`step/stop`、call/interrupt/subgraph 计数器、resume 列表 |

## 3. 关键调用链

1. 用户在 state schema 写 `Annotated[bool, IsLastStep]`。
2. `_get_channels`（见 graph-state-api）经 `_is_field_managed_value` 识别出该字段是托管值，记入 `managed_specs`（`ManagedValueMapping`），与普通通道一起注入编译图。
3. Pregel 执行节点时，构造 `PregelScratchpad(step=..., stop=..., ...)`（`_internal/_scratchpad.py:9`），对每个托管值字段调 `Spec.get(scratchpad)`（`is_last_step.py:11/20`）得到运行期值，注入节点输入。
4. `IsLastStep.get` 返回 `scratchpad.step == scratchpad.stop - 1`；`RemainingSteps.get` 返回 `scratchpad.stop - scratchpad.step`。

## 4. 配置项

| 配置项 | 默认 / 行为 | 位置 |
| --- | --- | --- |
| state 字段注解 `Annotated[T, SomeManagedValue]` | 声明该字段为托管值；值在每超级步由 `get(scratchpad)` 派生 | `managed/base.py:18` |
| `step` / `stop`（scratchpad） | 当前步与最大步；由 Pregel 循环维护 | `_internal/_scratchpad.py:10` |

## 5. 错误与重试语义

- 本叶子为纯派生逻辑，无错误路径、无重试、无 I/O。
- `get` 为静态方法，若 `scratchpad` 未就绪会抛 AttributeError，但正常由引擎保证其在节点执行前已构造。

## 6. 并发细节

- 无并发：`get` 是纯函数式读取 `PregelScratchpad` 字段，每个节点在自己的执行上下文里得到一份派生值。
- 托管值**不写入 checkpoint**（与通道值区分），每步由 scratchpad 现场重算。

## 7. 系统边界

**In-Scope（本仓库源码内）**
- `managed/base.py`（ABC + spec 类型）、`managed/is_last_step.py`（两个内置实现）

**Out-of-Scope（不在本仓库源码内）**
- `PregelScratchpad` 的构造与注入 → `pregel/`（`../pregel-engine/`）
- schema 识别托管值 → `graph/state.py`（`../graph-state-api/`）

## 8. 与相邻子系统交互

- **上游 ← graph-state-api**：`_add_schema`/`_get_channels` 识别 `is_managed_value` 并收编为 `ManagedValueMapping`。
- **下游 → pregel-engine**：引擎为每节点构造 `PregelScratchpad` 并调 `Spec.get(scratchpad)` 注入值。
- **下游 → internal-foundation**：依赖 `PregelScratchpad`（`_internal/_scratchpad.py`）。

## 9. 语言专项适配口径

本项目为纯 Python 框架，按纯 Python 专项清单分析：

1. **能力缝归组**：`ManagedValue` ABC 是"运行期派生值"能力缝——与通道（持久化状态）对照，托管值是引擎现场计算、不持久化的只读注入值。
2. **注册表**：无全局注册表；通过 `Annotated[T, SomeManagedValueClass]` 在字段注解中携带 spec，`is_managed_value` 做类型守卫识别。
3. **配置驱动分派**：字段注解里放哪个 `ManagedValue` 子类，决定节点读到的派生值语义。

## 10. 图表清单与质量

| 图 | 文件 | 类型 | archify 质量档 |
| --- | --- | --- | --- |
| 托管值结构 | `managed-values-architecture.html` | architecture | standard |
| ManagedValue 生命周期 | `managed-values-lifecycle.html` | lifecycle | standard |

- 降档说明：architecture 图降 standard；本次新增的 lifecycle 图（声明注册→init 初始化→注入节点→节点消费→步末清理）因"跨 lane 回边穿越节点"简化为主线后仍未全过 showcase，按规范降 standard 渲染（均成功，约 800KB 自包含）。
- JSON IR 源：`json/managed-values-architecture.json`、`json/managed-values-lifecycle.json`。
- 本次迭代新增 lifecycle 图，把托管值"现场派生、不持久化"的生命周期从文字化升级为状态机；不另出 sequence：托管值是单实体状态变迁而非多方消息交互。
