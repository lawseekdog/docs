---
title: auth-service（认证授权）
parent: 模块
nav_order: 10
---

# auth-service（认证授权）

## 定位

auth-service 负责系统的认证与授权：

- 登录/注册/Token（以当前实现为准）
- RBAC 权限校验（供其它服务调用）
- internal 鉴权边界（/api/v1/internal/**）

## 技术栈

- Java 21 + Spring Boot
- Postgres + Flyway（默认）

## 对外 API（/api/v1，摘要）

按 controller 命名大致包含：

- `AuthController`：登录/Token
- `RbacController`：权限/角色/资源（以实现为准）
- `AdminAuthController`：管理端接口

## 对内 API（/internal，摘要）

- `InternalAuthController`：服务间鉴权/权限校验入口（供 collector-service 等调用）

## 与其它服务的关系

- user-service：auth-service 通过 `UserAccountClient` 查询用户账号信息
- collector-service：通过 auth-service 做权限校验（collector:read / collector:manage）
- 其它 Java 服务：多为资源服务器形态（JWT 校验）+ internal api key（由 `ai-boot-framework` 提供基座）

> Token 细节与时效以 auth-service 配置为准；docs 不在此硬编码固定时长。
