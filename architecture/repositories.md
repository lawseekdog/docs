---
title: 仓库结构与拆分映射
parent: 架构
nav_order: 5
---

# 仓库结构与拆分映射

本页用于回答两个问题：

1) 现在的 LawSeekDog “完整项目组”由哪些仓库组成？
2) 多仓库之间有哪些关键依赖/构建约束？

## 当前仓库清单（以组织 lawseekdog 为准）

说明：

- “运行时”指会被部署到 K8s 集群的服务/应用（或其依赖的基础设施适配）。
- “工程化”指脚手架、CI 模板、文档等不会直接承载业务流量的仓库。

| 仓库 | 语言/形态 | 角色 | 备注 |
|------|----------|------|------|
| `frontend` | Node/Vue3 | 运行时 | 多端前端（admin/lawyer/firm） |
| `auth-service` | Java/Spring Boot | 运行时 | 登录/鉴权/RBAC（对外 `/api/**`；内部 `/api/v1/internal/**`） |
| `user-service` | Java/Spring Boot | 运行时 | 用户/画像（业务域用户） |
| `organization-service` | Java/Spring Boot | 运行时 | 组织/律所/成员 |
| `billing-service` | Java/Spring Boot | 运行时 | 订阅/用量/额度（MVP 形态） |
| `notification-service` | Java/Spring Boot | 运行时 | 通知（MVP 形态） |
| `platform-service` | Java/Spring Boot | 运行时 | 平台配置、SystemConfig、FeatureFlag、Tag 等（PlaybookConfig 为历史兼容） |
| `consultations-service` | Java/Spring Boot | 运行时 | 对话会话 + SSE；对接 `ai-engine` 的 NDJSON 流 |
| `matter-service` | Java/Spring Boot | 运行时 | 事项/待办/阶段推进；对接 `ai-engine` |
| `knowledge-service` | Java/Spring Boot | 运行时 | 知识库：文档/Chunk、原子检索、GraphRAG（ES/Neo4j 可选） |
| `files-service` | Java/Spring Boot | 运行时 | 文件元数据 + 对象存储适配（MinIO/S3） |
| `templates-service` | Java/Spring Boot | 运行时 | 模板/文书生成（与 `ai-engine` 交互） |
| `gateway-service` | Java/Spring Boot | 运行时（占位） | 当前更偏“样板/占位”，不等同于集群入口网关 |
| `ai-engine` | Python/FastAPI | 运行时 | AI 执行引擎（LangGraph workbench、skills、NDJSON 事件流） |
| `collector-service` | Python/FastAPI | 运行时 | Seed Packages（系统资源/字典/结构化种子）分发到各服务 |
| `shared-libs` | Python package | 运行时依赖 | Python 侧通用库（contracts、clients、config/logging） |
| `lawseekdog-seed-init` | Python CLI | 工具 | 调用 `collector-service` internal seed 接口的轻量 CLI |
| `e2e-tests` | Python/pytest | 工具 | 端到端测试 |
| `docs` | Jekyll | 工程化 | 本文档站 |
| `ai-boot-framework` | Java/Maven | 工程化 | 微服务脚手架（BOM/Starter/Archetype） |
| `infra-templates` | GitHub Actions | 工程化 | 复用工作流（CI/CD） |
| `infra-live` | Terraform/Helm | 工程化 | 阿里云（VPC + ECS + 自建 k3s）（Terraform）+ 整体发布（集中式 Deploy） |

## 现状差异与迁移注意点（按“代码真实情况”）

### 1) Python 服务的独立构建约束：shared-libs（跨仓库依赖）

剩余 Python 服务中仍有仓库依赖 `shared-libs`（独立仓库，私有），这类依赖需要继续收敛为服务内契约或 HTTP 边界。

当前采用的工程化策略是：

- Python 服务仓库使用 `infra-templates` 的复用工作流：`docker-service-ci.yml`
- CI 会额外 checkout `lawseekdog/shared-libs` 到工作区根目录（`./shared-libs/`），Dockerfile 直接 `COPY shared-libs ...` 完成安装

这意味着：

- Java 微服务：已完成“多仓库独立构建与发布”（容器镜像仓库 + Helm；默认 GHCR，可切换到阿里云 ACR）。
- Python 微服务：应避免通过私有共享包传递运行时契约；独立构建优先依赖本服务代码、OpenAPI/HTTP 契约与显式生成物。

### 2) 文档漂移提示

早期单仓库阶段的技术栈与运行方式与当前多仓库形态可能不同（例如前端框架、检索实现、部署方式等）。

本 docs 仓库会以“当前仓库实现”为准逐步修正漂移：

- “现状/已实现”必须可在代码或 OpenAPI 中验证
- “规划/待实现”必须明确标注
