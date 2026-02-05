---
title: matter-service（事项中心）
parent: 模块
nav_order: 4
---

# matter-service（事项中心）

## 定位

matter-service 是“事项（Matter）”的真源服务，负责法律服务的全生命周期承载与协同：

- Matter：事项主实体（状态、业务分类、服务标签、关联文件、承办信息等）
- Todo：待办任务（工作台派单、确认点、交付复核等）
- PhaseProgress：阶段/里程碑进度（用于时间线展示与过程追踪）
- Deliverables：交付件索引（file_id / presigned_url 等）
- 对内同步：接收 `ai-engine` 的结构化产物与决策字段同步

> 重要：事项的“流程编排/门禁/下一步做什么”由 `ai-engine` 的 LangGraph 工作流负责；matter-service 不承担 Playbook/编排器角色。

## 技术栈

- Java 21 + Spring Boot
- Postgres + Flyway（默认）
- Spring Data JPA

## 核心数据模型（概念）

以当前代码为准：

- Matter
  - `status`：例如 `DRAFT` / `ACTIVE` 等
  - `matter_category`：诉讼/仲裁/非诉…（决定大方向与入口）
  - `cause_of_action_code`：案由/争议类型字典码（建议对齐 [`architecture/law.md`](../architecture/law.md)）
  - `service_type_id`：服务标签（产品/运营维度；用于 UI/路由提示，不应作为底层强绑定键）
  - `file_ids`：事项关联材料文件
- MatterTodo
  - `todo_key`：语义幂等键（upsert/完成按 key 或 id）
  - actor/review_type/payload/result：用于卡片/确认点
- MatterPhaseProgress
  - `phase_id/phase_name/status/order_num`：阶段时间线（由系统/引擎写入）

## 对外 API（/api/v1）

对外 API 面向前端工作台/用户界面：

- `GET/POST/PUT ... /api/v1/matters...`（事项 CRUD/列表）
- `GET/POST ... /api/v1/matters/{id}/todos...`（待办查询/完成）
- 时间线/交付件：以 controller 命名为准（例如 phase timeline、deliverables）

> 具体入参/返回以 OpenAPI 为准（见 `api/openapi.md`）。

## 对内 API（/internal）

### 1) 从咨询创建事项

- `POST /api/v1/internal/matters/from-consultation`

该接口用于 consultations-service 在对话中确保绑定 matter：

- 若用户类型为 lawyer/firm_admin，matter 会直接绑定承办律师/律所（对齐现有逻辑）
- 会创建 intake 收集 todo；若 providerOrganizationId 存在但未绑定 lawyer，则创建派单 todo

### 2) 工作流 profile 与同步

- `GET  /api/v1/internal/matters/{matterId}`（内部读取）
- `GET  /api/v1/internal/matters/{matterId}/workflow/profile`（给 ai-engine 的白名单 profile）
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

## 与 platform-service 的关系（配置）

matter-service 的“服务类型配置”来自 platform-service 的 SystemConfig：

- 配置 key：`matters.service_types`
- value：JSON（包含 `service_types` / `resolved_service_types`）

注意：当前 matter-service 会在对外/对内输出中 **主动移除历史字段 `playbook_id`**，避免把服务类型强绑定到 Playbook。

## FlowBlueprint（流程蓝图，可选）

matter-service 支持“流程蓝图（FlowBlueprint）”配置查询（主要面向管理端/调试）：

- 加载来源：classpath `config/flow-blueprints/*.json`（可为空）
- 典型用途：为前端/运营提供“流程/阶段的展示或提示”元信息

> 说明：workbench-mode 的实际编排在 ai-engine LangGraph 中完成；FlowBlueprint 不作为运行时编排 DSL。

## 现状与边界

- 已具备“咨询自动生成事项 + todo 初始化 + internal 同步/提交 + 时间线/交付件”骨架。
- 流程编排在 ai-engine：goal/阶段/门禁以 LangGraph 子图表达；matter-service 只做真源承载与协同。
