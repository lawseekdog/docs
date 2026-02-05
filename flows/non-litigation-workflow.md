---
title: 非诉流程（示例：合同审查）
parent: 业务流程
nav_order: 4
---

# 非诉工作流（示例：合同审查）

本页以 workbench 目标 `contract_review` 为例，描述“非诉事项”在系统中的典型推进路径：需求确认 → 合同分析 → 报告交付。

流程定义在 `ai-engine` 的 LangGraph 工作流图中（非 Playbook）：

- `ai-engine/src/application/agent/graphs/workflow_graph.py`
  - `contract_review` 子图（非诉主链路）
  - `docgen` 子图（文书生成与质量门禁）

## 1) 子图总览（contract_review）

```mermaid
flowchart LR
  A[intake<br/>需求确认] --> B[analyze<br/>合同分析]
  B --> C[output<br/>报告/推荐文书]
  C --> D[docgen<br/>文书生成(可选)]
  D --> E[finish<br/>同步并结束]
```

## 2) 关键门禁与确定性路由

- 材料优先：有新附件时先进入 `materials` 子图（洞察/分类/解析），避免直接跑审查。
- Intake 门禁：当合同类型/审查重点等关键信息缺失时，优先执行 `contract-intake` 补齐 `profile.contract_type` 等字段。
- 输出新鲜度：输入 fingerprint 变化会触发 reset（清理旧的合同分析/风险条款等）后再重算。
- 文书生成：`output` 阶段会推荐两类交付件：
  - `contract_review_report`（合同审查报告）
  - `modification_suggestion`（合同修改建议书）
  满足条件时进入 `docgen` 子图生成交付件，并在必要时要求律师复核确认。

## 3) 典型产物（字段层面）

- `data.non_litigation.contract_analysis`：合同条款分析
- `data.non_litigation.risk_summary / risk_clauses`：风险摘要/风险条款清单
- `data.work_product.analysis_report`：报告（markdown）
- `data.work_product.recommended_documents / documents`：推荐/已生成文书
- `profile.decisions.*`：文书选择与复核确认
