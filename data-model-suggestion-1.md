# Data Model Suggestion 1: Entity-Centric Normalized Relational

> Project: Offboarding Automation · Created: 2026-05-12

## Philosophy

This model follows strict third-normal-form relational design: every concept gets its own table, relationships are expressed through foreign keys and junction tables, and reference data (jurisdictions, departure reasons, app catalogues) is stored in dedicated lookup tables. The schema is designed for maximum data integrity and queryability — any question about offboarding state can be answered with a standard SQL JOIN without parsing JSON or replaying events.

The normalized approach mirrors how enterprise HRIS and identity management systems (Workday, Okta, Entra ID) structure their data internally. It is well-suited to organisations that value explicit schema enforcement, have database administrators who manage migrations, and need complex cross-entity reporting (e.g., "show me all offboarding cases in the EU where SaaS deprovisioning took longer than 24 hours, grouped by app category").

**Best for:** Organisations with mature database teams that prioritise data integrity, complex cross-entity reporting, and schema-enforced compliance constraints.

**Trade-offs:**
- Pro: Full referential integrity; every relationship is explicit and enforced by the database
- Pro: Standard SQL tooling works out of the box — BI tools, reporting, ad hoc queries
- Pro: Schema changes are explicit migrations; no hidden structure drift
- Pro: Audit queries are straightforward JOINs across related tables
- Con: High table count (~35-40 tables) increases migration complexity
- Con: Adding jurisdiction-specific fields requires schema migrations, not runtime configuration
- Con: Many-to-many junction tables add query complexity for common operations
- Con: Less flexible for rapidly evolving requirements during early product development

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| SCIM 2.0 (RFC 7643) | `saas_applications` and `employee_app_access` tables mirror SCIM User resource attributes (`active`, `userName`, `externalId`); deprovisioning status tracks SCIM operations |
| ISO/IEC 27001:2022 A.6.5 | `offboarding_cases` table enforces that every departure triggers a structured case with mandatory access revocation steps |
| ISO/IEC 27001:2022 A.5.18 | `access_revocation_actions` table logs every access right removal with timestamps, satisfying control evidence requirements |
| NIST SP 800-53 AC-2 | `audit_log` table provides automated account management evidence; `access_revocation_actions` tracks timely removal |
| NIST SP 800-53 PS-4 | `offboarding_cases.access_revoked_at` enforces personnel termination access deadlines |
| GDPR Article 17 | `data_deletion_requests` table tracks erasure requests with status and completion timestamps |
| GDPR Article 5 | `data_retention_policies` table enforces storage limitation with configurable per-jurisdiction retention windows |
| SOC 2 CC6.2/CC6.3 | `audit_log` provides timestamped evidence of credential revocation across all systems |
| OAuth 2.0 (RFC 6749/7009) | `oauth_token_grants` table tracks discovered OAuth tokens; `token_revocation_status` tracks RFC 7009 revocation |
| ISO 3166 | `jurisdictions` reference table uses ISO 3166-1 alpha-2 country codes and ISO 3166-2 subdivision codes |

---

## Organisation & Tenant Management

```sql
CREATE TABLE organisations (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            VARCHAR(255) NOT NULL,
    slug            VARCHAR(100) NOT NULL UNIQUE,
    plan_tier       VARCHAR(50) NOT NULL DEFAULT 'free',  -- free, team, enterprise
    settings        JSONB NOT NULL DEFAULT '{}',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE org_members (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL REFERENCES organisations(id),
    user_id         UUID NOT NULL REFERENCES users(id),
    role            VARCHAR(50) NOT NULL DEFAULT 'member',  -- admin, hr_manager, it_admin, member
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (organisation_id, user_id)
);

CREATE TABLE users (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    email           VARCHAR(320) NOT NULL UNIQUE,
    name            VARCHAR(255) NOT NULL,
    password_hash   VARCHAR(255),  -- NULL if SSO-only
    sso_provider    VARCHAR(50),   -- okta, entra_id, google
    sso_external_id VARCHAR(255),
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

## Reference Data

```sql
CREATE TABLE jurisdictions (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    country_code    CHAR(2) NOT NULL,         -- ISO 3166-1 alpha-2
    subdivision_code VARCHAR(6),               -- ISO 3166-2 (e.g., US-CA)
    name            VARCHAR(255) NOT NULL,
    notice_period_days INTEGER,                -- statutory minimum notice period
    final_pay_deadline_days INTEGER,           -- days after termination for final pay
    gdpr_applicable BOOLEAN NOT NULL DEFAULT false,
    data_retention_years INTEGER DEFAULT 3,    -- default data retention period
    notes           TEXT,
    UNIQUE (country_code, subdivision_code)
);

CREATE TABLE departure_reasons (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL REFERENCES organisations(id),
    code            VARCHAR(50) NOT NULL,     -- voluntary, involuntary, redundancy, end_of_contract, retirement
    label           VARCHAR(255) NOT NULL,
    requires_notice BOOLEAN NOT NULL DEFAULT false,
    is_active       BOOLEAN NOT NULL DEFAULT true,
    UNIQUE (organisation_id, code)
);

CREATE TABLE departments (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL REFERENCES organisations(id),
    name            VARCHAR(255) NOT NULL,
    parent_id       UUID REFERENCES departments(id),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_departments_org ON departments(organisation_id);
CREATE INDEX idx_departments_parent ON departments(parent_id);
```

## Employee Management

```sql
CREATE TABLE employees (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id     UUID NOT NULL REFERENCES organisations(id),
    hris_external_id    VARCHAR(255),          -- ID from source HRIS (BambooHR, Workday, etc.)
    hris_source         VARCHAR(50),           -- bamboohr, workday, rippling, manual
    employee_number     VARCHAR(50),           -- internal employee number
    first_name          VARCHAR(255) NOT NULL,
    last_name           VARCHAR(255) NOT NULL,
    email               VARCHAR(320) NOT NULL,
    personal_email      VARCHAR(320),
    job_title           VARCHAR(255),
    department_id       UUID REFERENCES departments(id),
    manager_id          UUID REFERENCES employees(id),
    jurisdiction_id     UUID REFERENCES jurisdictions(id),
    employment_type     VARCHAR(50) NOT NULL DEFAULT 'full_time',  -- full_time, part_time, contractor, intern
    start_date          DATE NOT NULL,
    status              VARCHAR(50) NOT NULL DEFAULT 'active',     -- active, offboarding, offboarded, archived
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_employees_org ON employees(organisation_id);
CREATE INDEX idx_employees_status ON employees(organisation_id, status);
CREATE INDEX idx_employees_hris ON employees(hris_source, hris_external_id);
CREATE INDEX idx_employees_manager ON employees(manager_id);
```

## Offboarding Cases

```sql
CREATE TABLE offboarding_cases (
    id                      UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id         UUID NOT NULL REFERENCES organisations(id),
    employee_id             UUID NOT NULL REFERENCES employees(id),
    initiated_by            UUID NOT NULL REFERENCES users(id),
    departure_reason_id     UUID NOT NULL REFERENCES departure_reasons(id),
    departure_date          DATE NOT NULL,           -- last working day
    notice_date             DATE,                    -- date notice was given
    access_cutoff_at        TIMESTAMPTZ,             -- when all access must be revoked
    status                  VARCHAR(50) NOT NULL DEFAULT 'draft',
        -- draft, pending_approval, approved, in_progress, access_revoked, completed, cancelled
    priority                VARCHAR(20) NOT NULL DEFAULT 'normal',  -- urgent, high, normal, low
    risk_level              VARCHAR(20),             -- high, medium, low (computed from role/access)
    notes                   TEXT,
    approved_by             UUID REFERENCES users(id),
    approved_at             TIMESTAMPTZ,
    access_revoked_at       TIMESTAMPTZ,             -- timestamp when all access confirmed revoked
    completed_at            TIMESTAMPTZ,
    cancelled_at            TIMESTAMPTZ,
    cancellation_reason     TEXT,
    created_at              TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at              TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_cases_org_status ON offboarding_cases(organisation_id, status);
CREATE INDEX idx_cases_employee ON offboarding_cases(employee_id);
CREATE INDEX idx_cases_departure_date ON offboarding_cases(departure_date);
```

## Checklist & Task Management

```sql
CREATE TABLE checklist_templates (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL REFERENCES organisations(id),
    name            VARCHAR(255) NOT NULL,
    description     TEXT,
    is_default      BOOLEAN NOT NULL DEFAULT false,
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Conditions under which a template applies (e.g., department=Engineering AND employment_type=full_time)
CREATE TABLE checklist_template_conditions (
    id                      UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    checklist_template_id   UUID NOT NULL REFERENCES checklist_templates(id) ON DELETE CASCADE,
    field                   VARCHAR(100) NOT NULL,   -- department, employment_type, jurisdiction, role_seniority
    operator                VARCHAR(20) NOT NULL,    -- equals, not_equals, in, contains
    value                   VARCHAR(255) NOT NULL,
    created_at              TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE task_definitions (
    id                      UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    checklist_template_id   UUID NOT NULL REFERENCES checklist_templates(id) ON DELETE CASCADE,
    title                   VARCHAR(500) NOT NULL,
    description             TEXT,
    responsible_department   VARCHAR(100) NOT NULL,   -- hr, it, finance, facilities, legal, manager
    sort_order              INTEGER NOT NULL DEFAULT 0,
    due_offset_days         INTEGER NOT NULL DEFAULT 0,   -- days relative to departure_date (negative = before)
    is_required             BOOLEAN NOT NULL DEFAULT true,
    is_automatable          BOOLEAN NOT NULL DEFAULT false,
    automation_action       VARCHAR(100),            -- revoke_saas_access, generate_final_pay, send_exit_survey
    created_at              TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_task_defs_template ON task_definitions(checklist_template_id);

CREATE TABLE offboarding_tasks (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    offboarding_case_id UUID NOT NULL REFERENCES offboarding_cases(id) ON DELETE CASCADE,
    task_definition_id  UUID REFERENCES task_definitions(id),  -- NULL if ad hoc task
    title               VARCHAR(500) NOT NULL,
    description         TEXT,
    responsible_department VARCHAR(100) NOT NULL,
    assigned_to         UUID REFERENCES users(id),
    status              VARCHAR(50) NOT NULL DEFAULT 'pending',
        -- pending, in_progress, completed, skipped, failed, blocked
    due_date            DATE,
    completed_at        TIMESTAMPTZ,
    completed_by        UUID REFERENCES users(id),
    failure_reason      TEXT,
    sort_order          INTEGER NOT NULL DEFAULT 0,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_tasks_case ON offboarding_tasks(offboarding_case_id);
CREATE INDEX idx_tasks_status ON offboarding_tasks(status);
CREATE INDEX idx_tasks_assigned ON offboarding_tasks(assigned_to, status);
```

## SaaS Application & Access Management

```sql
CREATE TABLE saas_applications (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id     UUID NOT NULL REFERENCES organisations(id),
    name                VARCHAR(255) NOT NULL,
    slug                VARCHAR(100) NOT NULL,
    category            VARCHAR(100),             -- productivity, engineering, communication, finance, security
    icon_url            VARCHAR(500),
    scim_endpoint       VARCHAR(500),             -- SCIM 2.0 base URL if supported
    scim_auth_token_enc TEXT,                     -- encrypted SCIM bearer token
    supports_scim       BOOLEAN NOT NULL DEFAULT false,
    api_type            VARCHAR(50),              -- scim, rest, graphql, manual
    deprovisioning_method VARCHAR(50) NOT NULL DEFAULT 'manual',
        -- scim_deactivate, scim_delete, api_custom, manual
    risk_level          VARCHAR(20) NOT NULL DEFAULT 'medium',  -- critical, high, medium, low
    is_active           BOOLEAN NOT NULL DEFAULT true,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_saas_apps_org ON saas_applications(organisation_id);
CREATE UNIQUE INDEX idx_saas_apps_slug ON saas_applications(organisation_id, slug);

CREATE TABLE saas_connectors (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    saas_application_id UUID NOT NULL REFERENCES saas_applications(id) ON DELETE CASCADE,
    connector_type      VARCHAR(50) NOT NULL,     -- scim, oauth_api, webhook, manual
    config_encrypted    TEXT NOT NULL,             -- encrypted JSON with API keys, endpoints, scopes
    status              VARCHAR(50) NOT NULL DEFAULT 'active',  -- active, error, disabled
    last_sync_at        TIMESTAMPTZ,
    last_error          TEXT,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE employee_app_access (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    employee_id         UUID NOT NULL REFERENCES employees(id),
    saas_application_id UUID NOT NULL REFERENCES saas_applications(id),
    external_user_id    VARCHAR(255),             -- user ID in the external app
    username            VARCHAR(255),             -- username in the external app
    access_level        VARCHAR(100),             -- admin, member, viewer, owner
    discovery_method    VARCHAR(50) NOT NULL DEFAULT 'manual',
        -- sso_log, scim_sync, manual, ai_discovery, browser_extension
    is_shadow_it        BOOLEAN NOT NULL DEFAULT false,
    granted_at          TIMESTAMPTZ,
    status              VARCHAR(50) NOT NULL DEFAULT 'active',  -- active, suspended, revoked, unknown
    revoked_at          TIMESTAMPTZ,
    revoked_by          UUID REFERENCES users(id),
    revocation_method   VARCHAR(50),              -- scim_patch, scim_delete, api_call, manual
    revocation_error    TEXT,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_emp_access_employee ON employee_app_access(employee_id);
CREATE INDEX idx_emp_access_app ON employee_app_access(saas_application_id);
CREATE INDEX idx_emp_access_status ON employee_app_access(status);
CREATE UNIQUE INDEX idx_emp_access_unique ON employee_app_access(employee_id, saas_application_id)
    WHERE status = 'active';

CREATE TABLE oauth_token_grants (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    employee_id         UUID NOT NULL REFERENCES employees(id),
    saas_application_id UUID REFERENCES saas_applications(id),  -- NULL if unknown app
    app_name_discovered VARCHAR(255),             -- app name from discovery (may not match catalogue)
    scopes              TEXT[],
    grant_type          VARCHAR(50),              -- authorization_code, implicit, client_credentials
    discovery_source    VARCHAR(50) NOT NULL,     -- email_scan, calendar_scan, idp_log, browser_extension
    discovered_at       TIMESTAMPTZ NOT NULL DEFAULT now(),
    token_revoked       BOOLEAN NOT NULL DEFAULT false,
    revoked_at          TIMESTAMPTZ,
    revocation_method   VARCHAR(50),              -- rfc7009, idp_admin, manual
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_oauth_grants_employee ON oauth_token_grants(employee_id);
```

## Access Revocation Actions

```sql
CREATE TABLE access_revocation_actions (
    id                      UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    offboarding_case_id     UUID NOT NULL REFERENCES offboarding_cases(id),
    employee_app_access_id  UUID REFERENCES employee_app_access(id),
    oauth_token_grant_id    UUID REFERENCES oauth_token_grants(id),
    action_type             VARCHAR(50) NOT NULL,
        -- scim_deactivate, scim_delete, api_revoke, oauth_token_revoke, manual_revoke
    status                  VARCHAR(50) NOT NULL DEFAULT 'pending',
        -- pending, in_progress, completed, failed, retry_scheduled
    request_payload         TEXT,                  -- sanitised request sent to external API
    response_status_code    INTEGER,
    response_body           TEXT,
    error_message           TEXT,
    retry_count             INTEGER NOT NULL DEFAULT 0,
    max_retries             INTEGER NOT NULL DEFAULT 3,
    next_retry_at           TIMESTAMPTZ,
    executed_at             TIMESTAMPTZ,
    completed_at            TIMESTAMPTZ,
    created_at              TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_revocation_case ON access_revocation_actions(offboarding_case_id);
CREATE INDEX idx_revocation_status ON access_revocation_actions(status);
CREATE INDEX idx_revocation_retry ON access_revocation_actions(next_retry_at) WHERE status = 'retry_scheduled';
```

## Document Management

```sql
CREATE TABLE documents (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    offboarding_case_id UUID NOT NULL REFERENCES offboarding_cases(id),
    document_type       VARCHAR(100) NOT NULL,
        -- separation_agreement, nda_reminder, final_acknowledgement, exit_checklist, equipment_receipt
    title               VARCHAR(500) NOT NULL,
    file_path           VARCHAR(1000),            -- storage path (S3, local)
    file_size_bytes     BIGINT,
    mime_type           VARCHAR(100),
    requires_signature  BOOLEAN NOT NULL DEFAULT false,
    signed_at           TIMESTAMPTZ,
    signed_by_employee  BOOLEAN NOT NULL DEFAULT false,
    signed_by_employer  BOOLEAN NOT NULL DEFAULT false,
    uploaded_by         UUID REFERENCES users(id),
    status              VARCHAR(50) NOT NULL DEFAULT 'pending',
        -- pending, sent, viewed, signed, expired, rejected
    sent_at             TIMESTAMPTZ,
    expires_at          TIMESTAMPTZ,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_documents_case ON documents(offboarding_case_id);
CREATE INDEX idx_documents_status ON documents(status);
```

## Equipment & Asset Tracking

```sql
CREATE TABLE equipment (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id     UUID NOT NULL REFERENCES organisations(id),
    employee_id         UUID REFERENCES employees(id),
    asset_tag           VARCHAR(100),
    equipment_type      VARCHAR(100) NOT NULL,    -- laptop, phone, monitor, badge, key, parking_pass
    make                VARCHAR(255),
    model               VARCHAR(255),
    serial_number       VARCHAR(255),
    status              VARCHAR(50) NOT NULL DEFAULT 'assigned',
        -- available, assigned, return_pending, return_shipped, returned, lost, written_off
    assigned_at         TIMESTAMPTZ,
    return_requested_at TIMESTAMPTZ,
    return_tracking_number VARCHAR(255),
    returned_at         TIMESTAMPTZ,
    condition_on_return VARCHAR(50),              -- good, damaged, missing
    notes               TEXT,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_equipment_org ON equipment(organisation_id);
CREATE INDEX idx_equipment_employee ON equipment(employee_id);
CREATE INDEX idx_equipment_status ON equipment(status);
```

## Exit Interviews

```sql
CREATE TABLE exit_survey_templates (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL REFERENCES organisations(id),
    name            VARCHAR(255) NOT NULL,
    is_default      BOOLEAN NOT NULL DEFAULT false,
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE exit_survey_questions (
    id                      UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    exit_survey_template_id UUID NOT NULL REFERENCES exit_survey_templates(id) ON DELETE CASCADE,
    question_text           TEXT NOT NULL,
    question_type           VARCHAR(50) NOT NULL,   -- free_text, rating_1_5, multiple_choice, yes_no
    options                 TEXT[],                  -- for multiple_choice questions
    sort_order              INTEGER NOT NULL DEFAULT 0,
    is_required             BOOLEAN NOT NULL DEFAULT true,
    created_at              TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE exit_interviews (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    offboarding_case_id UUID NOT NULL REFERENCES offboarding_cases(id),
    template_id         UUID NOT NULL REFERENCES exit_survey_templates(id),
    status              VARCHAR(50) NOT NULL DEFAULT 'pending',
        -- pending, sent, in_progress, completed, declined
    sent_at             TIMESTAMPTZ,
    completed_at        TIMESTAMPTZ,
    ai_summary          TEXT,                     -- AI-generated summary of responses
    ai_themes           TEXT[],                   -- AI-extracted themes (e.g., 'manager_feedback', 'compensation')
    ai_sentiment_score  NUMERIC(3,2),             -- -1.00 to 1.00
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_exit_interviews_case ON exit_interviews(offboarding_case_id);

CREATE TABLE exit_interview_responses (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    exit_interview_id   UUID NOT NULL REFERENCES exit_interviews(id) ON DELETE CASCADE,
    question_id         UUID NOT NULL REFERENCES exit_survey_questions(id),
    response_text       TEXT,
    response_rating     INTEGER,
    response_choice     VARCHAR(255),
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_exit_responses_interview ON exit_interview_responses(exit_interview_id);
```

## Knowledge Transfer

```sql
CREATE TABLE knowledge_transfer_briefs (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    offboarding_case_id UUID NOT NULL REFERENCES offboarding_cases(id),
    status              VARCHAR(50) NOT NULL DEFAULT 'pending',
        -- pending, ai_generating, draft, sent_to_employee, in_progress, completed
    ai_generated_brief  TEXT,                     -- AI-generated knowledge transfer document
    employee_additions  TEXT,                     -- employee's manual additions
    successor_id        UUID REFERENCES employees(id),
    reviewed_by         UUID REFERENCES users(id),
    completed_at        TIMESTAMPTZ,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_kt_briefs_case ON knowledge_transfer_briefs(offboarding_case_id);

CREATE TABLE knowledge_transfer_items (
    id                      UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    knowledge_transfer_brief_id UUID NOT NULL REFERENCES knowledge_transfer_briefs(id) ON DELETE CASCADE,
    category                VARCHAR(100) NOT NULL,   -- project, process, relationship, credential, documentation
    title                   VARCHAR(500) NOT NULL,
    description             TEXT,
    discovery_source        VARCHAR(50),             -- ai_analysis, employee_input, manager_input
    priority                VARCHAR(20) NOT NULL DEFAULT 'medium',
    transfer_status         VARCHAR(50) NOT NULL DEFAULT 'pending',
        -- pending, documented, handed_over, verified
    assigned_to             UUID REFERENCES employees(id),  -- who receives this knowledge
    created_at              TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at              TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_kt_items_brief ON knowledge_transfer_items(knowledge_transfer_brief_id);
```

## HRIS Integrations

```sql
CREATE TABLE hris_connections (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id     UUID NOT NULL REFERENCES organisations(id),
    provider            VARCHAR(50) NOT NULL,     -- bamboohr, workday, rippling, merge_dev, manual
    api_endpoint        VARCHAR(500),
    auth_config_enc     TEXT,                     -- encrypted auth configuration
    webhook_secret_enc  TEXT,                     -- encrypted webhook verification secret
    sync_status         VARCHAR(50) NOT NULL DEFAULT 'active',
    last_sync_at        TIMESTAMPTZ,
    last_error          TEXT,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_hris_org ON hris_connections(organisation_id);

CREATE TABLE hris_webhook_events (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    hris_connection_id  UUID NOT NULL REFERENCES hris_connections(id),
    event_type          VARCHAR(100) NOT NULL,    -- employee.terminated, employee.status_changed
    payload_hash        VARCHAR(64) NOT NULL,     -- SHA-256 for deduplication
    payload             JSONB NOT NULL,           -- raw webhook payload
    processing_status   VARCHAR(50) NOT NULL DEFAULT 'received',
        -- received, processing, processed, failed, ignored
    offboarding_case_id UUID REFERENCES offboarding_cases(id),  -- created case, if any
    error_message       TEXT,
    received_at         TIMESTAMPTZ NOT NULL DEFAULT now(),
    processed_at        TIMESTAMPTZ
);
CREATE INDEX idx_hris_events_status ON hris_webhook_events(processing_status);
CREATE UNIQUE INDEX idx_hris_events_dedup ON hris_webhook_events(hris_connection_id, payload_hash);
```

## GDPR & Data Retention

```sql
CREATE TABLE data_deletion_requests (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    offboarding_case_id UUID NOT NULL REFERENCES offboarding_cases(id),
    employee_id         UUID NOT NULL REFERENCES employees(id),
    requested_at        TIMESTAMPTZ NOT NULL DEFAULT now(),
    deadline_at         TIMESTAMPTZ NOT NULL,     -- GDPR: within 1 month of request
    status              VARCHAR(50) NOT NULL DEFAULT 'pending',
        -- pending, in_progress, partially_completed, completed, rejected
    rejection_reason    TEXT,                     -- legal hold, tax retention requirement, etc.
    data_categories_deleted TEXT[],               -- personal_contact, access_logs, exit_survey, etc.
    data_categories_retained TEXT[],              -- with legal justification
    retention_justification TEXT,
    completed_at        TIMESTAMPTZ,
    completed_by        UUID REFERENCES users(id),
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_deletion_requests_status ON data_deletion_requests(status);
CREATE INDEX idx_deletion_requests_deadline ON data_deletion_requests(deadline_at) WHERE status = 'pending';

CREATE TABLE data_retention_policies (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id     UUID NOT NULL REFERENCES organisations(id),
    jurisdiction_id     UUID REFERENCES jurisdictions(id),
    data_category       VARCHAR(100) NOT NULL,    -- payroll_records, access_logs, exit_surveys, personal_contact
    retention_years     INTEGER NOT NULL,
    legal_basis         TEXT,                     -- e.g., "UK tax law requires 6 years payroll retention"
    auto_purge          BOOLEAN NOT NULL DEFAULT false,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE UNIQUE INDEX idx_retention_policies ON data_retention_policies(organisation_id, jurisdiction_id, data_category);
```

## Audit Log

```sql
CREATE TABLE audit_log (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id     UUID NOT NULL REFERENCES organisations(id),
    offboarding_case_id UUID REFERENCES offboarding_cases(id),
    actor_id            UUID REFERENCES users(id),    -- NULL for system actions
    actor_type          VARCHAR(50) NOT NULL DEFAULT 'user',  -- user, system, hris_webhook, ai_agent
    action              VARCHAR(100) NOT NULL,
        -- case.created, case.approved, task.completed, access.revoked, document.signed, etc.
    target_type         VARCHAR(100),             -- offboarding_case, offboarding_task, employee_app_access, document
    target_id           UUID,
    details             TEXT,
    ip_address          INET,
    user_agent          VARCHAR(500),
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_audit_org ON audit_log(organisation_id);
CREATE INDEX idx_audit_case ON audit_log(offboarding_case_id);
CREATE INDEX idx_audit_action ON audit_log(action);
CREATE INDEX idx_audit_created ON audit_log(created_at);
-- Partition by month for large deployments:
-- CREATE TABLE audit_log (...) PARTITION BY RANGE (created_at);

CREATE TABLE notifications (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id     UUID NOT NULL REFERENCES organisations(id),
    recipient_id        UUID NOT NULL REFERENCES users(id),
    offboarding_case_id UUID REFERENCES offboarding_cases(id),
    channel             VARCHAR(50) NOT NULL DEFAULT 'in_app',  -- in_app, email, slack, webhook
    title               VARCHAR(500) NOT NULL,
    body                TEXT,
    is_read             BOOLEAN NOT NULL DEFAULT false,
    sent_at             TIMESTAMPTZ NOT NULL DEFAULT now(),
    read_at             TIMESTAMPTZ
);
CREATE INDEX idx_notifications_recipient ON notifications(recipient_id, is_read);
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Organisation & Auth | 3 | `organisations`, `org_members`, `users` |
| Reference Data | 3 | `jurisdictions`, `departure_reasons`, `departments` |
| Employee Management | 1 | `employees` |
| Offboarding Cases | 1 | `offboarding_cases` |
| Checklist & Tasks | 4 | Templates, conditions, definitions, instance tasks |
| SaaS & Access Management | 5 | Apps, connectors, access records, OAuth grants, revocation actions |
| Documents | 1 | `documents` |
| Equipment | 1 | `equipment` |
| Exit Interviews | 4 | Templates, questions, interviews, responses |
| Knowledge Transfer | 2 | Briefs and individual items |
| HRIS Integration | 2 | Connections and webhook events |
| GDPR & Retention | 2 | Deletion requests and retention policies |
| Audit & Notifications | 2 | Audit log and notifications |
| **Total** | **31** | |

---

## Key Design Decisions

1. **Separate `employees` from `users`**: Employees are the subjects of offboarding; users are the people operating the platform (HR admins, IT admins). A departing employee is not necessarily a platform user.

2. **Template-instance pattern for checklists**: `checklist_templates` / `task_definitions` define reusable templates; `offboarding_tasks` are concrete instances generated for each case. This allows template evolution without affecting active cases.

3. **Explicit `access_revocation_actions` table**: Every API call to an external system (SCIM PATCH, OAuth revoke, custom API) is tracked as a separate record with status, retry count, and response data. This provides the granular audit evidence SOC 2 and ISO 27001 auditors require.

4. **Shadow IT modelled alongside official access**: `employee_app_access.is_shadow_it` and `discovery_method` distinguish between IT-provisioned access and AI-discovered access. Both go through the same revocation workflow.

5. **HRIS webhook events stored raw**: The `hris_webhook_events` table preserves the exact payload from upstream HRIS systems, with deduplication via SHA-256 hash. This provides an immutable integration audit trail.

6. **Jurisdictions as a first-class entity**: Rather than embedding jurisdiction rules in code, the `jurisdictions` table with `data_retention_policies` allows runtime configuration of jurisdiction-specific compliance requirements.

7. **AI outputs stored alongside human data**: Exit interview AI summaries, knowledge transfer briefs, and AI-discovered OAuth grants are stored in the same tables as human-generated data, with `discovery_source` or `ai_*` columns distinguishing provenance.

8. **Equipment tracking integrated into the case**: The `equipment` table links directly to employees, and equipment return becomes an offboarding task. This avoids requiring a separate asset management system for the common case.

9. **Audit log designed for partitioning**: The `audit_log` table is append-only with a `created_at` timestamp suitable for PostgreSQL range partitioning in high-volume deployments.

10. **Multi-tenant via `organisation_id` foreign keys**: Shared-table multi-tenancy with `organisation_id` on every tenant-scoped table. Row-level security policies can be added for additional isolation.
