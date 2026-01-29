---
title: ai-boot-framework（工程脚手架）
parent: 模块
nav_order: 16
---

# ai-boot-framework（工程脚手架）

## 定位

`ai-boot-framework` 是 LawSeekDog 的 Java/Spring Boot 微服务脚手架与工程规范仓库，用于把跨服务的“硬门槛”固化为可复用组件：

- 统一返回体/错误码
- `/api/**` 与 `/api/v1/internal/**` 边界
- internal api key 鉴权、JWT 资源服务器入口
- OpenAPI 输出策略（默认只暴露对外 API）
- JPA + Flyway 基座
- 幂等（Idempotency-Key）
- 可观测（Actuator/Prometheus）与 OTel tracing
- 内部 HTTP 客户端（超时/透传/一致性）
- 可选 Kafka/LLM OpenAI-compat 客户端（按需引入，不默认）

同时提供：

- BOM（版本对齐）
- Starter（开箱即用聚合依赖）
- Maven Archetype（生成新服务模板工程）

## 与业务仓库的关系

大多数 Java 微服务（auth/user/matter/knowledge/…）都是基于 archetype 生成，并通过 BOM/Starter 对齐依赖。

## 版本与依赖拉取（GitHub Packages）

ai-boot-framework 发布到 GitHub Packages；业务仓库通过：

- `.mvn/settings.xml`（读取 `GH_PACKAGES_USERNAME/GH_PACKAGES_TOKEN`）
- `.mvn/maven.config`（强制使用仓库内 settings）

来解决 CI 读取私有包的认证问题。

> 详细用法以 ai-boot-framework 仓库内文档为准；本 docs 仓库只记录系统组装关系与落地约束。
