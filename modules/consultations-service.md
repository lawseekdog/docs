---
title: consultations-service（咨询会话）
parent: 模块
nav_order: 3
---

# consultations-service（咨询会话）

## 定位

consultations-service 负责：

- 咨询会话（Session）管理
- 消息/附件管理
- 对前端提供 SSE 对话流（chat/resume）
- 对接 `ai-engine` 的 NDJSON 事件流并转发（delta/card/end 等）

它是“用户实时交互”的主入口之一，也是卡片中断机制的前端出口。

## 技术栈

- Java 21 + Spring Boot
- Postgres + Flyway（默认）
- SSE（Spring MVC StreamingResponseBody/SseEmitter）

## 核心数据模型（概念）

以当前代码为准（ID 多为 Long，organizationId 多为 String/可为空）：

- ConsultationSession：会话（可绑定 `matter_id`）
- ConsultationMessage：会话消息（USER/ASSISTANT 等角色）
- ConsultationAttachment：附件与文件 ID 列表

## 对外 API（/api/v1）

会话：

- `GET  /api/v1/consultations/sessions`（分页列表）
- `POST /api/v1/consultations/sessions`（创建）
- `GET  /api/v1/consultations/sessions/{id}`（详情）
- `PUT/PATCH /api/v1/consultations/sessions/{id}`（更新）
- `DELETE /api/v1/consultations/sessions/{id}`（归档/结束）

对话（流式）：

- `POST /api/v1/consultations/sessions/{id}/chat`（SSE）
- `POST /api/v1/consultations/sessions/{id}/resume`（SSE）
- `GET  /api/v1/consultations/sessions/{id}/pending_card`（断线重连兜底）

其它：

- attachments/canvas/citations/stats 等 API（以 OpenAPI 为准）

> 组织/用户上下文通常通过 `X-User-Id`、`X-Organization-Id` 传入（也支持 query param 兜底）。

## 对内 API（/internal）

主要用于服务间调用与运维/脚本：

- `GET  /api/v1/internal/sessions/{sessionId}`
- `GET  /api/v1/internal/sessions/{sessionId}/messages/recent`
- `POST /api/v1/internal/sessions/{sessionId}/messages/assistant`
- `POST /api/v1/internal/sessions/{sessionId}/messages/user-and-chat`（非流式 chat）
- `POST /api/v1/internal/sessions/{sessionId}/resume`（非流式 resume）
- `GET  /api/v1/internal/sessions/{sessionId}/intake-context`

## 与其它服务的交互（关键）

### 1) 自动绑定 matter（咨询即事项）

当 session 尚未绑定 `matter_id` 时，chat/resume 会触发：

- 调用 matter-service：`POST /api/v1/internal/matters/from-consultation`
- 写回 session.matter_id 与 playbook_id

这使得咨询对话天然拥有“事项工作流”的承载体（Matter）。

### 2) 调用 ai-engine（NDJSON）并转发 SSE

consultations-service 调用 ai-engine：

- `POST /api/v1/internal/ai/agent/execute/stream`
- `POST /api/v1/internal/ai/agent/resume/stream`

并将 NDJSON events 映射为前端 SSE events：

- `token(node=chat_respond)` → `delta`
- `card` → `card`
- `end` → `end`
- `progress/task_start/task_end`：可透传给前端用于 UI 反馈

### 3) playbook 获取

consultations-service 会在执行前获取/组装 playbook_config（通常来自 platform-service），再作为 context 传入 ai-engine。

## 现状与边界

- 该服务已具备完整的“流式对话 + 卡片中断”主链路实现。
- “咨询模式不创建 matter”的产品形态当前未实现；现状为咨询自动创建 DRAFT matter（见 `flows/consultation-to-matter.md`）。
