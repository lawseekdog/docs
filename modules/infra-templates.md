---
title: infra-templates（复用 CI/CD 工作流）
parent: 模块
nav_order: 17
---

# infra-templates（复用 CI/CD 工作流）

## 定位

`infra-templates` 用于把“Java 微服务的 CI/CD”集中维护，避免每个业务仓库重复写一套 workflows。

业务仓库只保留一个薄的入口工作流：

```yaml
uses: lawseekdog/infra-templates/.github/workflows/java-service-ci.yml@v1
secrets: inherit
```

对于非 Java（Python/Node）仓库，则使用：

```yaml
uses: lawseekdog/infra-templates/.github/workflows/docker-service-ci.yml@v1
secrets: inherit
```

## 当前覆盖范围（现状）

已对齐的 Java 服务仓库（示例）：

- auth-service、user-service、matter-service、consultations-service、knowledge-service、templates-service、files-service、platform-service 等

## 工作流能力（概览）

- PR：`mvn -q package` + `docker build` 校验（不推送镜像）
- main / tag（v*）：构建并推送到容器镜像仓库（默认 GHCR，可切换到阿里云 ACR）
  - 多架构：`linux/amd64` + `linux/arm64`
  - 标签：`main`、`vX.Y.Z`、`sha-<short>`
- 可选：tag（v*）后自动 Helm 部署到 ACK（默认关闭）

非 Java 仓库：

- PR：直接 `docker build` 校验（不推送镜像）
- main / tag（v*）：构建并推送容器镜像（同上）

## 重要约束：Actions 存储配额

组织 Actions 存储配额曾因缓存（Caches）占用过大导致失败；
因此当前工作流默认关闭缓存写入（牺牲构建速度换稳定性）：

- 不启用 `setup-java cache: maven`
- 不使用 `docker/build-push-action` 的 `type=gha` cache

如需恢复缓存，应配合清理策略与配额确认后再启用。
