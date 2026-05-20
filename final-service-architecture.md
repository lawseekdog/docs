# LawSeekDog Final Service Architecture

## Service Boundaries

`matter-service` owns lawyer work matter truth, not case truth:

- matters, service items, matter files, team members and matter tasks
- matter-owned evidence/fact/claim/issue/decision/deadline/hearing/risk/economic ledgers
- matter documents, workbench/today projections, activity logs and draft state
- external references `case_id` and `proceeding_id` only; no case-table foreign keys

`case-service` owns case truth:

- cases and case parties
- case proceedings and proceeding party roles
- proceeding outcome, lower judgment breakdown, appeal ground, procedural decision and issue continuity ledgers
- case intelligence links
- internal proceeding context and stage-gate facts used by matter-service

`assistant-service` owns XiaoJian user-facing state:

- assistant sessions, messages, attachments and scope links
- action cards, action executions, stream events and feedback

It never stores prompt text and never calls model providers directly. It calls `ai-engine-v2`.

`document-workspace-service` owns lawyer drafting workspace state:

- document workspace sessions, versions, annotations, revision ops
- exports, signoffs and workspace events

It never owns template catalog, binary files or matter/case truth.

`ai-engine-v2` owns AI execution:

- prompt registry
- run orchestration
- model gateway
- tool runtime
- traces, costs and structured outputs

`firm-service`, lawyer profile and organization/user services remain separate upstream truth sources. Their tables are not folded into matter-service or case-service.

## Table Ownership Hard Cut

All four Java services use one PostgreSQL Flyway initializer per service. MySQL migrations and MySQL profiles are removed from matter-service.

`matter-service`: `matter-service/src/main/resources/db/migration-postgresql/V1__init.sql`

- table count: 43
- owns matter/workbench/task/ledger truth
- does not create `cases`, `case_parties`, `case_proceedings`, proceeding ledgers or `case_intelligence_links`
- rejects case-owned pending-ledger targets with `case_pending_candidate_moved_to_case_service`

`case-service`: `case-service/src/main/resources/db/migration-postgresql/V1__init.sql`

- table count: 10
- owns `cases`, `case_parties`, `case_proceedings`, `proceeding_party_roles`,
  `proceeding_outcome_ledger`, `lower_judgment_breakdown_ledger`,
  `appeal_ground_ledger`, `procedural_decision_ledger`,
  `issue_continuity_ledger`, `case_intelligence_links`

`assistant-service`: `assistant-service/src/main/resources/db/migration-postgresql/V1__init.sql`

- table count: 8
- replaces old XiaoJian/matter runtime chat tables with assistant-owned tables

`document-workspace-service`: `document-workspace-service/src/main/resources/db/migration-postgresql/V1__init.sql`

- table count: 7
- replaces old matter document session, annotation, revision-op and signoff tables

Deleted from matter-service without replacement there:

- `matter_runtime_materializations`
- `matter_claim_specifications`
- `matter_product_supersession_log`
- document draft workspace tables
- matter AI run index tables
- old XiaoJian/runtime thread tables
- canvas ownership

## AI Product Sync

AI products split by target truth source:

- matter-owned candidate products sync to `matter-service`
- case/proceeding candidate products sync to `case-service`
- firm-owned candidate products sync to `firm-service`
- XiaoJian prompt execution and model calls stay in `ai-engine-v2`; `assistant-service` stores conversation state only

There is no double-write path. `matter-service` no longer writes case/proceeding ledgers, and it no longer reads case tables locally.

## Relationships

`matter-service` stores external references to case truth:

- `matters.case_id`
- `matters.proceeding_id`
- `service_items.source_case_id`
- `service_items.converted_case_id`
- `pending_ledger_candidates.case_id`
- `pending_ledger_candidates.proceeding_id`

These are identifiers across service boundaries, not local foreign keys.

`case-service` stores `case_proceedings.active_matter_id` as the active matter reference for our-firm representation. It is not a local matter foreign key.

## Call Flow

```text
front
  -> case-service
    -> case/proceeding truth

front
  -> matter-service
    -> matter/workbench/task truth
    -> case-service internal APIs for proceeding context and stage-gate facts

front
  -> assistant-service
    -> ai-engine-v2
      -> model provider

front
  -> document-workspace-service
    -> templates-service
    -> files-service
```
