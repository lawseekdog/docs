---
title: memory-service（记忆/事实）
parent: 模块
nav_order: 6
---

# memory-service（记忆/事实）

## 定位

memory-service 提供“事实（Fact）”的结构化存储与召回能力：

- 支持按 scope（`case` / `global`）组织与隔离
- 支持事实增删改查与召回（关键词/BM25 为主；可选接入 rerank-service 做重排）
- 提供 internal recall/extract 等接口供 ai-engine 调用

## 技术栈

- Python 3.11 + FastAPI
- PostgreSQL（事实源）

## 内部 API（/internal，摘录，当前实现）

内部接口统一：

- 路径前缀：`/api/v1/internal/**`
- 鉴权：`X-Internal-Api-Key`
- 多租户隔离：`X-Organization-Id`（写入/召回均要求）
- 返回体：`ApiResponse(code/message/data)`（分页为 `ApiResponse<PageResponse<T>>`）

核心事实接口（摘录）：

- `POST /api/v1/internal/memory/facts`
- `GET  /api/v1/internal/memory/facts/{factId}`
- `PUT  /api/v1/internal/memory/facts/{factId}`
- `DELETE /api/v1/internal/memory/facts/{factId}`

## 其他内部 API（/internal，摘录）

- `POST /api/v1/internal/memory/recall`
- `GET  /api/v1/internal/memory/users/{userId}/facts`（支持 `page/size`；兼容 `limit`）
- `GET  /api/v1/internal/memory/users/{userId}/conflicts`（支持 `page/size`；兼容 `limit`）
- `GET  /api/v1/internal/memory/users/{userId}/context`
- `POST /api/v1/internal/memory/extract`
- `POST /api/v1/internal/memory/refine`

## 现状与边界

- 结构化存储与查询已实现。
- “自动事实抽取（extract）”目前为占位实现；详见 `implementation/memory-extraction.md`。
