---
title: collector-service（Seed/采集分发服务）
parent: 模块
nav_order: 2
---

# collector-service（Seed/采集分发服务）

## 定位

collector-service 在当前系统里承担两类职责：

1) Seed Packages 分发（系统资源/Playbooks/结构化 seeds/模板资源）  
2) 采集/复核工作流（sources/runs/documents 的管理与回放；以当前实现为准）

对于“多服务配置一致性”，Seed Packages 是更关键的主链路（见 `implementation/seed-packages.md`）。

## 技术栈

- Python >= 3.11
- FastAPI
- SQLAlchemy + Postgres
- 内部鉴权：`X-Internal-Api-Key`（seed packages 列表/manifest/download 等）
- 权限校验：通过 auth-service 校验 permission（collector:read / collector:manage）

## API（摘要）

collector-service 同时挂载到：

- `/api/v1`（管理端）
- `/internal`（服务间/运维/CI）

### 1) Seed Packages（核心）

- `GET  /api/v1/internal/seed-packages`（internal key）
- `GET  /api/v1/internal/seed-packages/{package_id}/manifest`（internal key）
- `GET  /api/v1/internal/seed-packages/{package_id}/download`（internal key）
- `POST /api/v1/seed-packages/{package_id}/apply`（需要登录权限）
- `POST /api/v1/internal/seed-packages/apply-internal`（internal key）

### 2) Collector（sources/runs/documents）

- `/api/v1/collector/sources`、`/collector/runs`、`/collector/documents` 等（需要登录权限）

> 具体入参/返回以 OpenAPI/实现为准。

## 启动时自动 seed（env）

collector-service 支持启动时自动执行种子包：

- `SEED_PACKAGE_IDS_ON_STARTUP=...`（逗号分隔）
- 或 `SEED_RESOURCES_ON_STARTUP=true` 使用默认集合

在 dev/local/test 环境允许 seed 失败后继续启动（便于排错），生产环境会强制失败。

## 状态（工程化）

当前 `collector-service/Dockerfile` 仍保留 mono-repo 路径假设（需要对齐独立仓库构建）。
迁移说明参见：`architecture/repositories.md`。
