---
title: notification-service（通知）
parent: 模块
nav_order: 14
---

# notification-service（通知）

## 定位

notification-service 负责通知域能力（当前为 MVP 形态）：

- 通知发送/查询
- 用户通知偏好设置
- internal 通知接口（供其它服务触发）

## 技术栈

- Java 21 + Spring Boot
- Postgres + Flyway（默认）

## API（摘要）

对外（/api/v1）：

- `NotificationController`
- `NotificationPreferenceController`

对内（/internal）：

- `InternalNotificationController`

## 与其它服务的关系

典型触发方包括 matter-service/billing-service/consultations-service 等（按业务需要）。
