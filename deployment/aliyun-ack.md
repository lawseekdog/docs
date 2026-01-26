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

最小流程（本地，推荐 OSS 远端 state）：

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
- Secrets:
  - `GH_ORG_TOKEN`：用于 checkout 私有服务仓库（PAT，至少 `repo` 读权限）
  - `ALIYUN_ACCESS_KEY_ID` / `ALIYUN_ACCESS_KEY_SECRET`（或配置 OIDC 变量后不需要 AK/SK）
  - `GH_PACKAGES_USERNAME` / `GH_PACKAGES_TOKEN`（可选：创建 `ghcr-pull` 拉取 secret）
  - `INTERNAL_API_KEY`（可选：创建 `lawseekdog-secrets`，用于 `/internal/**` 互调）

触发方式：在 `infra-live` Actions 手动触发 `Deploy (ACK)`：

- `image_tag`：例如 `v0.2.16`（推荐）或 `main`
- `namespace`：默认 `lawseekdog`

## 3) “整体发版”建议流程（最小可落地）

由于各服务是独立仓库，推荐的最小发布流程是：

1) 给需要发布的服务仓库打同一个 tag（如 `v0.3.0`）→ 各仓库 CI 构建并推送镜像
2) 触发 `infra-live` 的 `Deploy (ACK)`，使用同一个 `image_tag=v0.3.0`

后续若要做到“一次 tag → 全部仓库自动 tag + 等待 CI 完成 + 自动部署”，可在 `infra-live` 增加一个 orchestrator 工作流（需要更高权限的 PAT），这里先保持最小闭环。
