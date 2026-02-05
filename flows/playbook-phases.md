---
title: Playbook 配置规范（Legacy，已废弃）
parent: 业务流程
nav_order: 99
---

# Playbook（Legacy，已废弃）

从 workbench-mode 开始，主链路不再注入/解析 `playbook_config`：流程由 `ai-engine` 的 LangGraph 工作流图直接编排。

- 现行入口：`ai-engine/src/application/agent/graphs/workflow_graph.py`
- 现行说明：`flows/workbench-goals.md`

本页仅保留用于：解释历史 Playbook-phase 模式与迁移注意点，避免旧文档/旧字段误导。

## 现状（以代码为准）

- `ai-engine` workbench-mode 会拒绝 `playbook_config` 输入（若上游仍注入会报错）。
- `consultations-service`/`matter-service` 不再需要依赖 Playbook 推进阶段；阶段/门禁在 LangGraph 子图里完成。
- `platform-service` 可能仍保留 PlaybookConfig 相关表/接口：用于历史兼容，建议逐步下线。

## 迁移建议（从 Playbook → LangGraph）

- 把“阶段/门禁/优先规则”迁移到 LangGraph 子图 + conditional edges（确定性路由）。
- 把“文书池/模板推荐”迁移到 `docgen` 子图的确定性推荐 + 用户确认（例如 `profile.decisions.selected_documents`）。
- 把 `service_type_id` 作为产品/运营维度（标签/入口），不要作为底层强绑定流程键；底层以稳定字典码与结构化字段为主（参见 [`architecture/law.md`](../architecture/law.md)）。
