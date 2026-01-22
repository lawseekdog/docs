# 微服务拓扑

## 服务依赖关系

```
                                    ┌─────────────┐
                                    │   Frontend  │
                                    └──────┬──────┘
                                           │
                                    ┌──────▼──────┐
                                    │ API Gateway │
                                    └──────┬──────┘
                                           │
         ┌─────────────────────────────────┼─────────────────────────────────┐
         │                                 │                                 │
         ▼                                 ▼                                 ▼
┌─────────────────┐              ┌─────────────────┐              ┌─────────────────┐
│  Auth Service   │◄─────────────│ User Service    │◄─────────────│  Org Service    │
│                 │              │                 │              │                 │
│ - JWT 签发/验证  │              │ - 用户 CRUD     │              │ - 组织管理       │
│ - 登录/注册     │              │ - 角色权限      │              │ - 成员管理       │
│ - Token 刷新    │              │ - 超级管理员    │              │ - 律所认证       │
└─────────────────┘              └─────────────────┘              └─────────────────┘
         │                                 │                                 │
         └─────────────────────────────────┼─────────────────────────────────┘
                                           │
                                           ▼
┌──────────────────────────────────────────────────────────────────────────────────┐
│                              Consultations Service                                │
│                                                                                  │
│  - 会话管理 (Session)           - 消息存储 (Message)                              │
│  - 实时对话                     - 卡片交互 (Card)                                 │
│  - AI Engine 调用               - 会话状态机                                      │
└──────────────────────────────────────────────────────────────────────────────────┘
         │                                 │                                 │
         │                                 ▼                                 │
         │                    ┌─────────────────────┐                        │
         │                    │    AI Engine        │                        │
         │                    │                     │                        │
         │                    │ - LangGraph Agent   │                        │
         │                    │ - Skill 编排        │                        │
         │                    │ - Playbook 驱动     │                        │
         │                    │ - Planner 决策      │                        │
         │                    └─────────┬───────────┘                        │
         │                              │                                    │
         │         ┌────────────────────┼────────────────────┐               │
         │         │                    │                    │               │
         │         ▼                    ▼                    ▼               │
         │  ┌─────────────┐     ┌─────────────┐     ┌─────────────┐          │
         │  │ Knowledge   │     │   Memory    │     │   Files     │          │
         │  │ Service     │     │   Service   │     │   Service   │          │
         │  │             │     │             │     │             │          │
         │  │ - 法规检索   │     │ - 事实提取  │     │ - 文件上传  │          │
         │  │ - 案例检索   │     │ - 用户画像  │     │ - 文件解析  │          │
         │  │ - 要素匹配   │     │ - 记忆召回  │     │ - OCR/PDF   │          │
         │  └──────┬──────┘     └──────┬──────┘     └──────┬──────┘          │
         │         │                   │                   │                 │
         │         ▼                   ▼                   ▼                 │
         │  ┌─────────────┐     ┌─────────────┐     ┌─────────────┐          │
         │  │ Weaviate    │     │   Qdrant    │     │   MinIO     │          │
         │  │ + ES        │     │             │     │             │          │
         │  └─────────────┘     └─────────────┘     └─────────────┘          │
         │                                                                   │
         ▼                                                                   ▼
┌─────────────────┐                                              ┌─────────────────┐
│ Matter Service  │◄─────────────────────────────────────────────│ Templates Svc   │
│                 │                                              │                 │
│ - 事项管理      │                                              │ - 文书模板      │
│ - 待办任务      │                                              │ - 模板渲染      │
│ - 阶段推进      │                                              │ - 变量替换      │
│ - 数据同步      │                                              │                 │
└─────────────────┘                                              └─────────────────┘
         │
         ▼
┌─────────────────┐              ┌─────────────────┐              ┌─────────────────┐
│ Billing Service │              │ Notification    │              │ Platform Svc    │
│                 │              │ Service         │              │                 │
│ - 订阅管理      │              │ - 消息推送      │              │ - 系统配置      │
│ - 用量计费      │              │ - 邮件/短信     │              │ - 功能开关      │
│ - 支付集成      │              │ - WebSocket     │              │ - 审计日志      │
└─────────────────┘              └─────────────────┘              └─────────────────┘
```

## 服务清单

| 服务 | 端口 | 技术栈 | 职责 |
|------|------|--------|------|
| api-gateway | 80 | Nginx | 路由、负载均衡、JWT 验证 |
| auth-service | 8080 | Java/Spring | 认证授权 |
| user-service | 8080 | Java/Spring | 用户管理 |
| organization-service | 8080 | Java/Spring | 组织管理 |
| consultations-service | 8080 | Java/Spring | 咨询会话 |
| matter-service | 8080 | Java/Spring | 事项管理 |
| knowledge-service | 8080 | Java/Spring | 知识库 |
| memory-service | 8080 | Python/FastAPI | 记忆服务 |
| files-service | 8080 | Java/Spring | 文件服务 |
| templates-service | 8080 | Java/Spring | 模板服务 |
| billing-service | 8080 | Java/Spring | 计费服务 |
| notification-service | 8080 | Java/Spring | 通知服务 |
| platform-service | 8080 | Java/Spring | 平台配置 |
| collector-service | 8080 | Python/FastAPI | 种子数据 |
| ai-engine | 8080 | Python/FastAPI | AI 引擎 |

## 服务间调用矩阵

| 调用方 ↓ / 被调用方 → | Auth | User | Org | Consult | Matter | Knowledge | Memory | Files | AI Engine |
|----------------------|------|------|-----|---------|--------|-----------|--------|-------|-----------|
| Gateway | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ | - | ✓ | - |
| Auth | - | ✓ | - | - | - | - | - | - | - |
| User | ✓ | - | ✓ | - | - | - | - | - | - |
| Consultations | ✓ | ✓ | - | - | ✓ | - | - | ✓ | ✓ |
| Matter | ✓ | ✓ | ✓ | ✓ | - | - | - | ✓ | - |
| AI Engine | - | ✓ | - | - | ✓ | ✓ | ✓ | ✓ | - |
| Knowledge | - | - | - | - | - | - | - | - | - |
| Memory | - | - | - | - | - | - | - | - | - |

## 数据库隔离

每个服务独立数据库，遵循微服务数据隔离原则：

```
PostgreSQL
├── auth-service        # 认证数据
├── user-service        # 用户数据
├── organization-service # 组织数据
├── consultations-service # 会话数据
├── matter-service      # 事项数据
├── knowledge-service   # 知识元数据
├── files-service       # 文件元数据
├── templates-service   # 模板数据
├── billing-service     # 计费数据
├── notification-service # 通知数据
└── platform-service    # 平台配置
```

## 外部依赖

| 服务 | 依赖 | 用途 |
|------|------|------|
| AI Engine | OpenRouter | LLM API 网关 |
| Knowledge Service | Weaviate | 向量存储 |
| Knowledge Service | Elasticsearch | 全文检索 |
| Memory Service | Qdrant | 记忆向量 |
| Files Service | MinIO | 对象存储 |
| All Services | Redis | 缓存/消息 |
