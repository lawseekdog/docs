---
title: 本地开发
parent: 部署与交付
nav_order: 1
---

# 本地开发（单服务/多服务）

## 1) Java 微服务（单仓库）

前置：

- Java 21
- Maven 3.9+
- PostgreSQL（本地推荐 Docker）

启动 Postgres（每个服务仓库都内置 `docker-compose.yml`）：

```bash
docker compose up -d
```

启动服务：

```bash
mvn -q test
mvn -q spring-boot:run
```

注意：

- 各服务默认把 Postgres 端口映射到 `5432`，多服务同时起会冲突；需要改端口或使用“共享 Postgres + 多库”方式编排。
- 内部接口密钥本地默认：`INTERNAL_API_KEY=test_internal_key`（见各服务 `application.yml`）。

## 2) 多服务联调（现状）

`law_tools_agent/` 属于 mono-repo 阶段的历史产物，当前已逐步废弃；其 `docker-compose.yml` 不再作为正式入口维护。

独立仓库形态下的“一键全栈”编排仍在对齐中（重点是 Python 服务镜像构建与依赖注入）。

推荐方向：

- 本地 K8s（kind/minikube）+ Helm：直接复用各服务的 `deploy/helm`，更接近生产运行形态
- 单独维护一个 `compose/infra` 仓库：用于多服务联调（统一端口/依赖/env 注入）
