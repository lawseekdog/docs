---
title: knowledge-service（知识库）
parent: 模块
nav_order: 5
---

# knowledge-service（知识库）

## 定位

knowledge-service 提供“知识的存储、索引与检索”能力：

- 文档与切片（Document/Chunk）管理
- 原子检索（keyword/vector/hybrid）
- GraphRAG（GraphStore + 原子检索融合）
- 结构化知识：诉讼要素/非诉 checklist/案由画像
- 搜索索引运维：reindex（best-effort）

该服务以 Postgres 为真源，并支持可选依赖（Elasticsearch/Neo4j）实现更强检索。

## 技术栈

- Java 21 + Spring Boot
- Postgres + Flyway（必选）
- Elasticsearch（可选）
- Neo4j（可选）

## 对外 API（/api/v1）

对外 API 面向前端/业务侧使用，主要包括：

- 知识库管理：`KnowledgeBaseController`
- 文件/知识文件管理：`KnowledgeFileController`
- 查询接口：`KnowledgeQueryController`
- 任务/作业：`KnowledgeJobController`

> 具体接口路径与入参以 OpenAPI 为准（见 `api/openapi.md`）。

## 对内 API（/internal）

内部接口供 ai-engine、collector-service 等调用：

### 1) Atomic Search（检索原语）

- `POST /api/v1/internal/atomic/keyword-search`
- `POST /api/v1/internal/atomic/vector-search`
- `POST /api/v1/internal/atomic/hybrid-search`
- `POST /api/v1/internal/atomic/document`
- `POST /api/v1/internal/atomic/section`

### 2) GraphRAG

- `POST /api/v1/internal/atomic/graph-query`
- `POST /api/v1/internal/atomic/graph-rag-search`

### 3) Seed 导入

- `POST /api/v1/internal/seed/structured/import`
- `POST /api/v1/internal/seed/system-kb-documents/import`
- `POST /api/v1/internal/seed/cause-of-action-profiles/import`

### 4) 索引运维

- `POST /api/v1/internal/search/reindex-by-file`
- `POST /api/v1/internal/search/reindex-batch`

## 搜索与降级策略（重要）

knowledge-service 的检索遵循“可用性优先”：

- Elasticsearch 启用时优先用 ES（keyword/vector/hybrid）
- ES 不可用或调用失败时，自动降级为 SQL keyword 候选召回 + 简单打分

启用 ES：

- `KNOWLEDGE_SEARCH_BACKEND=elasticsearch`
- `ELASTICSEARCH_URL=...`

Neo4j（GraphStore）默认 fail-open：未配置时 GraphRAG 会降级为纯原子检索。

详细机制参见：`implementation/knowledge-rag.md`。

## 与其它服务的关系

- ai-engine：主要 consumer（atomic/graph-rag-search/element 工具）
- collector-service：seed 包分发（Playbooks/结构化 seeds/系统 KB 文档清单）
- files-service：知识文件与对象存储关联（文件元信息/权限校验）
- user-service：用户信息（用于归属/审计等）
