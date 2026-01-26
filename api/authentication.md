---
title: 认证与鉴权
parent: API 与规范
nav_order: 2
---

# 认证与鉴权（JWT + Internal API Key）

本页描述 LawSeekDog 的“对外认证”与“对内调用鉴权”约定（以 `ai-boot-framework` 与现有服务配置为准）。

## 1) 对外：JWT（可按服务启用）

业务微服务默认具备 JWT 资源服务器能力入口（是否启用由配置决定）：

- 配置前缀：`ai.boot.security.jwt.*`
- 常用 claim：
  - `user_id`（默认）
  - `user_type`（可选）
  - `organization_id`（可选）

在 Java 服务内，JWT claims 会被桥接写入 `RequestContext`，用于：

- 审计字段（`created_by/updated_by/organization_id`）
- 日志链路字段（MDC 透传）

说明：

- 模板工程本地默认 `jwt.enabled=false`，便于快速启动；生产建议开启并把密钥/公钥放到 K8s Secret。

## 2) 对内：`X-Internal-Api-Key`（强制）

所有 Java 服务的 `/internal/**` 必须校验内部密钥：

- Header：`X-Internal-Api-Key: <shared-secret>`
- 失败：返回 `401 Unauthorized`（统一返回体）

该机制用于“服务间调用”与“运维/seed 导入”等内部链路。

建议（生产）：

- internal key 由平台统一下发（K8s Secret → env）
- 配合 NetworkPolicy/ServiceMesh 做网络层隔离（即使 key 泄露也缩小攻击面）

## 3) Python 服务（ai-engine / collector-service）

Python 服务属于 internal-only 服务形态：

- 入口一般不直接暴露给公网
- 上游（Java 服务）通过 internal 网络访问其 `/internal/**` 接口
- internal key 建议同样通过 header 透传（与 Java 侧一致），避免“跨语言两套鉴权”

> 具体 header 校验以 Python 侧实现为准；当前以“网络隔离 + 上游校验”为主。
