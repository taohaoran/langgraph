# cli-tooling 域总览

> 本域是 LangGraph 官方命令行工具 `langgraph-cli`，源码位于 `libs/cli/langgraph_cli/`（19 个源文件 / 约 8.4k 行）。
> 域级定位：把开发者本地的 `langgraph.json` 项目，经构建、Docker 打包、依赖追踪，部署到 LangGraph Platform；
> 同时提供本地运行图、打包归档、配置校验等子命令。

## 域职责

- **命令入口与配置**：`langgraph` 可执行入口、Typer/Click 命令树、`langgraph.json` 校验、schema 定义。
- **部署与执行**：`deploy`（构建并推送/远程构建）、`exec`（本地执行图）、`archive`（打包 tar 归档）。
- **Docker 与依赖**：Dockerfile 模板、buildx 镜像构建、宿主后端 API 客户端、uv 锁工作区规划、import 依赖追踪。

## 叶子索引

| 叶子 | 职责 | 设计文档 | 图 |
| --- | --- | --- | --- |
| cli-core | CLI 入口、命令分发、配置加载/校验、进度条、分析上报、文件忽略 | [cli-core.md](cli-core/cli-core.md) | [architecture](cli-core/cli-core-architecture.html) |
| cli-deploy-exec | deploy 四阶段编排、exec 本地执行、archive 打包 | [cli-deploy-exec.md](cli-deploy-exec/cli-deploy-exec.md) | [architecture](cli-deploy-exec/cli-deploy-exec-architecture.html) · [sequence](cli-deploy-exec/cli-deploy-exec-sequence.html) |
| docker-packaging | Docker 镜像构建、Dockerfile 模板、uv 锁、依赖追踪、宿主后端 | [docker-packaging.md](docker-packaging/docker-packaging.md) | [architecture](docker-packaging/docker-packaging-architecture.html) |

## 域级机制

- **命令架构**：`cli.py` 以 `NestedHelpGroup` 组织子命令（`up/build/dockerfile/dev/validate/new` + `deploy` 经 `add_command` 挂入），`OPT_*` 常量复用共享选项。
- **部署流水线**：`deploy` 分本地 buildx 推送与远程 GCS 归档两条路径，轮询 revision 状态（300s/1s 与 900s/3s 两套超时），推送失败重试 3 次。
- **Docker 打包与依赖追踪**：`templates.py` 生成 Dockerfile；`uv_lock.py` 规划 uv 工作区；`dependency_tracking.py` 按 import 分析需打包的运行时包（当前仅 `google-adk`）。

## 外部边界

- Docker / buildx、uv、PyPI、LangGraph Platform 远端控制面与 GCS——均不在本仓库源码内。

## 域级图

| 图 | 文件 | 类型 | 说明 |
|---|---|---|---|
| 域组件拓扑 | [cli-tooling-architecture.html](cli-tooling-architecture.html) | architecture | 入口/分发/配置/构建/部署分层与外部 Docker、远端平台边界 |
| 一次 CLI 命令执行时序 | [cli-tooling-sequence.html](cli-tooling-sequence.html) | sequence | 参数解析→配置校验→镜像构建→部署输出的核心链路 |
| 部署流程泳道 | [cli-tooling-workflow.html](cli-tooling-workflow.html) | workflow | 发起部署→解析配置→构建→推送→部署平台→输出地址的多角色流程 |

域级 JSON IR 源文件位于 `json/`。
