# CLI 核心入口与命令分发（cli-core）

> 本文是 `cli-tooling` 域下的叶子子系统文档。域级总览见 `../cli-tooling.md`。
> 本文只展开 CLI 入口、命令注册、配置加载、进度条、匿名上报与文件忽略规则；
> 部署/本地执行逻辑见 `../cli-deploy-exec/cli-deploy-exec.md`，Docker 镜像构建与依赖追踪见
> `../docker-packaging/docker-packaging.md`。
>
> 源码基准：`libs/cli/langgraph_cli/`（langgraph-cli，纯 Python，基于 click）。

## 1. 功能清单

| 能力 | 说明 | 源码路径 |
| --- | --- | --- |
| 包入口 | `python -m langgraph_cli` 调用 `cli()` | `__main__.py`（4 行，`from .cli import cli; cli()`） |
| 顶层命令组 | `@click.group(cls=NestedHelpGroup)`，挂版本号 | `cli.py:230-233` |
| 嵌套帮助渲染 | 把子命令（含二级 `deploy revisions`）拍平到顶层帮助一页显示 | `cli.py:185-227` `NestedHelpGroup.format_commands` |
| `up` 命令 | 校验配置 → docker pull → 通过 stdin 喂 compose → `docker compose up` | `cli.py:276-372` |
| `build` 命令 | 校验 build/install 命令合法性 → 校验配置 → `build_docker_image` | `cli.py:412-464` |
| `dockerfile` 命令 | 校验配置 → `config_to_docker` 写出 Dockerfile，可选补 compose/.env/.dockerignore | `cli.py:527-655` |
| `dev` 命令 | 内存模式热重载开发服务器（依赖 `langgraph-api[inmem]`） | `cli.py:758-862` |
| `validate` 命令 | 读 JSON → 未知键警告 → `validate_config_file` | `cli.py:870-904` |
| `new` 命令 | 调用 `create_new` 从模板脚手架 | `cli.py:918-922` |
| `deploy` 命令组 | 在 `deploy.py` 定义后 `cli.add_command(deploy)` 接入 | `cli.py:237`、`deploy.py:1542` |
| 配置加载校验 | 读 `langgraph.json` → `validate_config` 规范化 → 版本/依赖/图校验 | `config.py:608` `validate_config_file`、`config.py:323` `validate_config` |
| 未知键提示 | 拼写纠错式 `did you mean` 警告 | `config.py:593` `get_unknown_keys` |
| 配置 schema | `langgraph.json` 全部字段的 TypedDict 文档化 | `schemas.py:615` `Config` |
| 进度条 | 后台线程画 `|/-\` 旋转光标，TCI 感知 | `progress.py:7` `Progress` |
| 匿名上报 | `@log_command` 装饰器异步 POST 到 Supabase | `analytics.py:86` `log_command` |
| 文件忽略规则 | `.dockerignore`/`.gitignore` + 内置排除，gitwildmatch | `_ignore.py:38` `_build_ignore_spec` |
| 工具函数 | 空行清理、非 wolfi 发行版安全提示 | `util.py:8/12` |

## 2. 核心类型与接口清单

| 类型 / 函数 | 位置 | 职责 |
| --- | --- | --- |
| `NestedHelpGroup(click.Group)` | `cli.py:185` | 自定义 help formatter，递归收集一级子命令并拍平输出 |
| `cli`（click group） | `cli.py:230` | 顶层命令容器，`@click.version_option` 打印版本 |
| `Config`（TypedDict） | `schemas.py:615` | `langgraph.json` 顶层契约：`graphs/dependencies/env/auth/encryption/...` |
| `Distros` | `schemas.py:5` | `Literal["debian","wolfi","bookworm"]` 基础镜像发行版 |
| `validate_config(config)` | `config.py:323` | 纯函数：原始 dict → 规范化 Config dict，失败抛 `click.UsageError` |
| `validate_config_file(path)` | `config.py:608` | 读文件 + `validate_config` + `package.json` engines 校验 |
| `default_base_image(config, ...)` | `config.py:1554` | 按 node/python 与运行模式选 `langchain/langgraph-api`/`langgraphjs-api`/`langgraph-executor` |
| `docker_tag(config, ...)` | `config.py:1566` | 拼镜像 tag（含 `api_version` 范围解析、distro 后缀） |
| `Progress` | `progress.py:7` | 上下文管理器式 spinner，`__enter__` 返回 `set_message` 回调 |
| `log_command`（装饰器） | `analytics.py:86` | 包一层命令：后台线程上报后再执行原命令 |
| `LogData`（TypedDict） | `analytics.py:20` | 上报载荷结构（os/python/cli 版本 + 匿名化参数） |
| `_NegatedDockerignoreHints` | `_ignore.py:24` | 解析 `.dockerignore` 的 `!` 取反，决定哪些被忽略目录仍需遍历 |

## 3. 关键调用链

**链一：`langgraph up` 本地起服务**（`cli.py:278`）
1. `Runner()` 与 `Progress()` 上下文进入；`docker.check_capabilities(runner)` 探测 docker compose 形态（plugin/standalone）。
2. `prepare(...)` → `config.validate_config_file(config)` 校验配置；若 `pull` 则 `docker pull <docker_tag(...)>`。
3. `prepare_args_and_stdin(...)`：`docker.compose(...)` 生成基础 compose 片段，`config.config_to_compose(...)` 把用户配置（env、build、dockerfile_inline、watch）拼进 stdin。
4. args 追加 `up --remove-orphans`（及 `--force-recreate`/`--watch`/`--wait`）。
5. `runner.run(subp_exec(*compose_cmd, *args, input=stdin, on_stdout=...))`；`on_stdout` 监听 `"unpacking"`/`"Application startup complete"` 切换进度文案并打印访问地址。

**链二：`langgraph build` 构建镜像**（`cli.py:419`）
1. `install_command`/`build_command` 先过 `config.has_disallowed_build_command_content`（阻断引号、反斜杠、`$`、单 `&` 等）。
2. `shutil.which("docker")` 校验；`validate_config_file` + `warn_non_wolfi_distro`。
3. 按 `engine_runtime_mode` 决定 `effective_base_image`；调用 `docker.build_docker_image(...)`（见 docker-packaging 叶子）。

**链三：`langgraph validate`**（`cli.py:873`）
1. `json.load` 读配置，JSON 语法错抛 `UsageError`。
2. `get_unknown_keys(raw_config)` 先算未知键（拼写纠错）。
3. `validate_config_file` 失败时连同未知键警告一起打印并 `SystemExit(1)`；成功打印图数量。

## 4. 配置项

| flag / 配置 | 默认 / 行为 | 位置 |
| --- | --- | --- |
| `--config/-c` | `langgraph.json`（`constants.DEFAULT_CONFIG`） | `cli.py:41`、`constants.py:1` |
| `--port/-p` | `8123`（`constants.DEFAULT_PORT`） | `cli.py:102` |
| `--pull/--no-pull` | 默认 pull | `cli.py:123` |
| `--recreate/--no-recreate` | 默认不 recreate | `cli.py:117` |
| `--api-version` | 未指定用最新稳定；支持 `~=`/`>~=` 范围（查 PyPI `langgraph-api`） | `cli.py:159`、`config.py:250` |
| `--engine-runtime-mode` | `combined_queue_worker`；`distributed` 拆 orchestrator+executor | `cli.py:172` |
| `langgraph.json: python_version` | 默认 `3.11`，下限 3.11 | `config.py:81-82` |
| `langgraph.json: node_version` | 默认 `20`，下限 20 | `config.py:17-18` |
| `langgraph.json: image_distro` | 默认 `debian`；可选 `debian/wolfi/bookworm` | `config.py:84`、`schemas.py:5` |
| `LANGGRAPH_CLI_NO_ANALYTICS=1` | 关闭匿名上报 | `analytics.py:89` |
| `LANGGRAPH_CLI_ANALYTICS_SOURCE` | deploy 上报的 source 字段 | `analytics.py:35` |

## 5. 错误与重试语义

- 配置/用法错误一律抛 `click.UsageError`，由 click 转为非零退出与红色提示；`validate` 命令额外 `SystemExit(1)`。
- API 版本范围解析需访问 PyPI（`_get_pypi_versions`，`httpx.get` 10s 超时）；网络失败或无匹配版本时抛 `UsageError`，不自动重试（`config.py:230-294`）。
- 子进程执行由 `Runner`/`subp_exec` 负责（见 cli-deploy-exec 叶子），失败透传为非零退出。
- 匿名上报 `log_data` 捕获 `urllib.error.URLError` 直接忽略——上报失败绝不阻断命令（`analytics.py:80-83`）。
- `dev` 命令的 `from langgraph_api.cli import run_server` 失败时，按 `langgraph-api` 是否安装给出 `pip install "langgraph-cli[inmem]"` 指引（`cli.py:790-820`）。

## 6. 并发细节

- `Progress` 用单个 `threading.Thread` 跑 `spinner_task`，`threading.Event`（`_stop`/`_line_clear`）协调停止与行清空；`__exit__` 置 `_stop` 后 `join`，不泄漏线程（`progress.py:46-107`）。非 TTY 时不启动线程，改走 stderr 行输出。
- `log_command` 每次命令起一个**前台后台线程**跑 `log_data`（`analytics.py:101`），主线程立即继续执行命令——上报与命令并行，不等待上报结果。
- 无线程池/锁共享；各命令子进程为阻塞式 `runner.run`。

## 7. 系统边界

**In-Scope（本仓库源码内）**
- `cli.py` 命令注册与参数定义、`config.py` 配置校验与 compose/Dockerfile 拼装入口、`schemas.py` TypedDict 契约、`progress.py`/`analytics.py`/`util.py`/`_ignore.py` 基础设施。

**Out-of-Scope（不在本仓库源码内）**
- Docker / docker compose 二进制、PyPI 索引（`pypi.org`）、Supabase 上报后端（`constants.py:5-6`）——均为外部服务。
- `langgraph-api`（内存服务器 `run_server`）为可选依赖 `langgraph-cli[inmem]`，不在本仓库源码内。
- 实际镜像构建细节在 `docker.py`（docker-packaging 叶子）；deploy 远端平台交互在 `deploy.py`（cli-deploy-exec 叶子）。

## 8. 与相邻子系统交互

- 上游：终端用户 → `__main__.py` → click 解析 → `cli` group → 各子命令。
- 本叶子 → docker-packaging：`up`/`build`/`dockerfile` 调用 `docker.check_capabilities`、`docker.compose`、`docker.build_docker_image`、`config.config_to_docker`。
- 本叶子 → cli-deploy-exec：`cli.add_command(deploy)` 接入 deploy 命令组；`up` 经 `Runner`/`subp_exec` 执行子进程。
- 下游外部：docker CLI、PyPI、Supabase、`langgraph-api`。

## 9. 语言专项适配口径（纯 Python）

- **框架选型**：基于 click（非 Typer）；用装饰器栈叠加共享 `OPT_*` option，自定义 `NestedHelpGroup` 覆盖帮助渲染。
- **能力缝**：`config.py` 的纯函数 `validate_config`/`config_to_docker`/`config_to_compose` 与 IO 读文件分离，便于测试；TypedDict（`schemas.py`）承担 `langgraph.json` 契约文档。
- **可选依赖与懒加载**：`dev` 命令内 `from langgraph_api.cli import run_server` 延迟导入，缺失时给出安装指引；httpx 仅用于 API 版本解析。
- **配置驱动**：`default_base_image`/`docker_tag` 根据 node/python 检测与 `image_distro` 选择镜像，同一命令经不同配置产出不同 compose/Dockerfile。

## 10. 图表清单与质量

| 图 | 文件 | 类型 | archify 质量档 |
| --- | --- | --- | --- |
| CLI 入口与命令矩阵 | `cli-core-architecture.html` | architecture | standard（showcase 布局校验：dispatcher→共享基础设施的向下连线触发 endpoint-side-direction 约束，显式 `fromSide/toSide` 后 standard 通过；showcase 仍有间距/布局诊断，如实披露） |
| CLI 命令分发流程 | `cli-core-workflow.html` | workflow | showcase |

第二张 workflow（本次迭代新增）刻画用户输入→click 解析→路由子命令→校验配置→执行→打印结果的多角色泳道流程，对应第 3 节三条调用链的公共分发骨架。JSON IR 源：`json/cli-core-architecture.json`、`json/cli-core-workflow.json`。
