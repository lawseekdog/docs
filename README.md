---
title: README
nav_exclude: true
search_exclude: true
---

# LawSeekDog 技术文档（docs 仓库）

本仓库用于发布 LawSeekDog 的系统文档站点（GitHub Pages）：

- 站点地址：<https://lawseekdog.github.io/docs/>
- 首页：[`index.md`](./index.md)

文档内容以当前仓库代码现状为准；涉及“规划/待实现”的能力会明确标注，避免把旧实现/设想写成既成事实。

## 文档导航

### 架构

- [系统架构概览](./architecture/overview.md)
- [项目进度与现状](./architecture/progress.md)
- [微服务拓扑与依赖](./architecture/microservices.md)
- [数据流与协议（SSE/NDJSON/Internal API）](./architecture/data-flow.md)
- [仓库结构与拆分映射（law_tools_agent → 多仓库）](./architecture/repositories.md)

### 模块（按仓库/服务）

- AI 侧
  - [ai-engine（AI 执行引擎）](./modules/ai-engine.md)
  - [collector-service（Seed Packages 分发）](./modules/collector-service.md)
- 业务侧（Java 微服务）
  - [consultations-service](./modules/consultations-service.md)
  - [matter-service](./modules/matter-service.md)
  - [knowledge-service](./modules/knowledge-service.md)
  - [memory-service](./modules/memory-service.md)
  - [files-service](./modules/files-service.md)
  - [templates-service](./modules/templates-service.md)
  - [platform-service](./modules/platform-service.md)
  - [auth-service](./modules/auth-service.md)
  - [user-service](./modules/user-service.md)
  - [organization-service](./modules/organization-service.md)
  - [billing-service](./modules/billing-service.md)
  - [notification-service](./modules/notification-service.md)
  - [gateway-service](./modules/gateway-service.md)
- 工程化
  - [ai-boot-framework（脚手架/BOM/Starter/Archetype）](./modules/ai-boot-framework.md)
  - [infra-templates（复用 CI/CD）](./modules/infra-templates.md)

### 业务流程

- [咨询 → 事项（Consultation → Matter）](./flows/consultation-to-matter.md)
- [诉讼流程（示例：民事起诉）](./flows/litigation-workflow.md)
- [非诉流程（示例：合同审查）](./flows/non-litigation-workflow.md)
- [Playbook 配置规范（v2）](./flows/playbook-phases.md)

### 核心实现

- [Skill 技能系统](./implementation/skill-system.md)
- [Planner 决策引擎](./implementation/planner-engine.md)
- [卡片交互机制（ask_user / resume）](./implementation/card-interaction.md)
- [知识检索与 GraphRAG](./implementation/knowledge-rag.md)
- [记忆服务（事实/召回）](./implementation/memory-extraction.md)
- [Seed Packages（分发/导入/回归）](./implementation/seed-packages.md)

### API 参考

- [API 约定（Google REST + 统一返回体）](./api/conventions.md)
- [认证与鉴权（JWT + Internal API Key）](./api/authentication.md)
- [OpenAPI 与内部运维端点](./api/openapi.md)

### 部署与交付

- [本地开发（单服务/多服务）](./deployment/local-dev.md)
- [CI/CD（复用工作流 → 镜像构建/推送 + 可选 Helm 部署）](./deployment/ci-cd.md)
- [Helm（部署到 K8s）](./deployment/helm.md)
- [生产部署要点（配置/密钥/依赖）](./deployment/production.md)

## 技术栈（以现状为准）

### 后端

- Java 21 + Spring Boot 3.3（多数业务微服务）
- Python >= 3.11 + FastAPI + LangGraph（AI 执行引擎/Seed 分发）
- PostgreSQL + Flyway（各服务独立库；模板默认 Postgres，本地 `docker-compose.yml` 提供）
- Elasticsearch（可选：knowledge-service 的 keyword/vector/hybrid 检索；向量由 OpenAI 兼容 Embedding API 生成）
- Neo4j（可选：knowledge-service 的 GraphStore，用于 GraphRAG 扩展召回）
- MinIO/S3（files-service 对象存储适配）

### 前端

- Vue 3 + TypeScript（`frontend` 仓库）
- Vite + TailwindCSS
- Pinia + Vue Router
- Playwright（E2E）

### AI

- LangGraph（Agent/interrupt）
- OpenAI 兼容协议模型调用（默认可走 OpenRouter；具体模型以配置为准）

## 相关仓库（组织：lawseekdog）

建议以「仓库结构与拆分映射」为入口：[`architecture/repositories.md`](./architecture/repositories.md)
