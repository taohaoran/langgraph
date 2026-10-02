# sdk-auth-encryption（认证授权与静态加密契约）

> 本文是 `sdk-python` 域下的叶子子系统文档，覆盖 `Auth`（自定义认证/授权）与
> `Encryption`（at-rest 加解密）两个可插拔处理器注册表。它们由用户在独立文件中实例化
> 并注册 `langgraph.json`，再由 LangGraph Server 服务端加载回调。
>
> 源码基准：`libs/sdk-py/langgraph_sdk/auth/` 与 `encryption/`，git HEAD `e539ac12`（2026-09-09）。

## 1. 功能清单

| 能力 | 说明 | 源码路径 |
|---|---|---|
| 认证注册 | `@auth.authenticate` 注册 token 校验处理器 | `auth/__init__.py:13`（`Auth`） |
| 授权规则 | `@auth.on.<resource>.<action>` 分级注册 | `auth/__init__.py:732`（`_On`） |
| 资源动作挂点 | assistants/threads/crons/store 各 on.* | `auth/__init__.py:488/512/548/592` |
| 处理器注册 | `_register_handler`/`_validate_handler` | `auth/__init__.py:867/889` |
| studio 用户判断 | `is_studio_user` | `auth/__init__.py:918` |
| blob 加解密 | `@encrypt.blob`/`@decrypt.blob` | `encryption/__init__.py:70/137/208` |
| JSON 加解密 | `@encrypt.json`/`@decrypt.json`（保持 key） | `encryption/__init__.py:208` |
| beta 警告 | `LangGraphBetaWarning` | `encryption/__init__.py:25/30` |

## 2. 核心类型与接口清单

| 类型 / 函数 | 位置 | 职责 |
|---|---|---|
| `Auth` | `auth/__init__.py:13` | 认证/授权注册表，`on` 为资源动作挂点入口 |
| `_On` | `auth/__init__.py:732` | `auth.on` 命名空间，暴露 threads/assistants/crons/store |
| `_ResourceActionOn`/`_ResourceOn` | `auth/__init__.py:318/350` | 资源×动作级处理器挂点 |
| `Encryption` | `encryption/__init__.py:208` | 加解密注册表，`encrypt`/`decrypt` 为 blob/json 挂点 |
| `_EncryptDecorators`/`_DecryptDecorators` | `encryption/__init__.py:70/137` | 加解密装饰器集合 |
| `BaseUser`/`MinimalUserDict` | `auth/types.py` | 用户身份与权限类型契约 |

## 3. 关键调用链

**链一：请求处理授权匹配（`auth/__init__.py:96-106`）**

1. 每个请求先执行 `@auth.authenticate`（`auth/__init__.py:98`）。
2. 授权时按“最具体匹配”选处理器：精确 `on.<res>.<act>` → 资源级 `on.<res>` → 全局 `on` → 无则放行（`auth/__init__.py:99-103`）。
3. 处理器接收 `AuthContext`（含 `ctx.user`）与 value，可改写 value（如注入 owner metadata）。

**链二：落库 JSON 加密（`encryption/__init__.py:208`）**

1. 服务端落库前调 `@encrypt.json`，处理器对 dict 逐 key 变换值（`encryption/__init__.py:282-290`）。
2. 约束：JSON 加密**不得增删 key**，只变换值（`encryption/__init__.py:242-262`）。
3. 解密时 `@decrypt.json` 按前缀/标记还原；且 `(model, field)` 在加解密间可能不一致（`encryption/__init__.py:321-327`）。

## 4. 配置项

| flag / option | 默认 / 行为 | 位置 |
|---|---|---|
| `langgraph.json: auth.path` | 指向 `./auth.py:my_auth` 实例 | `auth/__init__.py:35-37` |
| `langgraph.json: encryption.path` | 指向 `./encryption.py:my_encryption` | `encryption/__init__.py:233-235` |
| 默认拒绝策略 | 注册全局 `on` 返回 False 实现 default-deny | `auth/__init__.py:61-64` |
| beta | Encryption 为 beta API，发 `LangGraphBetaWarning` | `encryption/__init__.py:25/30` |

## 5. 错误与重试语义

- **认证失败**：用户处理器抛 `Auth.exceptions.HTTPException(status_code=401)`（`auth/__init__.py:53-55`）。
- **重复注册**：Encryption 处理器重复注册抛 `DuplicateHandlerError`（`encryption/__init__.py:38`）。
- **处理器签名校验**：`_validate_handler` 校验签名（`auth/__init__.py:889`、`encryption/__init__.py:44`）。
- 无重试逻辑：本叶子是同步回调注册表，失败即中断请求/落库。

## 6. 并发细节

- 本叶子无独立线程/锁；处理器由 LangGraph Server 在请求/落库路径中按需调用。
- 处理器为 async 函数，在服务端事件循环中执行。

## 7. 系统边界

**In-Scope（本仓库源码内）**
- Auth/Encryption 注册表、装饰器挂点、类型契约、签名校验。

**Out-of-Scope（不在本仓库源码内）**
- **LangGraph Server**：实际加载 `langgraph.json`、在请求/落库路径回调处理器的服务端，不在本仓库源码内。
- **用户处理器实现**：`verify_token`、实际加解密算法由用户提供，不在本仓库源码内。
- **密码学后端**：AESGCM 等实际算法在用户处理器中；本仓库只定义契约。

## 8. 与相邻子系统交互

- LangGraph Server（外部）→ `Auth`/`Encryption`（本叶子）回调用户注册的处理器。
- 本叶子 `auth.types.BaseUser` → `sdk-client` 的 `runtime.py`（ServerRuntime.user）。
- 本叶子 → 用户代码（外部）。

## 9. 语言专项适配口径（纯 Python）

- **能力缝/注册表与装饰器扩展点**：`Auth`/`Encryption` 是典型“装饰器注册 + 命名空间挂点”能力缝——`auth.on.threads.create`、`encrypt.json` 是链式装饰器注册点。
- **配置驱动加载**：由 `langgraph.json` 的 `auth.path`/`encryption.path` 配置驱动服务端加载实例。
- **协议/泛型挂点**：`_ActionHandler(Protocol)`、`_ResourceActionOn(Generic)` 提供类型化挂点（`auth/__init__.py:309/318`）。
- 双轨不适用：本叶子无异步/同步双轨，处理器本身为 async。

## 10. 图表清单与质量

| 图 | 文件 | 类型 | archify 质量档 |
|---|---|---|---|
| 组件架构图 | `sdk-auth-encryption-architecture.html` | architecture | showcase |
| 认证加密时序 | `sdk-auth-encryption-sequence.html` | sequence | showcase |

- JSON IR 源文件位于 `json/` 目录。
- 省略 dataflow：加解密是回调契约，无独立数据管道。
- 省略 workflow/lifecycle：无多角色泳道分支，也无单实体状态机。
