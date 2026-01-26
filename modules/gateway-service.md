---
title: gateway-service（占位/样板服务）
parent: 模块
nav_order: 15
---

# gateway-service（占位/样板服务）

## 定位

`gateway-service` 当前更偏“样板/占位”：

- 用于承载统一工程骨架（ai-boot-framework）在最小服务上的落地
- 提供 request-id relay 等最小示例接口

它 **不是** 生产入口网关（Ingress/API Gateway）：

- 生产入口通常由 K8s Ingress 或专用网关产品承接（鉴权/路由/限流等）
- gateway-service 未来若要承担真正网关职责，需要明确是否采用 Spring Cloud Gateway 或引入网关产品替代

## 技术栈

- Java 21 + Spring Boot
