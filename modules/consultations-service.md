# Consultations Service 模块设计

## 概述

Consultations Service 负责管理用户与 AI 的对话会话，包括消息存储、卡片交互、AI Engine 调用等。

## 核心概念

### Session（会话）

```java
public class Session {
    private UUID id;
    private UUID userId;
    private UUID matterId;           // 关联的事项（可选）
    private String engagementMode;   // legal_consultation | start_service
    private String serviceTypeId;
    private String playbookId;
    private SessionStatus status;
    private JsonNode state;          // 会话状态快照
    private Instant createdAt;
}
```

### Message（消息）

```java
public class Message {
    private UUID id;
    private UUID sessionId;
    private MessageRole role;        // user | assistant | system
    private String content;
    private JsonNode metadata;
    private Instant createdAt;
}
```

### Card（卡片）

```java
public class Card {
    private UUID id;
    private UUID sessionId;
    private String todoKey;
    private String type;             // ask_user
    private String reviewType;       // clarify | confirm | select
    private String skillId;
    private CardStatus status;       // pending | completed | cancelled
    private JsonNode payload;
    private JsonNode result;
}
```

## 核心功能

### 1. 会话管理

```
POST   /api/v1/consultations/sessions           # 创建会话
GET    /api/v1/consultations/sessions/{id}      # 获取会话
DELETE /api/v1/consultations/sessions/{id}      # 结束会话
```

### 2. 对话交互

```
POST   /api/v1/consultations/sessions/{id}/chat # 发送消息
GET    /api/v1/consultations/sessions/{id}/messages # 获取消息历史
```

### 3. 卡片交互

```
GET    /api/v1/consultations/sessions/{id}/cards      # 获取卡片列表
POST   /api/v1/consultations/sessions/{id}/cards/{cardId}/submit # 提交卡片
```

## 对话流程

```
用户发送消息
    │
    ▼
┌─────────────────────────────────────┐
│      Consultations Service          │
│                                     │
│  1. 保存用户消息                     │
│  2. 构建 AI Engine 请求             │
│     - session_id                    │
│     - matter_id                     │
│     - user_message                  │
│     - attachment_file_ids           │
│     - state (from session/matter)   │
│                                     │
└────────────────┬────────────────────┘
                 │
                 ▼ POST /chat
┌─────────────────────────────────────┐
│          AI Engine                  │
│                                     │
│  执行 Agent 流程                     │
│  返回:                              │
│  - response                         │
│  - card (可选)                      │
│  - state_patch                      │
│                                     │
└────────────────┬────────────────────┘
                 │
                 ▼
┌─────────────────────────────────────┐
│      Consultations Service          │
│                                     │
│  3. 保存 AI 回复消息                 │
│  4. 处理卡片（如有）                 │
│  5. 更新会话状态                     │
│  6. 返回响应给前端                   │
│                                     │
└─────────────────────────────────────┘
```

## 卡片交互机制

### 卡片生命周期

```
AI Engine 返回 card
    │
    ▼
┌─────────┐
│ pending │ ◀─────────────────────────┐
└────┬────┘                           │
     │                                │
     │ submit()                       │ reopen()
     ▼                                │
┌───────────┐                    ┌────┴────┐
│ completed │                    │ reopened│
└───────────┘                    └─────────┘
```

### 卡片提交

```java
public CardSubmitResponse submitCard(UUID sessionId, UUID cardId, JsonNode answers) {
    Card card = cardRepository.findById(cardId);

    // 1. 验证卡片状态
    if (card.getStatus() != CardStatus.PENDING) {
        throw new BizException("卡片已处理");
    }

    // 2. 调用 AI Engine 处理 human_review
    AiEngineResponse response = aiEngineClient.humanReview(
        sessionId,
        card.getTaskKey(),
        answers
    );

    // 3. 更新卡片状态
    card.setStatus(CardStatus.COMPLETED);
    card.setResult(answers);

    // 4. 同步到 Matter（如果关联）
    if (session.getMatterId() != null) {
        matterClient.completeTask(session.getMatterId(), card.getTaskKey(), answers);
    }

    return new CardSubmitResponse(response);
}
```

## 会话状态管理

### 状态来源

```
┌─────────────────┐
│ Session.state   │ ◀─── 会话级状态（咨询模式）
└────────┬────────┘
         │
         │ 升级为事项后
         ▼
┌─────────────────┐
│ Matter.profile  │ ◀─── 事项级状态（服务模式）
│ Matter.data     │
└─────────────────┘
```

### 状态同步

```java
public JsonNode buildAiEngineState(Session session) {
    if (session.getMatterId() != null) {
        // 服务模式：从 Matter 获取状态
        Matter matter = matterClient.getMatter(session.getMatterId());
        return buildStateFromMatter(matter);
    } else {
        // 咨询模式：使用 Session 状态
        return session.getState();
    }
}
```

## Engagement Mode

### legal_consultation（法律咨询）

- 轻量级对话
- 不创建 Matter
- 状态存储在 Session
- 可升级为 start_service

### start_service（开始服务）

- 创建 Matter
- 状态存储在 Matter
- 支持完整 Playbook 流程
- 支持待办任务

### 模式切换

```java
public void upgradeToService(UUID sessionId, String serviceTypeId) {
    Session session = sessionRepository.findById(sessionId);

    // 1. 创建 Matter
    Matter matter = matterService.create(
        session.getUserId(),
        serviceTypeId,
        session.getState()
    );

    // 2. 更新 Session
    session.setMatterId(matter.getId());
    session.setEngagementMode("start_service");

    sessionRepository.save(session);
}
```

## 数据模型

```
┌─────────────────┐       ┌─────────────────┐
│     Session     │       │    Message      │
├─────────────────┤       ├─────────────────┤
│ id              │───┐   │ id              │
│ user_id         │   │   │ session_id      │───┐
│ matter_id       │   │   │ role            │   │
│ engagement_mode │   └──▶│ content         │   │
│ service_type_id │       │ metadata        │   │
│ playbook_id     │       │ created_at      │   │
│ status          │       └─────────────────┘   │
│ state (JSONB)   │                             │
│ created_at      │       ┌─────────────────┐   │
└─────────────────┘       │      Card       │   │
        │                 ├─────────────────┤   │
        │                 │ id              │   │
        └────────────────▶│ session_id      │───┘
                          │ task_key        │
                          │ type            │
                          │ review_type     │
                          │ skill_id        │
                          │ status          │
                          │ payload         │
                          │ result          │
                          └─────────────────┘
```

## 目录结构

```
consultations-service/
├── src/main/java/com/lawseekdog/consultations/
│   ├── api/
│   │   ├── controller/
│   │   │   ├── SessionController.java
│   │   │   └── CardController.java
│   │   └── dto/
│   ├── application/
│   │   └── service/
│   │       ├── SessionService.java
│   │       ├── MessageService.java
│   │       └── CardService.java
│   ├── domain/
│   │   ├── entity/
│   │   │   ├── Session.java
│   │   │   ├── Message.java
│   │   │   └── Card.java
│   │   └── repository/
│   └── infrastructure/
│       ├── client/
│       │   ├── AiEngineClient.java
│       │   └── MatterServiceClient.java
│       └── persistence/
└── src/main/resources/
```
