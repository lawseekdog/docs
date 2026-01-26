---
title: infra-live（环境 Terraform + 整体发布）
parent: 模块
nav_order: 99
---

# infra-live（环境 Terraform + 整体发布）

## 定位

`infra-live` 是“环境与交付”仓库：

- 负责在阿里云创建 ACK 等基础设施（Terraform）
- 负责把各微服务部署到 ACK（集中式 Helm 发布）

它与 `infra-templates` 的分工：

- `infra-templates`：每个业务仓库的 CI 模板（构建镜像、可选单仓库部署）
- `infra-live`：统一持有环境凭据，做“整体发布”（更适合私有仓库+无法使用组织级 Secrets 的限制）

## 目录结构

- `infra-live/terraform/`：VPC + ACK + NodePool（最小闭环）
- `infra-live/deploy/`：服务清单 + 公共 values + Deploy 工作流

## 关键工作流

- `infra-live/.github/workflows/terraform.yml`
  - push/PR：fmt + init + validate（不做 plan，避免缺少 tfvars/凭据导致失败）
  - workflow_dispatch：支持 `plan/apply`（需要 repo secrets 的阿里云凭据）
- `infra-live/.github/workflows/deploy.yml`
  - workflow_dispatch：按指定 `image_tag` 统一部署全部服务（从各服务仓库拉 Helm chart）
  - 通过 `CI_REGISTRY_PROVIDER` 选择镜像仓库（`aliyun-acr` 或 `ghcr`）

## 运行所需的变量/密钥（Repo-level）

Variables：

- `ALIYUN_REGION_ID`
- `ALIYUN_ACK_CLUSTER_ID`
- （可选）`ALIYUN_OIDC_PROVIDER_ARN` / `ALIYUN_ROLE_TO_ASSUME`

Secrets：

- `GH_ORG_TOKEN`：checkout 私有服务仓库（仅部署时需要）
- `ALIYUN_ACCESS_KEY_ID` / `ALIYUN_ACCESS_KEY_SECRET`（未使用 OIDC 时）
- （可选）`GH_PACKAGES_USERNAME` / `GH_PACKAGES_TOKEN`：创建 `ghcr-pull` 拉取 secret（私有 GHCR 镜像）
- （可选）`INTERNAL_API_KEY`：创建 `lawseekdog-secrets`（让 `/internal/**` 互调一致）
