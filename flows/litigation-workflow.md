---
title: 诉讼流程（示例：民事起诉）
parent: 业务流程
nav_order: 3
---

# 诉讼工作流（示例：民事起诉 / 原告）

本页以 workbench 目标 `case_analysis` 为主，描述“诉讼事项”在系统中的典型推进路径：受理 → 案由 → 证据 → 策略 → 交付。

流程定义在 `ai-engine` 的 LangGraph 工作流图中（非 Playbook）：

- `ai-engine/src/application/agent/graphs/workflow_graph.py`
  - `case_analysis` 子图（主链路）
  - `docgen` 子图（文书生成与质量门禁）
  - `policy_gate` 子图（交付门禁/确认）

## 1) 子图总览（case_analysis）

```mermaid
flowchart LR
  A[intake<br/>收案分析] --> B[cause<br/>案由确认]
  B --> C[evidence<br/>证据分析]
  C --> D[strategy<br/>争点/策略]
  D --> E[output<br/>报告/推荐文书]
  E --> F[docgen<br/>文书生成(可选)]
  F --> G[finish<br/>同步并结束]
```

## 2) 关键门禁与确定性路由

- 材料优先：有新附件时先进入 `materials` 子图做洞察/分类，避免直接推进主产物。
- 案由门禁：当“需要案由”且未确认时，强制执行 `cause-recommendation` 并通过卡片确认（落到 `profile.decisions.cause_confirmed` / `profile.cause_of_action_code` 等）。
- 证据准备：新文件未分析完成、或证据结构未就绪时，优先执行 `evidence-analysis`。
- 输出新鲜度：输入 fingerprint 变化会触发 reset（清理旧的争点/策略/风险）后再重算，避免“旧结论混入新材料”。
- 文书生成：`output` 阶段会给出 `recommended_documents`（例如起诉状/证据目录/策略报告等）；满足条件时进入 `docgen` 子图生成交付件，并在必要时要求律师复核确认。

## 3) 典型卡片交互点（ask_user）

- 案由确认：从候选案由中选择/确认。
- 文书质量门禁：生成后质量审核未通过或存在重要问题时，要求律师选择“继续交付/重新生成”。
- 交付门禁（Policy Gate）：高风险或需要确认的输出统一在交付前拦截，要求“继续/需要修改”。

## 4) 典型产物（字段层面）

- `data.work_product.analysis_report`：分析报告（markdown）
- `data.work_product.recommended_documents`：推荐文书清单（output_key/template_key）
- `data.work_product.documents`：已生成交付件（file_id/url 等）
- `profile.decisions.*`：关键确认点（案由、文书选择、复核状态等）
- `data.litigation.issues / strategies / risk_assessment`：结构化分析产物

## 5) 备注：work_plan / doc_drafting

- `work_plan` 目标更偏“办案计划与确认”（会强制策略选择与确认，并产出 `profile.work_plan`）。
- `doc_drafting` 目标用于“只生成某份文书”（必要时先做案由确认，再进入 `docgen`）。
