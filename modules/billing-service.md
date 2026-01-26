---
title: billing-service（计费/订阅）
parent: 模块
nav_order: 13
---

# billing-service（计费/订阅）

## 定位

billing-service 负责订阅/权益/额度等计费域能力（当前为 MVP 形态，能力边界以代码为准）。

## 技术栈

- Java 21 + Spring Boot
- Postgres + Flyway（默认）

## API（摘要）

对外（/api/v1）：

- Membership/Subscription/Credit 等 controller（以 OpenAPI 为准）
- 管理端接口：`AdminMembershipController`

对内（/internal）：

- `InternalBillingController`、`InternalMembershipController` 等

## 与其它服务的关系

目前 billing-service 的跨服务依赖较少；后续可与 consultations/matter 等结合实现用量计费/限额控制。
