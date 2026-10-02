# 输入输出与检查点 IO（pregel-io-checkpoint）

> 本文是 `core-runtime` 域下的叶子子系统文档。域级总览见 `../core-runtime.md`。
> 本文只展开 Pregel 引擎周边的**输入输出映射、通道读写 runnable、检查点构造与水化**这一片：
> `_io.py`（纯函数映射）、`_read.py`（ChannelRead/PregelNode）、`_write.py`（ChannelWrite）、
> `_checkpoint.py`（检查点 blob 构造与从检查点重建通道）。超级步如何调度这些 IO 见
> `../pregel-engine/pregel-engine.md`；通道本身的存储语义见 `../channels/channels.md`。
>
> 源码基准：`langgraph` 1.2.11，commit `e539ac12`，目录 `libs/langgraph/langgraph/pregel/`。

![输入输出与检查点层架构图](pregel-io-checkpoint-architecture.html)

## 1. 功能清单

本叶子是 Pregel 运行时的**数据通路胶水层**：把外部输入翻译成待写、把节点输出翻译成通道写入、
把内存通道状态序列化成检查点、以及反过来从检查点把通道水化出来。

| 能力 | 说明 | 源码路径 |
|---|---|---|
| 单通道读 | `read_channel`：读通道值，空通道按 `catch` 决定返回 None 还是抛 `EmptyChannelError` | `pregel/_io.py:23` |
| 多通道读 | `read_channels`：按 select 列表批量读，`skip_empty` 跳过空通道 | `pregel/_io.py:38` |
| Command 映射 | `map_command`：把 `Command(update/goto/resume)` 展开成 `(channel, value)` 待写元组 | `pregel/_io.py:56` |
| 输入映射 | `map_input`：把用户输入 chunk 按 input_channels 拆成待写元组 | `pregel/_io.py:81` |
| 输出值映射 | `map_output_values`：把待写还原成对外输出 chunk | `pregel/_io.py:100` |
| 输出更新映射 | `map_output_updates`：按任务分组节点返回（含 RETURN/错误/中断过滤） | `pregel/_io.py:118` |
| 通道读 runnable | `ChannelRead`：从 `CONFIG_KEY_READ` 闭包读通道，支持 mapper 变换 | `pregel/_read.py:25` |
| 节点执行单元 | `PregelNode`：包一组 ChannelRead/节点 runnable/ChannelWrite，是引擎调度的任务单元 | `pregel/_read.py:97` |
| 通道写 runnable | `ChannelWrite`：把节点返回组装成待写，经 `CONFIG_KEY_SEND` 投递 | `pregel/_write.py:46` |
| 写条目 | `ChannelWriteEntry`/`ChannelWriteTupleEntry`：静态声明的写目标与 mapper | `pregel/_write.py:26` / `:37` |
| 空检查点 | `empty_checkpoint()`：初始空 Checkpoint | `pregel/_checkpoint.py:28` |
| 构造检查点 | `create_checkpoint`：由上一检查点 + 活通道状态生成新 blob，处理 DeltaChannel 快照 | `pregel/_checkpoint.py:149` |
| 通道水化 | `channels_from_checkpoint`：从 blob 重建通道；DeltaChannel 触发祖先写入回放 | `pregel/_checkpoint.py:229` |
| 异步水化 | `achannels_from_checkpoint`：异步版本 | `pregel/_checkpoint.py:280` |

## 2. 核心类型与接口清单

| 类型 / 函数 | 位置 | 职责 |
|---|---|---|
| `ChannelRead` | `pregel/_read.py:25` | `RunnableCallable`；`do_read` 静态方法从 config 取 `CONFIG_KEY_READ` 闭包执行真实读取 |
| `PregelNode` | `pregel/_read.py:97` | 引擎可调度的节点单元（输入通道、可执行体、写出列表） |
| `ChannelWrite` | `pregel/_write.py:46` | `RunnableCallable`；`do_write` 校验后调用 `CONFIG_KEY_SEND` 闭包批量投递待写 |
| `ChannelWriteEntry` | `pregel/_write.py:26` | `(channel, value=PASSTHROUGH, skip_none, mapper)`；`PASSTHROUGH` 表示透传节点输入 |
| `map_command(cmd)` | `pregel/_io.py:56` | `Command.PARENT` 直接报错；`goto` 中的 Send 进 TASKS 通道、字符串进分支目标；`resume/update` 展开为待写 |
| `create_checkpoint(...)` | `pregel/_checkpoint.py:149` | 遍历通道：普通通道 `ch.checkpoint()`（MISSING 跳过）；Delta 通道按 `channels_to_snapshot` 写 `_DeltaSnapshot` |
| `channels_from_checkpoint(...)` | `pregel/_checkpoint.py:229` | 先挑出 `_needs_replay` 的 Delta 通道，一次 `saver.get_delta_channel_history` 批量取历史，再 `from_checkpoint` + `replay_writes` |
| `_assemble_writes(writes)` | `pregel/_write.py:172` | 把写条目列表拍平为待写元组序列 |

扩展点：`CONFIG_KEY_READ` / `CONFIG_KEY_SEND` 两个闭包由 Pregel 引擎注入 config，本层只通过 config
调用，不直接触碰通道字典本身——这是本层与引擎之间的能力缝。

## 3. 关键调用链

**调用链一：一次节点执行的写出路径。**

1. 节点 runnable 返回值进入 `ChannelWrite._write`（`_write.py:74`），把 `PASSTHROUGH` 条目替换为真实输入。
2. `ChannelWrite.do_write`（`_write.py:104`）先校验：写保留通道 `TASKS` 直接抛 `InvalidUpdateError`
   （`_write.py:113`），再取 `config[CONF][CONFIG_KEY_SEND]` 闭包调用 `_assemble_writes`（`_write.py:123`）。
3. 闭包把待写投递给 Pregel 引擎，引擎在超步末尾归集后调用各通道 `update`（见 channels 叶子调用链一）。

**调用链二：运行中落一个检查点。**

1. 超步边界，Pregel 调 `create_checkpoint`（`_checkpoint.py:149`）：带上一 checkpoint、当前 `channels`、
   step、以及本超步更新的通道集合与 Delta 快照集合。
2. 对每个通道：普通通道 `ch.checkpoint()` 返回 MISSING 则不进 `channel_values`（如 UntrackedValue）；
   属于 `channels_to_snapshot` 的 Delta 通道写入 `_DeltaSnapshot(ch.get())`（`_checkpoint.py:192`）。
3. 产物是新的 `Checkpoint(v=latest, id, ts, channel_values, channel_versions, versions_seen)`，
   交给 checkpointer 的 `put` 持久化（saver 不在本仓库源码内）。

**调用链三：从检查点恢复（水化）。**

1. 启动时 Pregel 调 `channels_from_checkpoint`（`_checkpoint.py:229`）。
2. `_needs_replay`（`_checkpoint.py:217`）挑出需要回放的 Delta 通道，若有则一次性
   `saver.get_delta_channel_history(config, channels=delta_channels)`（`_checkpoint.py:258`）批量取。
3. 对每个通道：有历史的走 `delta_spec.from_checkpoint(seed)` + `replay_writes(history["writes"])`
   （`_checkpoint.py:267`）；其余走 `spec.from_checkpoint(blob)`（`_checkpoint.py:272`）。

![状态输入输出与检查点数据流](pregel-io-checkpoint-dataflow.html)

## 4. 配置项

| 配置 / 参数 | 默认 / 行为 | 位置 |
|---|---|---|
| `catch`（read_channel） | `True`：空通道读返回 None；`False`：抛 `EmptyChannelError` | `_io.py:24` |
| `skip_empty`（read_channels） | `True`：空通道不出现在结果 dict | `_io.py:40` |
| `fresh`（ChannelRead） | `False`；为 True 时读"本超步新写"的新鲜值 | `_read.py:29` |
| `skip_none`（ChannelWriteEntry） | `False`；True 时值为 None 则跳过该写 | `_write.py:31` |
| `channels_to_snapshot`（create_checkpoint） | `set()`：哪些 Delta 通道本步写全量快照；由调用方按快照频率算 | `_checkpoint.py:157` |
| `updated_channels` | 本超步更新的通道名集合，用于版本推进 | `_checkpoint.py:156` |

## 5. 错误与重试语义

- **写保留通道**：`ChannelWrite.do_write` 发现写 `TASKS` 通道抛 `InvalidUpdateError`（`_write.py:113`）——
  这是用户图配置错误，不重试。
- **PARENT 越界**：`map_command` 遇到 `Command.PARENT` 但无父图时抛 `InvalidUpdateError`（`_io.py:59`）。
- **goto 类型错**：`Command.goto` 既非 Send 也非 str 时抛 `TypeError`（`_io.py:73`）。
- **输入类型错**：`map_input` 期望 dict 却给了别的类型抛 `TypeError`（`_io.py:93`）；多余输入键仅
  `logger.warning` 不中断（`_io.py:97`）。
- **空通道读取**：空通道读是正常控制流（`EmptyChannelError`），由 `catch/skip_empty` 转化为 None/跳过。
- 本层无重试/退避：纯映射与胶水，失败即抛，节点级重试与超步级容错由 pregel-engine 负责。

## 6. 并发细节

- 本层全部是**同步纯函数**（`_io.py`）与 `RunnableCallable` 包装；异步版本 `achannels_from_checkpoint`
  （`_checkpoint.py:280`）是为了等待 `saver` 的异步 IO，通道重建本身仍同步。
- 线程/协程安全：`ChannelRead/ChannelWrite` 通过 config 注入的 `CONFIG_KEY_READ/SEND` 闭包与引擎通信，
  自身无共享可变状态，可在同一事件循环内被多个任务调用。
- 批量回放优化：Delta 通道回放把所有待回放通道合并成一次 `saver.get_delta_channel_history` 调用
  （`_checkpoint.py:258`），避免 N 次远端往返——这是本层唯一明显的并发/IO 聚合优化点。
- 取消传播：本层不捕获 `asyncio.CancelledError`；水化在恢复路径上被中断时由引擎决定重试或丢弃。

## 7. 系统边界

**In-Scope（本仓库源码内）**
- `pregel/_io.py`：输入/输出/Command 的纯函数映射
- `pregel/_read.py`：`ChannelRead`、`PregelNode`
- `pregel/_write.py`：`ChannelWrite` 与写条目
- `pregel/_checkpoint.py`：检查点 blob 构造与通道水化（含 Delta 回放编排）

**Out-of-Scope（不在本仓库源码内）**
- 检查点的实际持久化（put/get/list）与 Delta 历史存储：`BaseCheckpointSaver` 及其实现
  （`libs/checkpoint-*` 库）
- 通道的存储语义（覆盖/追加/归约）：channels 原语族
- 超级步调度、任务并发执行、节点调用：Pregel 引擎（`pregel/main.py`、`_loop.py`、`_runner.py`）
- `BaseStore` 长期记忆：store 后端库

## 8. 与相邻子系统交互

- **pregel-engine → 本叶子**：引擎在节点执行前后注入 `CONFIG_KEY_READ/SEND` 闭包；超步边界调
  `create_checkpoint`；启动/恢复调 `channels_from_checkpoint`。
- **本叶子 → channels**：`create_checkpoint` 调 `ch.checkpoint()/ch.get()`；`channels_from_checkpoint`
  调 `spec.from_checkpoint()` 与 `DeltaChannel.replay_writes()`。
- **本叶子 → checkpointer**：`create_checkpoint` 产出 blob 交给 saver.put；水化时调
  `saver.get_delta_channel_history`。
- **graph-state-api → 本叶子**：编译期产生的 `PregelNode`（含 ChannelRead/ChannelWrite 声明）是本层
  runnable 的来源；`Command/Send` 对象经 `map_command` 在此被翻译成待写。

## 9. 语言专项适配口径（纯 Python）

按 `language-python.md` 清单落地：

1. **能力缝归组**：本层与引擎之间的能力缝是两个 config 闭包——`CONFIG_KEY_READ` /
   `CONFIG_KEY_SEND`（`_write.py:121`、`_read.py:69`）。本层面向闭包编程，引擎注入实现，这是典型的
   "控制反转"接缝。
2. **双轨实现**：同步/异步双轨——`_write` 与 `_awrite`、`_read` 与 `_aread`、
   `channels_from_checkpoint` 与 `achannels_from_checkpoint`（`_checkpoint.py:280`）成对出现。
3. **批量聚合优化**：Delta 通道回放把 N 次远端调用合并为一次 `get_delta_channel_history`，属生产者-
   消费者/批处理缝的应用。
4. **隐藏状态机在循环里**：检查点 blob 的构造与水化发生在 Pregel 每个超步边界，是超步循环内的固定
   阶段；本层用 dataflow 图把"输入→通道→写出→快照→落盘"这条固定管道显式化。
5. **外部边界**：本层不持久化、不调度，只做翻译与编排；真正存储在 checkpointer、真正计算在用户节点，
   均标注"不在本仓库源码内"。

## 10. 图表清单与质量

| 图 | 文件 | 类型 | archify 质量档 |
|---|---|---|---|
| 输入输出与检查点层架构图 | `pregel-io-checkpoint-architecture.html` | architecture | showcase（首跑因"复用 read_channels"标签间距不足被 composition/label-gap 拒；删去两条低价值"复用"虚线边、改为 read/write 直连 channels 后通过） |
| 状态输入输出与检查点数据流 | `pregel-io-checkpoint-dataflow.html` | dataflow | showcase（首跑因"每超步快照"连线压到同排 write 节点被 layout/constraint 拒；把 ChannelWrite 降到 stage2 第二行、write→channels 改 vertical-channel 后通过） |

JSON IR 源文件位于 `json/`（`pregel-io-checkpoint-architecture.json`、`pregel-io-checkpoint-dataflow.json`）。

**省略说明**：本叶子未生成 sequence 图——本层是胶水/纯函数，其调用先后已由 dataflow 图与第 3 节文字
调用链表达，无"多方按时间互发消息"的独立交互链值得单画；未生成 lifecycle 图——本层无单一实体的多
状态迁移（检查点 blob 的版本推进由引擎超步循环驱动，归 pregel-engine）。按资源节省原则省略。
