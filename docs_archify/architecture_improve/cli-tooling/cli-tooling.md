# cli-tooling（CLI 工具链）域总览

> 本域包含以下叶子子系统；各叶子详情见对应文档。
> 源码基准：langgraph monorepo main 分支，commit `e539ac12`；主语言纯 Python（libs/cli/langgraph_cli；ts 文件为示例）。

## 1. 域职责

cli-tooling 域（`libs/cli/langgraph_cli`）提供 `langgraph` 命令行：命令分发与配置加载、`deploy`/`exec`
部署编排、Docker 镜像构建参数拼装，支撑本地开发与远端平台部署。覆盖 3 个叶子：

- **cli-core**：typer 命令分发、配置校验与遥测（`cli.py`/`config.py`/`schemas.py`）；
- **cli-deploy-exec**：部署与执行编排、依赖追踪、归档与进度展示（`deploy.py`/`exec.py` 等）；
- **docker-packaging**：Docker 镜像构建参数生成（`docker.py`）。

## 2. 叶子索引

| 叶子 | 文档 | 架构图 | 第二图 | 职责一句话 |
|------|------|--------|--------|-----------|
| cli-core | [cli-core.md](cli-core/cli-core.md) | [架构图](cli-core/cli-core-architecture.html) | [工作流图](cli-core/cli-core-workflow.html) | 命令分发与配置加载 |
| cli-deploy-exec | [cli-deploy-exec.md](cli-deploy-exec/cli-deploy-exec.md) | [架构图](cli-deploy-exec/cli-deploy-exec-architecture.html) | [时序图](cli-deploy-exec/cli-deploy-exec-sequence.html) | 部署与执行编排 |
| docker-packaging | [docker-packaging.md](docker-packaging/docker-packaging.md) | [架构图](docker-packaging/docker-packaging-architecture.html) | [数据流图](docker-packaging/docker-packaging-dataflow.html) | 镜像构建 |

## 3. 域级机制细节

- **配置驱动静态分派**：`schemas.py` 的 `Config` 模型（dataclass）承载全部 CLI 配置，命令体按配置
  组装 compose/Docker 参数——纯 Python 配置面工程。
- **命令分发**：`cli.py` 以 typer 装饰器声明命令树（`__main__.py` 入口），遥测/版本检查作为命令体外
  装饰；失败统一 `SystemExit(1)`（见 cli-core-workflow 图）。
- **部署管线**：本地构建（uv 依赖锁定 → docker 镜像）→ 归档推送 → 远端创建部署 → 轮询状态；
  `host_backend.py` 区分本地/远端后端，`progress.py` 驱动进度条（见域级 workflow 图）。
- **外部边界**：Docker 引擎、uv 解析器、LangGraph 平台、langgraph SDK（另一域）均不在本域源码内。

## 4. 域级图（架构图/工作流图 showcase 档，时序图 standard 档，render 退出码全 0）

![cli-tooling 域架构图](cli-tooling-architecture.html)

![cli-tooling 域时序图](cli-tooling-sequence.html)

![cli-tooling 域工作流图](cli-tooling-workflow.html)

> 本域未生成 lifecycle 图：CLI 无单一实体状态机语义（部署状态机在远端平台承载），按资源节省原则省略。
