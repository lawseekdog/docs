# Controller Boundary Replan

This is the hard-cut replacement plan for the removed controller layer in
`document-workspace-service`, `matter-service`, and `case-service`.

## Rules

- No legacy routes are retained.
- No compatibility aliases are introduced.
- Controllers are transport adapters only: request parsing, actor extraction,
  application-service invocation, and response wrapping.
- Business state belongs to the owning service application/domain layer.
- Runtime AI state belongs to `ai-engine-v2`; other services may expose only
  read-through projections with explicit source product/run references.
- Frontend-facing APIs are grouped by screen capability, not by old database
  tables or historical class names.

## document-workspace-service

Boundary: document drafting workspace, template-backed document sessions,
document versions, document marks, annotations, revision operations, exports,
and signoff state.

Planned controllers:

- `LawyerDocumentWorkspaceController`
  - `POST /lawyer/document-workspaces`
  - `GET /lawyer/document-workspaces/{documentWorkspaceId}`
  - `GET /lawyer/document-workspaces/{documentWorkspaceId}/view`
  - `PATCH /lawyer/document-workspaces/{documentWorkspaceId}`
- `LawyerDocumentRevisionController`
  - `POST /lawyer/document-workspaces/{documentWorkspaceId}/revision-ops`
  - `POST /lawyer/document-workspaces/{documentWorkspaceId}/revision-ops/{revisionOpId}/apply`
  - `POST /lawyer/document-workspaces/{documentWorkspaceId}/annotations`
- `LawyerDocumentExportController`
  - `POST /lawyer/document-workspaces/{documentWorkspaceId}/exports`
  - `GET /lawyer/document-workspaces/{documentWorkspaceId}/exports/{exportId}`

The `/view` endpoint is the front-end document editor contract. It must return
document blocks, outline, variables, marks, risks, suggestions, citations, and
revision actions as one view object. AI-derived cards are read projections with
`source_run_id` and `source_product_ref`; document-workspace does not become
the AI truth source.

## matter-service

Boundary: matter truth, matter status, matter source registry, matter files
references, matter-level workbench read model, and matter-owned ledgers that
remain after case/document/assistant extraction.

Planned controllers:

- `LawyerMatterController`
  - `POST /lawyer/matters`
  - `GET /lawyer/matters`
  - `GET /lawyer/matters/{matterId}`
  - `PATCH /lawyer/matters/{matterId}`
  - `POST /lawyer/matters/{matterId}/archive`
- `LawyerMatterTodayController`
  - `GET /lawyer/today`
  - `GET /lawyer/today/timeline`
  - `GET /lawyer/today/list`
- `LawyerMatterWorkbenchController`
  - `GET /lawyer/matters/{matterId}/workbench-view`
  - `GET /lawyer/matters/{matterId}/timeline`
  - `GET /lawyer/matters/{matterId}/source-registry`
  - `PUT /lawyer/matters/{matterId}/source-registry`
- `LawyerMatterLedgerController`
  - `GET /lawyer/matters/{matterId}/ledger-view`
  - `POST /lawyer/matters/{matterId}/ledger-items`
  - `PATCH /lawyer/matters/{matterId}/ledger-items/{ledgerItemId}`
- `InternalMatterRuntimeController`
  - `GET /internal/matters/{matterId}/runtime-bundle`
  - `GET /internal/matters/{matterId}/access`

`matter-service` must not expose firm proxy routes, direct AI admin routes,
task ownership routes, document drafting routes, or case proceeding routes.
Those belong to their owning services.

## case-service

Boundary: legal case aggregate, case status, court/hearing/proceeding lineage,
case-stage facts, judgment/outcome breakdown, appeal grounds, and procedural
decision history.

Planned controllers:

- `LawyerCaseController`
  - `POST /lawyer/cases`
  - `GET /lawyer/cases`
  - `GET /lawyer/cases/{caseId}`
  - `PATCH /lawyer/cases/{caseId}`
  - `POST /lawyer/cases/{caseId}/archive`
- `LawyerCaseProceedingController`
  - `GET /lawyer/cases/{caseId}/proceeding-view`
  - `POST /lawyer/cases/{caseId}/proceedings`
  - `GET /lawyer/cases/{caseId}/proceedings/{proceedingId}`
  - `POST /lawyer/cases/{caseId}/proceedings/{proceedingId}/outcomes`
  - `POST /lawyer/cases/{caseId}/proceedings/{proceedingId}/procedural-decisions`
- `InternalCaseRuntimeController`
  - `GET /internal/cases/{caseId}/runtime-context`
  - `GET /internal/case-proceedings/{proceedingId}/runtime-context`

Case data must not be reintroduced into `matter-service` controllers. Matter may
reference a `case_id`; case facts and proceeding details are served by
`case-service`.

## Frontend API Ownership

- Today page: `matter-service`.
- Matter list and matter workbench shell: `matter-service`.
- Case detail, court timeline, hearing status, proceeding lineage:
  `case-service`.
- Document drafting and contract review editor: `document-workspace-service`.
- XiaoJian conversation and assistant action cards: `assistant-service`.
- AI runtime traces, prompt execution, and product truth: `ai-engine-v2`.
