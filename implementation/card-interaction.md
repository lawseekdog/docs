---
title: 卡片交互机制
parent: 核心实现
nav_order: 3
---

# 卡片交互机制（ask_user / resume）

本系统的人机协同遵循一个强约束：

- 任何需要人工补问/选择/确认的步骤，都必须由技能输出 `control.action == "ask_user"` 触发
- 外层不做“自动修正/兜底”，必须等待用户提交卡片后才能继续推进

这一机制最初沉淀在 `law_tools_agent`，当前在多仓库形态下由：

- `ai-engine` 负责“中断/恢复”（LangGraph interrupt）
- `consultations-service` 负责“前端 SSE 输出与转发”

## 1) 输出契约：ask_user

技能输出的 `control` 段示例（摘录）：

```json
{
  "control": {
    "action": "ask_user",
    "review_type": "clarify",
    "questions": [
      {
        "question": "请补充被告的身份证号",
        "field_key": "profile.defendant.id_number",
        "input_type": "text",
        "required": true
      }
    ]
  }
}
```

语义：

- `action=ask_user`：当前轮必须中断等待用户输入
- `questions[*].field_key`：用于把用户答案回填到 state（支持点路径）
- `review_type`：前端渲染/交互类型（clarify/select/confirm/phase_done）

## 2) 事件流：NDJSON → SSE

### 2.1 ai-engine：NDJSON 事件流

ai-engine internal API（以当前实现为准）：

- `GET  /internal/ai/agent/pending_card?thread_id=...`
- `POST /internal/ai/agent/execute/stream`（NDJSON）
- `POST /internal/ai/agent/resume/stream`（NDJSON）

流式输出为逐行 JSON：

```json
{"event":"card","data":{...}}
{"event":"end","data":{"output":""}}
```

### 2.2 consultations-service：SSE 转发

对前端：

- `POST /api/v1/consultations/sessions/{sessionId}/chat`（SSE）
- `POST /api/v1/consultations/sessions/{sessionId}/resume`（SSE）
- `GET  /api/v1/consultations/sessions/{sessionId}/pending_card`

转发策略（关键点）：

- `token` 事件只转发“用户可见节点”的 token（避免 planner/skill 等内部 JSON 推理内容污染前端）
- `card` 事件原样转发给前端，用于弹出悬浮卡片并锁定输入
- `end` 结束本轮流式连接

## 3) pending_card：断线重连兜底

SSE 链路常见问题是前端刷新/断线重连。为避免“卡片已经触发但前端丢失”，提供：

- `GET /api/v1/consultations/sessions/{sessionId}/pending_card`

其内部通常会根据 thread_id 去 ai-engine 查询：

- `GET /internal/ai/agent/pending_card?thread_id=...`

返回格式（概念）：

```json
{
  "has_pending_card": true,
  "card": { "review_type": "...", "questions": [...] }
}
```

## 4) resume：提交答案并继续执行

前端提交卡片答案后：

- 调用 consultations-service 的 `/resume`（SSE）
- consultations-service 将 `user_response` 透传给 ai-engine 的 `/agent/resume/stream`
- ai-engine 把答案回填到 state 后继续运行 LangGraph

consultations-service 的请求体结构（摘录）：

```json
{
  "user_id": 123,
  "user_response": {
    "answers": [
      { "field_key": "profile.defendant.id_number", "value": "..." }
    ]
  }
}
```

## 5) 与 Matter Todo 的关系（需要约束）

系统同时存在两种“待处理交互”形态：

1) 会话侧：pending_card（thread interrupt）
2) 事项侧：MatterTodo（待办任务，持久化在 matter-service）

二者的定位不同：

- pending_card：更偏“流程推进必须的即时交互”，与 LangGraph 执行强绑定
- MatterTodo：更偏“事项业务上的可追踪任务”，可用于工作台/派单/进度

为了避免前端出现“两个入口/两个待办互相打架”，需要产品/前端明确：

- 哪些交互必须走 pending_card（ask_user）
- 哪些交互可以落到 MatterTodo（例如阶段性确认、交付物审核）

当前实现中，两者会在部分场景同时存在（例如创建事项时自动生成 todo），后续建议做规则收敛。
