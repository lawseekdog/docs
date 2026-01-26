---
title: 阿里云 ACK（Terraform + 整体发布）
parent: 部署与交付
nav_order: 4
---

# 阿里云 ACK：环境（Terraform）与整体发布（infra-live）

目标：把多仓库微服务“发布到阿里云 K8s（ACK）”，同时尽量避免**每个业务仓库都配置一遍阿里云/集群密钥**。

当前做法：

- **构建/推送镜像**：各业务仓库沿用 `infra-templates`（CI 产物=容器镜像）
- **环境与整体发布**：集中在 `lawseekdog/infra-live`

## 1) 创建 ACK（Terraform）

仓库：`lawseekdog/infra-live`

目录：`infra-live/terraform/`

最小流程（推荐：在 GitHub Actions 一键 bootstrap 远端 state）：

1) 运行 `infra-live` 的 `Bootstrap Terraform Backend (OSS)` 工作流（会创建 OSS bucket + TableStore 锁表，并写入 `TF_STATE_*` Variables）
2) 再运行 `Terraform (Aliyun)` 工作流，选择 `apply`（会创建 ACK/NodePool，并自动写入 `ALIYUN_ACK_CLUSTER_ID`）

如果 `Bootstrap Terraform Backend (OSS)` 报错：

- `oss: ... StatusCode=403, ErrorCode=UserDisable`：通常意味着 **OSS 未开通/账号不可用/或 RAM 权限不足**。请先在阿里云控制台开通 OSS，并确保所用 AK/SK 对 OSS 有权限（建议临时给 `AliyunOSSFullAccess` 验证闭环，再做最小权限收敛）。

也可以本地执行（同样推荐 OSS 远端 state）：

```bash
cd infra-live/terraform
cd bootstrap
terraform init
terraform apply -var bucket_name=lawseekdog-tfstate-<unique>

cd ../
cp backend.hcl.example backend.hcl
vim backend.hcl
terraform init -backend-config=backend.hcl

cp env/prod.tfvars.example env/prod.tfvars
vim env/prod.tfvars

export ALICLOUD_ACCESS_KEY=...
export ALICLOUD_SECRET_KEY=...

terraform apply -var-file=env/prod.tfvars
terraform output ack_cluster_id
```

将输出的 `ack_cluster_id` 写入 `infra-live` 的 Actions Variables：

- `ALIYUN_ACK_CLUSTER_ID=<ack_cluster_id>`

## 1.1) 竞价（Spot）节点与“一键释放计算资源”

`infra-live/terraform` 支持把默认 NodePool 改成 Spot（抢占式）节点（更省钱，适合 MVP/压测）。

推荐直接用内置的 tfvars：

- `infra-live/terraform/env/mvp-spot.tfvars`

并且提供一个开关用于“释放计算资源但保留 ACK/VPC 等底座”：

- `create_node_pool=false`：会销毁默认 NodePool（释放 Spot/按量实例）

在 GitHub Actions 上执行方式：

1) `Terraform (Aliyun)` → `apply`，选择 `tfvars=env/mvp-spot.tfvars`
2) 需要释放计算资源时：同一个工作流 `apply`，额外填 `create_node_pool=false`
3) 需要彻底清理（含 VPC/ACK）：`Terraform (Aliyun)` → `destroy`

## 2) 整体发布（Deploy 工作流）

`infra-live` 提供集中式发布工作流：`.github/workflows/deploy.yml`

它会：

1) 连接 ACK（`ack-set-context`）
2) checkout 各个服务仓库（只取 Helm chart）
3) 逐个 `helm upgrade --install`（统一 namespace 与公共 values）

需要在 `infra-live` 配置（Repo-level）：

- Variables:
  - `ALIYUN_REGION_ID`（如 `cn-hangzhou`）
  - `ALIYUN_ACK_CLUSTER_ID`
  - `CI_REGISTRY_PROVIDER`：`aliyun-acr`（推荐）或 `ghcr`
  - `ALIYUN_ACR_LOGIN_SERVER`：例如 `registry.cn-guangzhou.aliyuncs.com`（当使用 ACR 时必填）
- Secrets:
  - `GH_ORG_TOKEN`：用于 checkout 私有服务仓库（PAT，至少 `repo` 读权限）
  - `ALIYUN_ACCESS_KEY_ID` / `ALIYUN_ACCESS_KEY_SECRET`（或配置 OIDC 变量后不需要 AK/SK）
  - `GH_PACKAGES_USERNAME` / `GH_PACKAGES_TOKEN`（可选：当使用 `ghcr` 且镜像为私有时，用于创建 `ghcr-pull` 拉取 secret）
  - `INTERNAL_API_KEY`（可选：创建 `lawseekdog-secrets`，用于 `/internal/**` 互调）

触发方式：在 `infra-live` Actions 手动触发 `Deploy (ACK)`：

- `image_tag`：例如 `v0.2.16`（推荐）或 `main`
- `namespace`：默认 `lawseekdog`

## 3) “整体发版”建议流程（最小可落地）

由于各服务是独立仓库，推荐的最小发布流程是：

1) 给需要发布的服务仓库打同一个 tag（如 `v0.3.0`）→ 各仓库 CI 构建并推送镜像
2) 触发 `infra-live` 的 `Deploy (ACK)`，使用同一个 `image_tag=v0.3.0`

后续若要做到“一次 tag → 全部仓库自动 tag + 等待 CI 完成 + 自动部署”，可在 `infra-live` 增加一个 orchestrator 工作流（需要更高权限的 PAT），这里先保持最小闭环。
