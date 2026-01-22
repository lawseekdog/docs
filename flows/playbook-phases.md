# Playbook 阶段设计

## 概述

Playbook 是驱动法律服务流程的配置文件，定义了案件处理的阶段、技能、检查点和流转条件。

## Playbook 结构

```json
{
  "id": "litigation_civil_prosecution",
  "name": "民事诉讼（原告）",
  "description": "原告发起民事诉讼的完整流程",
  "matter_category": "litigation",
  "client_role": "plaintiff",
  "phases": [...]
}
```

## 阶段定义

### Phase 结构

```json
{
  "id": "intake",
  "name": "收案阶段",
  "goal": "收集案件基本信息，建立案件档案",
  "allowed_skills": ["litigation-intake", "file-classify"],
  "priority_rules": [
    {
      "condition": "has_new_attachments && !data.files.files_classified",
      "skill": "file-classify"
    }
  ],
  "checkpoints": [
    "profile.summary",
    "profile.plaintiff.name",
    "profile.defendant.name",
    "profile.facts"
  ],
  "gate_field": "profile.intake_status",
  "gate_check": "equals:completed",
  "completion_conditions": {
    "required_fields": ["profile.summary", "profile.plaintiff", "profile.defendant"],
    "optional_fields": ["profile.claims"]
  }
}
```

### 字段说明

| 字段 | 说明 |
|------|------|
| id | 阶段唯一标识 |
| name | 阶段名称 |
| goal | 阶段目标描述 |
| allowed_skills | 该阶段可执行的技能列表 |
| priority_rules | 优先规则（条件 → 技能映射） |
| checkpoints | 检查点字段列表 |
| gate_field | 阶段完成判断字段 |
| gate_check | 完成条件表达式 |

## 标准阶段流程

### 民事诉讼（原告）

```
┌─────────┐     ┌─────────────┐     ┌──────────┐
│ intake  │────▶│ claim_path  │────▶│ evidence │
│ 收案    │     │ 案由确认    │     │ 证据分析 │
└─────────┘     └─────────────┘     └────┬─────┘
                                         │
┌─────────┐     ┌─────────────┐     ┌────▼─────┐
│execution│◀────│  documents  │◀────│ strategy │
│ 执行    │     │  文书准备   │     │ 策略规划 │
└─────────┘     └─────────────┘     └──────────┘
```

### 各阶段详情

#### 1. intake（收案）

**目标**：收集案件基本信息

**技能**：
- `litigation-intake` - 诉讼收案
- `file-classify` - 附件分类

**产出**：
- profile.summary - 案件摘要
- profile.plaintiff - 原告信息
- profile.defendant - 被告信息
- profile.facts - 案件事实
- profile.claims - 诉讼请求

**完成条件**：
```
profile.intake_status == "completed"
```

#### 2. claim_path（案由确认）

**目标**：确定案由和诉讼路径

**技能**：
- `cause-recommendation` - 案由推荐

**产出**：
- profile.cause_of_action_code - 案由代码
- profile.cause_of_action_name - 案由名称
- profile.decisions.cause_confirmed - 确认标记

**完成条件**：
```
profile.decisions.cause_confirmed == true
```

**卡片交互**：
- task_key: `confirm_claim_path`
- actor: `lawyer`
- review_type: `select`

#### 3. evidence（证据分析）

**目标**：分析证据材料，识别证据缺口

**技能**：
- `evidence-analysis` - 证据分析
- `evidence-gap-check` - 证据缺口检查

**产出**：
- data.evidence.list - 证据清单
- data.evidence.gaps - 证据缺口
- data.evidence.analysis_result - 分析结果

**完成条件**：
```
data.evidence.analysis_completed == true
```

#### 4. strategy（策略规划）

**目标**：制定诉讼策略

**技能**：
- `issue-analysis` - 争点分析
- `strategy-planning` - 策略规划

**优先规则**：
```json
{
  "condition": "!data.litigation.issues",
  "skill": "issue-analysis"
}
```

**产出**：
- data.litigation.issues - 争点列表
- data.litigation.strategies - 策略方案
- profile.decisions.selected_strategy_id - 选定策略

**完成条件**：
```
profile.decisions.strategy_confirmed == true
```

**卡片交互**：
- task_key: `confirm_strategy`
- actor: `lawyer`
- review_type: `confirm`

#### 5. documents（文书准备）

**目标**：准备诉讼文书

**技能**：
- `documents` - 文书清单
- `document-generation` - 文书生成

**产出**：
- profile.decisions.selected_documents - 选定文书
- data.work_product.documents - 已生成文书/交付物

**完成条件**：
```
profile.decisions.document_reviewed == true
```

#### 6. execution（执行）

**目标**：执行诉讼流程

**技能**：
- `filing-guide` - 立案指引
- `hearing-prep` - 庭审准备

## 阶段流转逻辑

### Planner 决策流程

```python
def decide_next_phase(state, playbook):
    current_phase = state.get("current_task_id")
    phases = playbook.get("phases", [])

    # 找到当前阶段
    current_idx = find_phase_index(phases, current_phase)
    current_phase_config = phases[current_idx]

    # 检查当前阶段是否完成
    if is_phase_complete(state, current_phase_config):
        # 进入下一阶段
        if current_idx < len(phases) - 1:
            return phases[current_idx + 1]["id"]

    return current_phase
```

### 阶段完成检查

```python
def is_phase_complete(state, phase_config):
    gate_field = phase_config.get("gate_field")
    gate_check = phase_config.get("gate_check")

    value = get_nested_value(state, gate_field)

    if gate_check.startswith("equals:"):
        expected = gate_check.split(":")[1]
        if expected == "true":
            return value is True
        elif expected == "completed":
            return value == "completed"

    elif gate_check == "not_empty":
        return value is not None and value != ""

    return False
```

## 不同服务类型的 Playbook

### 诉讼类

| Playbook ID | 名称 | 角色 |
|-------------|------|------|
| litigation_civil_prosecution | 民事诉讼（原告） | plaintiff |
| litigation_civil_defense | 民事诉讼（被告） | defendant |
| litigation_appeal | 上诉案件 | appellant |
| litigation_criminal_defense | 刑事辩护 | defendant |

### 非诉类

| Playbook ID | 名称 |
|-------------|------|
| non_litigation_contract_review | 合同审查 |
| non_litigation_legal_opinion | 法律意见书 |
| non_litigation_due_diligence | 尽职调查 |

### 仲裁类

| Playbook ID | 名称 |
|-------------|------|
| arbitration_commercial | 商事仲裁 |
| arbitration_labor | 劳动仲裁 |

## 配置文件位置

```
collector-service/
└── resources/
    └── seed_packages/
        └── matters_system_resources/
            └── data/
                └── playbooks/
                    ├── litigation_civil_prosecution.json
                    ├── litigation_civil_defense.json
                    ├── litigation_appeal.json
                    └── ...
```
