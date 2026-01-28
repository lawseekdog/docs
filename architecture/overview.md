# 系统架构概览

## 整体架构

LawSeekDog 是一个智能法律服务平台，采用微服务架构，核心由 AI 引擎驱动，支持法律咨询、案件管理、文书生成等全流程服务。

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                              Frontend (React)                                │
│                         律师端 / 客户端 / 管理后台                            │
└─────────────────────────────────────────────────────────────────────────────┘
                                      │
                                      ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                           API Gateway (Nginx)                                │
│                    路由 / 负载均衡 / JWT 验证 / 限流                          │
└─────────────────────────────────────────────────────────────────────────────┘
                                      │
        ┌─────────────────────────────┼─────────────────────────────┐
        │                             │                             │
        ▼                             ▼                             ▼
┌───────────────┐           ┌───────────────┐           ┌───────────────┐
│  Auth Service │           │ User Service  │           │  Org Service  │
│   认证授权     │           │   用户管理     │           │   组织管理     │
└───────────────┘           └───────────────┘           └───────────────┘
        │                             │                             │
        └─────────────────────────────┼─────────────────────────────┘
                                      │
┌─────────────────────────────────────────────────────────────────────────────┐
│                            业务服务层                                        │
├─────────────────┬─────────────────┬─────────────────┬─────────────────────┤
│ Consultations   │ Matter Service  │ Templates       │ Billing Service     │
│ 咨询会话        │ 事项管理        │ 文书模板        │ 计费订阅            │
└─────────────────┴─────────────────┴─────────────────┴─────────────────────┘
                                      │
                                      ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                           AI Engine (Python)                                 │
│              LangGraph Agent / Skill 编排 / Playbook 驱动                    │
└─────────────────────────────────────────────────────────────────────────────┘
                                      │
        ┌─────────────────────────────┼─────────────────────────────┐
        │                             │                             │
        ▼                             ▼                             ▼
┌───────────────┐           ┌───────────────┐           ┌───────────────┐
│Knowledge Svc  │           │ Memory Service│           │ Files Service │
│ 知识库检索     │           │ 记忆/画像      │           │ 文件存储解析   │
└───────────────┘           └───────────────┘           └───────────────┘
        │                             │                             │
        ▼                             ▼                             ▼
┌───────────────┐           ┌───────────────┐           ┌───────────────┐
│  Weaviate     │           │ PostgreSQL    │           │    MinIO      │
│  + ES         │           │               │           │               │
└───────────────┘           └───────────────┘           └───────────────┘
```

## 技术选型

### 后端服务

| 组件 | 技术栈 | 说明 |
|------|--------|------|
| Java 服务 | Spring Boot 3.3 + JDK 21 | 业务服务主体 |
| AI 引擎 | Python 3.12 + FastAPI + LangGraph | 智能决策核心 |
| 数据库 | PostgreSQL 16 | 主数据存储 |
| 缓存 | Redis 7 | 会话/缓存 |
| 搜索 | Elasticsearch 8 | 全文检索 |
| 向量库 | Weaviate（可选） | 语义检索（知识库） |
| 对象存储 | MinIO | 文件存储 |
| 消息队列 | Redis Streams | 异步任务 |

### 前端

| 组件 | 技术栈 |
|------|--------|
| 框架 | React 18 + TypeScript |
| 构建 | Vite 5 |
| 样式 | TailwindCSS |
| 状态 | Zustand |
| 路由 | React Router 6 |

### AI/ML

| 组件 | 说明 |
|------|------|
| LLM 网关 | OpenRouter |
| 默认模型 | DeepSeek V3.2 |
| Embedding | Qwen3-Embedding-8B (1024维) |
| Agent 框架 | LangGraph |

## 核心设计原则

### 1. 分层架构

所有 Java 服务遵循严格分层：

```
API Layer (Controller)
    ↓
Application Layer (Service)
    ↓
Domain Layer (Entity, Repository Interface)
    ↑
Infrastructure Layer (JPA, External Client)
```

依赖方向：`API → Application → Domain ← Infrastructure`

### 2. 统一接口规范

- 对外接口：`/api/v1/**`
- 内部接口：`/internal/**`
- 统一返回体：`ApiResponse<T>` / `PageResponse<T>`

```json
{
  "code": 0,
  "message": "OK",
  "data": { ... }
}
```

### 3. 可观测性

- 日志：Log4j2，包含 `request_id`, `user_id`, `organization_id`
- 指标：Prometheus + Actuator (`/internal/actuator`)
- 链路：OpenTelemetry (规划中)

### 4. 幂等性

涉及扣费、任务提交等写操作必须支持 `Idempotency-Key` 头。

## 服务通信

### 同步调用

服务间通过 HTTP REST 调用，使用 `shared-libs` 中的 `*ServiceClient`：

```java
// 示例：调用 User Service
UserServiceClient userClient = new UserServiceClient(baseUrl);
UserInfo user = userClient.getUser(userId);
```

### 异步通信

使用 Redis Streams 进行异步消息传递：

- 通知推送
- 异步任务处理
- 事件广播

## 数据存储策略

| 数据类型 | 存储 | 说明 |
|----------|------|------|
| 业务数据 | PostgreSQL | 事务一致性 |
| 会话缓存 | Redis | 高性能读写 |
| 文件内容 | MinIO | 对象存储 |
| 全文索引 | Elasticsearch | 法规/案例检索 |
| 语义向量 | Weaviate | 知识库 RAG |
| 记忆事实 | PostgreSQL | 偏好/摘要/证据线索（与 matter 分层） |

## 安全设计

### 认证

- JWT Token（Access + Refresh）
- Token 有效期：Access 30min，Refresh 7d

### 授权

- RBAC 角色权限模型
- 资源级别权限控制
- 组织隔离

### 数据安全

- 敏感数据加密存储
- 传输层 TLS
- 审计日志
