---
title: 非诉流程（示例：合同审查）
parent: 业务流程
nav_order: 3
---

# 非诉流程（示例：合同审查）

本页以当前 seed playbook `contract_review` 为例，描述“非诉事项”在系统中的阶段与技能编排方式。

来源：

- playbook JSON：`collector-service/resources/seed_packages/matters_system_resources/data/playbooks/contract_review.json`
- 运行时读取：同诉讼流程（collector → platform → matter/consultations → ai-engine）

## 1) 阶段总览

```mermaid
flowchart LR
  A[qualify<br/>需求确认] --> B[analyze<br/>合同分析]
  B --> C[execute<br/>报告交付]
```

## 2) 阶段配置要点（摘录）

### 2.1 全局优先规则：附件预处理

与诉讼流程一致，若有新附件且尚未分类，优先执行 `file-classify`：

```json
{
  "when": "state.attachment_file_ids.length > 0 && state.files_classified != true",
  "skill": "file-classify"
}
```

### 2.2 各阶段（phases）

| phase.id | 名称 | allowed_skills | gate（门控） |
|----------|------|----------------|--------------|
| `qualify` | 需求确认 | `contract-intake` |（无显式 gate；以 checkpoints/缺口判断为主） |
| `analyze` | 合同分析 | `contract-review` | `contract_analysis not_empty`（gate_check） |
| `execute` | 报告交付 | `document-generation` | `documents not_empty`（gate_check） |

## 3) 关键输出字段（示例）

- `profile.contract_type`、`profile.review_focus`
- `contract_analysis`、`risk_clauses`
- `documents`（合同审查报告/修改建议等）

> 注意：这里的 `contract_analysis`、`documents` 等属于“data 叶子字段名”，在 playbook 的 gate_field/checkpoints 中以字段级引用（不写 `data.*`）。

## 4) 文书池（document_pool）

该 playbook 提供候选文书：

- `contract_review_report`（合同审查报告）
- `modification_suggestion`（合同修改建议书）

最终生成哪些文书由技能输出与卡片确认决定。
