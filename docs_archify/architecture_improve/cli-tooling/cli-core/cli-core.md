# cli-core（CLI 入口与命令分发）

> 本文是 `cli-tooling` 域下的叶子子系统文档，覆盖 LangGraph CLI 的命令行入口、Click 命令注册、
> 配置文件加载与校验、匿名遥测等“命令分发与配置面”职责。deploy/exec 的具体部署执行链、
> Docker 镜像构建分别见 `cli-deploy-exec`、`docker-packaging` 叶子，本文不展开。
>
> 源码基准：`libs/cli/langgraph_cli/`，git HEAD `e539ac12`（2026-09-09）。

## 1. 功能清单

| 能力 | 说明 | 源码路径 |
|---|---|---|
| 包入口 | `python -m langgraph_cli` 直接调用 `cli()` | `__main__.py:1-4` |
| Click 命令组 | `@click.group` 定义顶层 `cli()`，自定义 `NestedHelpGroup` 把子命令拍平到顶层帮助 | `cli.py:185-233` |
| deploy 组注入 | 把 `deploy.py` 中定义的 `deploy` Click 组挂进顶层 | `cli.py:237`（`cli.add_command(deploy)`） |
| `up` 命令 | 本地拉起 LangGraph API 服务（docker compose up） | `cli.py:276-372` |
| `build` 命令 | 构建并打 Docker 镜像标签 | `cli.py:412-464` |
| `dockerfile` 命令 | 导出生成的 Dockerfile / compose | `cli.py:529` 起 |
| `dev` 命令 | 开发态热重载（inmem extra） | `cli.py:758` 起 |
| `validate` 命令 | 校验 langgraph.json 是否合法 | `cli.py:871-904` |
| `new` 命令 | 从模板创建新项目 | `cli.py:918-922` |
| compose 准备 | `prepare` / `prepare_args_and_stdin` 拼装 compose 参数与 stdin | `cli.py:930-1043` |
| 配置加载校验 | 读 langgraph.json、按 TypedDict 校验、Node/Python 版本兼容检查 | `config.py:608-648`（`validate_config_file`）、`config.py:323-393`（`validate_config`） |
| 配置 schema | `Config` 及 store/checkpointer/auth/encryption/http/webhooks 等 TypedDict | `schemas.py:615`（`Config`）起 |
| 匿名遥测 | `@log_command` 装饰器后台线程上报 Supabase | `analytics.py:86-105` |
| 版本与常量 | `__version__`、默认配置文件名、默认端口、Supabase 端点 | `version.py:1-10`、`constants.py:1-6` |
| Wolfi 安全提示 | 非 wolfi 发行版时打印安全建议 | `util.py:12-50` |

## 2. 核心类型与接口清单

| 类型 / 函数 | 位置 | 职责 |
|---|---|---|
| `cli()` | `cli.py:232` | Click 顶层命令组入口，`@click.version_option` 打印版本 |
| `NestedHelpGroup` | `cli.py:185` | 自定义 `click.Group`，递归收集一层子命令并拍平展示在 `--help` |
| `OPT_CONFIG/OPT_PORT/...` | `cli.py:29-177` | 可复用的 Click option 工厂（`-c/--config`、`-p/--port` 等），在各命令上叠加复用 |
| `validate_config_file(config_path)` | `config.py:608` | 读 JSON 文件 → `validate_config` → 额外校验 package.json 的 Node engines |
| `validate_config(config)` | `config.py:323` | 按 graphs 类型推断 node/python 版本默认值，归一化并校验配置字典 |
| `Config`（TypedDict） | `schemas.py:615` | langgraph.json 的完整类型面：dependencies/graphs/env/store/auth/encryption/http/webhooks 等 |
| `prepare(...)` | `cli.py:982` | 编排：校验配置 → 可选 `docker pull` → 调 `prepare_args_and_stdin` |
| `prepare_args_and_stdin(...)` | `cli.py:930` | 把 `docker.compose(...)` 与 `config.config_to_compose(...)` 拼成 `(args, stdin)` |
| `log_command(func)` | `analytics.py:86` | 命令装饰器：采集匿名参数，起后台线程 `log_data` 上报 |
| `get_anonymized_params(...)` | `analytics.py:29` | 把 kwargs 归一化为布尔/短值，避免泄漏路径与敏感参数 |

## 3. 关键调用链

**链一：`langgraph up` 本地起服务**

1. 用户输入 `langgraph up ...`，`__main__.py:4` 调 `cli()`，Click 解析参数后进入 `up()`（`cli.py:278`）。
2. `up()` 进入 `with Runner() as runner, Progress(...)`，先 `langgraph_cli.docker.check_capabilities(runner)` 探测 Docker compose 形态（`cli.py:300-301`）。
3. 调 `prepare(...)`（`cli.py:302`）：内部 `validate_config_file(config_path)` 加载校验配置（`cli.py:1001`）；若 `pull=True` 则 `docker pull` 拉取 `docker_tag(...)` 指定的镜像（`cli.py:1004-1012`）。
4. `prepare_args_and_stdin` 把 `docker.compose(...)`（基础服务）与 `config.config_to_compose(...)`（用户服务定义）拼成 stdin 字符串（`cli.py:950-978`）。
5. `up()` 追加 `up --remove-orphans` 及 `--recreate/--watch/--wait` 等开关，按 `capabilities.compose_type` 选择 `docker compose` 或 `docker-compose`，经 `subp_exec` 执行（`cli.py:319-372`）。
6. `on_stdout` 回调识别 `unpacking to docker.io` / `Application startup complete`，打印 API/Docs/Studio 地址后返回（`cli.py:336-357`）。

**链二：`langgraph validate` 配置校验**

1. `validate()`（`cli.py:873`）先 `json.load` 配置文件，捕获 `JSONDecodeError` 转 `UsageError`（`cli.py:876-880`）。
2. `config.get_unknown_keys(raw_config)` 收集未知键警告（`cli.py:883`）。
3. `config.validate_config_file(config)`（`cli.py:886`）；失败时打印错误与警告并 `SystemExit(1)`（`cli.py:887-893`）。
4. 成功则统计 `graphs` 数量打印合法信息（`cli.py:895-900`）。

**链三：匿名遥测上报**

1. 每个命令函数被 `@log_command` 装饰（如 `up` 在 `cli.py:277`）。
2. 装饰器先查环境变量 `LANGGRAPH_CLI_NO_ANALYTICS==1` 则跳过上报直接执行（`analytics.py:89`）。
3. 否则组装 `LogData`（os/python/cli 版本 + 匿名参数），`threading.Thread(target=log_data).start()` 异步 POST 到 Supabase，随后同步执行原命令（`analytics.py:92-103`）。

## 4. 配置项

| flag / option / 环境变量 | 默认 / 行为 | 位置 |
|---|---|---|
| `-c/--config` | 默认 `langgraph.json`（`DEFAULT_CONFIG`），`exists=True` 校验 | `cli.py:41-101`、`constants.py:1` |
| `-p/--port` | 默认 `8123`（`DEFAULT_PORT`） | `cli.py:102-116`、`constants.py:2` |
| `--pull/--no-pull` | 默认 `pull=True`（拉最新镜像） | `cli.py:123-135` |
| `--recreate/--no-recreate` | 默认 `False` | `cli.py:117-122` |
| `--watch` | 热重载 | `cli.py:142` |
| `--engine-runtime-mode` | `combined_queue_worker`，可选 `distributed` | `cli.py:172-177` |
| `--api-version` | 未指定用最新稳定版；支持 `~=0.11.0.dev5` 这类兼容区间 | `cli.py:159-170`、`config.py:158-295` |
| `LANGGRAPH_CLI_NO_ANALYTICS=1` | 关闭匿名遥测 | `analytics.py:89` |
| `LANGGRAPH_CLI_ANALYTICS_SOURCE` | deploy 命令上报的 source 标签 | `analytics.py:35-37` |
| `image_distro`（langgraph.json） | 默认 `debian`；切 `wolfi` 触发安全建议 | `config.py:338`、`util.py:23` |
| `python_version` / `node_version` | 按 graphs 是否含 Node 图自动推断默认 | `config.py:331-336` |

## 5. 错误与重试语义

- **配置 JSON 非法**：`validate()` 捕获 `json.JSONDecodeError` 抛 `click.UsageError`（`cli.py:879-880`）；`validate_config_file` 抛 `UsageError/ValueError` 时命令打印错误并 `SystemExit(1)`（`cli.py:887-893`）。
- **Node 版本不兼容**：`validate_config_file` 读 `package.json` 的 `engines.node`，主版本低于 `MIN_NODE_VERSION` 抛 `UsageError`（`config.py:628-641`）。
- **Docker 未安装**：`build()` 中 `shutil.which("docker") is None` 时抛 `UsageError("Docker not installed")`（`cli.py:443-444`）。
- **build/install_command 注入风险**：`has_disallowed_build_command_content` 检测命令串中的危险字符，命中抛 `UsageError`（`cli.py:430-441`、`config.py:72`）。
- **遥测失败静默吞掉**：`log_data` 的 `urllib.request.urlopen` 若 `URLError` 直接 `pass`，绝不影响主命令（`analytics.py:80-83`）。
- **无自动重试**：CLI 本身不重试 Docker/pull 操作，失败即由子进程非零退出码中断。

## 6. 并发细节

- **唯一后台线程**：遥测通过 `threading.Thread` 异步上报（`analytics.py:101`），主线程不等待；线程不做 join，进程退出即结束，属“发射后不管”。
- **子进程执行**：所有 Docker 调用经 `subp_exec`/`Runner`（见 `cli-deploy-exec` 叶子），`Runner` 上下文管理子进程生命周期与 stdout 回调（`up()` 的 `on_stdout` 在 `cli.py:336`）。
- **无线程锁/共享状态**：命令组之间无共享可变状态；配置对象是纯数据 TypedDict。

## 7. 系统边界

**In-Scope（本仓库源码内）**
- `cli.py`：Click 命令组与全部非 deploy 命令、compose 参数拼装。
- `config.py`：langgraph.json 加载、校验、版本兼容、本地依赖分析、compose/Dockerfile 文本生成。
- `schemas.py`：配置 TypedDict 类型面。
- `analytics.py` / `constants.py` / `version.py` / `util.py`：遥测、常量、版本、辅助提示。

**Out-of-Scope（不在本仓库源码内）**
- **Docker 引擎 / docker compose**：CLI 通过 `subp_exec` 调用本机 docker，镜像构建与容器运行在外部 Docker 守护进程，不在本仓库源码内。
- **Supabase 遥测后端**：匿名日志 POST 到外部 Supabase 项目（`constants.py:5-6`），不在本仓库源码内。
- **LangGraph API Server 镜像**：`langchain/langgraph-api` 等基础镜像由外部镜像仓库分发，不在本仓库源码内。
- deploy/exec 部署执行链、宿主后端、uv 锁解析：见 `cli-deploy-exec` 叶子。
- Dockerfile 生成与镜像构建核心：见 `docker-packaging` 叶子。

## 8. 与相邻子系统交互

- 用户终端 → `cli()` Click 组 → 分发到 up/build/dev/validate/new 命令（本叶子）。
- 本叶子命令 → `config.validate_config_file` → 读磁盘 `langgraph.json` 并按 `schemas.Config` 校验。
- 本叶子命令 → `docker.check_capabilities` / `docker.compose` / `build_docker_image`（→ `docker-packaging` 叶子）。
- 本叶子命令 → `deploy` 子命令组（由 `deploy.py` 注入，→ `cli-deploy-exec` 叶子）。
- 本叶子命令 → `@log_command` → 后台线程 → Supabase（外部）。

## 9. 语言专项适配口径（纯 Python）

- **能力缝归组**：本叶子是典型的“CLI 编排/适配层”——不承载业务计算，价值在于把 `langgraph.json`（用户配置）翻译成 Docker compose 文本并驱动本机 Docker。能力缝体现在三层：Click 命令组（`cli()`）、配置校验（`config.py`）、遥测装饰器（`analytics.log_command`）。
- **装饰器即扩展点**：`@log_command` 是横切关注点的装饰器缝，所有命令统一挂接；`OPT_*` option 工厂是复用缝，避免重复声明 Click 参数。
- **配置驱动分派**：运行时行为（用哪个基础镜像、Python/Node 版本、engine runtime mode、image_distro）全部由 `langgraph.json` 配置对象决定，而非硬编码——`validate_config` 根据 `graphs` 内容推断版本默认值（`config.py:328-336`），是配置驱动静态分派。
- **可选依赖懒加载**：`__version__` 经 `importlib.metadata.version` 惰性读取，包元数据缺失时降级为空串（`version.py:5-9`），避免安装态差异崩溃。
- **外部边界**：编排层定位——实际容器运行在外部 Docker、遥测在外部 Supabase，均标注“不在本仓库源码内”。

## 10. 图表清单与质量

| 图 | 文件 | 类型 | archify 质量档 |
|---|---|---|---|
| 组件架构图 | `cli-core-architecture.html` | architecture | showcase |
| 命令分发与配置加载流程 | `cli-core-workflow.html` | workflow | showcase |

- JSON IR 源文件位于 `json/` 目录。
- 省略 sequence 图：本叶子的核心顺序（up/validate/遥测）已在 workflow 泳道图中表达多角色分步流程，再画 sequence 会与 workflow 信息重复，按资源节省原则省略。
- 省略 dataflow/lifecycle：本叶子无“数据管道”语义，也无单一实体状态机（命令是一次性短流程），故不画。
