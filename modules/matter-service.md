# Matter Service 模块设计

## 概述

Matter Service 负责法律事项（案件）的全生命周期管理，包括事项创建、阶段推进、待办任务、数据同步等。

## 技术栈

- Java 21 + Spring Boot 3.3
- PostgreSQL + Flyway
- Spring Data JPA

## 核心概念

### Matter（事项）

法律服务的核心实体，代表一个案件或法律事务。

```java
public class Matter {
    private UUID id;
    private UUID organizationId;
    private UUID clientId;
    private UUID lawyerId;
    private String serviceTypeId;
    private String playbookId;
    private MatterStatus status;
    private String currentPhase;
    private JsonNode profile;      // 案件画像
    private JsonNode data;         // 业务数据
    private Instant createdAt;
    private Instant updatedAt;
}
```

### MatterTodo（待办任务）

事项中需要人工处理的任务节点。

```java
public class MatterTodo {
    private UUID id;
    private UUID matterId;
    private String todoKey;        // 语义键（幂等）
    private String skillId;
    private TodoStatus status;     // pending/completed/cancelled
    private String actor;          // client/lawyer
    private String reviewType;     // clarify/confirm/select
    private JsonNode payload;      // 卡片数据
    private JsonNode result;       // 完成结果
}
```

## 核心功能

### 1. 事项管理

```
POST   /api/v1/matters              # 创建事项
GET    /api/v1/matters/{id}         # 获取事项详情
PUT    /api/v1/matters/{id}         # 更新事项
DELETE /api/v1/matters/{id}         # 删除事项
GET    /api/v1/matters              # 列表查询
```

### 2. 待办任务

```
GET    /api/v1/matters/{id}/todos           # 获取待办列表
POST   /api/v1/matters/{id}/todos/{todoId}/complete  # 完成待办
```

### 3. 状态同步（内部接口）

```
POST   /internal/matters/{id}/sync          # AI Engine 同步状态
POST   /internal/matters/{id}/todos/upsert  # 创建/更新待办
```

## 状态同步机制

### 同步流程

```
AI Engine
    │
    │ POST /internal/matters/{id}/sync
    │ {
    │   "profile": { ... },
    │   "data": { ... },
    │   "card": { ... }
    │ }
    │
    ▼
┌─────────────────────────────────────┐
│         Matter Service              │
│                                     │
│  1. 合并 profile（深度合并）         │
│  2. 合并 data（按路径合并）          │
│  3. 处理 card（upsert todo）        │
│  4. 检查阶段完成条件                 │
│  5. 更新 current_phase              │
│                                     │
└─────────────────────────────────────┘
```

### Profile 合并策略

```java
public JsonNode mergeProfile(JsonNode existing, JsonNode patch) {
    // 深度合并，patch 中的非空值覆盖 existing
    ObjectNode result = existing.deepCopy();
    patch.fields().forEachRemaining(entry -> {
        if (!entry.getValue().isNull()) {
            result.set(entry.getKey(), entry.getValue());
        }
    });
    return result;
}
```

### Data 合并策略

```java
public JsonNode mergeData(JsonNode existing, JsonNode patch) {
    // 按 group.field 路径合并
    // data.evidence.list -> 合并到 existing.evidence.list
    ObjectNode result = existing.deepCopy();
    patch.fields().forEachRemaining(group -> {
        ObjectNode groupNode = (ObjectNode) result.get(group.getKey());
        if (groupNode == null) {
            groupNode = objectMapper.createObjectNode();
            result.set(group.getKey(), groupNode);
        }
        group.getValue().fields().forEachRemaining(field -> {
            groupNode.set(field.getKey(), field.getValue());
        });
    });
    return result;
}
```

## 待办任务处理

### Todo 生命周期

```
┌─────────┐     complete()     ┌───────────┐
│ pending │ ─────────────────▶ │ completed │
└────┬────┘                    └───────────┘
     │
     │ cancel()
     ▼
┌───────────┐
│ cancelled │
└───────────┘
```

### 完成待办

```java
public void completeTodo(UUID matterId, UUID todoId, JsonNode result) {
    MatterTodo todo = todoRepository.findById(todoId);

    // 1. 规范化结果（根据 todoKey 处理特定字段）
    NormalizedCompletion normalized = normalizeCompletion(todo, result);

    // 2. 更新 todo 状态
    todo.setStatus(TodoStatus.COMPLETED);
    todo.setResult(normalized.result());
    todo.setCompletedAt(Instant.now());

    // 3. 回写 Matter（如案由确认需要更新 profile.cause_of_action_code）
    if (normalized.matterUpdates() != null) {
        matter.setProfile(mergeProfile(matter.getProfile(), normalized.matterUpdates()));
    }

    // 4. 保存
    todoRepository.save(todo);
    matterRepository.save(matter);
}
```

### 语义 Todo Key

| todo_key | 说明 | 完成时处理 |
|----------|------|-----------|
| confirm_claim_path | 案由确认 | 更新 profile.cause_of_action_code |
| confirm_strategy | 策略确认 | 更新 profile.decisions.selected_strategy_id |
| confirm_documents | 文书确认 | 更新 profile.decisions.selected_documents |
| clarify_facts | 事实补充 | 无特殊处理 |

## 阶段管理

### 阶段检查

```java
public boolean isPhaseComplete(Matter matter, PlaybookPhase phase) {
    // 检查 gate_field 和 gate_check
    String gateField = phase.getGateField();
    String gateCheck = phase.getGateCheck();

    JsonNode value = getFieldValue(matter, gateField);

    return switch (gateCheck) {
        case "not_empty" -> value != null && !value.isNull();
        case "equals:true" -> value != null && value.asBoolean();
        case "equals:completed" -> "completed".equals(value.asText());
        default -> false;
    };
}
```

### 阶段流转

```java
public void advancePhase(Matter matter) {
    Playbook playbook = playbookRegistry.get(matter.getPlaybookId());
    String currentPhase = matter.getCurrentPhase();

    // 找到下一个阶段
    List<PlaybookPhase> phases = playbook.getPhases();
    int currentIndex = findPhaseIndex(phases, currentPhase);

    if (currentIndex < phases.size() - 1) {
        PlaybookPhase nextPhase = phases.get(currentIndex + 1);
        matter.setCurrentPhase(nextPhase.getId());
    }
}
```

## 数据模型

### ER 图

```
┌─────────────────┐       ┌─────────────────┐
│     Matter      │       │   MatterTodo    │
├─────────────────┤       ├─────────────────┤
│ id              │───┐   │ id              │
│ organization_id │   │   │ matter_id       │───┐
│ client_id       │   │   │ todo_key        │   │
│ lawyer_id       │   └──▶│ skill_id        │   │
│ service_type_id │       │ status          │   │
│ playbook_id     │       │ actor           │   │
│ status          │       │ review_type     │   │
│ current_phase   │       │ payload         │   │
│ profile (JSONB) │       │ result          │   │
│ data (JSONB)    │       │ created_at      │   │
│ created_at      │       │ completed_at    │   │
│ updated_at      │       └─────────────────┘   │
└─────────────────┘                             │
        │                                       │
        └───────────────────────────────────────┘
```

## 目录结构

```
matter-service/
├── src/main/java/com/lawseekdog/matter/
│   ├── api/
│   │   ├── controller/
│   │   │   ├── MatterController.java
│   │   │   └── MatterTodoController.java
│   │   └── dto/
│   ├── application/
│   │   └── service/
│   │       ├── MatterService.java
│   │       ├── MatterTodoService.java
│   │       └── MatterSyncService.java
│   ├── domain/
│   │   ├── entity/
│   │   │   ├── Matter.java
│   │   │   └── MatterTodo.java
│   │   └── repository/
│   └── infrastructure/
│       └── persistence/
└── src/main/resources/
    └── db/migration/
```
