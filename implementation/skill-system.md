# Skill 技能系统

## 概述

Skill 是 AI Engine 的核心执行单元，每个 Skill 封装了特定的法律服务能力，通过 LLM + Tools 实现智能化处理。

## 技能架构

```
┌─────────────────────────────────────────────────────────────────┐
│                        Skill Registry                           │
│                                                                 │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐          │
│  │ Skill Config │  │ Skill Config │  │ Skill Config │  ...     │
│  │ (YAML)       │  │ (YAML)       │  │ (YAML)       │          │
│  └──────┬───────┘  └──────┬───────┘  └──────┬───────┘          │
│         │                 │                 │                   │
│         └─────────────────┼─────────────────┘                   │
│                           │                                     │
│                    ┌──────▼──────┐                              │
│                    │Skill Runner │                              │
│                    │             │                              │
│                    │ - LLM 调用  │                              │
│                    │ - Tool 执行 │                              │
│                    │ - 输出解析  │                              │
│                    └─────────────┘                              │
└─────────────────────────────────────────────────────────────────┘
```

## 技能配置

### YAML 定义

```yaml
# skills/litigation-intake.yaml
id: litigation-intake
name: 诉讼收案
description: 收集原告起诉所需的基本信息，建立案件档案
category: intake
version: "1.0"

# 系统提示词
system_prompt: |
  你是一个专业的法律助理，负责收集诉讼案件的基本信息。

  ## 任务目标
  1. 收集当事人信息（原告、被告）
  2. 了解案件事实经过
  3. 明确诉讼请求
  4. 整理已有证据材料

  ## 输出要求
  - 使用规范的法律术语
  - 信息要完整准确
  - 对缺失信息进行追问

# 输出结构定义
output_schema:
  profile:
    summary:
      type: string
      description: 案件摘要（100字以内）
    plaintiff:
      type: object
      properties:
        name: { type: string }
        id_type: { type: string }
        id_number: { type: string }
        contact: { type: string }
    defendant:
      type: object
      properties:
        name: { type: string }
        type: { type: string, enum: [individual, company] }
    facts:
      type: string
      description: 案件事实
    claims:
      type: array
      items: { type: string }
    disputed_amount:
      type: number
  data:
    evidence_list:
      type: array
      items:
        type: object
        properties:
          name: { type: string }
          type: { type: string }
          status: { type: string }

# 可用工具
tools:
  - file__get_info
  - file__parse
  - element__extract

# 控制选项
control:
  max_turns: 5
  allow_ask_user: true
```

## 技能执行流程

```
输入状态 (UnifiedAgentState)
    │
    ▼
┌─────────────────────────────────────┐
│          Skill Runner               │
│                                     │
│  1. 加载技能配置                     │
│  2. 构建 System Prompt              │
│  3. 准备上下文（state + messages）  │
│  4. 调用 LLM                        │
│     │                               │
│     ├─▶ 文本回复                    │
│     │                               │
│     └─▶ Tool Calls                  │
│         │                           │
│         ▼                           │
│     执行工具，获取结果               │
│         │                           │
│         ▼                           │
│     继续 LLM 对话                   │
│                                     │
│  5. 解析输出（JSON Schema）         │
│  6. 返回 SkillResult                │
│                                     │
└─────────────────────────────────────┘
    │
    ▼
SkillResult {
  success: bool,
  response: string,
  data: { profile, data },
  action: "continue" | "ask_user" | "finish",
  questions: [...],
  envelope: {...}
}
```

## 输出结构

### 四段式输出

```json
{
  "response": "AI 回复文本，展示给用户",
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
    "action": "continue",
    "review_type": "clarify",
    "questions": [
      { "question": "请提供被告的联系方式" }
    ]
  }
}
```

### 字段说明

| 字段 | 说明 | 用途 |
|------|------|------|
| response | AI 回复文本 | 展示给用户 |
| profile | 案件画像更新 | 同步到 Matter.profile |
| data | 业务数据更新 | 同步到 Matter.data |
| control | 流程控制 | 决定下一步动作 |

### Action 类型

| action | 说明 | 后续处理 |
|--------|------|----------|
| continue | 继续流程 | 回到 Planner 决策 |
| ask_user | 需要用户输入 | 生成卡片，等待回复 |
| finish | 技能完成 | 结束当前技能 |

## 工具系统

### 工具分类

| 前缀 | 说明 | 示例 |
|------|------|------|
| file__ | 文件操作 | file__get_info, file__parse |
| element__ | 要素操作 | element__extract, element__rank_causes |
| knowledge__ | 知识检索 | knowledge__search_laws, knowledge__search_cases |
| memory__ | 记忆操作 | memory__recall, memory__store |

### 工具定义

```python
# tools/file_tools.py
@tool_handler("file__get_info")
async def file_get_info(file_id: str, user_id: str = None) -> dict:
    """获取文件信息"""
    client = FilesServiceClient(FILES_SERVICE_URL)
    info = await client.get_info(file_id, user_id)
    return info.model_dump()

@tool_handler("file__parse")
async def file_parse(file_id: str, extract_tables: bool = False) -> dict:
    """解析文件内容"""
    client = FilesServiceClient(FILES_SERVICE_URL)
    result = await client.parse(file_id, extract_tables)
    return result
```

### 工具调用流程

```
LLM 返回 Tool Call
    │
    ▼
┌─────────────────────────────────────┐
│         Tool Executor               │
│                                     │
│  1. 解析 tool_calls                 │
│  2. 查找 handler                    │
│  3. 执行工具函数                    │
│  4. 收集结果                        │
│  5. 返回给 LLM                      │
│                                     │
└─────────────────────────────────────┘
```

## 技能分类

### Intake 类（收案）

| 技能 ID | 名称 | 适用场景 |
|---------|------|----------|
| litigation-intake | 诉讼收案 | 原告起诉 |
| defense-intake | 被告收案 | 被告应诉 |
| appeal-intake | 上诉收案 | 上诉案件 |
| arbitration-intake | 仲裁收案 | 仲裁案件 |
| criminal-intake | 刑事收案 | 刑事辩护 |

### Claim Path 类（案由）

| 技能 ID | 名称 | 功能 |
|---------|------|------|
| cause-recommendation | 案由推荐 | 根据事实推荐案由 |

### Evidence 类（证据）

| 技能 ID | 名称 | 功能 |
|---------|------|------|
| evidence-analysis | 证据分析 | 分析证据材料 |
| evidence-gap-check | 证据缺口 | 检查证据完整性 |

### Strategy 类（策略）

| 技能 ID | 名称 | 功能 |
|---------|------|------|
| issue-analysis | 争点分析 | 识别案件争点 |
| strategy-planning | 策略规划 | 制定诉讼策略 |
| defense-planning | 被告策略 | 被告抗辩策略 |

### Documents 类（文书）

| 技能 ID | 名称 | 功能 |
|---------|------|------|
| documents-todo | 文书清单 | 生成文书清单 |
| document-draft | 文书起草 | 起草法律文书 |

### Utility 类（工具）

| 技能 ID | 名称 | 功能 |
|---------|------|------|
| file-classify | 文件分类 | 分类上传的文件 |
| chat-respond | 对话回复 | 通用对话回复 |

## 技能配置位置

```
collector-service/
└── resources/
    └── seed_packages/
        └── matters_system_resources/
            └── data/
                └── skills/
                    ├── intake/
                    │   ├── litigation-intake.yaml
                    │   ├── defense-intake.yaml
                    │   └── ...
                    ├── claim_path/
                    │   └── cause-recommendation.yaml
                    ├── evidence/
                    │   └── evidence-analysis.yaml
                    ├── strategy/
                    │   ├── issue-analysis.yaml
                    │   └── strategy-planning.yaml
                    └── documents/
                        ├── documents-todo.yaml
                        └── document-draft.yaml
```
