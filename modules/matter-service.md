---
title: matter-service（事项中心）
parent: 模块
nav_order: 4
---

# matter-service（事项中心）

## 定位

matter-service 是“事项（Matter）”的真源服务，负责法律服务的全生命周期承载：

- Matter：事项主实体（状态、服务类型、playbook、关联文件、承办信息等）
- Todo：待办任务（用于工作台/派单/阶段确认等）
- 阶段推进：与 Playbook 门控/确认点协同
- 结构化产物承载：证据分析、争点、策略、风险评估等（以当前字段与 API 为准）

在当前实现中，“咨询会话”会自动创建一个 DRAFT matter（参见 `flows/consultation-to-matter.md`）。

## 技术栈

- Java 21 + Spring Boot
- Postgres + Flyway（默认）
- Spring Data JPA

## 核心数据模型（概念）

以当前代码为准（ID 多为 Long）：

- Matter
  - `status`：例如 `DRAFT` 等
  - `service_type_id`：决定 playbook 与业务类型
  - `playbook_id`：从 platform-service 的 playbook configs 解析/缓存
  - `source_consultation_session_id`：关联咨询会话（自动创建时写入）
- MatterTodo
  - `todo_key`：语义幂等键（内部 upsert/完成按 key 或 id）
  - actor/review_type/payload/result：用于卡片/确认点
- MatterAnalysisVersion、MatterPhaseProgress 等（用于分析结果与阶段状态）

## 对外 API（/api/v1）

对外 API 面向前端工作台/用户界面：

- `GET/POST/PUT ... /api/v1/matters...`（事项 CRUD/列表）
- `GET/POST ... /api/v1/matters/{id}/todos...`（待办查询/完成）
- 管理端/律师端接口：以 controller 命名为准（例如 `LawyerMatterController`、`LawyerTodoController`）

> 具体入参/返回以 OpenAPI 为准（见 `api/openapi.md`）。

## 对内 API（/internal）

### 1) 从咨询创建事项

- `POST /api/v1/internal/matters/from-consultation`

该接口用于 consultations-service 在对话中确保绑定 matter：

- 若用户类型为 lawyer/firm_admin，matter 会直接绑定承办律师/律所（对齐 legacy 逻辑）
- 会创建 intake 收集 todo；若 providerOrganizationId 存在但未绑定 lawyer，则创建派单 todo

### 2) 工作流状态与同步

- `GET  /api/v1/internal/matters/{matterId}`（内部读取）
- `GET  /api/v1/internal/matters/{matterId}/workflow/profile`（给 ai-engine/consultations 用的工作流 profile）
- `POST /api/v1/internal/matters/{matterId}/sync/all`（内部同步入口，供 ai-engine 写入状态/产物）

### 3) 结构化分析结果提交（internal）

该组接口用于提交证据/争点/策略等产物（由 ai-engine skill/tool 调用）：

- `POST /api/v1/internal/matters/{matterId}/analysis/versions/allocate`
- `GET  /api/v1/internal/matters/{matterId}/analysis/versions/latest`
- `POST /api/v1/internal/matters/{matterId}/evidences/submit-analysis`
- `POST /api/v1/internal/matters/{matterId}/sufficiency/submit`
- `POST /api/v1/internal/matters/{matterId}/issues/submit`
- `POST /api/v1/internal/matters/{matterId}/strategies/submit`
- `POST /api/v1/internal/matters/{matterId}/risk-assessment/submit`
- `POST /api/v1/internal/matters/{matterId}/defense-strategy/submit`
- `POST /api/v1/internal/matters/{matterId}/appeal-strategy/submit`

## 与 platform-service 的关系（配置/Playbook）

matter-service 的 ServiceType 与 Playbook 都来自 platform-service：

- ServiceType 配置 key：`matters.service_types`
  - 由 collector-service 的 `matters_system_resources` seed 包写入 platform-service
  - matter-service 通过 `PlatformServiceTypeRegistry` 拉取并缓存
- PlaybookConfig：由 seed 包导入 platform-service
  - matter-service 通过 `PlatformPlaybookRegistry` 拉取并缓存

这使得“新增服务类型/调整 playbook”无需改 matter-service 代码，仅需更新 seed/config。

## 现状与边界

- 已具备“咨询自动生成事项 + todo 初始化 + internal 同步/提交”骨架。
- 阶段推进/门控规则依赖 ai-engine 的 playbook 校验与技能输出字段一致性（需要测试与回归保障）。
