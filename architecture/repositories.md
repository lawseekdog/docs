---
title: 仓库结构与拆分映射
parent: 架构
nav_order: 5
---

# 仓库结构与拆分映射（law_tools_agent → 多仓库）

本页回答两个问题：

1) 现在的 LawSeekDog “完整项目组”由哪些仓库组成？  
2) `law_tools_agent/`（原始 mono-repo）与当前多仓库之间如何对应？

## 当前仓库清单（以组织 lawseekdog 为准）

说明：

- “运行时”指会被部署到 K8s 集群的服务/应用（或其依赖的基础设施适配）。
- “工程化”指脚手架、CI 模板、文档等不会直接承载业务流量的仓库。

| 仓库 | 语言/形态 | 角色 | 备注 |
|------|----------|------|------|
| `frontend` | Node/Vue3 | 运行时 | 多端前端（admin/lawyer/firm） |
| `auth-service` | Java/Spring Boot | 运行时 | 登录/鉴权/RBAC（对外 `/api/**`；内部 `/internal/**`） |
| `user-service` | Java/Spring Boot | 运行时 | 用户/画像（业务域用户） |
| `organization-service` | Java/Spring Boot | 运行时 | 组织/律所/成员 |
| `billing-service` | Java/Spring Boot | 运行时 | 订阅/用量/额度（MVP 形态） |
| `notification-service` | Java/Spring Boot | 运行时 | 通知（MVP 形态） |
| `platform-service` | Java/Spring Boot | 运行时 | 平台配置、PlaybookConfig、FeatureFlag、Tag 等 |
| `consultations-service` | Java/Spring Boot | 运行时 | 对话会话 + SSE；对接 `ai-engine` 的 NDJSON 流 |
| `matter-service` | Java/Spring Boot | 运行时 | 事项/待办/阶段推进；对接 `ai-engine` |
| `knowledge-service` | Java/Spring Boot | 运行时 | 知识库：文档/Chunk、原子检索、GraphRAG（ES/Neo4j 可选） |
| `memory-service` | Java/Spring Boot | 运行时 | 记忆条目/事实（当前以结构化存储为主；抽取能力待完善） |
| `files-service` | Java/Spring Boot | 运行时 | 文件元数据 + 对象存储适配（MinIO/S3） |
| `templates-service` | Java/Spring Boot | 运行时 | 模板/文书生成（与 `ai-engine` 交互） |
| `gateway-service` | Java/Spring Boot | 运行时（占位） | 当前更偏“样板/占位”，不等同于集群入口网关 |
| `ai-engine` | Python/FastAPI | 运行时 | AI 执行引擎（LangGraph、skills、NDJSON 事件流） |
| `collector-service` | Python/FastAPI | 运行时 | Seed Packages（系统资源/Playbooks/结构化种子）分发到各服务 |
| `shared-libs` | Python package | 运行时依赖 | Python 侧通用库（contracts、clients、config/logging） |
| `lawseekdog-seed-init` | Python CLI | 工具 | 调用 `collector-service` internal seed 接口的轻量 CLI |
| `e2e-tests` | Python/pytest | 工具 | 端到端测试 |
| `docs` | Jekyll | 工程化 | 本文档站 |
| `ai-boot-framework` | Java/Maven | 工程化 | 微服务脚手架（BOM/Starter/Archetype） |
| `infra-templates` | GitHub Actions | 工程化 | 复用工作流（CI/CD） |
| `law_tools_agent` | mono-repo | 历史/对照 | 原始项目形态（Playbook/Skills/服务编排规范的来源） |

## law_tools_agent（原始项目）与多仓库的对应关系

`law_tools_agent/` 是最初的 mono-repo：包含前端、后端与一组微服务目录，并沉淀了 Skills/Playbooks/卡片中断的规范。

在 `law_tools_agent/services/` 下能看到“服务目录集合”，拆分后大体对应到顶层同名仓库（部分做了命名调整/域合并）。

### 1) 直接一一对应（或仅改名）

| mono-repo 路径 | 现在的仓库 |
|---------------|-----------|
| `law_tools_agent/services/ai-platform-service` | `ai-engine`（仓库名变化；`pyproject.toml` 仍叫 `ai-platform-service`） |
| `law_tools_agent/services/matters-service` | `matter-service`（单数化） |
| `law_tools_agent/services/users-service` | `user-service`（单数化） |
| `law_tools_agent/services/auth-service` | `auth-service` |
| `law_tools_agent/services/consultations-service` | `consultations-service` |
| `law_tools_agent/services/files-service` | `files-service` |
| `law_tools_agent/services/gateway-service` | `gateway-service` |
| `law_tools_agent/services/knowledge-service` | `knowledge-service`（注意：实现已从旧代码迁移/重写，能力边界以当前仓库为准） |
| `law_tools_agent/services/memory-service` | `memory-service` |
| `law_tools_agent/services/platform-service` | `platform-service` |
| `law_tools_agent/services/templates-service` | `templates-service` |
| `law_tools_agent/services/collector-service` | `collector-service` |
| `law_tools_agent/services/shared-libs` | `shared-libs` |
| `law_tools_agent/frontend` | `frontend` |

### 2) 域合并/重组（需要人工确认的部分）

| mono-repo 路径 | 现在的仓库 | 说明 |
|---------------|-----------|------|
| `law_tools_agent/services/membership-service` | `billing-service`（推测） | 订阅/权益/用量通常归入 billing 域；以当前代码为准 |

## 现状差异与迁移注意点（按“代码真实情况”）

### 1) Python 服务的 Dockerfile 仍保留 mono-repo 路径假设

当前 `ai-engine/Dockerfile`、`collector-service/Dockerfile` 仍使用类似 `COPY shared-libs ...`、`COPY ai-engine/...` 的路径写法，
这在“单仓库独立构建镜像”的 CI/CD 下会失败（因为 build context 里不存在这些前缀目录）。

这意味着：

- Java 微服务已经完成“多仓库独立构建与发布”（容器镜像仓库 + Helm；默认 GHCR，可切换到阿里云 ACR）。
- Python 服务仍处于“拆分后的工程化对齐中”（需要调整 Dockerfile/依赖方式/CI 工作流）。

### 2) 旧文档中的技术栈信息可能来自 mono-repo 阶段

例如：前端从 React 切到 Vue，向量检索从 Weaviate/Qdrant 迁移到 ES kNN 等。
本 docs 仓库会以当前仓库实现为准逐步修正这些漂移。
