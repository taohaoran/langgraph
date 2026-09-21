# Pregel IO 与 Checkpoint（pregel-io-checkpoint）

> 本文是 `core-runtime` 域下的叶子子系统文档。域级总览见 `../core-runtime.md`。
> 本文只展开输入/输出映射、通道读写桥接（`ChannelRead`/`ChannelWrite`）、checkpoint
> 构建与水化、消息/工具流回调；不重复展开 BSP 超级步调度（见
> `../pregel-engine/pregel-engine.md`）与通道语义（见 `../graph-state-api/graph-state-api.md`）。
>
> 源码基准：`langgraph` main 分支，commit `e539ac122f4126f6dd850581c1494948cf620e31`。

## 1. 功能清单

| 能力 | 说明 | 源码路径 |
| --- | --- | --- |
| 输入映射 `map_input` | 把用户输入 chunk 转成 `(channel, value)` 待写入；dict 输入按键白名单过滤 | `pregel/_io.py:81` |
| 命令映射 `map_command` | 把 `Command(goto/resume/update)` 转成待写入：Send→`TASKS`，str→`branch:to:<name>`，resume→`RESUME` 通道 | `pregel/_io.py:56` |
| 输出映射 `map_output_values` | values 模式：按输出通道读当前值 | `pregel/_io.py:100` |
| 输出映射 `map_output_updates` | updates 模式：按任务聚合写入，排除 ERROR/INTERRUPT 与隐藏任务，产出 `{node: update}` | `pregel/_io.py:118` |
| `read_channels`/`read_channel` | 读通道值，`EmptyChannelError` 时按 `catch` 返回 None 或上抛 | `pregel/_io.py:23` |
| `ChannelRead` | 把"读状态"包成 Runnable，从 `CONFIG_KEY_READ` 闭包取值，支持 `fresh`/`mapper` | `pregel/_read.py:25` |
| `PregelNode` | 节点容器：`channels`（输入）、`triggers`（触发）、`writers`（写桥）、`bound`（主逻辑）、`mapper` | `pregel/_read.py:97` |
| `ChannelWrite` | 把"写通道"包成 Runnable，经 `CONFIG_KEY_SEND` 闭包提交写入；`PASSTHROUGH` 透传输入、`skip_none`/`SKIP_WRITE` 跳过 | `pregel/_write.py:46` |
| 静态写入分析 | `get_static_writes`/`register_writer` 提取 writer 声明的写入，供图校验 | `_write.py:137`、`:159` |
| `empty_checkpoint` | 空 checkpoint（`LATEST_VERSION=4`，uuid6 id） | `_checkpoint.py:28` |
| `create_checkpoint` | 从前一 checkpoint + 实时通道构建新 Checkpoint；DeltaChannel 到点写 `_DeltaSnapshot` blob | `_checkpoint.py:149` |
| `channels_from_checkpoint`/`achannels_from_checkpoint` | 从 checkpoint 水化通道；DeltaChannel 缺值时批量调 `saver.get_delta_channel_history` 回放祖先 writes | `_checkpoint.py:229`、`:280` |
| Delta 快照判定 | `delta_channels_to_snapshot`：达 `snapshot_frequency` 或 `DELTA_MAX_SUPERSTEPS_SINCE_SNAPSHOT` 才全量快照 | `_checkpoint.py:50` |
| `StreamMessagesHandler` | stream_mode=messages 的 LangChain 回调，收集 LLM token 与节点消息事件 | `_messages.py:49` |
| `StreamToolCallHandler` | tools 流模式回调：发 tool-started/output-delta/finished/error，经 `_tool_call_writer` ContextVar 桥接 | `_tools.py:35` |
| 日志 | 模块级 `logging.getLogger("langgraph")` | `_log.py:3` |

## 2. 核心类型与接口清单

| 类型 / 函数 | 位置 | 职责 |
| --- | --- | --- |
| `ChannelRead` | `_read.py:25` | 读桥 Runnable（`do_read` 静态方法可命令式调用） |
| `ChannelWrite` | `_write.py:46` | 写桥 Runnable（`do_write` 静态方法） |
| `ChannelWriteEntry` / `ChannelWriteTupleEntry` | `_write.py:26` / `:37` | 写入条目（channel+value+mapper）/ 元组提取条目 |
| `PregelNode` | `_read.py:97` | 节点容器（不含执行，只持组件） |
| `Checkpoint`（外部类型） | `langgraph.checkpoint.base` | 检查点数据结构；本叶子负责构造与水化 |
| `_DeltaSnapshot` | `_checkpoint.py:14`（导入） | DeltaChannel 全量快照 blob 标记 |
| `StreamMessagesHandler` / `StreamToolCallHandler` | `_messages.py:49` / `_tools.py:35` | 流模式回调 |
| `_tool_call_writer` | `_tools.py:25` | ContextVar，桥接工具体输出增量 |

## 3. 关键调用链

**链一：输入如何进入通道**

1. 用户 `invoke(input)` → `map_input(input_channels, input)`（`_io.py:81`）把 dict 输入按白名单展开成 `(channel, value)`。
2. `map_command(Command)`（`_io.py:56`）把 `Command.goto` 的 Send 转 `(NULL_TASK_ID, TASKS, send)`、str 转 `(NULL_TASK_ID, branch:to:<name>, START)`；`resume` 写 `RESUME` 通道；`update` 写状态键。
3. 这些 pending writes 经 `ChannelWrite.do_write`（`_write.py:105`）调用 `config[CONFIG_KEY_SEND]` 闭包提交，由 Pregel 循环在下一超级步 `apply_writes` 落通道。

**链二：节点如何读写状态**

- 节点主逻辑 `bound` 执行前，`PregelNode.channels` 经 `ChannelRead.do_read`（`_read.py:73`）从 `CONFIG_KEY_READ` 闭包读入；执行后 `writers`（含 `ChannelWrite`）把返回值转成 `(channel, value)` 提交 `CONFIG_KEY_SEND`。`PASSTHROUGH` 表示直接用节点返回值写，`skip_none` 跳过 None。

**链三：checkpoint 水化（DeltaChannel 回放）**

1. 循环 `__enter__` 调 `channels_from_checkpoint(specs, checkpoint, saver, config)`（`_checkpoint.py:229`）。
2. 对每个 `DeltaChannel` 且 `channel_values[k]` 缺失（`_needs_replay`，`:217`），批量收集后一次调 `saver.get_delta_channel_history(config, channels)`（`:261`）拿 seed + writes。
3. `from_checkpoint(seed)` 重建通道实例，再 `replay_writes(history["writes"])` 经批式 reducer 累加，得到当前值（`:271-273`）。
4. 普通通道直接 `spec.from_checkpoint(channel_values[k])`。

## 4. 配置项

| 配置项 | 默认 / 行为 | 位置 |
| --- | --- | --- |
| `stream_mode` 含 `messages`/`tools` | 触发挂接 `StreamMessagesHandler`/`StreamToolCallHandler` | `_messages.py`、`_tools.py` |
| `ChannelRead(fresh=True)` | 强制读实时值而非本步快照 | `_read.py:31` |
| `DeltaChannel.snapshot_frequency` | 每 N 次更新写一次全量快照 | `channels/delta.py` |
| `DELTA_MAX_SUPERSTEPS_SINCE_SNAPSHOT` | 无更新时最多回放超级步数（默认 5000） | `_internal/_config.py` |
| `TAG_HIDDEN` / `TAG_NOSTREAM` | updates 模式排除隐藏任务；tools 回调跳过 no-stream | `_io.py:127`、`_tools.py:11` |
| `LATEST_VERSION=4` | checkpoint 结构版本 | `_checkpoint.py:23` |

## 5. 错误与重试语义

- **非法写入**：`ChannelWrite.do_write` 禁止写保留通道 `TASKS`，抛 `InvalidUpdateError`（`_write.py:114`）；`PASSTHROUGH` 未替换抛 `InvalidUpdateError`。
- **非法 Command**：`map_command` 遇 `Command.PARENT` 且无父图抛 `InvalidUpdateError`（`_io.py:58`）；`goto` 非 Send/str 抛 `TypeError`。
- **读空通道**：`EmptyChannelError` 按 `catch`/`skip_empty` 静默返回 None 或上抛（`_io.py:31`）。
- **水化失败**：DeltaChannel 回放依赖 saver 提供 `get_delta_channel_history`；缺失时该接口由 checkpoint 后端实现（外部库）。
- 本叶子不含重试逻辑，重试由 `pregel-engine` 的 `run_with_retry` 承载。

## 6. 并发细节

- 本叶子多为纯函数与 Runnable，并发语义由执行引擎提供。
- `StreamMessagesHandler.run_inline = True` / `StreamToolCallHandler.run_inline = True`：强制回调在主线程内联执行，保证流事件顺序确定、避免锁竞争（`_messages.py:57`、`_tools.py:53`）。
- `_tool_call_writer` 是 `ContextVar`：在 `on_tool_start` 设置、`on_end`/`on_error` 重置，使并发工具体经同一上下文读回正确 writer（`_tools.py:25`）。
- `copy_context()` 在执行引擎侧复制；本叶子通过 `CONFIG_KEY_READ`/`CONFIG_KEY_SEND` 闭包读写，闭包在循环创建时注入，任务间共享只读。

## 7. 系统边界

**In-Scope（本仓库源码内）**
- `pregel/_io.py`（输入/输出映射）、`_read.py`（ChannelRead/PregelNode）、`_write.py`（ChannelWrite）
- `pregel/_checkpoint.py`（checkpoint 构造/水化/Delta 回放）、`_messages.py`、`_tools.py`、`_log.py`

**Out-of-Scope（不在本仓库源码内）**
- `BaseCheckpointSaver` 接口与 Memory/SQLite/Postgres 实现 → `libs/checkpoint*/`，外部库
- `Checkpoint`/`CheckpointMetadata`/`_DeltaSnapshot` 等类型 → `langgraph.checkpoint.base`/`serde`，外部库
- LangChain `BaseCallbackHandler`、`_StreamingCallbackHandler` → 第三方依赖
- checkpoint 实际序列化（serde）→ `libs/checkpoint*/serde`

## 8. 与相邻子系统交互

- **上游 ← pregel-engine**：`PregelLoop` 调 `map_input`/`map_command` 注入输入，调 `create_checkpoint`/`channels_from_checkpoint` 落盘与水化。
- **下游 → graph-state-api**：`create_checkpoint` 依赖各通道的 `checkpoint()`/`from_checkpoint()` 契约；DeltaChannel 回放依赖 `replay_writes`。
- **下游 → pregel-engine**：`ChannelRead`/`ChannelWrite` 经 `CONFIG_KEY_READ`/`CONFIG_KEY_SEND` 与任务执行上下文耦合。
- **下游 → stream-framework**：`StreamMessagesHandler`/`StreamToolCallHandler` 把 LLM/工具事件推入流多路复用器。
- **外部 → checkpoint 后端**：`saver.put`/`get_tuple`/`get_delta_channel_history` 由 `libs/checkpoint*/` 实现。

## 9. 语言专项适配口径

本项目为纯 Python 框架，按纯 Python 专项清单分析：

1. **能力缝归组**：`CONFIG_KEY_READ`/`CONFIG_KEY_SEND` 闭包 + `ChannelRead`/`ChannelWrite` Runnable 是"注入式读写桥"能力缝——引擎在创建任务时把闭包塞进 RunnableConfig，节点代码无感知地通过 `Command`/返回值触发写入。
2. **注册表/工厂**：无全局注册表；`map_command` 是 `Command.goto` 字段值（Send/str）的分派工厂。
3. **双轨实现**：`channels_from_checkpoint` 与 `achannels_from_checkpoint` 是同步/异步双轨；普通通道"直存值"与 DeltaChannel"回放重建"是 checkpoint 水化的两条轨迹。
4. **可选依赖**：`_StreamingCallbackHandler`/`_V2StreamingCallbackHandler` 用 try/except ImportError 降级为 `object`（`_messages.py:24-33`）；LangChain 为重依赖。
5. **配置驱动分派**：`stream_mode` 是否含 `messages`/`tools` 决定挂接哪个回调；DeltaChannel 行为由 `snapshot_frequency` 配置驱动。

## 10. 图表清单与质量

| 图 | 文件 | 类型 | archify 质量档 |
| --- | --- | --- | --- |
| Pregel IO 与 Checkpoint 结构 | `pregel-io-checkpoint-architecture.html` | architecture | standard |
| Checkpoint 读写数据管道 | `pregel-io-checkpoint-dataflow.html` | dataflow | standard |

- 降档说明：architecture 图降 standard；本次新增的 dataflow 图（get_tuple→反序列化→通道水化→节点执行→apply_writes→序列化→put）因"同 stage 纵向边标签压节点/标签重叠"改用 vertical-channel 路由后仍未全过 showcase，按规范降 standard 渲染（均成功，约 806KB 自包含）。
- JSON IR 源：`json/pregel-io-checkpoint-architecture.json`、`json/pregel-io-checkpoint-dataflow.json`。
- 本次迭代新增 dataflow 图，把 checkpoint 读写管道从文字化升级为图；不另出 sequence：读写管道是数据流动而非消息交互，时序已由 pregel-engine 域级时序图覆盖。
