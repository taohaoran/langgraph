# 部署与本地执行（cli-deploy-exec）

> 本文是 `cli-tooling` 域下的叶子子系统文档。域级总览见 `../cli-tooling.md`。
> 本文只展开 `deploy` 命令组的部署编排、子进程执行基础设施与源码打包；
> CLI 命令注册与配置校验见 `../cli-core/cli-core.md`，Docker 镜像构建细节见
> `../docker-packaging/docker-packaging.md`。
>
> 源码基准：`libs/cli/langgraph_cli/`（`deploy.py` 2086 行、`exec.py` 174 行、`archive.py` 138 行）。

## 1. 功能清单

| 能力 | 说明 | 源码路径 |
| --- | --- | --- |
| `deploy` 命令组 | 构建并部署到 LangSmith Deployment；无参时直接执行部署 | `deploy.py:1542-1566`（`DeployGroup` + 转发 `_deploy_cmd`） |
| 部署编排四阶段 | 预检 → 解析/创建 deployment → 构建（本地/远端）→ 轮询结果 | `deploy.py:1571` `_deploy_cmd` |
| 本地构建路径 | docker buildx 构建 → 申请 push token → login/tag/push → update_deployment | `deploy.py:930` `_run_local_build` |
| 远端构建路径 | 打源码 tarball → 请求上传 URL → PUT GCS → 触发远端构建 | `deploy.py:1129` `_run_remote_build` |
| 构建模式自动选择 | 有 `--remote` 强制远端；`--no-remote` 强制本地；否则按能否本地构建自动判定 | `deploy.py:562` `_resolve_build_mode` |
| 源码打包 | 生成 `source.tar.gz`，含本地依赖与 monorepo 额外上下文，大小上限 200MB | `archive.py:62` `create_archive` |
| 子进程执行 | `asyncio` 异步子进程 + 按行监视 stdout/stderr + 信号转发 | `exec.py:31` `subp_exec`、`exec.py:126` `monitor_stream` |
| Runner 上下文 | `asyncio.Runner()` 跨版本兼容封装 | `exec.py:11` `Runner` |
| 结构化输出 | 文本/JSON 双模式 step/info/warn/result/heartbeat 输出 | `deploy.py:127` `_Emitter` |
| 结果抽象 | 把本地/远端构建统一为 `BuildResult` 喂给共享等待尾 | `deploy.py:105` `BuildResult` |
| revision 轮询 | 轮询 `get_revision` 至终态/超时，支持中断回调 | `deploy.py:703` `_poll_revision_status` |
| 运维子命令 | `deploy list` / `revisions list` / `delete` / `logs` | `deploy.py:1793/1840/1881/1972` |
| GCS 上传 | 带进度条的分片 PUT 到签名 URL | `deploy.py:860` `_upload_to_gcs` |
| push 重试 | `docker push` 失败最多重试 3 次 | `deploy.py:1080-1101` |

## 2. 核心类型与接口清单

| 类型 / 函数 | 位置 | 职责 |
| --- | --- | --- |
| `DeployGroup(NestedHelpGroup)` | `deploy.py:1400` | 覆盖 `parse_args`，支持无参转发与透传未知参数 |
| `_deploy_cmd` | `deploy.py:1571` | 部署主编排函数（被 `deploy` group 或单独调用） |
| `BuildResult`（dataclass） | `deploy.py:105` | 构建结果载体：updated/progress_message/超时/on_poll/on_interrupt |
| `_Emitter` | `deploy.py:127` | 输出抽象，`json_mode` 时写结构化 JSON 行 |
| `Runner`（contextmanager） | `exec.py:11` | 提供 asyncio 事件循环上下文 |
| `subp_exec(...)` | `exec.py:31` | 异步执行外部命令，可选 stdin、收集输出、行回调 |
| `monitor_stream(...)` | `exec.py:126` | 逐行读 `StreamReader`，处理 `LimitOverrunError` |
| `create_archive(...)` | `archive.py:62` | 上下文管理器，产出 `(archive_path, size, config_rel)` |
| `_tar_filter` | `archive.py:19` | 剔除符号/硬链接与 `..` 穿越路径（防 Zip-Slip） |
| `HostBackendClient` | `host_backend.py`（见 docker-packaging/域级） | 控制面 API 客户端：create/update/list/push_token |

## 3. 关键调用链

**链一：`langgraph deploy` 本地构建主路径**（`deploy.py:1571`）
1. **预检**：`validate_deploy_commands` 校验 install/build 命令；`validate_config_file` 校验配置；`_parse_env_from_config` 解析 env；`_secrets_from_env` 提取密钥；`_resolve_build_mode` 决定本地/远端。
2. **解析部署**：`_create_host_backend_client` → `_resolve_deployment`（按 id 或 name 查）→ 不存在则 `_create_deployment`。
3. **本地构建**（`_run_local_build`）：非 x86_64 用 `docker buildx build --platform linux/amd64 --load`；`request_push_token` 取 token 与 registry；`docker --config <临时> login -u oauth2accesstoken`；`docker tag` → `docker push`（最多 3 次重试）；`_resolve_pushed_image_digest` 取摘要；`update_deployment(image)`。
4. **等待**：`_poll_revision_status` 轮询 `get_revision`（超时 300s、间隔 1s），终态为 `BUILD_READY/ACTIVE/BUILD_FAILED`；失败且未 verbose 时拉最近 30 行构建日志。

**链二：远端构建分支**（`_run_remote_build`）
1. `create_archive` 打 `source.tar.gz`（含本地依赖与 monorepo 额外上下文，保留公共祖先相对布局）。
2. `request_upload_url` 取签名 URL → `_upload_to_gcs` 带进度上传。
3. `update_deployment_internal_source(source_tarball_path, config_path, ...)` 触发远端构建。
4. 轮询超时放宽到 900s、间隔 3s，`on_poll=_stream_build_logs` 流式拉构建日志；中断时提示"远端继续构建"。

**链三：子进程执行**（`exec.py`）
1. `Runner()` 进事件循环；`subp_exec` 用 `asyncio.create_subprocess_exec` 起进程。
2. `asyncio.gather` 并行监视 stdout/stderr（`monitor_stream`）并喂 stdin。
3. 注册 SIGINT/SIGTERM 处理器转发给子进程；非零退出（除 130 中断）抛 `click.exceptions.Exit(returncode)`。

## 4. 配置项

| flag / 选项 | 默认 / 行为 | 位置 |
| --- | --- | --- |
| `--api-key` / `--host-url` | 控制面凭证；host 默认 `https://api.host.langchain.com` | `deploy.py:1343` |
| `--deployment-id` / `--name` | 定位部署；都缺省时交互询问（`--no-input` 用目录名兜底） | `deploy.py:1621-1635` |
| `--remote` / `--no-remote` / 默认 | 强制远端 / 强制本地 / 自动按 `can_build_locally` | `deploy.py:562` |
| `--image` | 用现成镜像，强制本地构建且跳过 build | `deploy.py:1639` |
| `--no-wait` | 触发后不轮询直接返回 | `deploy.py:1731` |
| `--json-output` | `_Emitter` 输出结构化 JSON 行 | `deploy.py:1594` |
| `--no-input` | 关闭所有交互 prompt | `deploy.py:1595` |
| 本地构建轮询 | 超时 300s / 间隔 1s | `deploy.py:1123-1124` |
| 远端构建轮询 | 超时 900s / 间隔 3s | `deploy.py:1218-1219` |
| 归档大小 | 警告 50MB / 硬上限 200MB | `archive.py:15-16` |

## 5. 错误与重试语义

- `docker push` 失败按 `click.exceptions.Exit` 捕获，最多重试 3 次后才抛出（`deploy.py:1080-1101`）。
- `request_push_token` 返回 400 且提示 `internal_docker` 时，改写为友好报错（部署非本命令创建）（`deploy.py:1013-1024`）。
- `_poll_revision_status` 捕获 `KeyboardInterrupt`：调 `on_interrupt`（远端构建提示继续）后 `Exit(1)`；轮询网络错误不视为终态失败，继续循环到超时。
- 归档校验：写入后回读确认 `config_rel` 在包内，缺失抛 `ClickException`；超 200MB 直接拒绝。
- `tracked_packages` 扫描失败仅 warn，绝不阻断部署（`deploy.py:1682-1686`）。
- 子进程非零退出（除 130 中断）透传 `click.exceptions.Exit(returncode)`。

## 6. 并发细节

- 整个编排运行在 `asyncio.Runner` 事件循环中；`subp_exec` 用 `asyncio.gather` 并行监视 stdout/stderr/stdin 三路。
- SIGINT/SIGTERM 通过 `loop.add_signal_handler` 转发给子进程，退出时 `remove_signal_handler` 还原（`exec.py:72-121`）；Windows 走 `signal.signal` 兼容路径。
- 无线程池；`_upload_to_gcs` 为同步 HTTP 上传经 `_ProgressReader` 回调进度。轮询为同步循环（`while time.time() < deadline`）。
- 进度条 `Progress` 后台线程与 deploy 主流程并行（见 cli-core）。

## 7. 系统边界

**In-Scope（本仓库源码内）**
- `deploy.py` 编排与子命令、`exec.py` 子进程执行、`archive.py` 打包；调用 `host_backend.py` 客户端。

**Out-of-Scope（不在本仓库源码内）**
- LangSmith Deployment 控制面 API（`api.host.langchain.com`）、镜像 registry、GCS 签名上传、本地 Docker/docker buildx——均外部。
- 实际镜像构建逻辑在 `docker.py`（docker-packaging 叶子）；`can_build_locally`/`find_tracked_packages` 等辅助在本文件内。

## 8. 与相邻子系统交互

- 上游：`cli.py` 经 `cli.add_command(deploy)` 接入本命令组。
- 本叶子 → docker-packaging：`_run_local_build` 调 `docker.build_docker_image`；`_resolve_build_mode` 用 `can_build_locally`。
- 本叶子 → cli-core：复用 `Runner`/`Progress`/`log_command`/`validate_config_file`。
- 下游外部：HostBackendClient → 控制面 API；`subp_exec` → docker CLI；`_upload_to_gcs` → GCS。

## 9. 语言专项适配口径（纯 Python）

- **能力缝**：`BuildResult` 把本地/远端两条构建路径的差异收敛为数据字段，共享等待尾——策略对象 + 结果对象模式。`_Emitter` 抽象文本/JSON 双输出（同 `_async/_sync` 双轨思路）。
- **子进程模型**：`asyncio` 子进程 + 三路 `gather`，信号转发兼容跨平台；`monitor_stream` 自行处理 `LimitOverrunError`（标准库 `readline` 的边界修补）。
- **安全**：`_tar_filter` 剔除软/硬链接与 `..` 路径；push 用临时 `--config` 目录隔离系统 credential helper。
- **配置驱动**：`--remote`/`--no-remote`/`--image` 组合决定本地/远端分支，是命令级静态分派。

## 10. 图表清单与质量

| 图 | 文件 | 类型 | archify 质量档 |
| --- | --- | --- | --- |
| 部署流水线组件 | `cli-deploy-exec-architecture.html` | architecture | standard（向下边标签重叠，加 `labelDy` 后通过；showcase 仍有布局间距诊断） |
| deploy 生命周期时序 | `cli-deploy-exec-sequence.html` | sequence | standard（同通道消息需 ≥28px 间距，重排后通过；showcase 未全过） |

本叶子不补第三张图：部署流程的多角色泳道已由域级 `cli-tooling-workflow.html` 表达，且本地/远端构建分支已在 sequence 与 architecture 中覆盖，按资源节省原则省略。JSON IR 源：`json/cli-deploy-exec-architecture.json`、`json/cli-deploy-exec-sequence.json`。
