---
title: infra-live（环境 Terraform + 整体发布）
parent: 模块
nav_order: 99
---

# infra-live（环境 Terraform + 整体发布）

## 定位

`infra-live` 是“环境与交付”仓库，唯一的远端 runtime 是 `remote-cluster`，部署目标是腾讯云上的一台自建 k3s 主机：

- 声明运行拓扑（`deploy/runtime-topology.v5.json`）、服务清单、公共 values 与基础设施 addon（`deploy/`）
- 负责把各微服务发布到该集群（`scripts/release.py`，经 SSH 在目标机上执行 Helm）
- 描述腾讯云主机本身：`terraform/tencent` 只描述并导入 CVM、EIP、安全组与数据盘（不创建）；
  本机与 CI 对该主机的访问分别由 `terraform/tencent-operator-access`、`terraform/tencent-ci-release-access` 管理

它与 `infra-templates` 的分工：

- `infra-templates`：每个业务仓库的 CI 模板（构建并推送镜像）
- `infra-live`：统一持有发布与环境凭据，做“整体发布”

## 目录结构

- `infra-live/terraform/tencent*`：腾讯云主机的描述、导入与访问控制
- `infra-live/deploy/`：服务清单、公共 values、topology-scoped addon manifests、runbooks
- `infra-live/scripts/`：`release.py` 与发布、探测、备份等脚本

## 关键工作流

- `infra-live/.github/workflows/release.yml`（`Release`）：唯一受管发布路径，只用组织与服务身份，不使用个人令牌
  - 业务服务按版本 tag（`vMAJOR.MINOR.PATCH`，打在该服务 `main` 的提交上）发布，不按分支头发布
  - 各服务在 tag 推送时已把镜像构建并推送到 `ghcr.io/lawseekdog/<name>`；缺镜像则发布失败
  - `release.py` 在部署前执行全部闸门（CI 全绿、契约组合、运行稳定性、Owner 读取）
  - 只有人可以发起发布；发布按启动顺序逐个执行；支持紧急回滚与服务退役
- `infra-live/.github/workflows/tests.yml`（`Infra tests`）：每次 push 与 PR 运行 `scripts/tests`

## 凭据与变量

运行时密钥、发布 SSH、CI 身份与备份目标的归属、投影与轮换规则统一见 `infra-live/deploy/SECRET-MANAGEMENT.md`；
不要把运行时密钥放入 GitHub Secrets 或 Variables。

## 基础设施组件

Postgres、MinIO、Elasticsearch、可观测性与 Traefik 入口的 addon、镜像与注意事项见
`infra-live/deploy/README.md`；发布与备份恢复流程见 `infra-live/deploy/runbooks/`。
