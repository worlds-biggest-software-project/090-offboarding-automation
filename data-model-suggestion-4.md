# Data Model Suggestion 4: Workflow-Engine / State-Machine with Dual-Write Audit

> Project: Offboarding Automation · Created: 2026-05-12

## Philosophy

This model centres the architecture around a general-purpose workflow engine: offboarding cases are workflow instances, tasks are workflow steps, and access revocations are asynchronous jobs dispatched by the workflow. The data model explicitly represents state machines with defined states, transitions, guards, and side effects. Every state transition is atomically dual-written to both the mutable workflow state and an append-only audit ledger, giving the benefits of mutable relational state (fast reads, simple queries) with the compliance guarantees of an immutable audit trail.

This pattern is inspired by durable execution engines like Temporal and workflow orchestrators like Airflow, but implemented entirely in PostgreSQL without external dependencies. The key insight is that offboarding is fundamentally a multi-step, multi-party workflow with conditional branching (different paths for different jurisdictions, departure reasons, and risk levels), parallel execution (revoking access to 20 apps simultaneously), retry logic (SCIM calls fail and must be retried), and timeout handling (tasks that are not completed by the due date). A workflow-native data model handles these patterns naturally, whereas a pure relational model requires the application to manage workflow state in code.

The dual-write audit pattern (mutable state + immutable ledger) avoids the eventual-consistency complexity of event sourcing while still providing a tamper-evident audit trail. Auditors see the ledger; operators see the current state. Both are always consistent because they are written in the same transaction.

**Best for:** Organisations that need robust workflow orchestration (parallel execution, retries, timeouts, conditional branching) with strong audit trails, but want the simplicity of mutable relational state for day-to-day queries. Particularly suited to teams building a production-grade workflow engine without adopting external orchestration infrastructure.

**Trade-offs:**
- Pro: Workflow logic is declarative (state machine definitions in the database), not scattered across application code
- Pro: Dual-write audit gives both fast mutable reads and immutable compliance evidence
- Pro: Job queue with retry/timeout is built into the schema — no external queue dependency needed
- Pro: Parallel task execution, conditional branching, and dependency ordering are first-class concepts
- Pro: State machine definitions are versioned — workflow changes do not break in-progress cases
- Con: More complex schema than the JSONB hybrid (24 tables)
- Con: State machine definitions require careful design; incorrect transitions can leave cases in stuck states
- Con: Dual-write adds write amplification (every mutation writes to two tables)
- Con: Historical state requires querying the audit ledger — the mutable tables only show current state
- Con: Less flexible than event sourcing for temporal queries ("what was true at time T?")

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| SCIM 2.0 (RFC 7643) | SCIM operations dispatched as `revocation_jobs` with SCIM-specific payloads; job results recorded with SCIM response codes |
| ISO/IEC 27001:2022 A.5.18 / A.6.5 | `audit_ledger` provides immutable evidence of every access revocation; `workflow_instances` track completion of post-termination responsibilities |
| NIST SP 800-53 AC-2 | `revocation_jobs` provide automated account management evidence with timestamps and SLA tracking |
| NIST SP 800-53 PS-4 | State machine enforces that all access must be revoked before a case can transition to `completed`; the transition guard is auditable |
| SOC 2 CC6.2/CC6.3 | `audit_ledger` is append-only and cannot be modified; provides the tamper-evident credential revocation records auditors require |
| GDPR Article 17 | Data deletion modelled as a workflow step with its own state machine (requested -> in_progress -> completed) |
| OAuth 2.0 (RFC 7009) | OAuth token revocation is a job type in `revocation_jobs` with RFC 7009 endpoint configuration |
| ISO 3166 | `jurisdictions` reference table with jurisdiction-specific workflow step requirements |

---

## State Machine Definitions

```sql
-- Reusable state machine definitions (versioned)
CREATE TABLE state_machine_definitions (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            VARCHAR(100) NOT NULL,       -- offboarding_case, revocation_job, document_signing
    version         INTEGER NOT NULL DEFAULT 1,
    description     TEXT,
    initial_state   VARCHAR(50) NOT NULL,
    states          JSONB NOT NULL,
    -- states example for offboarding_case:
    -- {
    --   "draft": {"label": "Draft", "type": "initial"},
    --   "pending_approval": {"label": "Pending Approval", "type": "intermediate"},
    --   "approved": {"label": "Approved", "type": "intermediate"},
    --   "in_progress": {"label": "In Progress", "type": "active"},
    --   "access_revoked": {"label": "Access Revoked", "type": "intermediate"},
    --   "completed": {"label": "Completed", "type": "terminal"},
    --   "cancelled": {"label": "Cancelled", "type": "terminal"}
    -- }
    transitions     JSONB NOT NULL,
    -- transitions example:
    -- [
    --   {"from": "draft", "to": "pending_approval", "event": "submit", "guards": ["has_departure_date", "has_departure_reason"]},
    --   {"from": "pending_approval", "to": "approved", "event": "approve", "guards": ["actor_has_approval_permission"]},
    --   {"from": "pending_approval", "to": "draft", "event": "reject", "side_effects": ["notify_initiator"]},
    --   {"from": "approved", "to": "in_progress", "event": "start", "side_effects": ["generate_tasks", "discover_access", "start_revocations"]},
    --   {"from": "in_progress", "to": "access_revoked", "event": "all_access_revoked", "guards": ["all_revocation_jobs_completed"]},
    --   {"from": "access_revoked", "to": "completed", "event": "complete", "guards": ["all_tasks_completed", "all_documents_signed"]},
    --   {"from": ["draft", "pending_approval", "approved"], "to": "cancelled", "event": "cancel"}
    -- ]
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (name, version)
);

-- Insert default state machines
INSERT INTO state_machine_definitions (name, version, description, initial_state, states, transitions) VALUES
('offboarding_case', 1, 'Main offboarding workflow', 'draft',
 '{"draft":{"type":"initial"},"pending_approval":{"type":"intermediate"},"approved":{"type":"intermediate"},"in_progress":{"type":"active"},"access_revoked":{"type":"intermediate"},"completed":{"type":"terminal"},"cancelled":{"type":"terminal"}}',
 '[{"from":"draft","to":"pending_approval","event":"submit"},{"from":"pending_approval","to":"approved","event":"approve"},{"from":"approved","to":"in_progress","event":"start"},{"from":"in_progress","to":"access_revoked","event":"all_access_revoked"},{"from":"access_revoked","to":"completed","event":"complete"},{"from":["draft","pending_approval","approved"],"to":"cancelled","event":"cancel"}]'
),
('revocation_job', 1, 'SaaS access revocation job', 'queued',
 '{"queued":{"type":"initial"},"executing":{"type":"active"},"succeeded":{"type":"terminal"},"failed":{"type":"terminal"},"retry_scheduled":{"type":"intermediate"}}',
 '[{"from":"queued","to":"executing","event":"execute"},{"from":"executing","to":"succeeded","event":"success"},{"from":"executing","to":"failed","event":"fail","guards":["max_retries_exceeded"]},{"from":"executing","to":"retry_scheduled","event":"fail","guards":["retries_remaining"]},{"from":"retry_scheduled","to":"queued","event":"retry"}]'
),
('task', 1, 'Offboarding task', 'pending',
 '{"pending":{"type":"initial"},"in_progress":{"type":"active"},"completed":{"type":"terminal"},"skipped":{"type":"terminal"},"failed":{"type":"terminal"},"blocked":{"type":"intermediate"}}',
 '[{"from":"pending","to":"in_progress","event":"start"},{"from":"pending","to":"blocked","event":"block"},{"from":"blocked","to":"pending","event":"unblock"},{"from":"in_progress","to":"completed","event":"complete"},{"from":"in_progress","to":"failed","event":"fail"},{"from":["pending","blocked"],"to":"skipped","event":"skip"}]'
);
```

## Core Tables

```sql
CREATE TABLE organisations (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            VARCHAR(255) NOT NULL,
    slug            VARCHAR(100) NOT NULL UNIQUE,
    plan_tier       VARCHAR(50) NOT NULL DEFAULT 'free',
    settings        JSONB NOT NULL DEFAULT '{}',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE users (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL REFERENCES organisations(id),
    email           VARCHAR(320) NOT NULL UNIQUE,
    name            VARCHAR(255) NOT NULL,
    role            VARCHAR(50) NOT NULL DEFAULT 'member',
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_users_org ON users(organisation_id);

CREATE TABLE jurisdictions (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    country_code    CHAR(2) NOT NULL,
    subdivision_code VARCHAR(6),
    name            VARCHAR(255) NOT NULL,
    rules           JSONB NOT NULL DEFAULT '{}',
    UNIQUE (country_code, subdivision_code)
);

CREATE TABLE employees (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id     UUID NOT NULL REFERENCES organisations(id),
    email               VARCHAR(320) NOT NULL,
    first_name          VARCHAR(255) NOT NULL,
    last_name           VARCHAR(255) NOT NULL,
    job_title           VARCHAR(255),
    department          VARCHAR(255),
    manager_id          UUID REFERENCES employees(id),
    jurisdiction_id     UUID REFERENCES jurisdictions(id),
    employment_type     VARCHAR(50) NOT NULL DEFAULT 'full_time',
    start_date          DATE NOT NULL,
    status              VARCHAR(50) NOT NULL DEFAULT 'active',
    hris_source         VARCHAR(50),
    hris_external_id    VARCHAR(255),
    profile             JSONB NOT NULL DEFAULT '{}',
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_employees_org ON employees(organisation_id);
CREATE INDEX idx_employees_status ON employees(organisation_id, status);
```

## Workflow Instances (Offboarding Cases)

```sql
-- Generic workflow instance table; each offboarding case is a workflow instance
CREATE TABLE workflow_instances (
    id                      UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id         UUID NOT NULL,
    state_machine_name      VARCHAR(100) NOT NULL,
    state_machine_version   INTEGER NOT NULL,
    current_state           VARCHAR(50) NOT NULL,
    parent_instance_id      UUID REFERENCES workflow_instances(id),  -- for sub-workflows
    context                 JSONB NOT NULL DEFAULT '{}',  -- workflow-specific context data
    created_at              TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at              TIMESTAMPTZ NOT NULL DEFAULT now(),
    FOREIGN KEY (state_machine_name, state_machine_version)
        REFERENCES state_machine_definitions(name, version)
);
CREATE INDEX idx_wf_org_state ON workflow_instances(organisation_id, state_machine_name, current_state);
CREATE INDEX idx_wf_parent ON workflow_instances(parent_instance_id);

-- Offboarding-specific data linked to the workflow instance
CREATE TABLE offboarding_cases (
    id                  UUID PRIMARY KEY,  -- same ID as the workflow_instance
    workflow_instance_id UUID NOT NULL UNIQUE REFERENCES workflow_instances(id),
    organisation_id     UUID NOT NULL REFERENCES organisations(id),
    employee_id         UUID NOT NULL REFERENCES employees(id),
    initiated_by        UUID NOT NULL REFERENCES users(id),
    departure_reason    VARCHAR(50) NOT NULL,
    departure_date      DATE NOT NULL,
    notice_date         DATE,
    access_cutoff_at    TIMESTAMPTZ,
    priority            VARCHAR(20) NOT NULL DEFAULT 'normal',
    risk_level          VARCHAR(20),
    approved_by         UUID REFERENCES users(id),
    approved_at         TIMESTAMPTZ,
    completed_at        TIMESTAMPTZ,
    case_data           JSONB NOT NULL DEFAULT '{}',
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_cases_org ON offboarding_cases(organisation_id);
CREATE INDEX idx_cases_employee ON offboarding_cases(employee_id);
CREATE INDEX idx_cases_departure ON offboarding_cases(departure_date);
```

## Workflow Tasks

```sql
CREATE TABLE checklist_templates (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL REFERENCES organisations(id),
    name            VARCHAR(255) NOT NULL,
    conditions      JSONB NOT NULL DEFAULT '{}',
    tasks           JSONB NOT NULL,
    is_default      BOOLEAN NOT NULL DEFAULT false,
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Each task is also a workflow instance (with the 'task' state machine)
CREATE TABLE offboarding_tasks (
    id                  UUID PRIMARY KEY,
    workflow_instance_id UUID NOT NULL UNIQUE REFERENCES workflow_instances(id),
    offboarding_case_id UUID NOT NULL REFERENCES offboarding_cases(id) ON DELETE CASCADE,
    title               VARCHAR(500) NOT NULL,
    description         TEXT,
    responsible_dept    VARCHAR(100) NOT NULL,
    assigned_to         UUID REFERENCES users(id),
    due_date            DATE,
    sort_order          INTEGER NOT NULL DEFAULT 0,
    depends_on          UUID[] DEFAULT '{}',      -- task IDs this task depends on
    automation_action   VARCHAR(100),              -- revoke_saas_access, send_exit_survey, etc.
    automation_config   JSONB,                     -- action-specific configuration
    completed_at        TIMESTAMPTZ,
    completed_by        UUID REFERENCES users(id),
    failure_reason      TEXT,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_tasks_case ON offboarding_tasks(offboarding_case_id);
CREATE INDEX idx_tasks_assigned ON offboarding_tasks(assigned_to);
```

## Revocation Job Queue

```sql
-- Each revocation job is a workflow instance (with the 'revocation_job' state machine)
CREATE TABLE revocation_jobs (
    id                  UUID PRIMARY KEY,
    workflow_instance_id UUID NOT NULL UNIQUE REFERENCES workflow_instances(id),
    offboarding_case_id UUID NOT NULL REFERENCES offboarding_cases(id),
    task_id             UUID REFERENCES offboarding_tasks(id),
    employee_id         UUID NOT NULL REFERENCES employees(id),
    saas_application_id UUID NOT NULL REFERENCES saas_applications(id),
    external_user_id    VARCHAR(255),
    revocation_method   VARCHAR(50) NOT NULL,      -- scim_deactivate, scim_delete, api_custom, oauth_revoke, manual
    job_config          JSONB NOT NULL DEFAULT '{}',
    -- job_config example:
    -- {
    --   "scim_endpoint": "https://api.github.com/scim/v2/organizations/acme/Users/abc123",
    --   "scim_operation": "PATCH",
    --   "scim_body": {"schemas": ["urn:ietf:params:scim:api:messages:2.0:PatchOp"], "Operations": [{"op": "replace", "value": {"active": false}}]},
    --   "timeout_ms": 30000
    -- }
    priority            INTEGER NOT NULL DEFAULT 100,  -- lower = higher priority
    retry_count         INTEGER NOT NULL DEFAULT 0,
    max_retries         INTEGER NOT NULL DEFAULT 3,
    next_retry_at       TIMESTAMPTZ,
    last_error          TEXT,
    last_response_code  INTEGER,
    started_at          TIMESTAMPTZ,
    completed_at        TIMESTAMPTZ,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_jobs_case ON revocation_jobs(offboarding_case_id);
CREATE INDEX idx_jobs_queue ON revocation_jobs(priority, created_at)
    WHERE workflow_instance_id IN (
        SELECT id FROM workflow_instances WHERE current_state = 'queued'
    );
-- Functional index for the job queue polling query
CREATE INDEX idx_jobs_queued ON revocation_jobs(next_retry_at NULLS FIRST, priority, created_at);
```

### Job Queue Polling Query

```sql
-- Workers poll for the next available job using SELECT FOR UPDATE SKIP LOCKED
-- This is PostgreSQL's built-in job queue pattern — no external queue needed
WITH next_job AS (
    SELECT rj.id
    FROM revocation_jobs rj
    JOIN workflow_instances wi ON rj.workflow_instance_id = wi.id
    WHERE wi.current_state = 'queued'
      AND (rj.next_retry_at IS NULL OR rj.next_retry_at <= now())
    ORDER BY rj.priority, rj.created_at
    LIMIT 1
    FOR UPDATE OF rj SKIP LOCKED
)
UPDATE revocation_jobs
SET updated_at = now()
WHERE id = (SELECT id FROM next_job)
RETURNING *;
-- Application code then transitions the workflow instance to 'executing'
```

## SaaS Applications & Access

```sql
CREATE TABLE saas_applications (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id     UUID NOT NULL REFERENCES organisations(id),
    name                VARCHAR(255) NOT NULL,
    slug                VARCHAR(100) NOT NULL,
    category            VARCHAR(100),
    risk_level          VARCHAR(20) NOT NULL DEFAULT 'medium',
    connector_config    JSONB NOT NULL DEFAULT '{}',
    is_active           BOOLEAN NOT NULL DEFAULT true,
    last_sync_at        TIMESTAMPTZ,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE UNIQUE INDEX idx_saas_slug ON saas_applications(organisation_id, slug);

CREATE TABLE employee_app_access (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    employee_id         UUID NOT NULL REFERENCES employees(id),
    saas_application_id UUID NOT NULL REFERENCES saas_applications(id),
    external_user_id    VARCHAR(255),
    access_level        VARCHAR(100),
    discovery_method    VARCHAR(50) NOT NULL DEFAULT 'manual',
    is_shadow_it        BOOLEAN NOT NULL DEFAULT false,
    status              VARCHAR(50) NOT NULL DEFAULT 'active',
    revoked_at          TIMESTAMPTZ,
    access_details      JSONB NOT NULL DEFAULT '{}',
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_access_employee ON employee_app_access(employee_id);
CREATE INDEX idx_access_status ON employee_app_access(status);
CREATE UNIQUE INDEX idx_access_unique ON employee_app_access(employee_id, saas_application_id)
    WHERE status = 'active';
```

## Audit Ledger (Append-Only)

```sql
-- Immutable audit ledger: every state transition and significant action is recorded
-- This table is NEVER updated or deleted (except for GDPR PII redaction)
CREATE TABLE audit_ledger (
    id                  BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    organisation_id     UUID NOT NULL,
    workflow_instance_id UUID,
    offboarding_case_id UUID,
    event_type          VARCHAR(100) NOT NULL,
    -- State transition events:
    --   state_transition.offboarding_case
    --   state_transition.revocation_job
    --   state_transition.task
    -- Domain events:
    --   access.discovered, access.revoked, access.revocation_failed
    --   document.uploaded, document.signed
    --   exit_interview.completed, exit_interview.ai_analysed
    --   knowledge_transfer.completed
    --   gdpr.erasure_requested, gdpr.erasure_completed
    --   hris.webhook_received, hris.webhook_processed
    from_state          VARCHAR(50),              -- for state_transition events
    to_state            VARCHAR(50),              -- for state_transition events
    trigger_event       VARCHAR(100),             -- the event that caused this transition
    actor_id            UUID,
    actor_type          VARCHAR(50) NOT NULL DEFAULT 'user',  -- user, system, hris_webhook, ai_agent, scheduler
    details             JSONB NOT NULL DEFAULT '{}',
    -- details example (state transition):
    -- {
    --   "case_id": "uuid",
    --   "employee_name": "Jane Smith",
    --   "app_name": "GitHub",
    --   "revocation_method": "scim_deactivate",
    --   "response_code": 200,
    --   "duration_ms": 342
    -- }
    ip_address          INET,
    idempotency_key     VARCHAR(255),             -- prevent duplicate audit entries
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_ledger_org ON audit_ledger(organisation_id);
CREATE INDEX idx_ledger_case ON audit_ledger(offboarding_case_id);
CREATE INDEX idx_ledger_wf ON audit_ledger(workflow_instance_id);
CREATE INDEX idx_ledger_type ON audit_ledger(event_type);
CREATE INDEX idx_ledger_created ON audit_ledger(created_at);
CREATE UNIQUE INDEX idx_ledger_idempotency ON audit_ledger(idempotency_key) WHERE idempotency_key IS NOT NULL;

-- Partition by month for performance at scale
-- CREATE TABLE audit_ledger (...) PARTITION BY RANGE (created_at);
```

### Dual-Write Transaction Pattern

```sql
-- Every state transition writes to both the mutable table and the audit ledger atomically
BEGIN;

-- 1. Update mutable state
UPDATE workflow_instances
SET current_state = 'access_revoked',
    updated_at = now()
WHERE id = :workflow_instance_id
  AND current_state = 'in_progress';  -- optimistic concurrency

-- 2. Append to audit ledger (same transaction)
INSERT INTO audit_ledger (
    organisation_id, workflow_instance_id, offboarding_case_id,
    event_type, from_state, to_state, trigger_event,
    actor_id, actor_type, details
) VALUES (
    :org_id, :workflow_instance_id, :case_id,
    'state_transition.offboarding_case', 'in_progress', 'access_revoked', 'all_access_revoked',
    :actor_id, 'system',
    '{"total_apps_revoked": 12, "time_to_revoke_hours": 1.5}'
);

COMMIT;
```

## Exit Interviews & Knowledge Transfer

```sql
CREATE TABLE exit_interviews (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    offboarding_case_id UUID NOT NULL REFERENCES offboarding_cases(id),
    status              VARCHAR(50) NOT NULL DEFAULT 'pending',
    sent_at             TIMESTAMPTZ,
    completed_at        TIMESTAMPTZ,
    interview_data      JSONB NOT NULL DEFAULT '{}',
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_exit_case ON exit_interviews(offboarding_case_id);

CREATE TABLE knowledge_transfers (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    offboarding_case_id UUID NOT NULL REFERENCES offboarding_cases(id),
    status              VARCHAR(50) NOT NULL DEFAULT 'pending',
    successor_id        UUID REFERENCES employees(id),
    transfer_data       JSONB NOT NULL DEFAULT '{}',
    completed_at        TIMESTAMPTZ,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_kt_case ON knowledge_transfers(offboarding_case_id);
```

## Documents & Equipment

```sql
CREATE TABLE documents (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    offboarding_case_id UUID NOT NULL REFERENCES offboarding_cases(id),
    document_type       VARCHAR(100) NOT NULL,
    title               VARCHAR(500) NOT NULL,
    file_path           VARCHAR(1000),
    mime_type           VARCHAR(100),
    status              VARCHAR(50) NOT NULL DEFAULT 'pending',
    doc_data            JSONB NOT NULL DEFAULT '{}',
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_docs_case ON documents(offboarding_case_id);

CREATE TABLE equipment (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id     UUID NOT NULL REFERENCES organisations(id),
    employee_id         UUID REFERENCES employees(id),
    equipment_type      VARCHAR(100) NOT NULL,
    status              VARCHAR(50) NOT NULL DEFAULT 'assigned',
    equipment_data      JSONB NOT NULL DEFAULT '{}',
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_equip_org ON equipment(organisation_id);
CREATE INDEX idx_equip_employee ON equipment(employee_id);
```

## HRIS Integration

```sql
CREATE TABLE hris_connections (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL REFERENCES organisations(id),
    provider        VARCHAR(50) NOT NULL,
    connection_config JSONB NOT NULL DEFAULT '{}',
    is_active       BOOLEAN NOT NULL DEFAULT true,
    last_sync_at    TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE webhook_events (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    hris_connection_id  UUID REFERENCES hris_connections(id),
    source              VARCHAR(50) NOT NULL,
    event_type          VARCHAR(100) NOT NULL,
    payload             JSONB NOT NULL,
    payload_hash        VARCHAR(64) NOT NULL,
    processing_status   VARCHAR(50) NOT NULL DEFAULT 'received',
    offboarding_case_id UUID REFERENCES offboarding_cases(id),
    error_message       TEXT,
    received_at         TIMESTAMPTZ NOT NULL DEFAULT now(),
    processed_at        TIMESTAMPTZ
);
CREATE INDEX idx_webhook_status ON webhook_events(processing_status);
CREATE UNIQUE INDEX idx_webhook_dedup ON webhook_events(source, payload_hash);

-- Scheduled jobs: timeout checks, retry scheduling, SLA monitoring
CREATE TABLE scheduled_jobs (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    job_type        VARCHAR(100) NOT NULL,
    -- check_task_timeouts, retry_failed_revocations, sla_breach_notification,
    -- gdpr_erasure_deadline, data_retention_purge
    target_id       UUID,                         -- workflow_instance, offboarding_case, etc.
    scheduled_for   TIMESTAMPTZ NOT NULL,
    status          VARCHAR(50) NOT NULL DEFAULT 'scheduled',  -- scheduled, executing, completed, cancelled
    config          JSONB NOT NULL DEFAULT '{}',
    completed_at    TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_scheduled_jobs_due ON scheduled_jobs(scheduled_for) WHERE status = 'scheduled';
```

## Example: Complete Offboarding Flow

```sql
-- 1. HRIS webhook arrives -> create workflow instance + offboarding case
BEGIN;
INSERT INTO workflow_instances (id, organisation_id, state_machine_name, state_machine_version, current_state)
VALUES (:case_id, :org_id, 'offboarding_case', 1, 'draft');

INSERT INTO offboarding_cases (id, workflow_instance_id, organisation_id, employee_id, initiated_by, departure_reason, departure_date)
VALUES (:case_id, :case_id, :org_id, :emp_id, :system_user_id, 'voluntary', '2026-06-15');

INSERT INTO audit_ledger (organisation_id, workflow_instance_id, offboarding_case_id, event_type, to_state, trigger_event, actor_type, details)
VALUES (:org_id, :case_id, :case_id, 'state_transition.offboarding_case', 'draft', 'hris_webhook', 'hris_webhook',
        '{"hris_provider": "bamboohr", "webhook_event_id": "uuid"}');
COMMIT;

-- 2. HR manager approves -> transition to approved, then start
-- 3. Start generates tasks + revocation jobs as child workflow instances
-- 4. Workers poll revocation_jobs, execute SCIM calls, transition job states
-- 5. When all revocation jobs succeed -> case transitions to access_revoked
-- 6. When all tasks complete -> case transitions to completed
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Workflow Engine | 2 | `state_machine_definitions`, `workflow_instances` |
| Organisation & Auth | 2 | `organisations`, `users` |
| Reference Data | 1 | `jurisdictions` |
| Employees | 1 | `employees` |
| Offboarding Cases | 1 | `offboarding_cases` (linked to workflow instance) |
| Tasks | 2 | `checklist_templates`, `offboarding_tasks` (linked to workflow instance) |
| Revocation Jobs | 1 | `revocation_jobs` (linked to workflow instance) |
| SaaS & Access | 2 | `saas_applications`, `employee_app_access` |
| Exit & Knowledge | 2 | `exit_interviews`, `knowledge_transfers` |
| Documents & Equipment | 2 | `documents`, `equipment` |
| Integration | 3 | `hris_connections`, `webhook_events`, `scheduled_jobs` |
| Audit | 1 | `audit_ledger` (append-only) |
| **Total** | **20** | |

---

## Key Design Decisions

1. **State machine definitions stored in the database**: Workflow behaviour is data, not code. State machines are versioned, so changing the offboarding workflow (e.g., adding a "legal_review" state) creates a new version without affecting in-progress cases running on the old version.

2. **Every stateful entity is a workflow instance**: Offboarding cases, tasks, and revocation jobs all link to `workflow_instances`. This means the workflow engine code (state transition validation, guard evaluation, side effect dispatch) is written once and reused for all entity types.

3. **Dual-write audit in the same transaction**: Every state transition atomically writes to both the mutable `workflow_instances` table and the append-only `audit_ledger`. This avoids the eventual consistency of event sourcing while providing the same audit guarantees. If the transaction fails, neither the state nor the audit entry is written.

4. **PostgreSQL as the job queue**: Revocation jobs use `SELECT FOR UPDATE SKIP LOCKED` for concurrent, safe job dequeuing without an external queue (Redis, RabbitMQ, SQS). This reduces infrastructure dependencies and keeps all data in one database.

5. **Task dependency ordering via `depends_on` array**: Tasks can declare dependencies on other tasks (e.g., "transfer Drive files before deleting Google account"). The workflow engine checks dependencies before allowing a task to transition from `pending` to `in_progress`.

6. **Scheduled jobs for timeouts and SLA monitoring**: Rather than relying on application-level timers, the `scheduled_jobs` table allows the system to schedule future actions (task timeout checks, retry scheduling, GDPR erasure deadline enforcement). A background worker polls this table.

7. **Parent-child workflow instances**: Sub-workflows (e.g., a GDPR deletion workflow spawned from an offboarding case) use `parent_instance_id` to maintain the relationship. This enables hierarchical workflow orchestration.

8. **Optimistic concurrency on state transitions**: The `UPDATE ... WHERE current_state = :expected_state` pattern prevents race conditions. If two concurrent requests try to transition the same workflow, one succeeds and the other fails with zero rows updated, prompting a retry.

9. **Audit ledger uses `BIGINT GENERATED ALWAYS AS IDENTITY`**: Unlike UUIDs, sequential IDs on the audit ledger provide natural ordering and efficient range scans. The `GENERATED ALWAYS` constraint prevents manual ID insertion, reinforcing immutability.

10. **Hybrid JSONB for variable fields**: Following the pattern from Suggestion 3, domain-specific variable data (connector configs, case data, interview responses) uses JSONB columns, while the workflow engine itself is fully relational. This keeps the workflow engine generic while allowing domain flexibility.
