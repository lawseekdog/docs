---
title: organization-service（组织/律所）
parent: 模块
nav_order: 12
---

# organization-service（组织/律所）

## 定位

organization-service 负责组织域能力：

- 组织/律所信息
- 律所成员与管理侧接口
- internal 查询（供其它服务做组织归属与权限判断）

## 技术栈

- Java 21 + Spring Boot
- Postgres + Flyway（默认）

## API（摘要）

对外（/api/v1）：

- `OrganizationController`、`FirmController`、`LawFirmController`
- `AdminOrganizationController`（管理端）

对内（/internal）：

- `InternalOrganizationController`

## 与其它服务的关系

- user-service：组织成员/用户归属查询
- matter-service：事项的 providerOrganizationId 与派单相关流程
