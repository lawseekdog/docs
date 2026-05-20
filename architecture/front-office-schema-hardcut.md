---
title: 前台业务数据结构硬切方案
parent: 架构
nav_order: 8
---

# 前台业务数据结构硬切方案

本文是面向当前前端原型的目标表结构与代码改造清单，不表示当前实现已经完成。
本轮范围只梳理数据结构和代码边界，不定义新接口，不保留旧接口兼容层。

## 1. 当前前端反推的数据对象

当前 `frontend` 的前台页面主要覆盖四类业务对象：

| 前端区域 | 页面组件 | 主对象 | 真相源 |
| --- | --- | --- | --- |
| 今日 | `src/features/dashboard/Dashboard.tsx` | 今日待处理项投影 | 聚合投影，不拥有真相 |
| 任务白板 | `src/features/agenda/AgendaBoard.tsx` | 事项任务 | `matter-service` |
| 日历/临近期限 | `src/features/agenda/Schedule.tsx` | 人工日程、庭审、法律期限 | 人工日程在 `matter-service`，庭审/期限在 `case-service` |
| 客户与线索 | `src/features/clients/ClientsLeads.tsx` | 客户、线索、冲突检查、线索 AI 初筛 | `firm-service` |

核心业务流：

```text
线索 Intake
  -> 线索 AI 初筛 / 缺失材料识别
  -> 利益冲突检查
  -> 转客户 Client
  -> 建委托 Engagement
  -> 建服务请求 ServiceRequest
  -> 建事项 Matter
  -> 如进入诉讼/仲裁，再建案件 Case 与程序 Proceeding
  -> Matter 生成任务/日程/AI候选
  -> Case 承接案件事实、争点、证据链、期限、庭审
  -> 今日页只聚合投影
```

## 2. 服务边界

| 服务 | 保留真相 | 删除或迁出 |
| --- | --- | --- |
| `firm-service` | 客户、线索、委托、服务请求、冲突检查、签约、应收、律所商业 SLA | 案件事实、程序事实、事项任务、AI runtime |
| `matter-service` | 事项工作台、任务、人工日程、材料范围、文书工作单元、AI 候选确认流 | 客户主账、线索主账、案件主账、诉请/争点/证据链真相 |
| `case-service` | 案件、程序、一审/二审/执行、当事人、事实、案由、诉请、争点、证据链、期限、庭审、裁判结果 | 事项任务、客户线索、AI runtime |
| `ai-engine-v2` | run、cards、products、candidate outbox、runtime snapshot | 客户、案件、事项业务真相 |

跨服务引用只存外部 id，不建跨库外键。跨服务候选必须带：

```sql
source_owner varchar(64) not null,
source_type varchar(64) not null,
source_id varchar(128) not null,
source_revision varchar(128)
```

## 3. `firm-service` 目标表

`firm-service` 面向客户经营与签约前流程，承接“客户与线索”页面。

### 3.1 保留并收敛

| 目标表 | 来源/变化 | 用途 |
| --- | --- | --- |
| `legal_subjects` | 保留 | 法律主体主账 |
| `firm_clients` | 由 `clients` 改名 | 客户主账 |
| `firm_client_contacts` | 由 `client_contacts` 改名 | 客户联系人 |
| `firm_client_team_members` | 由 `client_team_members` 改名 | 客户负责律师 |
| `firm_intakes` | 保留并加 `intake_no` | 线索主账 |
| `firm_intake_files` | 保留 | 线索材料 |
| `firm_intake_subject_links` | 保留 | 线索相关主体 |
| `firm_intake_ai_reviews` | 新增 | AI 初筛结果 |
| `firm_intake_material_requests` | 新增 | 缺失材料清单 |
| `firm_conflict_index` | 保留 | 利益冲突索引 |
| `firm_conflict_checks` | 保留，`intake_id` 改 bigint | 利益冲突检查记录 |
| `firm_engagements` | 由 `client_engagements` 改名 | 委托关系 |
| `firm_service_requests` | 由 `client_service_requests` 改名 | 服务请求 |
| `fee_agreements` | 保留 | 收费协议 |
| `payment_schedules` | 保留 | 收款计划 |
| `receivable_ledger` | 保留 | 应收流水 |
| `service_sla_ledger` | 保留 | 服务 SLA 待办 |
| `firm_communication_threads` | 新增 | 今日页高优先级通讯线程 |
| `firm_communication_messages` | 新增 | 通讯消息摘要和外部消息引用 |
| `firm_pending_ledger_candidates` | 保留 | 只承接 firm owner 的候选 |
| `idempotency_records` | 保留 | 幂等 |

### 3.2 删除或暂不建

| 旧表 | 动作 | 原因 |
| --- | --- | --- |
| `firm_client_profile_snapshot` | 删除 | 派生快照，不是主账；客户列表可从主表和聚合查询生成 |
| `firm_client_preferences` | 删除 | 当前前端没有稳定业务闭环 |
| `firm_service_catalog` | 删除 | 先不做服务目录配置面 |
| `firm_sla_policies` | 删除 | 先保留 SLA ledger，不保留策略配置表 |
| `client_invoices` | 删除 | 当前只需要应收/收款计划，不需要发票主账 |
| `client_payments` | 删除 | 当前只需要应收流水，不需要付款明细主账 |

### 3.3 DDL 草案

```sql
create table legal_subjects (
    id bigserial primary key,
    organization_id varchar(64) not null,
    subject_type varchar(32) not null,
    display_name varchar(255) not null,
    identity_no varchar(128),
    unified_social_credit_code varchar(128),
    contact_json jsonb not null default '{}'::jsonb,
    status varchar(32) not null default 'active',
    created_at timestamptz not null,
    updated_at timestamptz not null,
    created_by bigint,
    updated_by bigint,
    constraint ck_legal_subjects_type check (subject_type in ('natural_person', 'organization'))
);

create table firm_clients (
    id bigserial primary key,
    organization_id varchar(64) not null,
    client_no varchar(64) not null,
    legal_subject_id bigint not null references legal_subjects(id),
    display_name varchar(255) not null,
    client_type varchar(32) not null,
    client_status varchar(32) not null,
    risk_level varchar(16) not null default 'normal',
    industry_code varchar(64),
    owner_user_id bigint,
    last_contact_at timestamptz,
    created_at timestamptz not null,
    updated_at timestamptz not null,
    created_by bigint,
    updated_by bigint,
    constraint ux_firm_clients_org_no unique (organization_id, client_no),
    constraint ck_firm_clients_status check (client_status in ('active', 'warning', 'new', 'inactive', 'lost'))
);

create table firm_intakes (
    id bigserial primary key,
    organization_id varchar(64) not null,
    intake_no varchar(64) not null,
    source varchar(32) not null,
    visitor_name varchar(128),
    contact_method varchar(128),
    opposing_subject_name varchar(255),
    requested_program_key varchar(128),
    summary text not null,
    urgency varchar(32) not null,
    intake_status varchar(32) not null,
    conflict_status varchar(32) not null,
    decision varchar(64),
    decision_reason text,
    converted_client_id bigint references firm_clients(id),
    converted_service_request_id bigint,
    converted_matter_id bigint,
    converted_case_id bigint,
    created_at timestamptz not null,
    updated_at timestamptz not null,
    created_by bigint,
    updated_by bigint,
    constraint ux_firm_intakes_org_no unique (organization_id, intake_no),
    constraint ck_firm_intakes_status check (intake_status in ('new', 'ai_reviewing', 'pending_human_review', 'conflict_checking', 'accepted', 'declined', 'converted'))
);

create table firm_intake_ai_reviews (
    id bigserial primary key,
    organization_id varchar(64) not null,
    intake_id bigint not null references firm_intakes(id) on delete cascade,
    review_status varchar(32) not null,
    cause_labels_json jsonb not null default '[]'::jsonb,
    fact_confidence numeric(5,2),
    risk_flags_json jsonb not null default '[]'::jsonb,
    material_gap_count integer not null default 0,
    recommended_action varchar(64),
    model_ref varchar(128),
    run_id varchar(128),
    reviewed_at timestamptz not null,
    created_at timestamptz not null,
    updated_at timestamptz not null,
    constraint ck_firm_intake_ai_reviews_status check (review_status in ('running', 'ready_for_review', 'failed', 'approved'))
);

create table firm_intake_material_requests (
    id bigserial primary key,
    organization_id varchar(64) not null,
    intake_id bigint not null references firm_intakes(id) on delete cascade,
    material_key varchar(128) not null,
    title varchar(255) not null,
    reason text,
    status varchar(32) not null,
    source_review_id bigint references firm_intake_ai_reviews(id) on delete set null,
    created_at timestamptz not null,
    updated_at timestamptz not null,
    constraint ux_firm_intake_material_requests_key unique (intake_id, material_key)
);

create table firm_engagements (
    id bigserial primary key,
    organization_id varchar(64) not null,
    client_id bigint not null references firm_clients(id) on delete cascade,
    engagement_type varchar(64) not null,
    engagement_status varchar(32) not null,
    service_scope_json jsonb not null default '{}'::jsonb,
    started_at timestamptz,
    ended_at timestamptz,
    created_at timestamptz not null,
    updated_at timestamptz not null,
    created_by bigint,
    updated_by bigint
);

create table firm_service_requests (
    id bigserial primary key,
    organization_id varchar(64) not null,
    request_no varchar(64) not null,
    client_id bigint not null references firm_clients(id) on delete cascade,
    engagement_id bigint references firm_engagements(id) on delete set null,
    source_intake_id bigint references firm_intakes(id) on delete set null,
    request_type varchar(64) not null,
    request_channel varchar(32) not null,
    requester_name varchar(128),
    request_summary text not null,
    request_payload_json jsonb not null default '{}'::jsonb,
    request_status varchar(32) not null,
    sla_due_at timestamptz,
    created_matter_id bigint,
    converted_case_id bigint,
    created_at timestamptz not null,
    updated_at timestamptz not null,
    created_by bigint,
    updated_by bigint,
    constraint ux_firm_service_requests_org_no unique (organization_id, request_no)
);

create table firm_communication_threads (
    id bigserial primary key,
    organization_id varchar(64) not null,
    thread_kind varchar(64) not null,
    title varchar(255) not null,
    client_id bigint references firm_clients(id) on delete set null,
    matter_id bigint,
    case_id bigint,
    proceeding_id bigint,
    external_channel varchar(64),
    external_thread_ref varchar(255),
    participants_json jsonb not null default '[]'::jsonb,
    priority varchar(16) not null,
    unread_count integer not null default 0,
    last_message_at timestamptz,
    status varchar(32) not null,
    created_at timestamptz not null,
    updated_at timestamptz not null,
    created_by bigint,
    updated_by bigint,
    constraint ck_firm_communication_threads_kind check (thread_kind in ('client', 'team', 'opposing_counsel', 'system')),
    constraint ck_firm_communication_threads_priority check (priority in ('low', 'medium', 'high')),
    constraint ck_firm_communication_threads_status check (status in ('open', 'muted', 'archived'))
);

create index idx_firm_communication_threads_today
    on firm_communication_threads (organization_id, status, priority, last_message_at desc, id desc);

create table firm_communication_messages (
    id bigserial primary key,
    organization_id varchar(64) not null,
    thread_id bigint not null references firm_communication_threads(id) on delete cascade,
    sender_display_name varchar(128) not null,
    sender_role varchar(64) not null,
    body_preview text not null,
    body_ref varchar(255),
    importance varchar(16) not null,
    sent_at timestamptz not null,
    source_owner varchar(64) not null,
    source_type varchar(64) not null,
    source_id varchar(128) not null,
    source_revision varchar(128),
    created_at timestamptz not null,
    constraint ck_firm_communication_messages_importance check (importance in ('low', 'medium', 'high'))
);
```

## 4. `matter-service` 目标表

`matter-service` 面向“办案工作台”和“任务/日程”，不再拥有案件事实。

### 4.1 保留并收敛

| 目标表 | 来源/变化 | 用途 |
| --- | --- | --- |
| `matters` | 保留，放宽 `case_id/proceeding_id` 可选关系 | 事项工作台主账 |
| `service_items` | 保留 | 服务型事项扩展 |
| `matter_team_members` | 保留 | 事项成员 |
| `matter_documents` | 保留 | 事项文书工作单元 |
| `matter_material_packages` | 保留 | 材料包版本 |
| `matter_source_scopes` | 保留 | 本次工作材料范围 |
| `matter_work_orders` | 保留 | 本次工作目标 |
| `matter_tasks` | 重做字段 | 任务白板主账 |
| `matter_task_questions` | 保留 | 卡片/任务问题 |
| `matter_task_question_options` | 保留 | 问题选项 |
| `matter_task_answer_items` | 保留 | 任务答案 |
| `matter_schedule_events` | 新增 | 人工日程、客户会议、团队会议、文书签署 |
| `matter_candidate_inbox` | 保留 | AI 候选确认队列 |
| `matter_activity_log` | 保留 | 工作台时间线 |
| `matter_lawyer_feedback` | 保留 | 律师反馈 |
| `matter_lawyer_revision` | 保留 | 律师修订 |
| `matter_run_comparison` | 保留 | run 对比 |
| `matter_material_change_events` | 保留 | 材料变更事件 |
| `matter_route_revision_requests` | 保留 | 路由重算请求 |
| `idempotency_records` | 保留 | 幂等 |

### 4.2 删除或迁出

| 旧表 | 动作 | 新 owner |
| --- | --- | --- |
| `matter_intake_profiles` | 删除 | `firm-service.firm_intakes` / `firm_intake_ai_reviews` |
| `matter_claims` | 删除 | `case-service.proceeding_claim_ledger` |
| `matter_evidence_item` | 删除 | `case-service.case_evidence_items` |
| `matter_evidence_chain` | 删除 | `case-service.proceeding_evidence_chain_ledger` |
| `matter_evidence_spine_projection` | 删除 | 从 `case-service` 读投影 |
| `matter_procedure_contexts` | 删除或收敛为只读外部上下文缓存 | `case-service.case_proceedings` 是真相 |
| `matter_work_plans` | 删除 | 由 `matter_work_orders` 承接 |
| `matter_draft_state` | 删除 | 文书状态归 `matter_documents`/runtime snapshot |

### 4.3 DDL 草案

```sql
create table matters (
    id bigserial primary key,
    organization_id varchar(64) not null,
    title varchar(255) not null,
    status varchar(32) not null,
    workspace_kind varchar(32) not null,
    matter_category varchar(32) not null,
    program_variant_key varchar(128),
    delivery_goal varchar(64),
    entry_mode varchar(32),
    case_id bigint,
    proceeding_id bigint,
    client_id bigint,
    legal_subject_id bigint,
    engagement_id bigint,
    firm_service_request_id bigint,
    current_work_goal text,
    workspace_stage varchar(128),
    active_run_id varchar(128),
    runtime_head_revision bigint not null default 0,
    provider_organization_id bigint,
    owner_user_id bigint,
    created_at timestamptz not null,
    updated_at timestamptz not null,
    created_by bigint,
    updated_by bigint,
    constraint ck_matters_workspace_kind check (workspace_kind in ('case_proceeding', 'service_item', 'document_delivery')),
    constraint ck_matters_case_context check (
        workspace_kind <> 'case_proceeding'
        or (case_id is not null and proceeding_id is not null)
    )
);

create table matter_tasks (
    id bigserial primary key,
    organization_id varchar(64) not null,
    matter_id bigint not null references matters(id) on delete cascade,
    task_no varchar(64) not null,
    task_key varchar(128) not null,
    title varchar(255) not null,
    description text,
    task_kind varchar(64) not null,
    board_status varchar(32) not null,
    priority varchar(16) not null,
    due_at timestamptz,
    actor varchar(32) not null,
    assignee_user_id bigint,
    case_id bigint,
    proceeding_id bigint,
    document_session_id varchar(128),
    source_owner varchar(64) not null,
    source_type varchar(64) not null,
    source_id varchar(128) not null,
    source_revision varchar(128),
    completed_at timestamptz,
    created_at timestamptz not null,
    updated_at timestamptz not null,
    created_by bigint,
    updated_by bigint,
    constraint ux_matter_tasks_matter_key unique (matter_id, task_key),
    constraint ux_matter_tasks_org_no unique (organization_id, task_no),
    constraint ck_matter_tasks_board_status check (board_status in ('todo', 'in_progress', 'review', 'done')),
    constraint ck_matter_tasks_priority check (priority in ('low', 'medium', 'high')),
    constraint ck_matter_tasks_actor check (actor in ('client', 'lawyer', 'system'))
);

create index idx_matter_tasks_board
    on matter_tasks (organization_id, board_status, priority, due_at, id);

create table matter_schedule_events (
    id bigserial primary key,
    organization_id varchar(64) not null,
    matter_id bigint references matters(id) on delete set null,
    case_id bigint,
    proceeding_id bigint,
    client_id bigint,
    event_kind varchar(64) not null,
    title varchar(255) not null,
    starts_at timestamptz not null,
    ends_at timestamptz,
    location varchar(255),
    notes text,
    status varchar(32) not null,
    source_owner varchar(64) not null,
    source_type varchar(64) not null,
    source_id varchar(128) not null,
    source_revision varchar(128),
    created_at timestamptz not null,
    updated_at timestamptz not null,
    created_by bigint,
    updated_by bigint,
    constraint ck_matter_schedule_events_kind check (event_kind in ('client_meeting', 'team_meeting', 'document_signing', 'manual')),
    constraint ck_matter_schedule_events_status check (status in ('scheduled', 'cancelled', 'done'))
);

create index idx_matter_schedule_events_calendar
    on matter_schedule_events (organization_id, status, starts_at, id);
```

## 5. `case-service` 目标表

`case-service` 是案件和程序事实 owner，承接“案件跑到哪个阶段、一审二审执行、庭审、期限、证据链”。

### 5.1 保留并强化

| 表 | 动作 | 用途 |
| --- | --- | --- |
| `cases` | 保留，加 `current_proceeding_id` | 案件档案 |
| `case_parties` | 保留 | 案件当事人 |
| `case_proceedings` | 保留 | 一审/二审/再审/执行/仲裁程序 |
| `case_proceeding_matters` | 保留 | case/proceeding 与 matter 的关系 |
| `proceeding_party_roles` | 保留 | 程序当事人角色 |
| `case_fact_ledger` | 保留 | 案件事实 |
| `case_cause_ledger` | 保留 | 案由/法律关系判断 |
| `case_evidence_items` | 保留 | 证据主账 |
| `proceeding_evidence_submissions` | 保留 | 程序证据提交 |
| `proceeding_evidence_admissions` | 保留 | 程序证据采信 |
| `proceeding_evidence_chain_ledger` | 保留 | 证据链 |
| `proceeding_issue_ledger` | 保留 | 争点 |
| `proceeding_claim_ledger` | 保留 | 诉请 |
| `proceeding_decision_ledger` | 保留 | 裁判/程序决定 |
| `proceeding_filing_ledger` | 新增 | 法院/仲裁机构提交、审核、受理结果 |
| `proceeding_deadline_ledger` | 保留，加 `case_id` | 法律期限 |
| `proceeding_hearing_ledger` | 保留，加 `case_id` | 庭审 |
| `proceeding_outcome_ledger` | 保留 | 程序结果 |
| `lower_judgment_breakdown_ledger` | 保留 | 下级裁判拆解 |
| `appeal_ground_ledger` | 保留 | 上诉理由 |
| `procedural_decision_ledger` | 保留 | 程序性决策 |
| `issue_continuity_ledger` | 保留 | 跨程序争点延续 |
| `case_intelligence_links` | 保留 | 外部情报链接 |
| `idempotency_records` | 保留 | 幂等 |

### 5.2 DDL 草案

```sql
create table cases (
    id bigserial primary key,
    organization_id varchar(64) not null,
    primary_client_id bigint not null,
    primary_legal_subject_id bigint not null,
    engagement_id bigint,
    case_no varchar(128),
    case_title varchar(255) not null,
    case_status varchar(32) not null,
    current_proceeding_id bigint,
    opened_at timestamptz not null,
    closed_at timestamptz,
    firm_context_snapshot_version varchar(64) not null,
    firm_context_snapshot_json jsonb not null,
    created_at timestamptz not null,
    updated_at timestamptz not null,
    created_by bigint,
    updated_by bigint
);

create table case_proceedings (
    id bigserial primary key,
    organization_id varchar(64) not null,
    case_id bigint not null references cases(id) on delete cascade,
    previous_proceeding_id bigint references case_proceedings(id) on delete restrict,
    active_matter_id bigint,
    instance_level varchar(32) not null,
    proceeding_phase varchar(128) not null,
    proceeding_role varchar(32) not null,
    client_role varchar(32),
    defense_posture varchar(32),
    proceeding_status varchar(32) not null,
    court_name varchar(255),
    tribunal_name varchar(255),
    case_no varchar(128),
    filed_at timestamptz,
    accepted_at timestamptz,
    judgment_at timestamptz,
    effective_at timestamptz,
    closed_at timestamptz,
    source_owner varchar(64) not null,
    source_type varchar(64) not null,
    source_id varchar(128) not null,
    source_revision varchar(128),
    confirmed_by bigint,
    confirmed_at timestamptz,
    created_at timestamptz not null,
    updated_at timestamptz not null,
    created_by bigint,
    updated_by bigint
);

create table proceeding_deadline_ledger (
    id bigserial primary key,
    organization_id varchar(64) not null,
    case_id bigint not null references cases(id) on delete cascade,
    proceeding_id bigint not null references case_proceedings(id) on delete cascade,
    deadline_key varchar(128) not null,
    deadline_kind varchar(64) not null,
    trigger_event varchar(128) not null,
    trigger_event_at timestamptz not null,
    rule_ref varchar(256) not null,
    start_at timestamptz not null,
    due_at timestamptz not null,
    calculation_text text not null,
    status varchar(32) not null,
    source_owner varchar(64) not null,
    source_type varchar(64) not null,
    source_id varchar(128) not null,
    source_revision varchar(128),
    confirmed_by bigint,
    confirmed_at timestamptz,
    created_at timestamptz not null,
    updated_at timestamptz not null,
    constraint ux_proceeding_deadline_ledger_key unique (proceeding_id, deadline_key)
);

create index idx_proceeding_deadline_ledger_today
    on proceeding_deadline_ledger (organization_id, status, due_at, case_id, proceeding_id);

create table proceeding_filing_ledger (
    id bigserial primary key,
    organization_id varchar(64) not null,
    case_id bigint not null references cases(id) on delete cascade,
    proceeding_id bigint not null references case_proceedings(id) on delete cascade,
    filing_key varchar(128) not null,
    filing_kind varchar(64) not null,
    title varchar(255) not null,
    document_ref varchar(128),
    filing_status varchar(32) not null,
    progress_percent integer not null default 0,
    submitted_at timestamptz,
    accepted_at timestamptz,
    rejected_at timestamptz,
    court_name varchar(255),
    status_reason text,
    source_owner varchar(64) not null,
    source_type varchar(64) not null,
    source_id varchar(128) not null,
    source_revision varchar(128),
    confirmed_by bigint,
    confirmed_at timestamptz,
    created_at timestamptz not null,
    updated_at timestamptz not null,
    constraint ux_proceeding_filing_ledger_key unique (proceeding_id, filing_key),
    constraint ck_proceeding_filing_progress check (progress_percent >= 0 and progress_percent <= 100),
    constraint ck_proceeding_filing_status check (filing_status in ('submitted', 'reviewing', 'accepted', 'rejected', 'withdrawn'))
);

create index idx_proceeding_filing_ledger_today
    on proceeding_filing_ledger (organization_id, filing_status, updated_at desc, case_id, proceeding_id);

create table proceeding_hearing_ledger (
    id bigserial primary key,
    organization_id varchar(64) not null,
    case_id bigint not null references cases(id) on delete cascade,
    proceeding_id bigint not null references case_proceedings(id) on delete cascade,
    hearing_key varchar(128) not null,
    scheduled_at timestamptz not null,
    ends_at timestamptz,
    court_room varchar(255),
    judge_name varchar(255),
    opposing_counsel varchar(255),
    session_kind varchar(64) not null,
    status varchar(32) not null,
    status_reason text,
    notes_doc_id varchar(128),
    transcript_doc_id varchar(128),
    source_owner varchar(64) not null,
    source_type varchar(64) not null,
    source_id varchar(128) not null,
    source_revision varchar(128),
    confirmed_by bigint,
    confirmed_at timestamptz,
    created_at timestamptz not null,
    updated_at timestamptz not null,
    constraint ux_proceeding_hearing_ledger_key unique (proceeding_id, hearing_key)
);

create index idx_proceeding_hearing_ledger_today
    on proceeding_hearing_ledger (organization_id, status, scheduled_at, case_id, proceeding_id);
```

## 6. 今日页读模型

今日页不拥有业务真相。可以新增只读投影表，或由 BFF 实时聚合。
若落库，表名使用 `lawyer_today_projection_items`，并明确它是 projection。
该表只解决今日页列表、时间线、关键任务等通用入口；`AI 洞察` 和 `合规监控`
需要更强结构，不应只塞进 `subtitle`：

```sql
create table lawyer_today_projection_items (
    id bigserial primary key,
    organization_id varchar(64) not null,
    user_id bigint not null,
    display_group varchar(64) not null,
    item_kind varchar(64) not null,
    title varchar(255) not null,
    subtitle text,
    priority varchar(16) not null,
    starts_at timestamptz,
    due_at timestamptz,
    status varchar(32) not null,
    source_owner varchar(64) not null,
    source_table varchar(128) not null,
    source_id varchar(128) not null,
    source_revision varchar(128),
    dedupe_key varchar(255) not null,
    payload_json jsonb not null default '{}'::jsonb,
    action_refs_json jsonb not null default '[]'::jsonb,
    route_target jsonb not null default '{}'::jsonb,
    projected_at timestamptz not null,
    expires_at timestamptz,
    constraint ux_lawyer_today_projection_items_dedupe unique (organization_id, user_id, dedupe_key)
);
```

投影来源：

| item_kind | source_owner | 来源 |
| --- | --- | --- |
| `task` | `matter-service` | `matter_tasks` |
| `manual_event` | `matter-service` | `matter_schedule_events` |
| `deadline` | `case-service` | `proceeding_deadline_ledger` |
| `hearing` | `case-service` | `proceeding_hearing_ledger` |
| `filing` | `case-service` | `proceeding_filing_ledger` |
| `document_preparation` | `matter-service` | `matter_documents` |
| `intake` | `firm-service` | `firm_intakes` / `firm_intake_ai_reviews` |
| `receivable` | `firm-service` | `receivable_ledger` / `payment_schedules` |
| `communication` | `firm-service` | `firm_communication_threads` / `firm_communication_messages` |
| `ai_signal` | `ai-engine-v2` | runtime cards/products/candidates |

### 6.1 AI 洞察投影

`AI 洞察` 是 runtime 派生建议，不是案件/事项/客户真相。ai-engine-v2 应产出
结构化 insight candidate/product，再由投影层按当前用户和作用域生成展示项。

```sql
create table lawyer_ai_insight_projection_items (
    id bigserial primary key,
    organization_id varchar(64) not null,
    user_id bigint not null,
    insight_kind varchar(64) not null,
    title varchar(255) not null,
    summary text not null,
    severity varchar(16) not null,
    confidence numeric(5,2),
    scope_owner varchar(64) not null,
    scope_type varchar(64) not null,
    scope_id varchar(128) not null,
    matter_id bigint,
    case_id bigint,
    proceeding_id bigint,
    client_id bigint,
    run_id varchar(128) not null,
    product_type varchar(128) not null,
    product_id varchar(128) not null,
    product_revision varchar(128),
    evidence_refs_json jsonb not null default '[]'::jsonb,
    action_refs_json jsonb not null default '[]'::jsonb,
    quality_json jsonb not null default '{}'::jsonb,
    status varchar(32) not null,
    projected_at timestamptz not null,
    expires_at timestamptz,
    constraint ck_lawyer_ai_insight_kind check (insight_kind in ('precedent_alert', 'optimization_ready', 'material_gap', 'risk_shift', 'deadline_risk', 'quality_warning')),
    constraint ck_lawyer_ai_insight_severity check (severity in ('low', 'medium', 'high')),
    constraint ck_lawyer_ai_insight_status check (status in ('active', 'dismissed', 'converted_to_task', 'expired')),
    constraint ux_lawyer_ai_insight_product unique (organization_id, user_id, product_id, scope_owner, scope_type, scope_id)
);

create index idx_lawyer_ai_insight_today
    on lawyer_ai_insight_projection_items (organization_id, user_id, status, severity, projected_at desc);
```

ai-engine-v2 侧输出最小契约：

```json
{
  "product_type": "lawyer_ai_insight",
  "insight_kind": "precedent_alert",
  "title": "先例预警",
  "summary": "第九巡回法院数字隐私准则出现语义偏差，可能影响当前 2 个案件。",
  "severity": "high",
  "confidence": 0.86,
  "scope": {
    "owner": "case-service",
    "type": "case",
    "ids": ["123", "456"]
  },
  "evidence_refs": [],
  "action_refs": []
}
```

### 6.2 合规监控投影

合规监控不是业务主账。它汇总 `firm_conflict_checks`、平台审计日志、候选确认流和关键操作审计。

```sql
create table lawyer_compliance_projection_items (
    id bigserial primary key,
    organization_id varchar(64) not null,
    user_id bigint not null,
    monitor_kind varchar(64) not null,
    status varchar(32) not null,
    title varchar(255) not null,
    summary text not null,
    source_owner varchar(64) not null,
    source_type varchar(64) not null,
    source_id varchar(128) not null,
    source_revision varchar(128),
    payload_json jsonb not null default '{}'::jsonb,
    projected_at timestamptz not null,
    constraint ck_lawyer_compliance_monitor_kind check (monitor_kind in ('conflict_check', 'audit_recording', 'candidate_review', 'data_access')),
    constraint ck_lawyer_compliance_status check (status in ('clear', 'warning', 'blocked'))
);
```

### 6.3 今日页板块覆盖表

| Dashboard 板块 | 数据是否足够 | 需要的表/产物 |
| --- | --- | --- |
| 今日日程计划 | 足够 | `matter_schedule_events` + `proceeding_hearing_ledger` + `proceeding_deadline_ledger` |
| 高优先级通讯 | 原方案不够，已补 | `firm_communication_threads` + `firm_communication_messages` |
| AI 洞察 | 原方案不够，已补 | ai-engine `lawyer_ai_insight` product + `lawyer_ai_insight_projection_items` |
| 法庭立案状态 | 原方案不够，已补并修正边界 | `proceeding_filing_ledger` 只管已提交后的法院流转；`matter_documents` 只管提交前文书准备 |
| 关键任务 | 足够 | 重做后的 `matter_tasks` |
| 合规监控 | 原方案不够，已补 | `lawyer_compliance_projection_items` + `firm_conflict_checks` + audit source |

### 6.4 法庭立案状态面板的展示规则

当前前端的“法庭立案状态”面板展示的是面向律师的工作进度，不等于单一表。
它应该由今日投影合并两类来源：

| 前端展示状态 | 真相源 | 说明 |
| --- | --- | --- |
| `起草中`、`待审核`、`已定稿` | `matter-service.matter_documents` | 文书生产状态，尚未形成法院提交事件 |
| `已提交`、`审核中`、`已受理`、`被退回`、`已撤回` | `case-service.proceeding_filing_ledger` | 法院/仲裁机构流转状态，属于程序事实 |

因此不能把 `drafting` 放进 `proceeding_filing_ledger`。
`proceeding_filing_ledger` 只在文书被提交到法院/仲裁机构后产生记录。
如果前端仍保留“法庭立案状态”这个标题，投影层可以把 `matter_documents`
里的“拟提交文书准备进度”和 `proceeding_filing_ledger` 里的“已提交后流转进度”
合并成同一个面板，但每一行必须带 `source_owner/source_table/source_id`，不得双写。

## 7. 代码去留清单

### 7.1 `firm-service`

| 代码 | 动作 | 说明 |
| --- | --- | --- |
| `FirmService` | 拆分 | 现在同时管 client、intake、catalog、conflict、conversion，后续拆成 `ClientLedgerService`、`IntakeLedgerService`、`ConflictCheckService`、`EngagementService` |
| `FirmCommercialLedgerService` | 收缩保留 | 保留 `fee_agreements`、`payment_schedules`、`receivable_ledger`、`service_sla_ledger`；删除 invoice/payment 细表逻辑 |
| `FirmClientProfileService` | 删除 | 依赖 `firm_client_profile_snapshot`，该表不是主账 |
| `FirmPendingLedgerCandidateService` | 保留并收窄 | 只处理 firm owner 候选，不接 case/matter truth |
| `FirmDtos` | 后续重写 DTO | 当前 DTO 含 catalog/preference/profile 等待删除概念 |
| `FirmCommercialDtos` | 后续收缩 | 删除 invoice/payment DTO，保留签约、收款计划、应收、SLA 今日项 |
| `LawyerFirmClientController` | 本阶段不动 | 接口后续按新表重开 |
| `LawyerFirmCommercialController` | 本阶段不动 | 接口后续按新表重开 |

### 7.2 `matter-service`

| 代码 | 动作 | 说明 |
| --- | --- | --- |
| `MatterTaskEntity` | 重写字段 | 增加 `task_no/board_status/priority/due_at/source_*`，删除 `organization_ref_id` 命名 |
| `MatterTask` | 重写 record | 与新 `matter_tasks` 对齐 |
| `JpaMatterTaskRepository` | 重写查询 | 从 inbox 查询改成 board/due/assignee 查询 |
| `MatterTaskService` | 保留并拆掉 profile 写入 | 不再通过 task 更新 `matter_intake_profiles` |
| `MatterIntakeProfileStore` | 删除 | intake truth 迁到 `firm-service` |
| `MatterIntakeProfileEntity` / repository | 删除 | 对应表删除 |
| `MatterEvidenceService` | 删除或迁到 case 调用层 | evidence truth 迁到 `case-service` |
| `MatterEvidenceItemEntity` / repository | 删除 | 对应表删除 |
| `MatterEvidenceBindingEntity` / repository | 删除 | 对应表删除 |
| `MatterClaimEntity` / repository | 删除 | claim truth 迁到 `case-service` |
| `MatterContextBundleService` | 改造 | 删除对 `matter_intake_profiles` 的读取，改由 firm work context + case context 组装 |
| `PendingLedgerCandidateService` | 保留 | 作为 AI candidate inbox 控制面 |
| `CandidateOwnerRouter` | 保留 | 继续把 case/firm/matter owner 分清楚 |
| `MatterCaseTruthLedgerService` | 收窄 | 只记录候选/owner 交接，不拥有 case truth |
| `MatterMaterialRevisionService` | 保留 | 材料变化仍属于事项范围 |
| `MatterDocumentService` | 保留 | 文书工作单元仍属于事项范围 |
| `MatterScheduleEventEntity` | 新增 | 对应 `matter_schedule_events` |
| `MatterScheduleEventRepository` | 新增 | 支撑人工日程 |

### 7.3 `case-service`

| 代码 | 动作 | 说明 |
| --- | --- | --- |
| `LegalCaseEntity` | 更新 | 对齐 `cases` 新字段 |
| `CasePartyEntity` | 更新 | 保持案件当事人主账 |
| `CaseWorkspaceService` | 保留 | 作为 case 总览读模型，不承接 matter task |
| `CaseProceedingService` | 保留 | 程序主账写入入口 |
| `CaseTruthCandidateService` | 保留 | case owner 候选提交主控面 |
| `CaseProceedingDtos` | 后续更新 | 增加 `case_id` deadline/hearing 聚合需要的字段 |
| `CaseTruthCandidateDtos` | 后续更新 | 对齐 `source_owner/source_revision` |

## 8. 实施顺序

1. 先重写三服务 `V1__init.sql`，不叠 `V2 alter`。
2. 先改 `matter-service` 的 JPA 强绑定对象，因为 validate 最容易先爆。
3. 再改 `firm-service` 的大 SQL service，把旧表名一次性换掉。
4. 再改 `case-service` 的 SQL ledger，把 `case_id/source_owner/source_revision` 补齐。
5. 最后重开接口和前端 API，不从旧 controller 上加兼容字段。

## 9. 禁止事项

- 不建新旧双表。
- 不保留旧字段别名。
- 不在接口层拼装 fallback。
- 不让 `matter-service` 继续拥有 case truth。
- 不让 `firm-service` 写 matter/case 业务状态，只写外部 id 和 conversion result。
- 不让今日页落业务真相。
