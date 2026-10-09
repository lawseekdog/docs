---
title: auth-service（已并入 user-service）
parent: 模块
nav_order: 10
---

# auth-service（已并入 user-service）

auth-service 已于 2026-10-07 并入 user-service 并退役，不再是运行中的服务。认证、授权与服务间鉴权由
[user-service](./user-service.md) 提供：

- 登录/Token：`/api/v1/auth/**`
- RBAC 权限/角色/资源：`/api/v1/rbac/**`
- 服务间鉴权与权限校验：`/api/v1/internal/auth/**`（例如 collector-service 通过 `validate-token` 校验 collector:read / collector:manage）

契约以 user-service 实现为准；auth-service 仓库与 `auth_service` 数据库仅作历史证据保留。
