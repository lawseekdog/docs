---
title: Workflow 路由与执行（Workbench）
parent: 核心实现
nav_order: 2
---

# Workflow 路由与执行（ai-engine / workbench-mode）

本页描述 `ai-engine` 在每一轮对话/事项推进中如何决定“下一步做什么”（选择目标、执行 skill、同步、门禁、中断恢复）。

结论：当前主链路为 **LangGraph workbench 工作流**，路由逻辑内聚在 `workflow_graph.py`，不再依赖 Playbook/Phase DSL。

## 1) 输入：state + context（不支持 playbook_config）

workbench-mode 的最小输入：

- `state`：统一状态（thread state），包含 `profile`、`data.*`、附件/文件信息、当前 task_id 等
- `context`：每轮上下文（来自 consultations-service/matter-service），常见字段：
  - `user_id` / `tenant_id` / `session_id(thread_id)` / `matter_id`
  - `service_type_id`（可选：路由提示/标签）
  - `attachment_file_ids`（本轮新附件）
  - `matter_profile`（事项画像快照，白名单字段）

重要约束：

- workbench-mode **拒绝** `playbook_config` 注入（若上游仍传会报错）。

## 2) 顶层图：节点与子图

主入口图（简化）：

- 系统节点：`ingest` / `hydrate` / `memory` / `router`
- 执行基础设施节点：`dispatch` / `run_skill` / `parallel_skills` / `sync_data` / `human_review` / `extract`
- workbench 子图：
  - `materials`（新材料洞察/分类/解析）
  - `goal_gate`（目标选择/确认）
  - `documents_stale`（材料变更 → 输出过期 → 二次确认）
  - `policy_gate`（交付门禁/确认）
  - per-goal 子图：`case_analysis` / `contract_review` / `legal_opinion` / `doc_drafting` / `judgment_prediction` / `work_plan`
  - `docgen`（草稿/生成/质量门禁/可选人工复核）

## 3) 路由优先级（简化）

路由应尽量确定性，且只依赖当前 state（幂等）。典型优先级：

1. 若存在新附件/新材料 → 先走 `materials` 子图
2. 若 goal 未确定 → 走 `goal_gate` 子图（必要时 ask_user）
3. 若文书/产物过期（stale）且未确认 → 走 `documents_stale` 子图
4. 否则 → 进入当前 goal 子图推进（必要时生成交付件）
5. 输出前统一经过 `policy_gate` 与 `docgen`（如果有推荐/选择的文书）

## 4) ask_user / resume（中断恢复）

- 任何需要用户/律师确认的点，统一通过卡片（`ask_user`）进入 `human_review` 中断。
- 关键原则：中断前必须先 `sync_data`，把 card 与必要状态落库，保证断线可恢复。
- `/resume` 后继续以同一个 `thread_id` 从 checkpoint 状态推进。

## 5) Legacy：Planner/Playbook 模式

早期设计曾通过 “Playbook + PhaseManager + Planner” 做流程 DSL。

当前 workbench-mode 以 LangGraph graph/subgraph 替代该模式；相关字段/接口可能仍在部分服务中保留用于历史兼容，但不应作为主链路依赖。

## 参考（代码真源）

- `ai-engine/src/application/agent/graphs/workflow_graph.py`
- `ai-engine/src/application/execution/base.py`
- `ai-engine/docs/unified_agent_state.md`
