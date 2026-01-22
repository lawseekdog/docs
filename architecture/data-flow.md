# 数据流架构

## 核心数据流

### 1. 咨询对话流程

```
用户消息
    │
    ▼
┌─────────────────┐
│ Consultations   │ ─── 创建/更新 Session
│ Service         │ ─── 存储 Message
└────────┬────────┘
         │
         ▼ HTTP POST /chat
┌─────────────────┐
│   AI Engine     │
│                 │
│ ┌─────────────┐ │
│ │  Planner    │ │ ─── 决策下一步技能
│ └──────┬──────┘ │
│        │        │
│ ┌──────▼──────┐ │
│ │ Run Skill   │ │ ─── 执行技能
│ └──────┬──────┘ │
│        │        │
│ ┌──────▼──────┐ │
│ │ Sync Data   │ │ ─── 同步状态到 Matter
│ └─────────────┘ │
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│ 返回结果        │
│ - response      │ ─── AI 回复文本
│ - card          │ ─── 交互卡片（可选）
│ - state_patch   │ ─── 状态更新
└─────────────────┘
```

### 2. 知识检索流程

```
用户查询
    │
    ▼
┌─────────────────┐
│   AI Engine     │
│ (Skill: search) │
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│ Knowledge Svc   │
│                 │
│ 1. Query Rewrite│ ─── 查询改写
│ 2. Hybrid Search│ ─── 混合检索
│    ├─ ES BM25   │     ├─ 关键词匹配
│    └─ Weaviate  │     └─ 语义向量
│ 3. Rerank       │ ─── 重排序
│ 4. Return Top-K │
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│ 检索结果        │
│ - laws[]        │ ─── 法规条文
│ - cases[]       │ ─── 相关案例
│ - elements[]    │ ─── 要素匹配
└─────────────────┘
```

### 3. 记忆提取与召回

```
对话上下文
    │
    ▼
┌─────────────────┐
│ Memory Service  │
│                 │
│ ┌─────────────┐ │
│ │ Fact Extract│ │ ─── 从对话提取事实
│ └──────┬──────┘ │
│        │        │
│ ┌──────▼──────┐ │
│ │ Embedding   │ │ ─── 向量化
│ └──────┬──────┘ │
│        │        │
│ ┌──────▼──────┐ │
│ │ Store Qdrant│ │ ─── 存储到向量库
│ └─────────────┘ │
└─────────────────┘

后续对话
    │
    ▼
┌─────────────────┐
│ Memory Service  │
│                 │
│ 1. Query Embed  │ ─── 查询向量化
│ 2. Vector Search│ ─── 相似度检索
│ 3. Return Facts │ ─── 返回相关事实
└─────────────────┘
```

### 4. 事项状态同步

```
AI Engine 技能执行完成
    │
    ▼
┌─────────────────┐
│ skill_output    │
│ - profile       │ ─── 案件画像更新
│ - data          │ ─── 业务数据更新
│ - card          │ ─── 待办卡片
└────────┬────────┘
         │
         ▼ HTTP POST /internal/matters/{id}/sync
┌─────────────────┐
│ Matter Service  │
│                 │
│ 1. Merge Profile│ ─── 合并画像字段
│ 2. Merge Data   │ ─── 合并业务数据
│ 3. Upsert Todo  │ ─── 创建/更新待办
│ 4. Update Phase │ ─── 更新阶段状态
└─────────────────┘
```

## 状态存储

### Session State (Consultations Service)

```json
{
  "session_id": "uuid",
  "matter_id": "uuid",
  "user_id": "uuid",
  "engagement_mode": "legal_consultation | start_service",
  "service_type_id": "litigation_civil_prosecution",
  "playbook_id": "pb_xxx",
  "current_task_id": "intake",
  "profile": {
    "client_role": "plaintiff",
    "summary": "...",
    "plaintiff": { "name": "..." },
    "defendant": { "name": "..." }
  },
  "data": {
    "litigation": { "issues": [...] },
    "evidence": { "list": [...] }
  },
  "messages": [...]
}
```

### Matter State (Matter Service)

```json
{
  "matter_id": "uuid",
  "organization_id": "uuid",
  "client_id": "uuid",
  "lawyer_id": "uuid",
  "service_type_id": "litigation_civil_prosecution",
  "playbook_id": "pb_xxx",
  "status": "active",
  "current_phase": "intake",
  "profile": { ... },
  "data": { ... },
  "todos": [
    {
      "id": "uuid",
      "todo_key": "confirm_claim_path",
      "status": "pending",
      "actor": "lawyer",
      "payload": { ... }
    }
  ]
}
```

## 数据一致性

### 最终一致性模型

- AI Engine 是状态的"生产者"
- Matter Service 是状态的"持久化层"
- 通过 `sync_data` 节点实现异步同步
- 使用 `todo_key` 实现幂等更新

### 冲突解决

1. **Profile 合并**：深度合并，新值覆盖旧值
2. **Data 合并**：按 group.field 路径合并
3. **Todo 更新**：基于 `todo_key` 幂等 upsert

## 缓存策略

| 数据类型 | 缓存位置 | TTL | 失效策略 |
|----------|----------|-----|----------|
| 用户信息 | Redis | 5min | 更新时失效 |
| 会话状态 | Redis | 30min | 活动时续期 |
| 知识检索 | Redis | 1h | LRU |
| Playbook | 内存 | 启动加载 | 重启刷新 |
