# 微服务拓扑与依赖

本页以“当前多仓库工程组”为准，给出核心服务拓扑与依赖关系（细节以各仓库 README/OpenAPI 为准）。

## 1) 关键拓扑（简化）

```mermaid
flowchart TB
  FE[frontend (Vue3)] -->|REST + SSE| CONS[consultations-service]
  FE -->|REST| MAT[matter-service]
  FE -->|REST| TPL[templates-service]
  FE -->|REST| FILES[files-service]
  FE -->|REST| AUTH[auth-service]
  FE -->|REST| USER[user-service]
  FE -->|REST| ORG[organization-service]
  FE -->|REST| PLAT[platform-service]

  CONS -->|NDJSON| AIE[ai-engine]
  MAT -->|internal| AIE
  TPL -->|internal| AIE

  AIE -->|internal| KNOW[knowledge-service]
  AIE -->|internal| MEM[memory-service]
  AIE -->|internal| FILES
  AIE -->|internal| MAT

  KNOW -. optional .-> RR[rerank-service]
  COL[collector-service] -->|internal seed| PLAT
  COL -->|internal seed| KNOW
  COL -->|internal seed| TPL
```

## 2) 服务清单（摘要）

| 服务/仓库 | 技术栈 | 职责 |
|---|---|---|
| `consultations-service` | Java/Spring Boot | 会话/SSE/卡片；转发 ai-engine NDJSON |
| `matter-service` | Java/Spring Boot | 事项/待办/阶段进度/交付件；internal 同步 |
| `ai-engine` | Python/FastAPI/LangGraph | workbench 工作流编排 + skills 执行 + checkpoint/trace |
| `knowledge-service` | Java/Spring Boot | 知识检索/GraphRAG（可选 ES/Neo4j） |
| `memory-service` | Python/FastAPI | 事实/记忆存储与召回（能力按实现） |
| `files-service` | Java/Spring Boot | 文件元数据 + 对象存储适配 |
| `templates-service` | Java/Spring Boot | 模板/文书生成（与 ai-engine 协作） |
| `platform-service` | Java/Spring Boot | 配置/字典/标签/feature flags（PlaybookConfig 为历史兼容） |
| `collector-service` | Python/FastAPI | seed packages 分发与导入 |
| `rerank-service` | (按实现) | 检索结果重排（可选） |

> 其它基础域服务（auth/user/org/billing/notification 等）见 `architecture/repositories.md`。
