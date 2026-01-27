---
title: 阿里云（ECS + 自建 k3s）（Terraform + 整体发布）
parent: 部署与交付
nav_order: 4
---

# 阿里云（ECS + 自建 k3s）：环境（Terraform）与整体发布（infra-live）

目标：把多仓库微服务发布到阿里云（自建 K8s），同时尽量避免每个业务仓库都配置一遍“阿里云/部署密钥”。

当前做法：

- 构建/推送镜像：各业务仓库沿用 `infra-templates`（CI 产物=容器镜像）
- 环境与整体发布：集中在 `lawseekdog/infra-live`

## 1) 创建自建 k3s 集群（Terraform）

仓库：`lawseekdog/infra-live`

目录：`infra-live/terraform/`

最小流程（推荐：GitHub Actions 一键 bootstrap 远端 state）：

1) 运行 `infra-live` 的 `Bootstrap Terraform Backend (OSS)` 工作流（创建 OSS bucket + 可选 TableStore 锁表，并写入 `TF_STATE_*` Variables）
2) 配置 `infra-live` 的 Repo-level Variables/Secrets（见 1.1）
3) 运行 `Terraform (Aliyun)` 工作流，选择 `apply`（创建 VPC + ECS + k3s；Deploy 会从 Terraform state 自动解析 master IP，不强依赖 `K8S_MASTER_PUBLIC_IP` 变量）

在中国大陆网络环境下的额外注意：

- k3s 默认会从 `docker.io` 拉取部分系统镜像（例如 `rancher/mirrored-pause`），可能出现超时导致 Pod 一直 `ContainerCreating`。
- `infra-live` 已在 k3s user-data 中配置了 `docker.io` 镜像加速（优先 `registry.aliyuncs.com`）。

如果 `Bootstrap Terraform Backend (OSS)` 报错：

- `oss: ... StatusCode=403, ErrorCode=UserDisable`：通常是 OSS 未开通/账号不可用/或 RAM 权限不足。请先在阿里云控制台开通 OSS，并确保 AK/SK 对 OSS 有权限（可先临时授予 `AliyunOSSFullAccess` 验证闭环）。

本地执行（同样推荐 OSS 远端 state）：

```bash
cd infra-live/terraform/bootstrap
terraform init
terraform apply -var region_id=cn-guangzhou -var bucket_name=lawseekdog-tfstate-<unique>

cd ../
cp backend.hcl.example backend.hcl
vim backend.hcl
terraform init -backend-config=backend.hcl

cp env/prod.tfvars.example env/prod.tfvars
vim env/prod.tfvars

export ALICLOUD_ACCESS_KEY=...
export ALICLOUD_SECRET_KEY=...
export TF_VAR_ssh_public_key="$(cat ~/.ssh/id_rsa.pub)"

terraform apply -var-file=env/prod.tfvars
terraform output master_public_ip
```

## 1.1) infra-live 需要的 Repo-level Variables / Secrets（最小闭环）

Variables：

- `ALIYUN_REGION_ID`：如 `cn-guangzhou`（registry/脚本使用）
- `SSH_PUBLIC_KEY`：Terraform 用（创建 ECS KeyPair）
- `TF_STATE_BUCKET` / `TF_STATE_PREFIX` / `TF_STATE_KEY` / `TF_STATE_REGION`
- （可选）`TF_STATE_TABLESTORE_ENDPOINT` / `TF_STATE_TABLESTORE_TABLE`：启用 state lock
- `CI_REGISTRY_PROVIDER=ghcr|aliyun-acr`（默认 ghcr）
- `ALIYUN_ACR_LOGIN_SERVER`：当使用 `aliyun-acr` 时需要（例如 `registry.cn-guangzhou.aliyuncs.com`）
  - `K8S_SSH_USER`：默认 `root`
  - `K8S_MASTER_PUBLIC_IP`：可选（Deploy 会从 Terraform state 自动解析；写入该变量仅用于兜底/加速）

Secrets：

- `ALIYUN_ACCESS_KEY_ID` / `ALIYUN_ACCESS_KEY_SECRET`：Terraform 用（建议后续换最小权限 RAM 用户 / OIDC）
- `K8S_SSH_PRIVATE_KEY`：Deploy SSH 到 master
- `GH_ORG_TOKEN`：Deploy 时 checkout 私有服务仓库（只读 repo 即可）
- （推荐）`GH_PACKAGES_TOKEN`：私有 GHCR 拉取 secret（需要 `read:packages`；私有仓库通常还要 `repo`）
- （可选）`GH_PACKAGES_USERNAME`：未配置时 Deploy 默认用触发人 `github.actor`
- （可选）`INTERNAL_API_KEY`：创建 `lawseekdog-secrets`（内部互调）

## 1.2) 竞价（Spot）与“一键释放计算资源”

`infra-live/terraform` 支持 Spot（抢占式）ECS（更省钱，适合 MVP/压测）。

推荐直接使用：

- `infra-live/terraform/env/mvp-k3s-spot.tfvars`

释放计算资源（保留 VPC/数据盘等底座）：

- `create_instances=false`

在 GitHub Actions 上执行方式：

1) `Terraform (Aliyun)` → `apply`，选择 `tfvars=env/mvp-k3s-spot.tfvars`
2) 需要释放计算资源：同一个工作流 `apply`，额外填 `create_instances=false`
3) 需要彻底清理（含 VPC/数据盘等）：`Terraform (Aliyun)` → `destroy`

## 2) 整体发布（Deploy 工作流，SSH 到 master）

`infra-live` 提供集中式发布工作流：`.github/workflows/deploy.yml`

它会：

1) checkout 各个服务仓库（只取 Helm chart）
2) 打包 charts/values bundle 并上传到 k3s master
3) SSH 到 master 执行 `kubectl/helm`，逐个 `helm upgrade --install`

触发方式：在 `infra-live` Actions 手动触发 `Deploy (Self-Managed K8s)`：

- `image_tag`：例如 `v0.2.16`（推荐）或 `main`
- `namespace`：默认 `lawseekdog`

## 3) “整体发版”建议流程（最小可落地）

由于各服务是独立仓库，推荐的最小发布流程：

1) 给需要发布的服务仓库打同一个 tag（如 `v0.3.0`）→ 各仓库 CI 构建并推送镜像
2) 触发 `infra-live` 的 `Deploy (Self-Managed K8s)`，使用同一个 `image_tag=v0.3.0`

## 4) RAM 最小权限（建议）

如果不使用主账号的 AK/SK，建议创建一个专用 RAM 用户（给 GitHub Actions 用），并附加最小系统策略（先跑通再收敛）：

- `AliyunECSFullAccess`
- `AliyunVPCFullAccess`
- `AliyunOSSFullAccess`（Terraform remote state）

> 自建 k3s 不依赖 ACK 服务角色，因此不需要 `AliyunCS*` 相关授权。
