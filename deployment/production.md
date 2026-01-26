---
title: 生产部署要点
parent: 部署与交付
nav_order: 5
---

# 生产部署要点（配置/密钥/依赖）

本页提供“把服务跑在生产 K8s”时的最低检查清单（以当前模板与框架为准）。

## 1) 必配密钥

- `INTERNAL_API_KEY`：用于 `/internal/**` 调用鉴权（所有服务必须一致）
- JWT（若启用）：
  - `AI_BOOT_SECURITY_JWT_ENABLED=true`
  - `AI_BOOT_SECURITY_JWT_HMAC_SECRET=<secret>`

## 2) 数据库与迁移

- 每个服务独立库（推荐）
- Flyway：生产必须开启（默认开启），并把迁移脚本与版本发布绑定

## 3) 可观测性

- 日志：确保输出包含 `request_id`（以及 trace/span id）
- 指标：`/internal/actuator/prometheus`
- 追踪：按需配置 OTLP exporter（对接 OpenTelemetry Collector）

## 4) 网络与安全

- `/internal/**`：只允许集群内访问（NetworkPolicy/Ingress 规则）
- 对外入口：建议通过 Ingress/Gateway 统一鉴权、限流与审计

## 5) 镜像与发布

- 推荐以 tag 发布（`vX.Y.Z`），并把 Helm values 固定到具体 tag（避免“main 漂移”）
