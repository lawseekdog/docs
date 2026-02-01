---
title: Playbook 配置规范（v2）
parent: 业务流程
nav_order: 4
---

# Playbook 配置规范（v2，以当前实现为准）

Playbook 是驱动咨询/事项流程的“配置”，由 `ai-engine` 的 Planner/PhaseManager 解析执行。

Playbook 主要来源：

- `collector-service/resources/seed_packages/matters_system_resources/data/playbooks/*.json`
- 由 `collector-service` 分发并写入 `platform-service`（PlaybookConfig），供业务服务查询与下发

本页以当前 `ai-engine` 的校验逻辑与现有 playbook JSON 为准（允许 `gate_check` 与 `gate_value` 并存）。

## 1) Playbook 顶层结构

以 `litigation_civil_prosecution` 为例（精简字段）：

> 注：下方示例中的 `allowed_skills` / `priority_rules` 属于历史字段（兼容/提示用途），不再作为候选技能过滤与确定性裁决源；以代码实现为准。

```json
{
  "id": "litigation_civil_prosecution",
  "version": "2.0",
  "title": "民事起诉（原告）",
  "matter_category": "litigation",
  "description": "民事诉讼原告一审流程",

  "allowed_skills": ["file-classify", "agentic-search"],
  "priority_rules": [
    {
      "when": "state.attachment_file_ids.length > 0 && state.files_classified != true",
      "skill": "file-classify",
      "reason": "新附件到达，先做文件分类"
    }
  ],

  "phases": [ ... ],

  "knowledge_bases": ["sys_cases", "sys_laws"],
  "document_pool": [ ... ]
}
```

字段约定（核心）：

- （已废弃/仅兼容）`allowed_skills`：历史上用于“全局技能白名单”。当前实现中，Planner 的候选技能集合由 `Skill.availability` 自动发现（见 `ai-engine/src/application/skill_executor/planner.py`），不再依赖该字段。
- （兼容/仅作为提示）`priority_rules`：历史上用于“全局优先规则”。当前确定性优先规则来自 `Skill.priority_rule`（见 `ai-engine/src/application/agent/planner/strategies/priority_rules.py`）；但 LLM 兜底规划器仍会把 playbook/phase 的 `priority_rules` 作为软提示注入 prompt（见 `ai-engine/src/application/skill_executor/llm_planner.py`）。
- `phases`：阶段数组（定义阶段目标、可用技能、阶段门控条件）

## 2) Phase 结构

示例（来自现有 playbook）：

```json
{
  "id": "strategy",
  "name": "策略与计划",
  "goal": "制定诉讼策略并完成确认",

  "priority_rules": [
    {
      "when": "state.issues.length == 0",
      "skill": "issue-analysis",
      "reason": "先梳理争点"
    }
  ],

  "checkpoints": ["issues", "strategies", "risk_assessment", "profile.decisions.selected_strategy_id"],

  "gate_field": "profile.decisions.strategy_confirmed",
  "gate_value": true,

  "allowed_skills": ["issue-analysis", "strategy-planning"]
}
```

字段说明：

- （已废弃/仅诊断）`allowed_skills`：历史上用于“阶段内技能白名单”。当前实现不再用它过滤候选技能；它主要用于阶段推进校验/诊断：当技能属于 phase.allowed_skills 且本轮宣称 continue/finish 时，引擎会校验该技能是否填补了当前阶段缺口（见 `ai-engine/src/application/skill_executor/validators/phase_validation.py`）。
- （兼容/仅作为提示）`priority_rules`：历史字段；确定性优先规则来自 `Skill.priority_rule`；playbook/phase.priority_rules 仅作为 LLM prompt 的软提示。
- `checkpoints`：阶段完成校验的字段（用于“缺口检查”与提示）
- `gate_field` + (`gate_value` 或 `gate_check`)：阶段门控条件

## 3) gate_field / gate_value / gate_check（门控）

### 3.1 gate_field 是“点路径”，不是条件表达式

`gate_field` 与 `checkpoints` 写法遵循以下规则（由 `ai-engine` 强校验）：

- 不能写 `state.` 或 `output.` 前缀（这是点路径，不是条件表达式）
- 允许：
  - `profile.<field>`（且字段必须在允许列表内）
  - `profile.decisions.<field>`（只允许到字段级，且字段必须在允许列表内）
  - data 侧“叶子字段名”（例如 `issues`、`documents`、`evidence_analysis` 等）
- 禁止：`data.<group>.<field>`（必须直接写叶子字段名）

### 3.2 gate_value：等值判断（常见）

```json
{
  "gate_field": "profile.decisions.cause_confirmed",
  "gate_value": true
}
```

### 3.3 gate_check：内置检查器（兼容写法）

现有 playbooks 中也使用 `gate_check`：

```json
{
  "gate_field": "documents",
  "gate_check": "not_empty"
}
```

> 说明：当前实现允许 `gate_value` 与 `gate_check` 二选一；如果写了 `gate_field` 却两者都没有，会被校验拒绝（禁止默认兜底）。

## 4) priority_rules.when（条件表达式，兼容字段）

`priority_rules[*].when` 是表达式（不是点路径），必须遵循约束：

- 必须显式使用 `state.<field>` 引用状态字段（禁止隐式字段引用）
- 禁止引用 `output.*`
- 禁止隐式 `data.*`（data 叶子字段也要写成 `state.<leaf>`）
- 禁止 `in [...]` 语法
- 仅允许少量函数（例如 `has_uploaded_files()`；以代码为准）

示例：

```json
{
  "when": "state.attachment_file_ids.length > 0 && state.files_classified != true",
  "skill": "file-classify"
}
```

## 5) 阶段推进语义（执行侧）

阶段推进的核心语义：

1. 先根据 `Skill.priority_rule` 尝试确定性选择技能（命中即执行）
2. 若未命中，Planner 会在“当前阶段可用技能集合”（由 `Skill.availability` 自动发现 + requires/visibility/api_call_only 过滤）内选择下一技能（包含 LLM 兜底策略）
3. 每轮技能执行后，PhaseManager 会根据 `gate_*` 与 `checkpoints` 评估是否可推进到下一阶段
4. 若 `control.action == ask_user`，进入中断等待用户补充（卡片），完成后 `/resume` 继续

> 实现细节参见：`implementation/planner-engine.md` 与 `implementation/card-interaction.md`。

## 6) 现有 Playbook 列表（来源：seed packages）

以 `collector-service/resources/seed_packages/matters_system_resources/data/playbooks/` 为准，包含：

- 诉讼：`litigation_civil_prosecution`、`litigation_civil_defense`、`litigation_civil_appeal_appellant`、`litigation_civil_appeal_appellee`、`litigation_criminal` 等
- 非诉：`contract_review`、`due_diligence`、`legal_opinion`、`document_drafting` 等
- 咨询：`consultation_general`
