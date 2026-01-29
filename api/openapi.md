---
title: OpenAPI 与运维端点
parent: API 与规范
nav_order: 3
---

# OpenAPI 与内部运维端点

## 1) OpenAPI 输出（Java 服务）

各 Java 微服务基于 SpringDoc 输出 OpenAPI：

- OpenAPI JSON：`GET /api/v1/v3/api-docs`
- Swagger UI（若依赖启用）：`GET /api/v1/swagger-ui/index.html`

框架侧默认会过滤路径：

- 只保留对外路径：`/api/**`
- 默认隐藏内部路径：`/api/v1/internal/**`（即对外表现为 `/api/v1/internal/**`）

对应配置（可覆盖）：

- `ai.boot.openapi.enabled`
- `ai.boot.openapi.include-path-prefixes`
- `ai.boot.openapi.exclude-path-prefixes`

## 2) Actuator（内部运维端点）

框架默认把 Actuator 放到内部路径，便于统一保护：

- 基础路径：`/api/v1/internal/actuator`
- 常用端点：
  - `/api/v1/internal/actuator/health`
  - `/api/v1/internal/actuator/health/liveness`
  - `/api/v1/internal/actuator/health/readiness`
  - `/api/v1/internal/actuator/prometheus`

说明：

- Helm Chart 默认把 liveness/readiness 指向上述路径（见各服务 `deploy/helm/values.yaml`）。
