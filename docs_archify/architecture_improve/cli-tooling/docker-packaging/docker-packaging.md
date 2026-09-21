# Docker 打包与依赖追踪（docker-packaging）

> 本文是 `cli-tooling` 域下的叶子子系统文档。域级总览见 `../cli-tooling.md`。
> 本文只展开 Docker 镜像/compose 生成、能力探测、uv 工作区解析、依赖追踪与主机后端 HTTP 客户端；
> 命令注册见 `../cli-core/cli-core.md`，部署编排见 `../cli-deploy-exec/cli-deploy-exec.md`。
>
> 源码基准：`libs/cli/langgraph_cli/`（`docker.py` 406、`host_backend.py` 205、`uv_lock.py` 1055、
> `dependency_tracking.py` 141、`templates.py` 186 行）。

## 1. 功能清单

| 能力 | 说明 | 源码路径 |
| --- | --- | --- |
| Docker 能力探测 | 解析 docker/compose 版本，判断 compose 是 plugin 还是 standalone | `docker.py:100` `check_capabilities` |
| 本地可构建判定 | 查 docker 二进制、守护进程、（非 x86_64 时）buildx | `docker.py:52` `can_build_locally` |
| compose 字典生成 | 拼 Redis6 + pgvector/pg16 + langgraph-api（+debugger）服务 | `docker.py:190` `compose_as_dict` |
| compose → YAML | 自实现 dict→YAML（避免引入 PyYAML 依赖） | `docker.py:169` `dict_to_yaml` |
| 镜像构建执行 | pull 基础镜像 → 拼 Dockerfile stdin → `docker build/buildx build` | `docker.py:333` `build_docker_image` |
| Dockerfile 生成（依赖路径） | 由 `config.py` 的 `python_config_to_docker`/`node_config_to_docker` 产出 | `config.py:1263/1482` |
| uv 工作区解析 | 解析 `pyproject.toml`/`uv.lock` 工作区，产出 `UvLockPlan` | `uv_lock.py:623` `_plan_uv_lock_workspace` |
| Dockerfile 行累加器 | `DockerBuildPlan` 累加 RUN/COPY/ENV 后 `render()` | `uv_lock.py:66` `DockerBuildPlan` |
| 主机后端 HTTP 客户端 | 部署/revision/push-token/upload-url/build-logs REST 封装 | `host_backend.py:19` `HostBackendClient` |
| 跟踪包检测 | 扫描本地依赖的 uv.lock/pyproject/requirements，产出 `<pkg>:<ver>` | `dependency_tracking.py:98` `find_tracked_packages` |
| 模板脚手架 | 下载 GitHub 模板 zip 并解压为新项目 | `templates.py:128` `create_new` |

## 2. 核心类型与接口清单

| 类型 / 函数 | 位置 | 职责 |
| --- | --- | --- |
| `DockerCapabilities`（NamedTuple） | `docker.py:29` | docker/compose 版本 + healthcheck 能力 + compose 形态 |
| `compose_as_dict(...)` | `docker.py:190` | 返回 compose 服务字典（顺序与单测绑定，勿随意重排） |
| `build_docker_image(...)` | `docker.py:333` | 镜像构建入口；支持 `docker_command` 覆盖为 buildx |
| `HostBackendClient` | `host_backend.py:19` | httpx 同步客户端，`X-Api-Key`/`X-Tenant-ID` 头，retries=3、30s 超时 |
| `HostBackendError` | `host_backend.py:11` | 带 `status_code` 的 ClickException |
| `UvLockPlan` / `DockerBuildPlan` | `uv_lock.py:53/66` | uv 工作区部署计划 / Dockerfile 行累加器 |
| `UvLockPackage` | `uv_lock.py:32` | 工作区单包：依赖名、工作区依赖、是否启用 |
| `TRACKED_PACKAGES` | `dependency_tracking.py:17` | 当前仅 `google-adk`，与服务端严格同步 |
| `TEMPLATE_ID_TO_CONFIG` | `templates.py:28` | 模板 id → (名称, 语言, zip URL) 注册表 |

## 3. 关键调用链

**链一：`build_docker_image` 构建镜像**（`docker.py:333`）
1. 若 `pull`，`docker pull <docker_tag(config_json, base_image, api_version)>`。
2. 判定 JS/Python 工程决定 build context（JS 且带 install/build 命令用 cwd，否则用 config 父目录）。
3. `copy.deepcopy(config_json)` 后 `config_to_docker(...)` 生成 Dockerfile stdin 与 `additional_contexts`。
4. 追加 `--build-context k=v`；默认 `docker build -f - -t <tag> <context>`（deploy 本地构建时覆盖为 `docker buildx build --platform linux/amd64 --load`）。

**链二：uv 工作区 Dockerfile**（`uv_lock.py`）
1. `_plan_uv_lock_workspace` 从 `source.root` 发现 pyproject.toml → `_discover_uv_lock_workspace_packages` 解析工作区成员。
2. `_infer_uv_lock_target_package` 推断目标包，计算 `install_order` 与容器内根映射。
3. `python_config_to_docker_uv_lock` 用 `DockerBuildPlan` 累加 COPY/uv sync/RUN 指令，`_update_uv_lock_graph_paths` 把图路径改写为容器内路径。

**链三：主机后端调用**（`host_backend.py`）
1. `HostBackendClient._request` 统一 request → `raise_for_status` → JSON 解码；HTTPStatusError 包装为带 status_code 的 `HostBackendError`，TransportError 也包装。
2. 暴露 create/get/list/delete deployment、push-token、upload-url、update（镜像/tarball 两路径）、list/get revision、build/deploy logs。

## 4. 配置项

| flag / 配置 | 默认 / 行为 | 位置 |
| --- | --- | --- |
| `langgraph.json: pip_installer` | `auto`：支持的基础镜像用 uv，否则 pip | `config.py:410`、`schemas.py:658` |
| `langgraph.json: source` | `{kind:"uv", root, package?}` 走 uv 工作区部署 | `schemas.py:593` `UvSource` |
| `langgraph.json: keep_pkg_tools` | 默认清理 pip/setuptools/wheel | `config.py:1242` |
| compose 内置服务 | Redis6 + pgvector/pg16 + langgraph-api | `docker.py:219-299` |
| `DEFAULT_POSTGRES_URI` | 内置 postgres 连接串 | `docker.py:15` |
| HostBackendClient | retries=3、timeout=30s | `host_backend.py:30-43` |
| `TRACKED_PACKAGES` | 仅 `google-adk` | `dependency_tracking.py:17` |
| 跟踪文件读取上限 | 5MB | `dependency_tracking.py:19` |
| 模板源 | GitHub 各 `*-template` 仓库 zip main 分支 | `templates.py:10-25` |

## 5. 错误与重试语义

- `HostBackendClient` 传输层重试由 httpx `HTTPTransport(retries=3)` 处理；业务错误统一 `HostBackendError`（带 status_code），由 deploy 编排层按状态码改写（见 cli-deploy-exec）。
- `check_capabilities` 失败抛 `UsageError`（docker 未安装/未运行/compose 缺失）。
- uv 工作区校验失败抛 `UsageError`：`[project].dependencies` 必须是字符串列表、PEP 508 必须带显式包名。
- `templates.create_new`：目标目录非空直接退出 1；下载 HTTPError 打印后退出 1。
- 跟踪包扫描单文件读取截断到 5MB，解码错误 `errors="replace"`，绝不阻断部署（编排层再兜底）。

## 6. 并发细节

- `HostBackendClient` 是**同步** httpx.Client（阻塞），与 deploy 编排的同步轮询模型一致；不引入异步。
- `check_capabilities` 用 `runner.run(subp_exec(... collect=True))` 同步收集输出。
- uv_lock 解析为纯 CPU 同步遍历文件；无线程/锁。
- 模板下载 `urllib.request.urlopen` 为阻塞 IO。

## 7. 系统边界

**In-Scope（本仓库源码内）**
- `docker.py` 能力/compose/构建、`host_backend.py` HTTP 客户端、`uv_lock.py` 工作区解析、`dependency_tracking.py`、`templates.py`。

**Out-of-Scope（不在本仓库源码内）**
- Docker 守护进程/buildx、镜像 registry、GCS、LangSmith 主机后端服务（`api.host.langchain.com`）、GitHub 模板仓库、PyPI——均外部。
- 实际 Dockerfile 拼装的其余部分在 `config.py`（cli-core 域引用）。

## 8. 与相邻子系统交互

- 上游：`cli.py`（up/build/dockerfile）与 `deploy.py` 调用 `check_capabilities`/`compose`/`build_docker_image`/`HostBackendClient`。
- 本叶子 → cli-core：复用 `config.py` 的 `config_to_docker`/`docker_tag`/`default_base_image`。
- 下游外部：Docker CLI、registry、GCS、主机后端 API、GitHub、PyPI。

## 9. 语言专项适配口径（纯 Python）

- **能力缝**：`HostBackendClient` 用一个 `_request` 私有方法收敛全部 REST 调用（method+path+payload），公开方法只是语义封装——薄客户端 + 统一错误包装。
- **配置驱动**：`pip_installer=auto`、`source.kind=uv` 决定走 pip 还是 uv 工作区 Dockerfile，是构建路径的静态分派；`compose_as_dict` 按 capabilities（docker≥25 才有 `start_interval`）调整 healthcheck。
- **避免重依赖**：自实现 `dict_to_yaml` 替代 PyYAML；uv_lock 用标准库 `tomllib`（3.10 回落 `tomli`）。
- **数据结构即计划**：`UvLockPlan`/`DockerBuildPlan` 用 frozen dataclass 把"工作区解析结果 → Dockerfile 行"建模为不可变计划对象。

## 10. 图表清单与质量

| 图 | 文件 | 类型 | archify 质量档 |
| --- | --- | --- | --- |
| 镜像构建与主机后端组件 | `docker-packaging-architecture.html` | architecture | standard（多条边标签与邻近组件重叠，去标签后通过；showcase 布局诊断未全过） |
| 镜像构建与推送流程 | `docker-packaging-workflow.html` | workflow | showcase |

第二张 workflow（本次迭代新增）刻画配置就位→生成 Dockerfile/compose→docker build 构建镜像→推送 registry 的多角色泳道流程，对应第 3 节链一 `build_docker_image` 调用链。JSON IR 源：`json/docker-packaging-architecture.json`、`json/docker-packaging-workflow.json`。
