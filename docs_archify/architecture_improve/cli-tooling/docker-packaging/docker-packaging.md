# docker-packaging（Docker 镜像构建与 compose 生成）

> 本文是 `cli-tooling` 域下的叶子子系统文档，覆盖 Docker 能力探测、compose YAML 生成、
> 镜像构建入口。Dockerfile 文本的具体生成逻辑位于 `config.py`（`config_to_docker`/
> `python_config_to_docker`），本文聚焦 `docker.py` 对外编排；命令入口见 `cli-core`，
> deploy 远端构建见 `cli-deploy-exec`。
>
> 源码基准：`libs/cli/langgraph_cli/docker.py`，git HEAD `e539ac12`（2026-09-09）。

## 1. 功能清单

| 能力 | 说明 | 源码路径 |
|---|---|---|
| 本地构建可行性探测 | 检查 docker 是否安装/守护进程运行/非 x86_64 是否需 buildx | `docker.py:52`（`can_build_locally`） |
| Docker 能力采集 | `docker info` 解析 server/compose 版本，区分 plugin/standalone compose | `docker.py:100`（`check_capabilities`） |
| 调试服务 compose | 生成 langgraph-debugger 服务段 | `docker.py:145`（`debugger_compose`） |
| dict→YAML | 手写递归字典转 YAML 字符串 | `docker.py:169`（`dict_to_yaml`） |
| compose 字典生成 | 按端口/postgres/image/base_image 拼 docker-compose 服务字典 | `docker.py:190`（`compose_as_dict`） |
| compose 输出 | 字典转 YAML 字符串（作为 `docker compose -f -` 的 stdin） | `docker.py:304`（`compose`） |
| 镜像构建 | pull 基础镜像 → 生成 Dockerfile stdin → `docker build` | `docker.py:333`（`build_docker_image`） |

## 2. 核心类型与接口清单

| 类型 / 函数 | 位置 | 职责 |
|---|---|---|
| `Version(NamedTuple)` | `docker.py:20` | 语义版本元组，支持 `>=` 比较 |
| `DockerCapabilities(NamedTuple)` | `docker.py:29` | 承载 docker/compose 版本、healthcheck_start_interval、compose_type |
| `check_capabilities(runner)` | `docker.py:100` | 探测 Docker 与 compose 形态，返回 `DockerCapabilities` |
| `can_build_locally()` | `docker.py:52` | 返回 `(bool, reason)`，供 deploy 决定本地/远端构建 |
| `compose_as_dict(...)` | `docker.py:190` | 生成基础 compose 服务字典（postgres、langgraph-api、debugger） |
| `build_docker_image(...)` | `docker.py:333` | 镜像构建主入口 |

## 3. 关键调用链

**链一：`langgraph build` 镜像构建（`build_docker_image`，`docker.py:333`）**

1. 若 `pull=True`，先 `docker pull docker_tag(config_json, base_image, api_version)` 拉取基础镜像（`docker.py:351-359`）。
2. `set("Building...")` 更新进度（`docker.py:360`）。
3. 判断 JS/Python 项目：`is_js_project = node_version 且无 python_version`（`docker.py:369-371`）；JS 项目带 install/build 命令时构建上下文取当前目录，否则取 config 父目录（`docker.py:374-377`）。
4. `copy.deepcopy(config_json)` 后调 `config.config_to_docker(...)` 生成 Dockerfile stdin 与 additional_contexts（`docker.py:381-390`）。
5. 把 additional_contexts 拼成 `--buildcontext k=v` 参数（`docker.py:392-394`）。
6. `subp_exec(*cmd, "-f", "-", "-t", tag, ..., build_context, input=stdin)` 执行 `docker build`（`docker.py:395-406`）。

**链二：能力探测（`check_capabilities`，`docker.py:100`）**

1. `shutil.which("docker")` 缺失抛 `UsageError`（`docker.py:102-103`）。
2. `docker info -f {{json .}}` 取信息；JSON 解析失败抛 `UsageError("Docker not installed or not running")`（`docker.py:105-111`）。
3. 从 `ClientInfo.Plugins` 找 `compose` 插件，找不到则回退 `docker-compose` standalone（`docker.py:117-130`）。
4. 解析版本，返回 `DockerCapabilities`（`docker.py:137-142`）。

## 4. 配置项

| flag / option | 默认 / 行为 | 位置 |
|---|---|---|
| `pull` | 默认 True，构建前拉最新基础镜像 | `docker.py:351` |
| `tag` | `-t` 指定镜像标签（必填，来自 build 命令） | `docker.py:366` |
| `base_image` / `api_version` | 决定 `docker_tag(...)` 拉取的基础镜像 | `docker.py:356`、`config.docker_tag` |
| `healthcheck_start_interval` | docker 版本 ≥25.0.0 才启用 | `docker.py:140` |
| `docker_command` | 可替换默认 `("docker","build")` | `docker.py:395` |

## 5. 错误与重试语义

- **Docker 未安装/未运行**：`check_capabilities` 抛 `click.UsageError`（`docker.py:103/111/114`）。
- **compose 缺失**：既无插件也无 standalone 时抛 `UsageError("Docker Compose not installed")`（`docker.py:124-125`）。
- **本地构建不可行**：`can_build_locally` 返回 `(False, reason)`，deploy 据此回退远端构建（`docker.py:60-97`）。
- **无重试**：构建命令失败由 `subp_exec` 非零退出码中断，CLI 不自动重试。

## 6. 并发细节

- 本叶子本身不创建线程/协程；构建执行经 `Runner`/`subp_exec`（见 `cli-deploy-exec`）异步驱动子进程。
- `can_build_locally` 用同步 `subprocess.run(timeout=10)` 做快速探测（`docker.py:69-85`）。

## 7. 系统边界

**In-Scope（本仓库源码内）**
- `docker.py`：能力探测、compose YAML 生成、镜像构建入口编排。

**Out-of-Scope（不在本仓库源码内）**
- **Docker 引擎/守护进程**：build/pull/容器运行在外部 Docker，不在本仓库源码内。
- **基础镜像仓库**：`langchain/langgraph-api` 等镜像从外部镜像仓库拉取。
- **Dockerfile 文本生成**：`config_to_docker`/`python_config_to_docker` 在 `config.py`（`cli-core` 配置面），本文只作为其调用方。

## 8. 与相邻子系统交互

- `cli-core` 的 `up`/`build` 命令 → `check_capabilities`/`compose`/`build_docker_image`（本叶子）。
- 本叶子 → `config.docker_tag`/`config_to_docker`（`cli-core` 配置面）生成镜像标签与 Dockerfile。
- 本叶子 → `subp_exec`（`cli-deploy-exec`）执行本机 docker。
- 本叶子 → Docker 引擎（外部）。

## 9. 语言专项适配口径（纯 Python）

- **能力缝归组**：本叶子是“外部工具编排层”——把配置对象翻译成 Docker CLI 调用，本身不做镜像构建计算。能力缝：能力探测（`check_capabilities`）与 compose/Dockerfile 文本生成（`compose_as_dict`/`config_to_docker`）。
- **配置驱动分派**：Python vs JS 项目、plugin vs standalone compose、是否需要 buildx，均由探测结果/配置驱动选择执行路径（`docker.py:369-377`、`docker.py:117-130`）。
- **外部边界**：实际构建在外部 Docker 引擎与镜像仓库，均标注“不在本仓库源码内”。

## 10. 图表清单与质量

| 图 | 文件 | 类型 | archify 质量档 |
|---|---|---|---|
| 组件架构图 | `docker-packaging-architecture.html` | architecture | showcase |
| 镜像构建数据流 | `docker-packaging-dataflow.html` | dataflow | showcase |

- JSON IR 源文件位于 `json/` 目录。
- 省略 sequence：镜像构建是“配置→Dockerfile→docker build→镜像”的单向管道，已由 dataflow 表达，跨方消息交互弱，故不画 sequence。
- 省略 workflow/lifecycle：构建是单次线性流程，无多角色泳道分支，也无单实体状态机，故不画。
