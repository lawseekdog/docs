---
title: 项目进度与现状
parent: 架构
nav_order: 2
---

# 项目进度与现状（从 mono-repo 到多仓库）

本页用于回答“目前整体做到哪一步了、哪些是可用的、哪些还需要补齐”的问题（以当前代码仓库为准）。

## 1) 拆分完成度（law_tools_agent → 多仓库）

已完成：

- 主要服务已拆分为独立仓库（Java 微服务 + Python AI 服务 + 前端 + infra/docs/e2e）
- Java 微服务工程骨架统一：`ai-boot-framework`（BOM/Starter/Archetype）
- 统一 CI/CD：`infra-templates` 复用工作流，推送容器镜像（默认 GHCR，可切换到阿里云 ACR）
- 每个 Java 服务仓库具备：
  - `.mvn/settings.xml`（从 env 读取 GitHub Packages 凭据）
  - `Dockerfile`（以 `target/*.jar` 构建运行镜像）
  - `deploy/helm`（可直接用于 K8s 部署）

待补齐（重点）：

- Python 服务（`ai-engine` / `collector-service`）的 Dockerfile 仍保留 mono-repo build context 假设（`COPY shared-libs ...`、`COPY ai-engine/...`），需要对齐为“独立仓库可构建镜像”。
- 多仓库的“本地一键起全栈”编排尚未沉淀为独立仓库/统一入口（原 `law_tools_agent` 的 compose/部署脚本已逐步废弃，需要迁移到专用的 infra/compose 仓库）。

## 2) 运行时链路可用性（按关键路径）

- 对话链路（Frontend → consultations-service SSE → ai-engine NDJSON）：骨架已具备（需结合环境配置验证）。
- 事项链路（Matter/Phase/Todo + Playbook）：matter-service 作为真源，playbook 由 platform-service 承载；ai-engine 负责技能推进。
- 知识链路（knowledge-service）：已具备 Postgres 真源 + 可选 ES/Neo4j 的检索/GraphRAG 能力，且支持 seed 导入。
- 记忆链路（memory-service）：以结构化存储为主；“抽取”能力目前处于占位/待完善状态（见 `implementation/memory-extraction.md`）。

## 3) 工程化与交付

已具备：

- Maven 私有依赖（GitHub Packages）拉取方案：`.mvn/settings.xml` + `GH_PACKAGES_*` env
- Tag 发布与镜像构建：各服务 `on: push tags: v*`（由 `infra-templates` 复用工作流实现）
- 多架构镜像：main/tag 默认 `linux/amd64,linux/arm64`

风险与约束：

- GitHub Actions Artifacts/Caches 有组织级配额限制；当前工作流默认不上传构建制品、也不启用 Actions Cache，以降低配额风险（以容器镜像为交付物）。
