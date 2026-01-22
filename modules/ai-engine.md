# AI Engine 模块设计

## 概述

AI Engine 是系统的智能核心，基于 LangGraph 实现 Agent 编排，通过 Skill 技能系统和 Playbook 配置驱动法律服务流程。

## 技术栈

- Python 3.12
- FastAPI
- LangGraph
- OpenRouter (LLM 网关)

## 核心架构

```
┌─────────────────────────────────────────────────────────────────┐
│                        AI Engine                                 │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ┌─────────────┐    ┌─────────────┐    ┌─────────────┐         │
│  │   Router    │───▶│   Planner   │───▶│  Run Skill  │         │
│  │             │    │             │    │             │         │
│  │ 路由决策     │    │ 技能选择    │    │ 技能执行    │         │
│  └─────────────┘    └─────────────┘    └──────┬──────┘         │
│                                               │                 │
│                     ┌─────────────┐    ┌──────▼──────┐         │
│                     │ Chat Respond│◀───│  Sync Data  │         │
│                     │             │    │             │         │
│                     │ 生成回复    │    │ 状态同步    │         │
│                     └─────────────┘    └─────────────┘         │
│                                                                 │
├─────────────────────────────────────────────────────────────────┤
│                      Skill Registry                             │
│  ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────┐          │
│  │ intake   │ │ cause-   │ │ evidence │ │ strategy │ ...      │
│  │ skills   │ │ recommend│ │ analysis │ │ planning │          │
│  └──────────┘ └──────────┘ └──────────┘ └──────────┘          │
│                                                                 │
├─────────────────────────────────────────────────────────────────┤
│                      Tool Handlers                              │
│  ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────┐          │
│  │ file__*  │ │ element__│ │ knowledge│ │ memory__ │ ...      │
│  │          │ │ *        │ │ __*      │ │ *        │          │
│  └──────────┘ └──────────┘ └──────────┘ └──────────┘          │
└─────────────────────────────────────────────────────────────────┘
```

## LangGraph 节点

### 1. Router Node

路由决策，判断请求类型：

```python
def router_node(state: UnifiedAgentState) -> dict:
    # 判断是否需要 AI 处理
    # - query: 简单问答
    # - workflow: 流程推进
    # - human_review: 人工审核回调
    return {"_route": "workflow"}
```

### 2. Planner Node

技能选择决策，采用策略链模式：

```python
class PlannerChain:
    strategies = [
        ForceSkillStrategy(),      # 强制指定技能
        FilePreprocessStrategy(),  # 附件预处理
        PriorityRulesStrategy(),   # 阶段优先规则
        DeterministicStrategy(),   # 确定性规则
        LLMPlannerStrategy(),      # LLM 兜底决策
    ]

    def decide(self, context: PlannerContext) -> PlannerDecision:
        for strategy in self.strategies:
            if decision := strategy.decide(context):
                return decision
        raise RuntimeError("No strategy matched")
```

### 3. Run Skill Node

执行选定的技能：

```python
async def run_skill_node(state: UnifiedAgentState) -> dict:
    skill_id = state.get("next_skill")
    skill = registry.get(skill_id)

    result = await runner.run(skill, state)

    # 处理 action
    if result.action == "ask_user":
        return {"_card": {...}, "awaiting_user_input": True}

    return {"skill_output": result.data}
```

### 4. Sync Data Node

同步状态到 Matter Service：

```python
async def sync_data_node(state: UnifiedAgentState) -> dict:
    matter_id = state.get("matter_id")
    skill_output = state.get("skill_output")

    await matter_client.sync_state(matter_id, skill_output)
    return {}
```

## Skill 技能系统

### 技能定义

```yaml
# skills/litigation-intake.yaml
id: litigation-intake
name: 诉讼收案
description: 收集原告起诉所需的基本信息
category: intake

system_prompt: |
  你是一个专业的法律助理，负责收集诉讼案件的基本信息...

output_schema:
  profile:
    - summary
    - plaintiff
    - defendant
    - facts
    - claims
  data:
    evidence_list:
      - name
      - type
      - status

tools:
  - file__get_info
  - file__parse
  - element__extract
```

### 技能分类

| 类别 | 技能 | 说明 |
|------|------|------|
| intake | litigation-intake | 诉讼收案 |
| intake | defense-intake | 被告应诉收案 |
| intake | arbitration-intake | 仲裁收案 |
| intake | criminal-intake | 刑事收案 |
| claim_path | cause-recommendation | 案由推荐 |
| evidence | evidence-analysis | 证据分析 |
| strategy | issue-analysis | 争点分析 |
| strategy | strategy-planning | 策略规划 |
| documents | documents | 文书清单 |
| documents | document-draft | 文书起草 |

### 技能输出结构

```json
{
  "response": "AI 回复文本",
  "profile": {
    "summary": "案件摘要",
    "plaintiff": { "name": "张三" }
  },
  "data": {
    "evidence": {
      "list": [...]
    }
  },
  "control": {
    "action": "continue | ask_user | finish",
    "review_type": "clarify | confirm | select",
    "questions": [...]
  }
}
```

## Playbook 配置

### 结构定义

```json
{
  "id": "litigation_civil_prosecution",
  "name": "民事诉讼（原告）",
  "phases": [
    {
      "id": "intake",
      "name": "收案阶段",
      "goal": "收集案件基本信息",
      "allowed_skills": ["litigation-intake", "file-classify"],
      "priority_rules": [...],
      "checkpoints": ["profile.summary", "profile.plaintiff"],
      "gate_field": "profile.intake_status",
      "gate_check": "equals:completed"
    },
    {
      "id": "claim_path",
      "name": "案由确认",
      "allowed_skills": ["cause-recommendation"],
      "gate_field": "profile.decisions.cause_confirmed",
      "gate_check": "equals:true"
    }
  ]
}
```

### 阶段流转

```
intake ──▶ claim_path ──▶ evidence ──▶ strategy ──▶ documents ──▶ execution
   │           │             │            │             │
   ▼           ▼             ▼            ▼             ▼
 收案        案由确认      证据分析     策略规划      文书准备
```

## 卡片交互机制

### 卡片类型

| review_type | 说明 | 示例 |
|-------------|------|------|
| clarify | 信息补充 | 请补充原告联系方式 |
| confirm | 确认操作 | 确认案由选择 |
| select | 多选一 | 选择诉讼策略 |
| phase_done | 阶段完成 | 收案阶段完成确认 |

### 语义 Task Key

```python
_SEMANTIC_TASK_KEY_BY_SKILL = {
    "cause-recommendation": "confirm_claim_path",
    "dispute-strategy-planning": "confirm_strategy",
    "documents": "confirm_documents",
    "work-plan": "confirm_work_plan",
}
```

## 目录结构

```
ai-engine/
├── src/
│   ├── api/                    # FastAPI 路由
│   ├── application/
│   │   ├── agent/
│   │   │   ├── nodes/          # LangGraph 节点
│   │   │   │   ├── router.py
│   │   │   │   ├── planner.py
│   │   │   │   ├── run_skill.py
│   │   │   │   └── sync_data.py
│   │   │   └── planner/        # Planner 策略
│   │   │       ├── chain.py
│   │   │       └── strategies/
│   │   └── skill_executor/     # 技能执行器
│   ├── domain/
│   │   └── agent/
│   │       └── entities.py     # 状态定义
│   └── infrastructure/
│       ├── providers/          # LLM Provider
│       └── tools/              # 工具实现
├── tests/
└── pyproject.toml
```
