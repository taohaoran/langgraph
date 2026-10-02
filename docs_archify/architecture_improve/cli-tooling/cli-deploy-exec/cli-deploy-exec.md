# cli-deploy-exec（deploy/exec 部署执行链）

> 本文是 `cli-tooling` 域下的叶子子系统文档，覆盖 `langgraph deploy` 命令组、子进程执行器、
> 宿主后端 HTTP 客户端、源码打包、uv 锁解析、依赖追踪与进度展示。命令入口与配置加载见
> `cli-core` 叶子；Dockerfile 生成与镜像构建核心见 `docker-packaging` 叶子，本文不展开。
>
> 源码基准：`libs/cli/langgraph_cli/`，git HEAD `e539ac12`（2026-09-09）。

## 1. 功能清单

| 能力 | 说明 | 源码路径 |
|---|---|---|
| deploy 命令组 | `@click.group` 定义 `deploy`，含 deploy/list/revisions/delete/logs 子命令 | `deploy.py:1542-1558`、`deploy.py:1816-1972` |
| 部署编排主流程 | `_deploy_cmd`：预检→解析/创建 deployment→本地或远端构建→轮询终态 | `deploy.py:1571-1777` |
| 构建模式分派 | `_resolve_build_mode` 决定本地构建还是远端构建 | `deploy.py:562` |
| 本地构建 | `_run_local_build`：调 Docker 构建并推送 image_uri | `deploy.py:930` |
| 远端构建 | `_run_remote_build`：打包 tarball → 取签名 URL → 上传 GCS → 触发后端构建 | `deploy.py:1129` |
| 子进程执行器 | `subp_exec`/`Runner`：asyncio 子进程、信号转发、stdout 回调 | `exec.py:31-123` |
| 流监控 | `monitor_stream`：按行消费 stdout/stderr，处理 `LimitOverrunError` | `exec.py:126-174` |
| 宿主后端客户端 | `HostBackendClient`：httpx 同步 JSON 客户端，封装 /v2/deployments REST | `host_backend.py:19-205` |
| 源码打包 | `create_archive`：tar.gz 打包项目与本地依赖，含 .dockerignore 过滤与大小校验 | `archive.py:62-138` |
| uv 锁计划 | `_plan_uv_lock_workspace`/`python_config_to_docker_uv_lock`：解析 uv.lock 工作区并生成 Docker 指令 | `uv_lock.py:623`、`uv_lock.py:868` |
| 依赖追踪 | `find_tracked_packages`：扫描本地依赖中的受追踪包（如 google-adk）版本 | `dependency_tracking.py:98-141` |
| 进度 spinner | `Progress`：后台线程旋转光标，TTY/非 TTY 双模式 | `progress.py:7-108` |
| 结果/表格格式化 | deployments/revisions/logs 表格、时间戳、日志级别着色 | `deploy.py:414-489` |

## 2. 核心类型与接口清单

| 类型 / 函数 | 位置 | 职责 |
|---|---|---|
| `DeployGroup` / `NestedHelpGroup` | `deploy.py:1355-1428` | Click 组，拍平子命令帮助；`deploy` 组在 `cli.py:237` 被 add_command |
| `_deploy_cmd` | `deploy.py:1571` | deploy 主命令体，四阶段编排 |
| `BuildResult` | `deploy.py:105` | 数据类：承载 build 后的 updated/progress_message/timeout/on_poll 等 |
| `_Emitter` | `deploy.py:127` | 输出抽象（普通文本 / JSON 模式），`em.note/info/error/log/warn` |
| `HostBackendClient` | `host_backend.py:19` | 同步 httpx 客户端；`create_deployment/update_deployment/request_upload_url/list_revisions/get_build_logs` 等 |
| `HostBackendError` | `host_backend.py:11` | `click.ClickException` 子类，携带 status_code |
| `Runner()` | `exec.py:12` | contextmanager，封装 `asyncio.Runner`（旧 Python 回退自实现） |
| `subp_exec(...)` | `exec.py:31` | 异步执行外部命令，支持 stdin/stdout 回调、verbose、collect |
| `create_archive(...)` | `archive.py:62` | contextmanager，产出 `(archive_path, file_size, config_rel)` |
| `DockerBuildPlan` | `uv_lock.py:66` | 累加 Dockerfile 指令（`add_instruction/render`），供 uv 工作区注入 |
| `UvLockPlan` | `uv_lock.py:53` | uv.lock 工作区解析计划 |

## 3. 关键调用链

**链一：`langgraph deploy` 主流程（`_deploy_cmd`，`deploy.py:1571`）**

1. **预检**：`validate_deploy_commands` 校验 install/build 命令串（`deploy.py:1605`）；`validate_config_file` 加载配置（`deploy.py:1616`）；`_parse_env_from_config` 提取环境变量（`deploy.py:1619`）；`_resolve_build_mode` 决定本地/远端（`deploy.py:1642`）。
2. **解析/创建 deployment**：`_create_host_backend_client` 建 `HostBackendClient`（`deploy.py:1651`）；`_resolve_deployment` 按 id/name 查找，未命中则 `_create_deployment` POST `/v2/deployments`（`deploy.py:1654-1674`）。
3. **依赖扫描**：`find_tracked_packages` 扫描本地依赖（失败仅告警不阻断，`deploy.py:1682-1686`）。
4. **构建（分叉）**：远端走 `_run_remote_build`（`deploy.py:1690`）；本地走 `_run_local_build`（`deploy.py:1703`）。
5. **等待与结果**：`_poll_revision_status` 轮询 revision 状态直到终态（`deploy.py:1735`）；失败且非 verbose 时拉取最近 30 行构建日志打印（`deploy.py:1748-1769`）；`_print_deployment_result` 输出 dashboard 链接。

**链二：本地构建（`_run_local_build`，`deploy.py:930`）**

1. 经 `Runner`/`subp_exec` 调 `docker build`（复用 `docker-packaging` 的 `build_docker_image`）。
2. `_resolve_pushed_image_digest` 解析推送后的镜像 digest（`deploy.py:895`）。
3. `HostBackendClient.update_deployment` PATCH image_uri（`host_backend.py:120`）。

**链三：远端构建（`_run_remote_build`，`deploy.py:1129`）**

1. `create_archive` 生成 `source.tar.gz`（`archive.py:62`）。
2. `request_upload_url` 取签名 GCS URL（`host_backend.py:113`）。
3. `_upload_to_gcs` PUT 上传 tarball（`deploy.py:860`）。
4. `update_deployment_internal_source` PATCH 触发后端远端构建（`host_backend.py:141`）。

## 4. 配置项

| flag / option / 环境变量 | 默认 / 行为 | 位置 |
|---|---|---|
| `--remote/--no-remote` | `remote_build_flag`，未指定时按 build_mode 自动 | `deploy.py:1588`、`_resolve_build_mode:562` |
| `--image` | 用既有镜像，禁止与 `--remote` 同用 | `deploy.py:1639-1640` |
| `--deployment-name` / `--deployment-id` | 名称为空时交互 prompt 或用目录名 | `deploy.py:1621-1635` |
| `--no-wait` | 不轮询，提交即返回 | `deploy.py:1587`、`deploy.py:1731` |
| `--json` | JSON 模式输出（`_Emitter(json_mode=True)`） | `deploy.py:1590`、`deploy.py:1594` |
| `--no-input` | 非交互，名称用默认目录名 | `deploy.py:1591`、`deploy.py:1625` |
| host backend timeout | 30s，`httpx.HTTPTransport(retries=3)` | `host_backend.py:30-42` |
| 归档大小 | 警告阈值 50MB，硬上限 200MB | `archive.py:15-16` |
| API key / host URL | 经 `OPT_HOST_API_KEY`/`OPT_HOST_URL` 或环境变量 | `host_backend.py` 注入 `X-Api-Key` |

## 5. 错误与重试语义

- **HTTP 错误**：`HostBackendClient._request` 把 `httpx.HTTPStatusError` 包装为 `HostBackendError`（含 status_code），`TransportError` 包装为纯文本错误（`host_backend.py:55-62`）。
- **传输重试**：`httpx.HTTPTransport(retries=3)` 对底层连接重试 3 次（`host_backend.py:30`）。
- **子进程失败**：`subp_exec` 在 returncode 非 0 且非 130（Ctrl-C）时打印 stdout/stderr 并 `raise click.exceptions.Exit(returncode)`（`exec.py:94-101`）。
- **归档超限**：>200MB 抛 `ClickException`，>50MB 仅警告（`archive.py:121-132`）。
- **依赖扫描失败不阻断**：`find_tracked_packages` 异常被 `except Exception` 捕获，仅 warn（`deploy.py:1684-1686`）。
- **构建失败日志**：`BUILD_FAILED` 时自动拉取最近构建日志辅助排障（`deploy.py:1748-1769`）。

## 6. 并发细节

- **asyncio 子进程**：`subp_exec` 用 `asyncio.create_subprocess_exec`，`asyncio.gather` 并发监控 stdout、stderr 与 stdin 写入三路（`exec.py:79-92`）。
- **信号转发**：注册 SIGINT/SIGTERM 处理器转发给子进程 `proc.terminate()`；Windows 仅处理 SIGINT（`exec.py:58-75`）；`finally` 中恢复原信号处理器（`exec.py:109-123`）。
- **进度线程**：`Progress.spinner_task` 是独立 `threading.Thread`，用 `threading.Event`（`_stop`/`_line_clear`）协调停止与清行（`progress.py:21-70`）。
- **同步 HTTP**：`HostBackendClient` 是阻塞式 `httpx.Client`，在 deploy 主流程中同步调用（CLI 场景无需并发）。
- **流式行处理**：`monitor_stream` 用 `asyncio.to_thread` 把 `on_line` 回调移出事件循环（`exec.py:167`）。

## 7. 系统边界

**In-Scope（本仓库源码内）**
- `deploy.py`：deploy 命令组与四阶段编排、本地/远端构建分叉、轮询与结果输出。
- `exec.py`：asyncio 子进程执行器与流监控。
- `host_backend.py`：宿主后端 REST 的同步 httpx 客户端。
- `archive.py`：源码 tarball 打包与过滤。
- `uv_lock.py`：uv.lock 工作区解析与 Docker 指令计划。
- `dependency_tracking.py`：受追踪包版本扫描。
- `progress.py`：终端进度 spinner。

**Out-of-Scope（不在本仓库源码内）**
- **LangSmith 部署后端（host backend）**：/v2/deployments REST、远端构建服务本身，是远端服务，不在本仓库源码内。
- **GCS（Google Cloud Storage）**：tarball 上传的对象存储，外部服务。
- **Docker 引擎**：本地 build/push 在外部 Docker 守护进程执行。
- **Dockerfile 生成与镜像构建核心**：见 `docker-packaging` 叶子。

## 8. 与相邻子系统交互

- `cli-core` 的 `cli.py:237` → `cli.add_command(deploy)` 把本叶子 deploy 组挂入顶层命令。
- 本叶子 → `config.validate_config_file`（`cli-core`）加载校验 langgraph.json。
- 本叶子 → `docker.check_capabilities`/`build_docker_image`（`docker-packaging`）执行本地镜像构建。
- 本叶子 → `HostBackendClient` → LangSmith 部署后端（外部 REST）。
- 本叶子 → GCS（外部对象存储）上传 tarball。
- 本叶子 → `subp_exec` → 本机 Docker（外部）。

## 9. 语言专项适配口径（纯 Python）

- **能力缝归组**：本叶子是“部署编排/适配层”——本身不构建镜像，而是编排“本地 Docker 构建”与“远端后端构建”两条路径。能力缝：`_Emitter`（文本/JSON 输出缝）、`HostBackendClient`（远端 API 客户端缝）、`Runner`/`subp_exec`（子进程执行缝）。
- **双轨实现模式**：本地构建（`_run_local_build`，本地 docker build）与远端构建（`_run_remote_build`，打包上传 GCS + 后端 build）是同一“产出 revision”能力的两条实现，由 `_resolve_build_mode` 配置/环境自动选择——典型配置驱动分派。
- **装饰器/回调缝**：`on_stdout` 回调（`exec.py:38`）与 `on_poll`（`deploy.py:1742`）是注入式扩展点，把业务关注点（识别“Application startup complete”、轮询打印）与执行器解耦。
- **外部边界**：编排层定位——实际构建在外部 Docker / LangSmith 后端 / GCS，均标注“不在本仓库源码内”。

## 10. 图表清单与质量

| 图 | 文件 | 类型 | archify 质量档 |
|---|---|---|---|
| 组件架构图 | `cli-deploy-exec-architecture.html` | architecture | showcase |
| deploy 执行链时序 | `cli-deploy-exec-sequence.html` | sequence | showcase |

- JSON IR 源文件位于 `json/` 目录。
- 省略 workflow：deploy 的多阶段顺序已由 sequence 时序图表达（参与方明确、消息按时间序），再画泳道流程会重复，按资源节省原则省略。
- 省略 dataflow/lifecycle：本叶子无独立数据管道，也无单实体状态机（revision 状态机在远端后端，不在本仓库源码内），故不画。
