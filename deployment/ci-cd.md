---
title: CI/CD
parent: 部署与交付
nav_order: 2
---

# CI/CD（复用工作流 → 镜像构建/推送 + 可选 Helm 部署）

LawSeekDog 的 Java 微服务采用“复用工作流”模式：

- 公共工作流：`lawseekdog/infra-templates`
- 业务仓库只保留薄入口：`.github/workflows/ci.yml`

目前区分两类仓库：

- Java 服务：使用 `java-service-ci.yml`（Maven 构建 + 镜像构建/推送 + 可选 Helm 部署）
- 非 Java（Python/Node 等）：使用 `docker-service-ci.yml`（直接构建镜像 + 可选 Helm 部署）

## 1) 触发规则（业务仓库）

默认触发：

- push `main`
- push tag `v*`
- pull request → `main`

行为：

- PR：构建（不推送镜像）
- main/tag：构建并推送镜像到容器镜像仓库（默认 GHCR，可切换到阿里云 ACR）
- 部署：推荐由 `infra-live` 统一执行（避免每个业务仓库都持有部署凭据）

## 2) 镜像命名与标签

- 默认镜像仓库：`ghcr.io/<org>/<repo>`
- 可切换到：阿里云 ACR（示例：`registry.cn-hangzhou.aliyuncs.com/<namespace>/<repo>`）
- 标签由 `docker/metadata-action` 生成，常见包括：
  - `main`
  - `vX.Y.Z`
  - `sha-<short>`

## 2.1) 切换到阿里云 ACR（推荐 OIDC）

`infra-templates` 的工作流支持 `aliyun-acr`（阿里云 ACR）。两种方式：

1) 推荐：OIDC（无长期 AK/SK，安全/可审计）
2) 备选：仓库 Secrets 放 AK/SK（简单但需要妥善轮换）

### 开启方式（无需改 workflow，只用 Variables/Secrets）

在**业务仓库**配置 Variables（Actions → Variables）：

- `CI_REGISTRY_PROVIDER=aliyun-acr`
- `ALIYUN_ACR_LOGIN_SERVER=registry.cn-hangzhou.aliyuncs.com`（示例）
- `ALIYUN_REGION_ID=cn-hangzhou`（示例）
- `ALIYUN_OIDC_PROVIDER_ARN=acs:ram::<uid>:oidc-provider/<name>`
- `ALIYUN_ROLE_TO_ASSUME=acs:ram::<uid>:role/<roleName>`

并确保业务仓库 `.github/workflows/ci.yml` 包含：

```yaml
permissions:
  contents: read
  packages: write
  id-token: write
```

> `id-token: write` 是 OIDC 所必需的权限；没有它将无法获取阿里云 STS 临时凭据。

### AK/SK fallback（不推荐长期使用）

在业务仓库配置 Secrets（Actions → Secrets）：

- `ALIYUN_ACCESS_KEY_ID`
- `ALIYUN_ACCESS_KEY_SECRET`

## 3) Maven 私有依赖（GitHub Packages）

若业务仓库依赖 `ai-boot-framework` 的私有 Maven 包，需要在**每个业务仓库**配置：

- `GH_PACKAGES_TOKEN`：PAT（至少 `read:packages`；私有仓库通常还需要 `repo`）
- `GH_PACKAGES_USERNAME`：可选（不配时会回退到 `github.actor`）

模板工程已内置：

- `.mvn/settings.xml`：从 `env.GH_PACKAGES_*` 读取凭据
- `.mvn/maven.config`：强制 Maven 使用仓库内 settings

## 4) 关于 Actions 存储配额（Artifacts/Caches）

为降低 `Failed to CreateArtifact: Artifact storage quota has been hit` 风险：

- Java 服务 CI 不上传构建制品（以容器镜像为交付物）
- 默认不启用 Actions Cache（Maven/Docker），避免缓存也占用配额

## 5) 集中式部署（infra-live，推荐用于“私有仓库 + 无法用组织级 Secrets”的场景）

当组织无法对私有仓库使用 Organization secrets（套餐限制）时，把“部署凭据”分散到每个业务仓库会非常痛苦。

因此我们新增了 `lawseekdog/infra-live`：

- 业务仓库：只负责构建并推送镜像（CI 产物=镜像）
- `infra-live`：集中持有阿里云凭据与集群 SSH 凭据，负责“整体发布”（见 `deployment/aliyun-ack.md`）

这样做的收益：

- 阿里云/部署密钥只在一个仓库配置
- 业务仓库仍可保持最小 CI 入口（引用 `infra-templates`）
