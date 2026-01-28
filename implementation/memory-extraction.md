---
title: 记忆服务（现状与规划）
parent: 核心实现
nav_order: 5
---

# 记忆服务（memory-service）：事实/召回/抽取（现状与规划）

本页以 `memory-service` 当前实现为准，描述“记忆/事实”能力的边界，并明确哪些能力仍是占位。

## 1) 当前已实现：结构化记忆条目（Postgres 真源）

memory-service 目前以结构化存储为主：

- `MemoryItem`：通用记忆条目
  - scope：`SESSION` / `MATTER` / `USER`
  - item_type：`FACT` 等
  - content：文本内容
  - source_json：来源/实体/关键词等附加信息（JSON）

存储：Postgres（JPA + Flyway）。

## 2) 内部 API（摘录）

内部接口（给 ai-engine / 服务间调用）：

- `POST /internal/memory/facts`（create fact）
- `GET  /internal/memory/facts/{factId}`
- `PUT  /internal/memory/facts/{factId}`
- `DELETE /internal/memory/facts/{factId}`
- `POST /internal/memory/store`（兼容接口）
- `POST /internal/memory/recall`
- `GET  /internal/memory/user/{userId}`（兼容接口）
- `POST /internal/memory/extract`
- `POST /internal/memory/migrate`
- `GET  /internal/memory/users/{userId}/context`
- `GET  /internal/memory/users/{userId}/facts`

> 具体入参/返回以 OpenAPI 为准（见 `api/openapi.md`）。

## 3) 召回（recall）：当前策略

现状（以代码为准）：

- recall 基于结构化 facts 的筛选/搜索
- 不依赖向量库（未引入 Qdrant/Weaviate 等）
- 适用于“短事实/可直接检索关键词”的场景

## 4) 抽取（extract）：当前为占位

memory-service 暴露了抽取入口：

- `POST /internal/memory/extract`

但当前实现返回空结果（占位），尚未集成 LLM/ai-engine 做真正的事实抽取：

- `MemoryService.extractFacts(...)` 当前直接返回 `ExtractFactsResponse(true, [], 0, null)`

这意味着：

- 可以先用“业务方显式写入 FACT”支撑基本体验
- “自动抽取 → 写入事实 → 后续召回”仍需后续建设

## 5) 规划建议（不等于已实现）

要把记忆服务升级为“可用且可控”的生产能力，建议明确以下硬约束：

1. 抽取来源：由 ai-engine 负责抽取还是 memory-service 自己直连模型？
   - 建议：抽取逻辑集中在 ai-engine（skills），memory-service 只做存储与查询
2. 幂等与去重：同一事实重复抽取如何处理？
   - 建议：引入 entity_key + normalized_content 的幂等键（memory-service 已有部分实现思路）
3. 召回策略：关键词/结构化过滤 +（可选）向量召回
   - 若引入向量：再评估 ES kNN vs 专用向量库；并与知识检索保持一致的“可选依赖/可降级”策略
