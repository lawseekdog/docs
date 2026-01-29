---
title: ai-engine（AI 执行引擎）
parent: 模块
nav_order: 1
---

# ai-engine（AI 执行引擎）

## 定位

`ai-engine` 是 LawSeekDog 的 AI 执行引擎，负责：

- 基于 LangGraph 执行 agent 图（planner → run_skill → …）
- 维护 thread state（Postgres checkpoint）
- 对外（internal）提供 agent 执行 API（含 NDJSON 流式事件）
- 管理 skills（.skills 目录）与工具调用（tool handlers）

该服务默认以 internal API 形式被 `consultations-service`、`matter-service`、`templates-service` 等调用。

## 技术栈

- Python >= 3.11
- FastAPI
- LangGraph + Postgres checkpointer
- OpenAI 兼容模型调用（默认 OpenRouter；以 provider 配置为准）

## 主要组件（代码层）

- 应用入口：`ai-engine/src/main.py`
  - 初始化数据库
  - 初始化 provider registry
  - 初始化 SkillRegistry
  - 初始化 AgentExecutionService（LangGraph）
- API 路由：`ai-engine/src/api/internal_routes.py`
  - skills 列表/查询/执行
  - agent execute/resume（流式/非流式）
  - pending_card、timeline、trace 查询等

## 关键 API（internal）

以下路径在运行时前缀为 `/api/v1/internal/ai/...`（因为 FastAPI router prefix=`/ai` 且 include_router prefix=`/internal`）。

### 1) Skill API

- `GET  /api/v1/internal/ai/skills`：列出可用 skills（过滤 internal/api_call_only）
- `GET  /api/v1/internal/ai/skills/{skill_name}`：查询 skill 详情
- `POST /api/v1/internal/ai/skills/{skill_name}/execute`：技能直跑（`skill_only=true`，强制执行指定 skill）

### 2) Agent 执行 API

- `GET  /api/v1/internal/ai/agent/pending_card?thread_id=...`
- `POST /api/v1/internal/ai/agent/execute`：非流式执行（供服务间调用）
- `POST /api/v1/internal/ai/agent/resume`：非流式恢复
- `POST /api/v1/internal/ai/agent/execute/stream`：流式执行（NDJSON）
- `POST /api/v1/internal/ai/agent/resume/stream`：流式恢复（NDJSON）

NDJSON 单行格式：

```json
{"event":"token","data":{"node":"chat_respond","content":"..."}}
```

### 3) Trace/Timeline（可观测）

- `GET /api/v1/internal/ai/timeline?...`：按 session/matter/thread 查询轮次摘要
- `GET /api/v1/internal/ai/traces?...`：执行 trace 列表
- `GET /api/v1/internal/ai/traces/{trace_id}`：trace 详情

> 具体字段以实现为准；consultations-service 会把部分事件透传到前端（progress/task_start/task_end 等）。

## 依赖与配置（现状）

必选：

- Postgres（用于 LangGraph checkpoint/trace）

可选/按需：

- 模型 provider：OpenRouter 或其它 OpenAI 兼容网关

内部鉴权：

- 对外 internal API 建议由网络策略隔离 + `X-Internal-Api-Key`（与全局 internal key 一致）

## 与其它服务的关系

- consultations-service：对外 SSE；内部把对话请求转成 ai-engine 的 NDJSON 流并转发
- matter-service：事项状态承载；ai-engine 在 skill/tool 中会读写 matter
- knowledge-service：知识检索（atomic/GraphRAG）
- memory-service：事实/记忆（当前抽取能力占位）
- files-service：文件信息/解析（供技能读取材料）
- platform-service：playbook/config/tag/feature flag 等配置读取

## 状态（工程化）

当前 `ai-engine` 的 Dockerfile 仍保留 mono-repo 的路径假设（需要在“独立仓库构建镜像”场景下做对齐）。
详见 `architecture/repositories.md` 的迁移注意点。
