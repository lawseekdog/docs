---
title: user-service（用户）
parent: 模块
nav_order: 11
---

# user-service（用户）

## 定位

user-service 负责用户域数据：

- 用户信息（User）
- 律师画像（LawyerProfile）
- 管理端用户管理
- internal 用户查询（供 auth/matter/files/organization 等服务调用）

## 技术栈

- Java 21 + Spring Boot
- Postgres + Flyway（默认）

## API（摘要）

对外（/api/v1）：

- `UserController`：用户信息
- `LawyerProfileController`：律师画像
- `AdminUserController`：管理端接口

对内（/internal）：

- `InternalUserController`
- `InternalLawyerController`

## 与其它服务的关系

- auth-service：查询用户账号信息/权限绑定
- matter-service：创建事项时用于判断创建者类型（lawyer/firm_admin）并绑定承办律师/律所
- files-service/organization-service：用于权限/归属校验
