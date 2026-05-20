# LawSeekDog 系统文档

> 智能法律服务平台技术文档中心

[![GitHub Pages](https://img.shields.io/badge/docs-GitHub%20Pages-blue)](https://lawseekdog.github.io/docs/)

## 📚 文档导航

### 系统架构
- [系统架构概览](./architecture/overview.md) - 整体架构设计与技术选型
- [微服务拓扑](./architecture/microservices.md) - 服务间依赖与通信
- [数据流架构](./architecture/data-flow.md) - 数据流转与存储设计

### 核心模块设计
| 模块 | 说明 | 文档 |
|------|------|------|
| AI Engine | AI 智能引擎（LangGraph workbench + skills） | [设计文档](./modules/ai-engine.md) |
| Matter Service | 事项管理（案件全生命周期） | [设计文档](./modules/matter-service.md) |
| Consultations Service | 咨询会话（实时对话、卡片交互） | [设计文档](./modules/consultations-service.md) |
| Knowledge Service | 知识库（法规、案例、要素检索） | [设计文档](./modules/knowledge-service.md) |
| Auth Service | 认证授权（JWT、RBAC） | [设计文档](./modules/auth-service.md) |
| User Service | 用户管理 | [设计文档](./modules/user-service.md) |
| Organization Service | 组织/律所管理 | [设计文档](./modules/organization-service.md) |
| Files Service | 文件存储（MinIO、解析） | [设计文档](./modules/files-service.md) |
| Templates Service | 文书模板 | [设计文档](./modules/templates-service.md) |
| Billing Service | 计费订阅 | [设计文档](./modules/billing-service.md) |
| Notification Service | 通知推送 | [设计文档](./modules/notification-service.md) |
| Platform Service | 平台配置 | [设计文档](./modules/platform-service.md) |
| Collector Service | 种子数据/资源包管理 | [设计文档](./modules/collector-service.md) |

### 业务流程
- [咨询到事项转化流程](./flows/consultation-to-matter.md)
- [工作台目标与工作流（Workbench）](./flows/workbench-goals.md)
- [诉讼案件处理流程](./flows/litigation-workflow.md)
- [非诉业务处理流程](./flows/non-litigation-workflow.md)
- [Playbook 配置规范（Legacy，已废弃）](./flows/playbook-phases.md)

### 核心实现
- [Skill 技能系统](./implementation/skill-system.md)
- [Workflow 路由与执行](./implementation/planner-engine.md)
- [卡片交互机制](./implementation/card-interaction.md)
- [知识检索 RAG](./implementation/knowledge-rag.md)

### API 参考
- [API 设计规范](./api/conventions.md)
- [认证与授权](./api/authentication.md)
- [OpenAPI 文档](./api/openapi.md)

### 部署运维
- [本地开发环境](./deployment/local-dev.md)
- [Docker Compose 部署](./deployment/docker-compose.md)
- [生产环境部署](./deployment/production.md)

## 🏗️ 技术栈

### 后端
- **Java 21** + Spring Boot 3.3
- **Python** + FastAPI + LangGraph
- **PostgreSQL** + Flyway 迁移
- **Elasticsearch（可选）** keyword/vector/hybrid
- **Neo4j（可选）** GraphRAG/关系图
- **Redis** 缓存
- **MinIO** 对象存储

### 前端
- **Vue3** + TypeScript
- **Vite** 构建

### AI/ML
- **LangGraph** Agent 编排
- **OpenRouter** LLM 网关
- 默认模型/Embedding：以运行时配置为准

## 📁 仓库结构

```
docs/
├── README.md                 # 本文件
├── architecture/             # 架构设计
├── modules/                  # 模块设计文档
├── flows/                    # 业务流程
├── implementation/           # 核心实现
├── api/                      # API 参考
└── deployment/               # 部署文档
```

## 🔗 相关仓库

| 仓库 | 说明 |
|------|------|
| [ai-engine](https://github.com/lawseekdog/ai-engine) | AI 智能引擎 |
| [matter-service](https://github.com/lawseekdog/matter-service) | 事项管理服务 |
| [consultations-service](https://github.com/lawseekdog/consultations-service) | 咨询会话服务 |
| [knowledge-service](https://github.com/lawseekdog/knowledge-service) | 知识库服务 |
| [frontend](https://github.com/lawseekdog/frontend) | 前端应用 |
| ... | 其他服务 |

## 📝 贡献指南

1. Fork 本仓库
2. 创建特性分支 (`git checkout -b feature/xxx`)
3. 提交更改 (`git commit -m 'Add xxx'`)
4. 推送分支 (`git push origin feature/xxx`)
5. 创建 Pull Request

## 📄 License

MIT License
