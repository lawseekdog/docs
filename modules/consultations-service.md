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

以当前代码为准：

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
- 写回 session.matter_id

这使得咨询对话天然拥有“事项态”的承载体（Matter），便于落库、协同与审计。

### 2) 调用 ai-engine（NDJSON）并转发 SSE

consultations-service 调用 ai-engine：

- `POST /api/v1/internal/ai/agent/execute/stream`
- `POST /api/v1/internal/ai/agent/resume/stream`

并将 NDJSON events 映射为前端 SSE events：

- `token(node=chat_respond)` → `delta`
- `card` → `card`
- `end` → `end`
- `progress/task_start/task_end`：可透传给前端用于 UI 反馈

### 3) matter_profile 白名单（不注入 playbook_config）

workbench-mode 下，ai-engine 以 `context.matter_profile` 作为事项画像快照输入。

- consultations-service 会从 matter-service 获取 workflow profile
- 仅保留 ai-engine 允许的白名单字段
- **不再注入/透传 `playbook_config`**（ai-engine workbench-mode 会拒绝该字段）

## 现状与边界

- 已具备完整的“流式对话 + 卡片中断 + 自动绑定事项”主链路。
- “咨询模式不创建 matter”的产品形态当前未实现；现状为咨询自动创建 DRAFT matter（见 `flows/consultation-to-matter.md`）。
