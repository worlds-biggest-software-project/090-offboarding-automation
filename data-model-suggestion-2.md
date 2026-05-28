# Data Model Suggestion 2: Event-Sourced / Audit-First (CQRS)

> Project: Offboarding Automation · Created: 2026-05-12

## Philosophy

This model treats every state change in the offboarding lifecycle as an immutable event appended to a central event store. The event stream is the single source of truth; all queryable state (dashboards, reports, compliance evidence) is derived from materialised read models (projections) that can be rebuilt at any time by replaying the event stream. This is the CQRS (Command Query Responsibility Segregation) pattern: commands write events, queries read projections.

This architecture is a natural fit for offboarding automation because the domain has an inherent requirement for complete, tamper-proof audit trails. SOC 2, ISO 27001, and NIST SP 800-53 all require timestamped evidence of every access revocation action. Rather than bolting an audit log onto a mutable relational model, this approach makes auditability the foundational architectural principle — the audit trail IS the data model.

Event sourcing also enables temporal queries that are difficult with mutable state: "What access did this employee have at 3pm on Tuesday?" or "Replay all offboarding events for departures in Q1 to identify process bottlenecks." These queries are essential for AI-powered analytics on offboarding patterns and for litigation support where the exact sequence of events matters.

**Best for:** Organisations where compliance evidence, temporal queries, and tamper-proof audit trails are the primary requirements — particularly regulated industries (financial services, healthcare, government) subject to SOX, FINRA, or NIST frameworks.

**Trade-offs:**
- Pro: Complete, immutable audit trail is built into the architecture — not an afterthought
- Pro: Temporal queries ("what was true at time T?") are trivial — replay to any point
- Pro: AI analytics can process the event stream directly for pattern detection
- Pro: Projections can be rebuilt if requirements change, without migrating data
- Pro: Natural fit for distributed/async architectures with multiple integration workers
- Con: Higher implementation complexity — requires event versioning, projection management, and eventual consistency handling
- Con: Read models are eventually consistent; stale-read windows must be managed in the UI
- Con: Event schema evolution requires upcasting logic for old events
- Con: Debugging requires understanding both the event stream and the current projection state
- Con: Storage grows monotonically; event compaction/archival strategy needed for long-running deployments

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| SCIM 2.0 (RFC 7643) | SCIM API calls generate `AccessRevocationRequested` and `AccessRevoked` events; SCIM response payloads are stored as event metadata |
| ISO/IEC 27001:2022 A.5.18 / A.6.5 | The event stream itself IS the access control audit evidence — every access change is an immutable, timestamped event |
| NIST SP 800-53 AC-2 / PS-4 | Event replay provides complete account management history; PS-4 compliance can be verified by querying the time between `OffboardingApproved` and `AllAccessRevoked` events |
| SOC 2 CC6.2/CC6.3 | Auditors can be given read access to the event stream directly — no separate audit log to maintain or reconcile |
| GDPR Article 17 | `DataDeletionRequested` and `DataDeletionCompleted` events track erasure workflow; note: GDPR-required deletions apply to projections, not to the event stream itself (legal basis for event retention must be documented) |
| OAuth 2.0 (RFC 7009) | OAuth token discovery and revocation are captured as `OAuthTokenDiscovered` and `OAuthTokenRevoked` events |
| SOX / FINRA | Immutable event store satisfies financial services requirements for tamper-proof records of privileged access changes |

---

## Event Store (Write Side)

```sql
-- Central event store: append-only, immutable
CREATE TABLE events (
    event_id        UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    stream_id       UUID NOT NULL,               -- aggregate root ID (e.g., offboarding case ID)
    stream_type     VARCHAR(100) NOT NULL,        -- offboarding_case, employee, saas_access, exit_interview
    event_type      VARCHAR(200) NOT NULL,        -- e.g., OffboardingCaseCreated, AccessRevoked
    event_version   INTEGER NOT NULL,             -- schema version of this event type (for upcasting)
    sequence_number BIGINT NOT NULL,              -- monotonic within a stream
    payload         JSONB NOT NULL,               -- event-specific data
    metadata        JSONB NOT NULL DEFAULT '{}',  -- actor, IP, correlation_id, causation_id
    organisation_id UUID NOT NULL,                -- tenant isolation
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (stream_id, sequence_number)
);

-- Global ordering for projections that span multiple streams
CREATE SEQUENCE events_global_seq;
ALTER TABLE events ADD COLUMN global_position BIGINT NOT NULL DEFAULT nextval('events_global_seq');

CREATE INDEX idx_events_stream ON events(stream_id, sequence_number);
CREATE INDEX idx_events_type ON events(event_type);
CREATE INDEX idx_events_org ON events(organisation_id);
CREATE INDEX idx_events_global ON events(global_position);
CREATE INDEX idx_events_created ON events(created_at);

-- Partition by month for large deployments
-- CREATE TABLE events (...) PARTITION BY RANGE (created_at);

-- Event type catalogue for documentation and validation
CREATE TABLE event_type_registry (
    event_type      VARCHAR(200) PRIMARY KEY,
    current_version INTEGER NOT NULL DEFAULT 1,
    description     TEXT NOT NULL,
    payload_schema  JSONB NOT NULL,              -- JSON Schema for the event payload
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

### Event Type Examples

```sql
-- Example: OffboardingCaseCreated event payload
-- {
--   "employee_id": "uuid",
--   "employee_name": "Jane Smith",
--   "employee_email": "jane@company.com",
--   "departure_reason": "voluntary",
--   "departure_date": "2026-06-15",
--   "notice_date": "2026-05-15",
--   "jurisdiction": "US-CA",
--   "initiated_by_user_id": "uuid",
--   "priority": "normal",
--   "risk_level": "high"
-- }

-- Example: AccessRevocationRequested event payload
-- {
--   "app_name": "GitHub Enterprise",
--   "app_id": "uuid",
--   "employee_app_user_id": "octocat-jane",
--   "revocation_method": "scim_deactivate",
--   "scim_endpoint": "https://api.github.com/scim/v2/...",
--   "risk_level": "critical"
-- }

-- Example: AccessRevoked event payload
-- {
--   "app_name": "GitHub Enterprise",
--   "app_id": "uuid",
--   "revocation_method": "scim_deactivate",
--   "scim_response_status": 200,
--   "duration_ms": 342,
--   "confirmed_at": "2026-05-12T14:30:00Z"
-- }

-- Example: ExitInterviewCompleted event payload
-- {
--   "responses": [
--     {"question_id": "uuid", "question": "Reason for leaving?", "response": "Better opportunity"},
--     {"question_id": "uuid", "question": "Manager rating (1-5)", "response": 2}
--   ],
--   "ai_summary": "Employee cites lack of growth opportunity and poor manager relationship...",
--   "ai_themes": ["career_growth", "manager_feedback"],
--   "ai_sentiment_score": -0.35
-- }
```

### Complete Event Type Catalogue

```sql
INSERT INTO event_type_registry (event_type, description, payload_schema) VALUES
-- Offboarding Case Lifecycle
('OffboardingCaseCreated',          'New offboarding case initiated', '{}'),
('OffboardingCaseApprovalRequested','Case submitted for approval', '{}'),
('OffboardingCaseApproved',         'Case approved by authorised user', '{}'),
('OffboardingCaseRejected',         'Case approval rejected', '{}'),
('OffboardingCaseCancelled',        'Case cancelled before completion', '{}'),
('OffboardingCaseCompleted',        'All tasks and revocations completed', '{}'),

-- Task Management
('TaskGenerated',                   'Task created from template for this case', '{}'),
('TaskAdHocCreated',                'Ad hoc task added to case', '{}'),
('TaskAssigned',                    'Task assigned to a user', '{}'),
('TaskStarted',                     'Task work begun', '{}'),
('TaskCompleted',                   'Task marked complete', '{}'),
('TaskFailed',                      'Task execution failed', '{}'),
('TaskSkipped',                     'Task skipped (not applicable)', '{}'),

-- Access Discovery
('EmployeeAccessDiscovered',        'App access found for employee (SSO, SCIM, AI)', '{}'),
('ShadowITAccessDiscovered',        'Unofficial app access discovered by AI', '{}'),
('OAuthTokenDiscovered',            'Personal OAuth grant discovered', '{}'),

-- Access Revocation
('AccessRevocationRequested',       'Revocation request sent to external system', '{}'),
('AccessRevocationRetried',         'Failed revocation retried', '{}'),
('AccessRevoked',                   'Access successfully revoked in external system', '{}'),
('AccessRevocationFailed',          'Revocation failed after all retries', '{}'),
('OAuthTokenRevoked',               'OAuth token revoked via RFC 7009 or admin API', '{}'),
('AllAccessRevoked',                'All known access for employee confirmed revoked', '{}'),

-- Documents
('DocumentUploaded',                'Separation document uploaded', '{}'),
('DocumentSentForSignature',        'Document sent to employee for signature', '{}'),
('DocumentSigned',                  'Document signed by employee or employer', '{}'),
('DocumentExpired',                 'Document signature deadline passed', '{}'),

-- Equipment
('EquipmentReturnRequested',        'Equipment return initiated', '{}'),
('EquipmentReturnShipped',          'Return shipping label sent / item shipped', '{}'),
('EquipmentReturned',               'Equipment received and checked in', '{}'),

-- Exit Interview
('ExitSurveySent',                  'Exit interview survey sent to employee', '{}'),
('ExitInterviewCompleted',          'Employee completed exit interview', '{}'),
('ExitInterviewAIAnalysed',         'AI analysis of exit interview completed', '{}'),

-- Knowledge Transfer
('KnowledgeTransferBriefGenerated', 'AI generated knowledge transfer brief', '{}'),
('KnowledgeTransferItemAdded',      'Knowledge item identified for transfer', '{}'),
('KnowledgeTransferCompleted',      'Knowledge successfully transferred to successor', '{}'),

-- GDPR
('DataDeletionRequested',           'GDPR Article 17 erasure request received', '{}'),
('DataDeletionCompleted',           'Personal data deleted per erasure request', '{}'),
('DataRetentionExtended',           'Data retained beyond standard period (legal hold)', '{}'),

-- HRIS Integration
('HRISWebhookReceived',             'Webhook received from upstream HRIS', '{}'),
('HRISWebhookProcessed',            'HRIS webhook successfully processed', '{}'),
('HRISWebhookFailed',               'HRIS webhook processing failed', '{}');
```

## Read Models (Projections)

Projections are mutable tables rebuilt from the event stream. They can be dropped and rebuilt at any time.

### Offboarding Case Projection

```sql
-- Materialised from OffboardingCase* events
CREATE TABLE proj_offboarding_cases (
    case_id             UUID PRIMARY KEY,
    organisation_id     UUID NOT NULL,
    employee_id         UUID NOT NULL,
    employee_name       VARCHAR(255) NOT NULL,
    employee_email      VARCHAR(320) NOT NULL,
    department          VARCHAR(255),
    job_title           VARCHAR(255),
    jurisdiction        VARCHAR(10),
    departure_reason    VARCHAR(50) NOT NULL,
    departure_date      DATE NOT NULL,
    notice_date         DATE,
    access_cutoff_at    TIMESTAMPTZ,
    status              VARCHAR(50) NOT NULL,
    priority            VARCHAR(20) NOT NULL,
    risk_level          VARCHAR(20),
    initiated_by        UUID NOT NULL,
    approved_by         UUID,
    approved_at         TIMESTAMPTZ,
    total_tasks         INTEGER NOT NULL DEFAULT 0,
    completed_tasks     INTEGER NOT NULL DEFAULT 0,
    total_apps          INTEGER NOT NULL DEFAULT 0,
    revoked_apps        INTEGER NOT NULL DEFAULT 0,
    failed_revocations  INTEGER NOT NULL DEFAULT 0,
    access_revoked_at   TIMESTAMPTZ,
    completed_at        TIMESTAMPTZ,
    last_event_at       TIMESTAMPTZ NOT NULL,
    last_event_position BIGINT NOT NULL           -- for resuming projection
);
CREATE INDEX idx_proj_cases_org_status ON proj_offboarding_cases(organisation_id, status);
CREATE INDEX idx_proj_cases_departure ON proj_offboarding_cases(departure_date);
```

### Task Board Projection

```sql
-- Materialised from Task* events
CREATE TABLE proj_tasks (
    task_id             UUID PRIMARY KEY,
    case_id             UUID NOT NULL,
    organisation_id     UUID NOT NULL,
    title               VARCHAR(500) NOT NULL,
    responsible_dept    VARCHAR(100) NOT NULL,
    assigned_to         UUID,
    assigned_to_name    VARCHAR(255),
    status              VARCHAR(50) NOT NULL,
    due_date            DATE,
    completed_at        TIMESTAMPTZ,
    failure_reason      TEXT,
    sort_order          INTEGER NOT NULL DEFAULT 0,
    last_event_at       TIMESTAMPTZ NOT NULL
);
CREATE INDEX idx_proj_tasks_case ON proj_tasks(case_id);
CREATE INDEX idx_proj_tasks_assigned ON proj_tasks(assigned_to, status);
CREATE INDEX idx_proj_tasks_org_status ON proj_tasks(organisation_id, status);
```

### Access Revocation Dashboard Projection

```sql
-- Materialised from Access* and OAuth* events
CREATE TABLE proj_access_status (
    id                  UUID PRIMARY KEY,
    case_id             UUID NOT NULL,
    organisation_id     UUID NOT NULL,
    employee_id         UUID NOT NULL,
    app_name            VARCHAR(255) NOT NULL,
    app_id              UUID,
    external_user_id    VARCHAR(255),
    access_level        VARCHAR(100),
    is_shadow_it        BOOLEAN NOT NULL DEFAULT false,
    discovery_method    VARCHAR(50),
    revocation_method   VARCHAR(50),
    revocation_status   VARCHAR(50) NOT NULL,     -- pending, in_progress, revoked, failed
    retry_count         INTEGER NOT NULL DEFAULT 0,
    last_error          TEXT,
    requested_at        TIMESTAMPTZ,
    revoked_at          TIMESTAMPTZ,
    last_event_at       TIMESTAMPTZ NOT NULL
);
CREATE INDEX idx_proj_access_case ON proj_access_status(case_id);
CREATE INDEX idx_proj_access_org_status ON proj_access_status(organisation_id, revocation_status);
```

### Compliance Evidence Projection

```sql
-- Optimised for auditor queries: one row per completed revocation with timestamps
CREATE TABLE proj_compliance_evidence (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id     UUID NOT NULL,
    case_id             UUID NOT NULL,
    employee_name       VARCHAR(255) NOT NULL,
    employee_email      VARCHAR(320) NOT NULL,
    departure_date      DATE NOT NULL,
    app_name            VARCHAR(255) NOT NULL,
    revocation_method   VARCHAR(50) NOT NULL,
    case_created_at     TIMESTAMPTZ NOT NULL,
    revocation_requested_at TIMESTAMPTZ NOT NULL,
    revocation_completed_at TIMESTAMPTZ,
    time_to_revoke_seconds INTEGER,               -- computed
    status              VARCHAR(50) NOT NULL,
    nist_control        VARCHAR(20) DEFAULT 'AC-2',
    iso_control         VARCHAR(20) DEFAULT 'A.5.18',
    last_event_at       TIMESTAMPTZ NOT NULL
);
CREATE INDEX idx_proj_compliance_org ON proj_compliance_evidence(organisation_id);
CREATE INDEX idx_proj_compliance_date ON proj_compliance_evidence(departure_date);
```

### Exit Interview Analytics Projection

```sql
-- Aggregated themes across all exit interviews for cohort analysis
CREATE TABLE proj_exit_themes (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id     UUID NOT NULL,
    case_id             UUID NOT NULL,
    department          VARCHAR(255),
    departure_reason    VARCHAR(50),
    departure_month     DATE,                     -- truncated to month for cohort grouping
    theme               VARCHAR(100) NOT NULL,    -- manager_feedback, compensation, career_growth, etc.
    sentiment_score     NUMERIC(3,2),
    ai_summary_excerpt  TEXT,
    last_event_at       TIMESTAMPTZ NOT NULL
);
CREATE INDEX idx_proj_themes_org ON proj_exit_themes(organisation_id);
CREATE INDEX idx_proj_themes_dept ON proj_exit_themes(organisation_id, department, departure_month);
CREATE INDEX idx_proj_themes_theme ON proj_exit_themes(organisation_id, theme);
```

## Supporting Tables (Non-Event-Sourced)

Reference data and configuration tables are standard relational — they do not need event sourcing because they change infrequently and do not require temporal queries.

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
    email           VARCHAR(320) NOT NULL UNIQUE,
    name            VARCHAR(255) NOT NULL,
    organisation_id UUID NOT NULL REFERENCES organisations(id),
    role            VARCHAR(50) NOT NULL DEFAULT 'member',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE saas_app_catalogue (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id     UUID NOT NULL REFERENCES organisations(id),
    name                VARCHAR(255) NOT NULL,
    slug                VARCHAR(100) NOT NULL,
    category            VARCHAR(100),
    scim_endpoint       VARCHAR(500),
    supports_scim       BOOLEAN NOT NULL DEFAULT false,
    api_type            VARCHAR(50),
    connector_config_enc TEXT,
    risk_level          VARCHAR(20) NOT NULL DEFAULT 'medium',
    is_active           BOOLEAN NOT NULL DEFAULT true,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE UNIQUE INDEX idx_saas_cat_slug ON saas_app_catalogue(organisation_id, slug);

CREATE TABLE checklist_templates (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL REFERENCES organisations(id),
    name            VARCHAR(255) NOT NULL,
    description     TEXT,
    conditions      JSONB NOT NULL DEFAULT '[]',
    tasks           JSONB NOT NULL,
    -- tasks example:
    -- [
    --   {"title": "Revoke GitHub access", "dept": "it", "due_offset_days": 0, "automatable": true, "action": "revoke_saas_access"},
    --   {"title": "Collect laptop", "dept": "facilities", "due_offset_days": -1, "automatable": false},
    --   {"title": "Process final pay", "dept": "finance", "due_offset_days": 5, "automatable": false}
    -- ]
    is_default      BOOLEAN NOT NULL DEFAULT false,
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE hris_connections (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL REFERENCES organisations(id),
    provider        VARCHAR(50) NOT NULL,
    api_endpoint    VARCHAR(500),
    auth_config_enc TEXT,
    webhook_secret_enc TEXT,
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE exit_survey_templates (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL REFERENCES organisations(id),
    name            VARCHAR(255) NOT NULL,
    questions       JSONB NOT NULL,
    -- questions example:
    -- [
    --   {"id": "uuid", "text": "Primary reason for leaving?", "type": "free_text", "required": true},
    --   {"id": "uuid", "text": "Rate your manager (1-5)", "type": "rating_1_5", "required": true}
    -- ]
    is_default      BOOLEAN NOT NULL DEFAULT false,
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Projection checkpoint: tracks where each projection has read up to
CREATE TABLE projection_checkpoints (
    projection_name VARCHAR(100) PRIMARY KEY,
    last_position   BIGINT NOT NULL DEFAULT 0,
    last_updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

## Example Queries

### Replay case state at a specific point in time

```sql
-- "What was the state of case X at 2pm on May 10th?"
SELECT event_type, payload, created_at
FROM events
WHERE stream_id = '...'  -- case ID
  AND stream_type = 'offboarding_case'
  AND created_at <= '2026-05-10T14:00:00Z'
ORDER BY sequence_number ASC;
-- Application code replays these events to reconstruct the state at that moment
```

### Generate SOC 2 compliance report

```sql
-- All access revocations with timing for a date range
SELECT
    employee_name,
    app_name,
    departure_date,
    revocation_requested_at,
    revocation_completed_at,
    time_to_revoke_seconds,
    CASE WHEN time_to_revoke_seconds <= 86400 THEN 'PASS' ELSE 'FAIL' END AS sla_status
FROM proj_compliance_evidence
WHERE organisation_id = '...'
  AND departure_date BETWEEN '2026-01-01' AND '2026-03-31'
ORDER BY departure_date, app_name;
```

### Exit interview theme analysis across cohorts

```sql
-- Top departure themes by department in the last 6 months
SELECT
    department,
    theme,
    COUNT(*) AS frequency,
    AVG(sentiment_score) AS avg_sentiment
FROM proj_exit_themes
WHERE organisation_id = '...'
  AND departure_month >= '2025-12-01'
GROUP BY department, theme
ORDER BY department, frequency DESC;
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Event Store | 2 | `events`, `event_type_registry` |
| Projections | 5 | Cases, tasks, access status, compliance evidence, exit themes |
| Organisation & Auth | 2 | `organisations`, `users` |
| Configuration (non-event-sourced) | 4 | SaaS catalogue, checklist templates, HRIS connections, exit survey templates |
| Infrastructure | 1 | `projection_checkpoints` |
| **Total** | **14** | Plus projections can be added/removed without schema migration |

---

## Key Design Decisions

1. **Single `events` table with `stream_type` discriminator**: All domain events go into one table, partitioned by time. This simplifies cross-aggregate queries (e.g., "all events for org X in the last hour") while still allowing stream-level replay via the `stream_id` + `sequence_number` unique constraint.

2. **`global_position` sequence for cross-stream ordering**: Projections that span multiple aggregates (like the compliance evidence projection) need a total ordering of events. The `global_position` sequence provides this without requiring distributed coordination.

3. **Event versioning via `event_version` field**: Every event carries its schema version. When the payload structure of `AccessRevoked` changes (e.g., adding a new field), old events retain their original version. Projection code includes upcasting functions that transform v1 events into the current shape during replay.

4. **Projections are disposable read models**: Any `proj_*` table can be dropped and rebuilt by replaying the event stream from position 0. This means schema changes to projections are zero-downtime: deploy new projection code, rebuild the table in the background, swap when ready.

5. **Metadata envelope for cross-cutting concerns**: Every event carries a `metadata` JSONB field with `actor_id`, `actor_type`, `ip_address`, `correlation_id` (which request triggered this), and `causation_id` (which prior event caused this). This supports causal ordering and actor-level audit queries without polluting the domain payload.

6. **Reference data stays relational**: Configuration tables (templates, SaaS catalogue, HRIS connections) use standard relational design because they change infrequently and do not need temporal queries. Only the offboarding domain — where audit trails are legally required — is event-sourced.

7. **GDPR tension acknowledged**: The event store is append-only and immutable, but GDPR requires the ability to delete personal data. Resolution: personal data is stripped from events during a GDPR deletion request and replaced with a `DataRedacted` event. The projections (which contain the readable personal data) are updated to remove PII. The event structure and metadata remain for audit continuity.

8. **Projection checkpoints enable resumable processing**: Each projection tracks its `last_position` in the event stream. If a projection process crashes, it resumes from the checkpoint rather than replaying the entire stream.

9. **Lower table count (14 vs. 31)**: By storing domain state as events rather than normalised tables, the total table count drops significantly. The complexity moves from schema to application code (event handlers, projections, upcasters).

10. **AI analytics on the event stream**: The event stream is a natural input for AI-powered analysis. Pattern detection ("cases where access revocation took >24h"), anomaly detection ("unusual number of shadow IT apps discovered"), and process mining ("what is the actual offboarding workflow path, vs. the intended one?") can all operate directly on the event data.
