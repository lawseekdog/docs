---
title: infra-live（环境 Terraform + 整体发布）
parent: 模块
nav_order: 99
---

# infra-live（环境 Terraform + 整体发布）

## 定位

`infra-live` 是“环境与交付”仓库：

- 负责在阿里云创建运行环境（VPC + ECS + 自建 k3s）（Terraform）
- 负责把各微服务部署到自建集群（集中式 Helm 发布，SSH 到 master 执行）

它与 `infra-templates` 的分工：

- `infra-templates`：每个业务仓库的 CI 模板（构建镜像、可选单仓库部署）
- `infra-live`：统一持有环境凭据，做“整体发布”（更适合私有仓库+无法使用组织级 Secrets 的限制）

## 目录结构

- `infra-live/terraform/`：VPC + ECS + k3s（最小闭环）
- `infra-live/deploy/`：服务清单 + 公共 values + Deploy 工作流

## 关键工作流

- `infra-live/.github/workflows/terraform.yml`
  - push/PR：fmt + init + validate（不做 plan，避免缺少 tfvars/凭据导致失败）
  - workflow_dispatch：支持 `plan/apply/destroy`；并可覆盖 `create_instances=false` 以释放计算资源（需要 repo secrets 的阿里云凭据）
  - apply 后会把 `K8S_MASTER_PUBLIC_IP`/`K8S_MASTER_PRIVATE_IP` 写入 repo variables；当实例被释放/回收导致 IP 为空时，会删除这些变量以避免“陈旧 IP”
- `infra-live/.github/workflows/deploy.yml`
  - workflow_dispatch：按指定 `image_tag` 统一部署全部服务（从各服务仓库拉 Helm chart）
  - 可选 `addons`：统一安装基础设施组件（Postgres/MinIO/Qdrant/Elasticsearch/api-gateway/OnlyOffice Document Server）
  - 通过 `CI_REGISTRY_PROVIDER` 选择镜像仓库（`ghcr` 或 `aliyun-acr`）
  - 部署阶段会创建/刷新 `imagePullSecret`，并在最后执行 `kubectl rollout status` 校验发布健康

- `infra-live/.github/workflows/publish-base-images.yml`
  - 发布基础设施镜像到 GHCR（用于国内网络下避免拉取 docker.io 不稳定）
  - 当前覆盖：postgres、minio、qdrant、elasticsearch、nginx、onlyoffice-documentserver（通常为 amd64-only）

## OnlyOffice（可选）

启用方式：在 `Deploy (Self-Managed K8s)` 工作流输入 `addons` 时追加 `onlyoffice-documentserver`。

本地访问（最简单）：通过端口转发暴露 Document Server，然后 templates-service 的默认配置即可工作：

```bash
kubectl -n lawseekdog port-forward svc/onlyoffice-documentserver 18010:80
```

如需公网/域名访问，请在集群侧自行提供 Ingress/反代，并设置 `templates-service` 的 `TEMPLATES_ONLYOFFICE_PUBLIC_URL`（对应 `templates.onlyoffice.public-url`）。

## MinIO（重要）

前端下载附件使用的是 `GET /api/v1/files/{fileId}/download-url` 返回的 MinIO **预签名 URL**。因此：

- MinIO 必须有一个浏览器可访问的公网 endpoint
- `files-service` 需要配置 `MINIO_PUBLIC_ENDPOINT`（用于生成预签名 URL 的 host）

在 `infra-live` 的默认部署里：

- `addons/minio.yaml` 使用 k3s 的 ServiceLB 暴露 `9000/9001`（Node 公网 IP + 端口）
- Deploy 工作流会把 `MINIO_PUBLIC_ENDPOINT` 默认写成 `${master_public_ip}:9000`（写入 `lawseekdog-secrets`，所有服务通过 `envFrom` 注入）

注意：需要在阿里云安全组放通 `9000/9001`（Terraform 已包含规则）。

## Postgres（重要）

在 `infra-live` 的默认部署里：

- `addons/postgres.yaml` 内置了一个 initdb `ConfigMap`，首次初始化数据目录时会创建各微服务数据库
- Deploy 工作流在发布时也会做一次“确保 DB 存在”（`kubectl exec psql ...`），用于兼容后续新增服务/数据库的情况

## 资源与稳定性（MVP 单机）

- `infra-live/terraform/env/mvp-k3s-spot.tfvars` 默认使用 `ecs.g7.xlarge`（4c16g），更适合“全量微服务 + ES + MinIO + Qdrant”一键起环境
- Deploy 会在 `lawseekdog-secrets` 中写入默认 `JAVA_TOOL_OPTIONS=-Xms128m -Xmx512m`，避免多 Spring Boot 服务并发启动导致节点 OOM

## Frontend 访问（www.lawseekdog.com）

`frontend` 服务默认是 `ClusterIP`，通过 k3s 自带的 Traefik Ingress 暴露到公网：

- Ingress：`infra-live/deploy/values/services/frontend.yaml`
  - `ingress.enabled=true`
  - `ingress.className=traefik`
  - `ingress.host=www.lawseekdog.com`

## DNS（Spot 场景自动跟随公网 IP）

Spot master 公网 IP 会变化。`infra-live/terraform/` 提供可选的 AliDNS 记录自动更新：

- `enable_dns=true`
- `domain_name="lawseekdog.com"`
- `subdomain="www"`

配置在：`infra-live/terraform/env/mvp-k3s-spot.tfvars`

## 运行所需的变量/密钥（Repo-level）

Variables：

- `ALIYUN_REGION_ID`（用于镜像推送/registry 配置）
- `SSH_PUBLIC_KEY`（Terraform：创建 ECS KeyPair）
- `K8S_MASTER_PUBLIC_IP`（Terraform apply 后自动写入；Deploy 目标；变量缺失时 Deploy 会自动从 Terraform state 解析）
- `K8S_SSH_USER`（默认 `root`）
- `CI_REGISTRY_PROVIDER`（`ghcr` 或 `aliyun-acr`）
- `ALIYUN_ACR_LOGIN_SERVER`（当使用 `aliyun-acr` 时需要）
- `ALIYUN_CR_REGION_ID`（可选；`aliyun/acr-login` 调用 CR API 的 region，默认 `cn-hangzhou`）

Secrets：

- `GH_ORG_TOKEN`：checkout 私有服务仓库（仅部署时需要）
- `ALIYUN_ACCESS_KEY_ID` / `ALIYUN_ACCESS_KEY_SECRET`（未使用 OIDC 时）
- `K8S_SSH_PRIVATE_KEY`：Deploy SSH 到 master
- （推荐）`GH_PACKAGES_TOKEN`：私有 GHCR 拉取镜像（需要 `read:packages`；私有仓库通常还要 `repo`）
  - `GH_PACKAGES_USERNAME` 可选（未配置时 deploy 默认用触发人 `github.actor`）
  - 若你的 `GH_ORG_TOKEN` 同时具备 `read:packages`，也可复用它来拉取 GHCR
- `INTERNAL_API_KEY`：创建 `lawseekdog-secrets`（让 `/internal/**` 互调一致，必填）
- `AI_BOOT_SECURITY_JWT_HMAC_SECRET`：Auth JWT HMAC secret（>=32 bytes，必填）
- `MINIO_ACCESS_KEY` / `MINIO_SECRET_KEY`：MinIO 根账号（必填）
- `OPENROUTER_API_KEY` 或 `DEEPSEEK_API_KEY`：LLM 调用密钥（二选一必填）
- （可选）`ONLYOFFICE_AI_API_KEY`、`ALIYUN_ACCESS_KEY_ID` / `ALIYUN_ACCESS_KEY_SECRET`、`TAVILY_API_KEY`

说明：
- 其他非敏感配置（如 `DEFAULT_LLM_PROVIDER`、`KNOWLEDGE_*`、`RERANK_*`、`FILES_OCR_*` 等）可通过 Repo Variables 设置；
- Deploy 会把这些键值写入 `lawseekdog-secrets` 并通过 `envFrom` 注入到各服务；
- 变量命名与根目录 `env.txt` 保持一致（生产模板参考）。
