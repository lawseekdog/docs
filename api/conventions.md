---
title: API 约定
parent: API 与规范
nav_order: 1
---

# API 约定（Google REST + 统一返回体）

本页描述 LawSeekDog 各 Java 微服务对外/对内 API 的统一约定（以 `ai-boot-framework` 规范与现有服务实现为准）。

## 1) API 边界（强制）

- 对外：`/api/v1/**`（面向前端/第三方/业务调用）
- 对内：`/api/v1/internal/**`（Service-to-Service 调用）
- **禁止**对外暴露 `/api/v1/internal/**`（生产应由网络策略 + 内部鉴权双保险）

## 2) 资源导向（Google REST/AIP 风格）

- URI 表示资源（名词），动作表达在 HTTP 方法与子资源上
- 集合：`/api/v1/matters`
- 单体：`/api/v1/matters/{matterId}`
- 子资源：`/api/v1/matters/{matterId}/todos`

方法语义（强制）：

- `GET` 读取（幂等）
- `POST` 创建或触发非幂等动作
- `PATCH` 局部更新（推荐）
- `DELETE` 删除（幂等）

## 3) 统一返回体

各服务使用 `ai-boot-core` 的统一返回体（示例字段）：

- 非分页：`ApiResponse<T>`（`code/message/data`）
- 分页：`PageResponse<T>`（`code/message/data/meta`）

约束：

- HTTP 状态码必须正确表达语义（不能只靠 `code`）
- 错误必须可定位（至少有 `X-Request-Id` 贯穿链路）

## 4) 常用请求头（跨服务一致）

- `Authorization: Bearer <jwt>`：对外认证（由 `auth-service` 签发；各服务可按需启用资源服务器校验）
- `X-Request-Id`：链路追踪 ID（服务端可生成并回写；日志 MDC 必须透传）
- `X-Organization-Id`：组织/律所上下文（是否强制由配置决定；JPA 审计会使用）
- `Idempotency-Key`：幂等键（涉及扣费/任务/写外部系统等写操作必须支持）
- `X-Internal-Api-Key`：内部接口鉴权（仅允许 `/api/v1/internal/**` 使用）

## 5) 列表/过滤/分页/排序（建议统一）

- 列表：`GET /resources`
- 过滤：query string（不要用请求体）
- 分页：项目内统一一种风格（`page/size` 或 `page_size/page_token`）
- 排序：`order_by=created_at desc,updated_at asc`

> 当前模板工程更偏“易用优先”，但正式阶段建议逐步收敛到 token 分页，避免深翻页性能与一致性问题。

## 6) 错误与状态码（强制）

常用 HTTP 状态码（最低要求）：

- `200/201/204` 成功
- `400` 参数错误
- `401` 未认证（含 internal key 校验失败）
- `403` 无权限
- `404` 资源不存在
- `409` 冲突（幂等键冲突、状态机冲突等）
- `429` 限流/配额不足
- `500` 服务端异常

## 7) 幂等（强制：关键写操作）

统一约定：

- 客户端生成 `Idempotency-Key`
- 服务端必须记录幂等键与结果（或通过唯一约束 + 409 冲突表达）

框架侧已提供幂等回放基座（`ai-boot-spring-idempotency`）；业务侧应明确哪些接口必须启用。
