---
title: platform-service（平台配置与控制面）
parent: 模块
nav_order: 9
---

# platform-service（平台配置与控制面）

## 定位

platform-service 承载“平台级配置/字典/审计/标签”等控制面能力，并为其它服务提供可查询的配置来源：

- SystemConfig（key/value 配置；如 `matters.service_types`）
- FeatureFlag（功能开关）
- Tag/TagGroup/BusinessCategory/Domain（分类/标签体系）
- AuditLog（审计）
- AI 侧管理代理：列 skills、usage 统计、graph 配置等（对接 ai-engine）

历史兼容（逐步下线）：

- PlaybookConfig（旧架构遗留；workbench-mode 不再注入/解析 `playbook_config`）

## 技术栈

- Java 21 + Spring Boot
- Postgres + Flyway（默认）

## 与其它服务的关系

- matter-service：通过 `getConfig(maters.service_types)` 拉取并缓存服务类型
- consultations-service：主要用于组织/标签/配置读取（不再需要 playbook_config）
- collector-service：通过 seed packages 批量导入/更新配置与字典
- ai-engine：平台侧对 AI 的“查询/统计/配置”可通过 platform-service 做统一入口代理

## 对外 API（/api/v1，摘要）

按 controller 命名大致包含：

- tags/domains/feature_flags/system_config/audit_log 等业务接口
- `GET /api/v1/ai/skills`、`GET /api/v1/ai/skills/{skillName}`（代理 ai-engine）
- `GET /api/v1/ai/admin/usage`、`/usage/stats`（AI 使用统计，代理 ai-engine 或基于平台数据聚合）

## 对内 API（/internal，摘要）

seed/批量导入与内部查询：

- `GET  /api/v1/internal/platform/configs/{key}`（读取 SystemConfig）
- `POST /api/v1/internal/platform/configs/batch`（批量写入/更新）

> 其它 internal 控制器以 OpenAPI 为准。

## 关键配置：matters.service_types

事项中心的“服务类型选择配置”存于 platform-service 的 SystemConfig：

- key：`matters.service_types`
- value：JSON（包含 `service_types` / `resolved_service_types`）

该配置通常由 seed packages 写入/更新（见 `implementation/seed-packages.md`）。
