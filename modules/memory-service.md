---
title: memory-service（记忆/事实）
parent: 模块
nav_order: 6
---

# memory-service（记忆/事实）

## 定位

memory-service 提供“记忆条目（MemoryItem）与事实（Fact）”的存储与查询能力：

- 支持按 scope（SESSION/MATTER/USER）组织与隔离
- 支持事实增删改查、查询/搜索、迁移（session → matter）
- 提供 internal recall/extract 等接口供 ai-engine 调用

## 技术栈

- Java 21 + Spring Boot
- Postgres + Flyway（默认）

## 对外 API（/api/v1，摘录）

- `GET  /api/v1/memory/items`
- `POST /api/v1/memory/items`
- `DELETE /api/v1/memory/items/{id}`
- `POST /api/v1/memory/facts`
- `GET  /api/v1/memory/facts/{factId}`
- `PUT  /api/v1/memory/facts/{factId}`
- `DELETE /api/v1/memory/facts/{factId}`
- `POST /api/v1/memory/facts/query`
- `POST /api/v1/memory/facts/search`
- `POST /api/v1/memory/extract`（当前为占位）

## 对内 API（/internal，摘录）

- `POST /internal/memory/recall`
- `POST /internal/memory/extract`（当前为占位）
- `POST /internal/memory/migrate`
- `GET  /internal/memory/users/{userId}/context`

## 现状与边界

- 结构化存储与查询已实现。
- “自动事实抽取（extract）”目前为占位实现；详见 `implementation/memory-extraction.md`。
