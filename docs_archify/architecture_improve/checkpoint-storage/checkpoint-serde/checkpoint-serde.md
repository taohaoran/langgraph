# checkpoint-serde（checkpoint-serde）

> 本文是 `checkpoint-storage` 域下的叶子子系统文档（**本轮新增独立成叶**）。本叶子是 LangGraph
> 检查点的**序列化层**：定义对象↔`(type, bytes)` 的类型化编解码协议、默认的 `JsonPlusSerializer`
> （基于 ormsgpack）、加密序列化包装、安全反序列化白名单与事件钩子。它被 `BaseCheckpointSaver`
> 及其所有后端（见 checkpoint-base / postgres / sqlite 叶子）通过 `self.serde` 调用。
>
> 源码基准：`langgraph-checkpoint`（libs/checkpoint），commit `e539ac12`，版本 4.2.0。

## 1. 功能清单

| 能力 | 说明 | 源码路径 |
|---|---|---|
| 序列化协议 | `SerializerProtocol`（`runtime_checkable` Protocol）：`dumps_typed(obj)->(type,bytes)` / `loads_typed((type,bytes))->obj`；`UntypedSerializerProtocol`（旧式 dumps/loads） | `serde/base.py:15`、`:6` |
| 旧式适配 | `SerializerCompat`：把旧式 dumps/loads 适配成 typed 协议；`maybe_add_typed_methods` 自动判定是否需要包装 | `serde/base.py:29`、`:40` |
| 加密协议 | `CipherProtocol`：`encrypt(plaintext)->(ciphername, bytes)` / `decrypt(ciphername, bytes)->plaintext` | `serde/base.py:51` |
| 默认序列化器 | `JsonPlusSerializer`：以 ormsgpack 为主，type tag 分流 null/bytes/bytearray/msgpack/pickle；反序列化走 `_reviver` + ext_hook | `serde/jsonplus.py:82`、`:258`、`:273` |
| msgpack 扩展类型 | ext code 0–7：单参/位置参/关键字参构造、单参方法调用、pydantic v1/v2、numpy 数组、`_DeltaSnapshot` 增量快照 | `serde/jsonplus.py:295-302`、`:305`、`:633` |
| 安全白名单 | `SAFE_MSGPACK_TYPES` / `SAFE_MSGPACK_METHODS` 内置安全类型集合；`allowed_json_modules`/`allowed_msgpack_modules` 构造参数；`LANGGRAPH_STRICT_MSGPACK` 环境变量 | `serde/_msgpack.py:21`、`:90`、`:12` |
| lc:2 JSON 复活 | `_revive_lc2`：把 LangChain lc:2 envelope 经 `importlib.import_module` 复活为对象；`_check_allowed_json_modules` 白名单校验 | `serde/jsonplus.py:181`、`:223` |
| 加密序列化 | `EncryptedSerializer`：先 inner serde 编码再 cipher 加密，type tag 写成 `"{typ}+{ciphername}"`；`from_pycryptodome_aes` AES-EAX 工厂 | `serde/encrypted.py:8`、`:17`、`:38` |
| 类型常量 | 特殊通道写入标记 `ERROR/SCHEDULED/INTERRUPT/RESUME/TASKS`；`_DeltaSnapshot` NamedTuple；`ChannelProtocol`/`SendProtocol` 协议镜像 | `serde/types.py:12-16`、`:19`、`:39`、`:58` |
| 事件钩子 | `SerdeEvent` + 模块级监听器列表（带锁），白名单/反序列化事件可被外部观测 | `serde/event_hooks.py:13`、`:26`、`:41` |

## 2. 核心类型与接口清单

| 类型 / 函数 | 位置 | 职责 |
|---|---|---|
| `SerializerProtocol` | `serde/base.py:15` | **契约核心**。typed 编解码协议；`BaseCheckpointSaver.serde` 的类型约束位置在消费方基类。 |
| `UntypedSerializerProtocol` | `serde/base.py:6` | 旧式 `dumps/loads` 协议，供 `SerializerCompat` 适配旧 pickle/json/orjson 用法。 |
| `maybe_add_typed_methods` | `serde/base.py:40` | 工厂函数：若传入对象未实现 typed 协议则包一层 `SerializerCompat`，否则原样返回。 |
| `CipherProtocol` | `serde/base.py:51` | 加解密抽象；`EncryptedSerializer` 依赖它，具体 cipher（Pycryptodome AES）是外部注入。 |
| `JsonPlusSerializer` | `serde/jsonplus.py:82` | 默认实现。`__init__`（`:97`）接收 `pickle_fallback`/`allowed_json_modules`/`allowed_msgpack_modules`/自定义 ext_hook；`with_msgpack_allowlist`（`:128`）返回合并白名单的浅克隆。 |
| `_msgpack_default` / `_create_msgpack_ext_hook` | `serde/jsonplus.py:305`、`:546` | msgpack 编码时的对象→Ext 转换器；解码时的 code→对象复活器（内含 `_check_allowed` 白名单）。 |
| `EncryptedSerializer` | `serde/encrypted.py:8` | 装饰器式包装：`dumps_typed` 编码后加密、type tag 拼 cipher 名；`loads_typed` 按 `"+"` 分隔判断是否加密。 |
| `_DeltaSnapshot` | `serde/types.py:19` | 增量通道快照 blob，经 ext code 7 落库；祖先链回溯遇它即停止（见 checkpoint-base）。 |
| `register_serde_event_listener` | `serde/event_hooks.py:26` | 返回注销函数；监听器失败被隔离并 warning，不影响主流程。 |

## 3. 关键调用链

**写链 `JsonPlusSerializer.dumps_typed`（`jsonplus.py:258`）：**
1. `None` → 返回 `("null", b"")`；`bytes`/`bytearray` 原样带 type tag 返回（`:259-264`）。
2. 其他对象尝试 `("msgpack", _msgpack_enc(obj))`；`_msgpack_default`（`:305`）按对象类型映射到 ext code（pydantic v2→5、普通构造→0/1/2、numpy→6、`_DeltaSnapshot`→7）。
3. 若抛 `ormsgpack.MsgpackEncodeError` 且 `pickle_fallback=True`，降级为 `("pickle", pickle.dumps(obj))`；否则原样抛出（`:268-271`）。

**读链 `JsonPlusSerializer.loads_typed`（`jsonplus.py:273`）：**
1. 按 type tag 分流：`null`→None、`bytes`→原样、`bytearray`→包装、`json`→`json.loads(object_hook=self._reviver)`、`msgpack`→`ormsgpack.unpackb(ext_hook=self._unpack_ext_hook)`（`:275-286`）。
2. msgpack 分支经 `ext_hook`（`:633`）按 code 复活：code 7→`_DeltaSnapshot`、code 0/1/2→构造器、code 3→方法调用（仅 `SAFE_MSGPACK_METHODS`）、code 4/5→pydantic、code 6→numpy。
3. `json` 分支经 `_reviver`（`:160`）处理 lc:2 envelope：先 `_is_safe_json_type` 判定安全集合，再 `_check_allowed_json_modules`（`:223`）拒绝未授权 import。

**加密读写 `EncryptedSerializer`（`encrypted.py:17` / `:26`）：**
1. 写：`self.serde.dumps_typed(obj)` 得 `(typ, data)` → `self.cipher.encrypt(data)` 得 `(ciphername, ciphertext)` → 返回 `(f"{typ}+{ciphername}", ciphertext)`（`:20-24`）。
2. 读：若 type 不含 `"+"` 说明未加密，直接走 inner `loads_typed`；否则按 `"+"` 拆出原 type 与 ciphername，`cipher.decrypt` 后交 inner 反序列化（`:29-36`）。

## 4. 配置项

| flag / option | 默认 / 行为 | 位置 |
|---|---|---|
| `pickle_fallback` | `False`；为 True 时 msgpack 编码失败降级 pickle，反序列化接受 `"pickle"` tag | `jsonplus.py:100`、`:269`、`:287` |
| `allowed_json_modules` | `None`→仅安全集合可复活 JSON 构造器；`True`→全放行（危险）；具体元组白名单→精确匹配 | `jsonplus.py:101`、`:233-255` |
| `allowed_msgpack_modules` | 未传时：`LANGGRAPH_STRICT_MSGPACK=true`→None（仅安全集合）；否则 `True`（全放行+告警） | `jsonplus.py:107-114` |
| `LANGGRAPH_STRICT_MSGPACK` | 默认 false；true 时反序列化限制在 `SAFE_MSGPACK_TYPES` | `_msgpack.py:12` |
| `LANGGRAPH_AES_KEY` | AES 密钥环境变量；必须 16/24/32 字节；`from_pycryptodome_aes` 未显式传 key 时读取 | `encrypted.py:54-59` |
| AES 模式 | 默认 `AES.MODE_EAX`，可 kwargs 覆盖 | `encrypted.py:62-63` |
| `__unpack_ext_hook__` | 注入自定义 ext 复活钩子；注入后 `with_msgpack_allowlist` 不再重建默认 hook | `jsonplus.py:105`、`:121` |

## 5. 错误与重试语义

- **反序列化安全失败**：未授权模块在 `_check_allowed_json_modules` 抛 `InvalidModuleError`（`jsonplus.py:234`/`:249`）；非安全类型的 lc:2 envelope 复活失败时仅 `logger.warning` 并返回 None（`:171-177`、`:214-221`），不中断恢复。
- **编码失败**：`ormsgpack.MsgpackEncodeError` 时若 `pickle_fallback=False` 直接上抛（`:268`）；开启 fallback 才降级 pickle。
- **未知 type tag**：`loads_typed` 末尾抛 `NotImplementedError(f"Unknown serialization type: {type_}")`（`:290`）。
- **加密密钥缺失**：`from_pycryptodome_aes` 未装 pycryptodome 抛 `ImportError`（提示 `pip install pycryptodome`，`:46`）；无 key 且无 env 抛 `ValueError`（`:55`）。
- **监听器容错**：`emit_serde_event` 中单个监听器异常被捕获并 warning，不影响其他监听器与主流程（`event_hooks.py:48-52`）。
- **无重试**：纯编解码 CPU 操作，无网络 IO，无退避重试。

## 6. 并发细节

- **事件钩子锁**：`event_hooks.py` 用模块级 `Lock()` 保护监听器列表；`emit` 时先在锁内拷贝监听器 tuple 再锁外逐个调用，缩短临界区（`:46-52`）。
- **告警去重**：`_warned_unregistered_types`/`_warned_blocked_types` 集合做进程内告警去重，上限 `_MAX_WARNED_TYPES=1000`；注释明确"竞态线程可能各打印一次"，是 best-effort 而非强一致（`jsonplus.py:55-57`）。
- **无共享可变状态**：`JsonPlusSerializer` 实例的 allowlist 为不可变 frozenset/tuple；`with_msgpack_allowlist` 返回浅克隆而非原地修改（`jsonplus.py:151`），便于并发 saver 派生。
- **同步纯函数**：dumps/loads 均为同步 CPU 操作，无线程/协程启停；后端在 async 路径直接调用（内存无阻塞），DB 后端在连接内调用。

## 7. 系统边界

**In-Scope（本仓库源码内）**
- `libs/checkpoint/langgraph/checkpoint/serde/`：协议、默认序列化器、加密包装、安全白名单、事件钩子、类型常量。

**Out-of-Scope（不在本仓库源码内）**
- `ormsgpack`：msgpack 高速编解码库（PyPI 依赖，顶层硬 import `jsonplus.py:30`）。
- `pycryptodome`（`Crypto.Cipher.AES`）：仅 `from_pycryptodome_aes` 时按需 import，未装抛 ImportError。
- `langchain_core.load.load.Reviver` / `BaseMessage` 等：langchain_core 外部库，被复活。
- 实际存储介质（Postgres/SQLite/内存表）：由各后端叶子负责，本层只产出 `(type, bytes)`。

## 8. 与相邻子系统交互

- **上游 → 本叶子**：`BaseCheckpointSaver.__init__` 接收可选 `serde`，默认 `JsonPlusSerializer()`，经 `maybe_add_typed_methods` 包装（见 checkpoint-base 叶子 `base/__init__.py:210-217`）。
- **本叶子 → 下游**：所有后端（InMemory/Postgres/Sqlite）把 `serde.dumps_typed` 产物 `(type, bytes)` 落库；读时把库里的 `(type, bytes)` 交回 `serde.loads_typed`。
- **本叶子 → 外部库**：编码调用 `ormsgpack`；加密路径调用注入的 `CipherProtocol`（Pycryptodome AES）；复活 lc:2 经 `importlib` 动态 import langchain_core 类。
- **反向扩展**：`BaseCheckpointSaver.with_allowlist`（checkpoint-base 叶子）把业务侧额外允许的类经 `JsonPlusSerializer.with_msgpack_allowlist` 合并进白名单。

## 9. 语言专项适配口径

本叶子是纯 Python 框架序列化层的典型样本：
- **能力缝归组**：以 `SerializerProtocol` Protocol 定义"对象编解码"能力缝，`EncryptedSerializer` 用装饰器模式套在任意 inner serde 外（类型 tag 拼 `"+"` 实现透明加解密），`SerializerCompat` 适配旧式接口——三种形态围绕同一 Protocol。
- **注册表/分派**：`dumps_typed`/`loads_typed` 按 **type tag 字符串分派**（null/bytes/bytearray/msgpack/json/pickle），msgpack 内部再按 **ext code 整数分派**（0–7）——是"字符串/整数→处理函数"的注册表路由，而非类继承。
- **可选依赖与懒加载**：`ormsgpack` 是硬依赖（顶层 import）；`pycryptodome` 是**真正的可选依赖**——`from_pycryptodome_aes` 内 try import，缺失才抛错（`encrypted.py:43-48`），符合"延迟到使用时才触发 ImportError"的懒加载口径。
- **安全白名单即配置驱动分派**：反序列化行为由构造参数 `allowed_*_modules` + 环境变量 `LANGGRAPH_STRICT_MSGPACK` 决定，同一 serializer 代码在"严格/宽松"两种安全 posture 下走不同分支——配置驱动的静态多态。
- **双轨**：本层无同步/异步双轨（纯 CPU），双轨在后端存储层（见 postgres/sqlite 叶子）。

## 10. 图表清单与质量

| 图 | 文件 | 类型 | archify 质量档 |
|---|---|---|---|
| 序列化器族谱架构图 | `checkpoint-serde-architecture.html` | architecture | standard |
| 序列化数据流图 | `checkpoint-serde-dataflow.html` | dataflow | showcase |

降档披露：架构图落 **standard**（原因：升级 archify 的 `composition/desktop-readability` 合成检查对 9 组件+9 连线的族谱密度敏感；已按诊断加宽 viewBox 至 `[1440,820]`）。数据流图一次过 **showcase**（首轮因 5 stage 横向超宽报错，加宽 `viewBox` 至 `[1200,520]` 后通过）。两图 render 退出码均 0、HTML 均 >700KB。

JSON IR 源文件位于 `json/` 目录。

省略说明：本叶子不补 sequence 图——dumps/loads 是单进程内函数调用，无多参与方跨网络消息交互；不补 lifecycle 图——序列化对象本身无多状态机（type tag 分派已在 dataflow 图中表达）。
