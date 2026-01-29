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

- `POST /internal/memory/facts`
- `GET  /internal/memory/facts/{factId}`
- `PUT  /internal/memory/facts/{factId}`
- `DELETE /internal/memory/facts/{factId}`

## 其他内部 API（/internal，摘录）

- `POST /internal/memory/recall`
- `GET  /internal/memory/users/{userId}/facts`
- `GET  /internal/memory/users/{userId}/context`
- `POST /internal/memory/extract`
- `POST /internal/memory/refine`

## 现状与边界

- 结构化存储与查询已实现。
- “自动事实抽取（extract）”目前为占位实现；详见 `implementation/memory-extraction.md`。
