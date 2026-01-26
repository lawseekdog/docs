---
title: Helm（部署到 K8s）
parent: 部署与交付
nav_order: 3
---

# Helm（部署到 K8s）

各 Java 微服务仓库默认内置 Helm Chart：

- 目录：`deploy/helm`
- 默认镜像：`ghcr.io/lawseekdog/<service>`（也可切换到阿里云 ACR）
- 默认 tag：`main`（与 CI 对齐）

## 1) 安装/升级（示例）

以 `user-service` 为例：

```bash
helm upgrade --install user-service ./deploy/helm \
  --namespace user-service --create-namespace \
  --set image.repository=ghcr.io/lawseekdog/user-service \
  --set image.tag=main
```

若使用阿里云 ACR（示例）：

```bash
helm upgrade --install user-service ./deploy/helm \
  --namespace user-service --create-namespace \
  --set image.repository=registry.cn-hangzhou.aliyuncs.com/lawseekdog/user-service \
  --set image.tag=v0.1.0
```

## 2) 配置注入（建议）

Chart 提供：

- `env`：显式环境变量列表
- `envFrom`：从 ConfigMap/Secret 批量注入

生产建议把以下敏感配置放到 Secret：

- `INTERNAL_API_KEY`
- `SPRING_DATASOURCE_URL/USERNAME/PASSWORD`
- `AI_BOOT_SECURITY_JWT_HMAC_SECRET`（若启用 JWT）
- OTLP endpoint / tracing 相关参数（若启用）

> 具体 values 字段以各服务 `deploy/helm/values.yaml` 为准。
