---
title: memory-service（记忆/事实）
parent: 模块
nav_order: 6
---

# memory-service（记忆/事实）

## 定位

memory-service 提供“事实（Fact）”的结构化存储与召回能力：

- 支持按 scope（`case` / `global`）组织与隔离
- 支持事实增删改查与召回（hybrid：BM25 + 向量可选）
- 提供 internal recall/extract 等接口供 ai-engine 调用

## 技术栈

- Python 3.11 + FastAPI
- PostgreSQL（事实源）
- 可选：Qdrant（向量索引，允许重建，best-effort）

## 对外 API（/api/v1，摘录，当前实现）

- `POST /api/v1/memory/facts`
- `GET  /api/v1/memory/facts/{factId}`
- `PUT  /api/v1/memory/facts/{factId}`
- `DELETE /api/v1/memory/facts/{factId}`

兼容接口（给旧调用方 / shared-libs 客户端）：

- `POST /api/v1/memory/store`
- `POST /api/v1/memory/recall`
- `GET  /api/v1/memory/user/{userId}`

## 对内 API（/internal，摘录）

- `POST /internal/memory/recall`
- `GET  /internal/memory/users/{userId}/facts`
- `GET  /internal/memory/users/{userId}/context`
- `POST /internal/memory/extract`
- `POST /internal/memory/refine`

## 现状与边界

- 结构化存储与查询已实现。
- “自动事实抽取（extract）”目前为占位实现；详见 `implementation/memory-extraction.md`。
