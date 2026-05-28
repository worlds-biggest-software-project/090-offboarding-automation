# Data Model Suggestion 3: Hybrid Relational + JSONB

> Project: Offboarding Automation · Created: 2026-05-12

## Philosophy

This model uses relational tables with fixed columns for core, universal fields that every offboarding case shares, and JSONB columns for variable, domain-specific data that changes by jurisdiction, departure reason, connector type, or organisation preference. The approach acknowledges that offboarding requirements vary significantly across jurisdictions (GDPR erasure workflows in the EU vs. WARN Act compliance in the US vs. minimal requirements in other regions) and across organisations (a 50-person startup and a 10,000-person enterprise have fundamentally different checklist needs).

Rather than modelling every possible variation as a separate table or column (which leads to sparse, migration-heavy schemas), the hybrid approach puts the stable skeleton in relational columns (with indexes, foreign keys, and constraints) and the variable flesh in JSONB (with runtime validation via JSON Schema). This is the pattern used by modern platforms like Stripe (metadata fields), Linear (custom properties), and Shopify (metafields) — products that serve diverse customers from a single schema.

This approach is particularly well-suited for an MVP/early-stage product where requirements are evolving rapidly. New jurisdiction-specific fields, new connector configurations, and new checklist items can be added without database migrations — just update the JSON Schema validation and the UI form.

**Best for:** Early-stage products, multi-jurisdiction deployments, and teams that want rapid iteration without constant database migrations. Ideal when the platform must accommodate diverse customer configurations from day one.

**Trade-offs:**
- Pro: Lowest migration burden — new fields are added to JSONB, not to table schemas
- Pro: Jurisdiction-specific and connector-specific data modelled naturally without sparse columns
- Pro: Fewer tables (~20) than the fully normalised approach
- Pro: JSONB is fully indexable in PostgreSQL (GIN indexes, expression indexes)
- Pro: Fastest path to MVP — schema changes do not require deployment
- Con: JSONB fields are not self-documenting — requires external JSON Schema definitions and documentation
- Con: No foreign key constraints inside JSONB — referential integrity must be enforced in application code
- Con: Complex JSONB queries can be slower than indexed relational columns for large datasets
- Con: Reporting tools may not understand JSONB structure without custom view layers
- Con: Risk of schema drift if JSON Schema validation is not strictly enforced

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| SCIM 2.0 (RFC 7643) | Connector configs stored as JSONB with SCIM-specific fields (`scim_endpoint`, `scim_auth_token`, `user_schema_mapping`) |
| ISO/IEC 27001:2022 | `audit_entries` table provides structured audit evidence; JSONB `details` field accommodates variable audit data per action type |
| NIST SP 800-53 AC-2 / PS-4 | Compliance tracking via `offboarding_cases.compliance_data` JSONB field storing per-control evidence |
| GDPR Article 17 | Jurisdiction-specific erasure requirements stored in `jurisdictions.gdpr_config` JSONB |
| ISO 3166 | `jurisdictions` table uses ISO 3166 codes; jurisdiction-specific rules in JSONB |
| JSON Schema 2020-12 | All JSONB columns validated against registered JSON Schema definitions at the application layer |
| OpenAPI 3.1 | API responses document JSONB field structures using OpenAPI schema references |
| SOC 2 CC6.2/CC6.3 | Audit entries with JSONB details provide flexible but complete compliance evidence |

---

## Core Tables

```sql
CREATE TABLE organisations (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            VARCHAR(255) NOT NULL,
    slug            VARCHAR(100) NOT NULL UNIQUE,
    plan_tier       VARCHAR(50) NOT NULL DEFAULT 'free',
    settings        JSONB NOT NULL DEFAULT '{}',
    -- settings example:
    -- {
    --   "default_access_cutoff_hours": 24,
    --   "require_approval_for_offboarding": true,
    --   "auto_discover_shadow_it": true,
    --   "exit_survey_mandatory": false,
    --   "notification_channels": ["email", "slack"],
    --   "data_retention_default_years": 3
    -- }
    branding        JSONB NOT NULL DEFAULT '{}',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE users (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL REFERENCES organisations(id),
    email           VARCHAR(320) NOT NULL UNIQUE,
    name            VARCHAR(255) NOT NULL,
    role            VARCHAR(50) NOT NULL DEFAULT 'member',  -- admin, hr_manager, it_admin, finance, facilities, member
    permissions     JSONB NOT NULL DEFAULT '[]',
    -- permissions example:
    -- ["cases.create", "cases.approve", "access.revoke", "reports.compliance", "settings.manage"]
    sso_provider    VARCHAR(50),
    sso_external_id VARCHAR(255),
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_users_org ON users(organisation_id);

CREATE TABLE jurisdictions (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    country_code    CHAR(2) NOT NULL,            -- ISO 3166-1 alpha-2
    subdivision_code VARCHAR(6),                  -- ISO 3166-2
    name            VARCHAR(255) NOT NULL,
    rules           JSONB NOT NULL DEFAULT '{}',
    -- rules example (US-CA):
    -- {
    --   "notice_period_days": 0,
    --   "final_pay_deadline": "same_day_if_fired_72h_if_quit",
    --   "final_pay_deadline_days": 0,
    --   "warn_act_threshold": 75,
    --   "gdpr_applicable": false,
    --   "data_retention_years": 4,
    --   "required_documents": ["final_pay_stub", "cobra_notice"],
    --   "required_offboarding_steps": ["final_pay_calculation", "cobra_notification", "edd_reporting"]
    -- }
    --
    -- rules example (DE - Germany):
    -- {
    --   "notice_period_days": 28,
    --   "final_pay_deadline_days": 30,
    --   "gdpr_applicable": true,
    --   "gdpr_erasure_deadline_days": 30,
    --   "data_retention_years": 6,
    --   "works_council_notification_required": true,
    --   "required_documents": ["arbeitszeugnis", "lohnsteuerbescheinigung"],
    --   "required_offboarding_steps": ["works_council_notification", "final_pay_calculation", "tax_certificate", "reference_letter"]
    -- }
    UNIQUE (country_code, subdivision_code)
);
```

## Employees

```sql
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
    -- profile example:
    -- {
    --   "employee_number": "EMP-1234",
    --   "cost_center": "ENG-001",
    --   "division": "Engineering",
    --   "office_location": "San Francisco",
    --   "phone": "+1-555-0100",
    --   "personal_email": "jane.personal@email.com",
    --   "seniority_level": "senior",
    --   "is_people_manager": true,
    --   "direct_reports_count": 5,
    --   "custom_fields": {
    --     "badge_number": "B-5678",
    --     "parking_spot": "A-23"
    --   }
    -- }
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_employees_org ON employees(organisation_id);
CREATE INDEX idx_employees_status ON employees(organisation_id, status);
CREATE INDEX idx_employees_hris ON employees(hris_source, hris_external_id);
CREATE INDEX idx_employees_dept ON employees(organisation_id, department);
```

## Offboarding Cases

```sql
CREATE TABLE offboarding_cases (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id     UUID NOT NULL REFERENCES organisations(id),
    employee_id         UUID NOT NULL REFERENCES employees(id),
    initiated_by        UUID NOT NULL REFERENCES users(id),
    departure_reason    VARCHAR(50) NOT NULL,     -- voluntary, involuntary, redundancy, end_of_contract, retirement
    departure_date      DATE NOT NULL,
    notice_date         DATE,
    access_cutoff_at    TIMESTAMPTZ,
    status              VARCHAR(50) NOT NULL DEFAULT 'draft',
    priority            VARCHAR(20) NOT NULL DEFAULT 'normal',
    risk_level          VARCHAR(20),
    approved_by         UUID REFERENCES users(id),
    approved_at         TIMESTAMPTZ,
    completed_at        TIMESTAMPTZ,
    case_data           JSONB NOT NULL DEFAULT '{}',
    -- case_data example:
    -- {
    --   "risk_factors": ["admin_access_to_production", "access_to_customer_data", "people_manager"],
    --   "jurisdiction_steps": [
    --     {"step": "cobra_notification", "status": "pending", "due_date": "2026-06-01"},
    --     {"step": "final_pay_calculation", "status": "completed", "completed_at": "2026-05-10"}
    --   ],
    --   "final_pay": {
    --     "accrued_pto_days": 12,
    --     "accrued_pto_value": 4615.38,
    --     "severance_weeks": 0,
    --     "final_pay_date": "2026-06-15"
    --   },
    --   "notes": "Employee requested alumni programme opt-in",
    --   "cancellation_reason": null
    -- }
    compliance_data     JSONB NOT NULL DEFAULT '{}',
    -- compliance_data example:
    -- {
    --   "iso_27001": {
    --     "a_5_18_access_revoked": true,
    --     "a_6_5_post_termination_communicated": true,
    --     "evidence_exported_at": "2026-06-16T10:00:00Z"
    --   },
    --   "nist_800_53": {
    --     "ac_2_accounts_disabled": true,
    --     "ps_4_access_revoked_within_sla": true,
    --     "time_to_full_revocation_hours": 2.3
    --   },
    --   "gdpr": {
    --     "erasure_requested": false,
    --     "erasure_deadline": null,
    --     "data_categories_retained": ["payroll", "audit_log"],
    --     "retention_legal_basis": "UK tax law (6 years)"
    --   }
    -- }
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_cases_org_status ON offboarding_cases(organisation_id, status);
CREATE INDEX idx_cases_employee ON offboarding_cases(employee_id);
CREATE INDEX idx_cases_departure ON offboarding_cases(departure_date);
CREATE INDEX idx_cases_compliance ON offboarding_cases USING GIN (compliance_data);
```

## Tasks

```sql
CREATE TABLE checklist_templates (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL REFERENCES organisations(id),
    name            VARCHAR(255) NOT NULL,
    description     TEXT,
    conditions      JSONB NOT NULL DEFAULT '{}',
    -- conditions example:
    -- {
    --   "match_all": [
    --     {"field": "department", "operator": "equals", "value": "Engineering"},
    --     {"field": "employment_type", "operator": "in", "values": ["full_time", "part_time"]}
    --   ]
    -- }
    tasks           JSONB NOT NULL,
    -- tasks: array of task definitions (title, dept, due_offset_days, automatable, action, etc.)
    is_default      BOOLEAN NOT NULL DEFAULT false,
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE offboarding_tasks (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    offboarding_case_id UUID NOT NULL REFERENCES offboarding_cases(id) ON DELETE CASCADE,
    title               VARCHAR(500) NOT NULL,
    description         TEXT,
    responsible_dept    VARCHAR(100) NOT NULL,
    assigned_to         UUID REFERENCES users(id),
    status              VARCHAR(50) NOT NULL DEFAULT 'pending',
    due_date            DATE,
    sort_order          INTEGER NOT NULL DEFAULT 0,
    completed_at        TIMESTAMPTZ,
    completed_by        UUID REFERENCES users(id),
    task_data           JSONB NOT NULL DEFAULT '{}',
    -- task_data example (for an automated revocation task):
    -- {
    --   "automation_action": "revoke_saas_access",
    --   "target_app_id": "uuid",
    --   "target_app_name": "GitHub",
    --   "revocation_method": "scim_deactivate",
    --   "execution_log": [
    --     {"attempt": 1, "status": "failed", "error": "timeout", "at": "2026-05-12T10:00:00Z"},
    --     {"attempt": 2, "status": "success", "response_code": 200, "at": "2026-05-12T10:05:00Z"}
    --   ]
    -- }
    -- task_data example (for a manual task):
    -- {
    --   "template_task_id": "uuid",
    --   "failure_reason": null,
    --   "notes": "Laptop collected from desk; monitor left for replacement employee"
    -- }
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_tasks_case ON offboarding_tasks(offboarding_case_id);
CREATE INDEX idx_tasks_assigned ON offboarding_tasks(assigned_to, status);
CREATE INDEX idx_tasks_status ON offboarding_tasks(status);
```

## SaaS Access & Connectors

```sql
CREATE TABLE saas_applications (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id     UUID NOT NULL REFERENCES organisations(id),
    name                VARCHAR(255) NOT NULL,
    slug                VARCHAR(100) NOT NULL,
    category            VARCHAR(100),
    risk_level          VARCHAR(20) NOT NULL DEFAULT 'medium',
    is_active           BOOLEAN NOT NULL DEFAULT true,
    connector_config    JSONB NOT NULL DEFAULT '{}',
    -- connector_config example (SCIM app):
    -- {
    --   "type": "scim",
    --   "scim_version": "2.0",
    --   "base_url": "https://api.github.com/scim/v2/organizations/acme",
    --   "auth": {
    --     "method": "bearer_token",
    --     "token_encrypted": "enc:abc123..."
    --   },
    --   "deprovisioning": {
    --     "method": "patch_active_false",
    --     "transfer_data_before_delete": true,
    --     "data_transfer_endpoint": "/admin/transfer"
    --   },
    --   "user_schema_mapping": {
    --     "external_id_field": "userName",
    --     "email_field": "emails[0].value"
    --   }
    -- }
    --
    -- connector_config example (OAuth API app):
    -- {
    --   "type": "oauth_api",
    --   "api_base_url": "https://www.googleapis.com/admin/directory/v1",
    --   "auth": {
    --     "method": "oauth2_service_account",
    --     "client_id": "...",
    --     "credentials_encrypted": "enc:xyz789..."
    --   },
    --   "deprovisioning": {
    --     "method": "custom_api",
    --     "steps": [
    --       {"action": "suspend_user", "endpoint": "/users/{userId}", "method": "PATCH", "body": {"suspended": true}},
    --       {"action": "transfer_drive", "endpoint": "/datatransfer/v1/transfers", "method": "POST"},
    --       {"action": "delete_user", "endpoint": "/users/{userId}", "method": "DELETE", "delay_hours": 24}
    --     ]
    --   }
    -- }
    --
    -- connector_config example (manual app):
    -- {
    --   "type": "manual",
    --   "instructions": "Log into admin console at https://app.example.com/admin, navigate to Users, search for the employee, click Deactivate.",
    --   "admin_url": "https://app.example.com/admin/users"
    -- }
    sync_status         VARCHAR(50) DEFAULT 'active',
    last_sync_at        TIMESTAMPTZ,
    last_error          TEXT,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE UNIQUE INDEX idx_saas_slug ON saas_applications(organisation_id, slug);
CREATE INDEX idx_saas_org ON saas_applications(organisation_id);

CREATE TABLE employee_app_access (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    employee_id         UUID NOT NULL REFERENCES employees(id),
    saas_application_id UUID NOT NULL REFERENCES saas_applications(id),
    offboarding_case_id UUID REFERENCES offboarding_cases(id),
    external_user_id    VARCHAR(255),
    access_level        VARCHAR(100),
    discovery_method    VARCHAR(50) NOT NULL DEFAULT 'manual',
    is_shadow_it        BOOLEAN NOT NULL DEFAULT false,
    status              VARCHAR(50) NOT NULL DEFAULT 'active',
    revoked_at          TIMESTAMPTZ,
    access_data         JSONB NOT NULL DEFAULT '{}',
    -- access_data example:
    -- {
    --   "username": "jsmith",
    --   "groups": ["engineering", "team-backend"],
    --   "roles": ["admin"],
    --   "oauth_scopes": ["repo", "admin:org"],
    --   "last_login_at": "2026-05-01T09:00:00Z",
    --   "discovery_details": {
    --     "source": "sso_log",
    --     "discovered_at": "2026-05-12T08:00:00Z",
    --     "confidence": 0.95
    --   },
    --   "revocation": {
    --     "method": "scim_deactivate",
    --     "request_at": "2026-05-12T10:00:00Z",
    --     "response_status": 200,
    --     "retry_count": 0,
    --     "confirmed_at": "2026-05-12T10:00:01Z"
    --   }
    -- }
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_access_employee ON employee_app_access(employee_id);
CREATE INDEX idx_access_case ON employee_app_access(offboarding_case_id);
CREATE INDEX idx_access_status ON employee_app_access(status);
CREATE UNIQUE INDEX idx_access_unique ON employee_app_access(employee_id, saas_application_id)
    WHERE status = 'active';
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
    -- interview_data example:
    -- {
    --   "template_id": "uuid",
    --   "template_name": "Standard Exit Survey",
    --   "responses": [
    --     {
    --       "question_id": "uuid",
    --       "question": "What is your primary reason for leaving?",
    --       "type": "free_text",
    --       "response": "Found a better growth opportunity elsewhere"
    --     },
    --     {
    --       "question_id": "uuid",
    --       "question": "Rate your relationship with your manager (1-5)",
    --       "type": "rating_1_5",
    --       "response": 2
    --     }
    --   ],
    --   "ai_analysis": {
    --     "summary": "Employee cites lack of career growth and poor manager relationship...",
    --     "themes": ["career_growth", "manager_feedback", "compensation"],
    --     "sentiment_score": -0.35,
    --     "retention_risk_factors": ["manager_relationship", "promotion_timeline"],
    --     "recommended_actions": ["Review promotion criteria for Engineering dept"]
    --   }
    -- }
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_exit_case ON exit_interviews(offboarding_case_id);
-- GIN index for querying AI themes across all interviews
CREATE INDEX idx_exit_themes ON exit_interviews USING GIN ((interview_data->'ai_analysis'->'themes'));

CREATE TABLE knowledge_transfers (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    offboarding_case_id UUID NOT NULL REFERENCES offboarding_cases(id),
    status              VARCHAR(50) NOT NULL DEFAULT 'pending',
    successor_id        UUID REFERENCES employees(id),
    completed_at        TIMESTAMPTZ,
    transfer_data       JSONB NOT NULL DEFAULT '{}',
    -- transfer_data example:
    -- {
    --   "ai_brief": "Jane is the primary maintainer of the billing service...",
    --   "items": [
    --     {
    --       "id": "uuid",
    --       "category": "project",
    --       "title": "Billing service ownership",
    --       "description": "Primary maintainer; knows the Stripe integration deeply",
    --       "priority": "critical",
    --       "status": "handed_over",
    --       "assigned_to": "uuid",
    --       "source": "ai_analysis"
    --     },
    --     {
    --       "id": "uuid",
    --       "category": "relationship",
    --       "title": "Key contact at Acme Corp",
    --       "description": "Monthly sync with their CTO on integration roadmap",
    --       "priority": "high",
    --       "status": "documented",
    --       "source": "employee_input"
    --     }
    --   ],
    --   "employee_additions": "Also have notes on the legacy data migration scripts in /docs/migration"
    -- }
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
    -- doc_data example:
    -- {
    --   "requires_signature": true,
    --   "signed_by_employee": true,
    --   "signed_at": "2026-05-14T11:00:00Z",
    --   "signed_by_employer": false,
    --   "employer_signer_id": "uuid",
    --   "expires_at": "2026-06-14T11:00:00Z",
    --   "file_size_bytes": 245678
    -- }
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
    -- equipment_data example:
    -- {
    --   "asset_tag": "LAP-2024-0567",
    --   "make": "Apple",
    --   "model": "MacBook Pro 16\" M3 Max",
    --   "serial_number": "C02XY1234567",
    --   "assigned_at": "2024-01-15",
    --   "return_tracking": {
    --     "shipping_label_url": "https://fedex.com/...",
    --     "tracking_number": "7961 0000 1234",
    --     "shipped_at": "2026-05-14",
    --     "received_at": null
    --   },
    --   "condition_on_return": null,
    --   "notes": null
    -- }
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_equip_org ON equipment(organisation_id);
CREATE INDEX idx_equip_employee ON equipment(employee_id);
```

## HRIS Integration & Audit

```sql
CREATE TABLE hris_connections (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL REFERENCES organisations(id),
    provider        VARCHAR(50) NOT NULL,
    connection_config JSONB NOT NULL DEFAULT '{}',
    -- connection_config example:
    -- {
    --   "api_endpoint": "https://api.bamboohr.com/api/gateway.php/acme/v1",
    --   "auth": {
    --     "method": "api_key",
    --     "api_key_encrypted": "enc:abc123..."
    --   },
    --   "webhook": {
    --     "enabled": true,
    --     "endpoint": "/webhooks/hris/bamboohr",
    --     "secret_encrypted": "enc:xyz789...",
    --     "events": ["employmentStatus.changed"]
    --   },
    --   "field_mapping": {
    --     "employee_id": "employeeId",
    --     "email": "workEmail",
    --     "status": "employmentStatus",
    --     "departure_date": "terminationDate"
    --   }
    -- }
    is_active       BOOLEAN NOT NULL DEFAULT true,
    last_sync_at    TIMESTAMPTZ,
    last_error      TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_hris_org ON hris_connections(organisation_id);

CREATE TABLE webhook_events (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    hris_connection_id  UUID REFERENCES hris_connections(id),
    source              VARCHAR(50) NOT NULL,     -- bamboohr, workday, rippling, internal
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

CREATE TABLE audit_entries (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id     UUID NOT NULL,
    offboarding_case_id UUID,
    actor_id            UUID,
    actor_type          VARCHAR(50) NOT NULL DEFAULT 'user',
    action              VARCHAR(100) NOT NULL,
    target_type         VARCHAR(100),
    target_id           UUID,
    details             JSONB NOT NULL DEFAULT '{}',
    -- details example:
    -- {
    --   "app_name": "GitHub Enterprise",
    --   "revocation_method": "scim_deactivate",
    --   "response_status": 200,
    --   "duration_ms": 342,
    --   "previous_state": "active",
    --   "new_state": "revoked"
    -- }
    ip_address          INET,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_audit_org ON audit_entries(organisation_id);
CREATE INDEX idx_audit_case ON audit_entries(offboarding_case_id);
CREATE INDEX idx_audit_action ON audit_entries(action);
CREATE INDEX idx_audit_created ON audit_entries(created_at);
-- GIN index for searching audit details
CREATE INDEX idx_audit_details ON audit_entries USING GIN (details);
```

## JSON Schema Validation Registry

```sql
-- Application-layer schema validation for all JSONB columns
CREATE TABLE json_schema_registry (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    schema_name     VARCHAR(200) NOT NULL UNIQUE,
    -- e.g., "organisations.settings.v1", "saas_applications.connector_config.scim.v1"
    version         INTEGER NOT NULL DEFAULT 1,
    json_schema     JSONB NOT NULL,              -- JSON Schema 2020-12 definition
    description     TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

---

## Example Queries

### Find all offboarding cases where GDPR erasure was requested

```sql
SELECT id, employee_id, departure_date,
       compliance_data->'gdpr'->>'erasure_requested' AS erasure_requested,
       compliance_data->'gdpr'->>'erasure_deadline' AS erasure_deadline
FROM offboarding_cases
WHERE organisation_id = '...'
  AND (compliance_data->'gdpr'->>'erasure_requested')::boolean = true
  AND status != 'completed';
```

### Exit interview theme frequency across the organisation

```sql
SELECT theme, COUNT(*) AS frequency
FROM exit_interviews,
     jsonb_array_elements_text(interview_data->'ai_analysis'->'themes') AS theme
WHERE offboarding_case_id IN (
    SELECT id FROM offboarding_cases
    WHERE organisation_id = '...'
      AND departure_date >= '2025-12-01'
)
  AND status = 'completed'
GROUP BY theme
ORDER BY frequency DESC;
```

### SCIM connector health check

```sql
SELECT name, slug,
       connector_config->>'type' AS connector_type,
       connector_config->'deprovisioning'->>'method' AS deprov_method,
       sync_status, last_sync_at, last_error
FROM saas_applications
WHERE organisation_id = '...'
  AND connector_config->>'type' = 'scim'
  AND sync_status != 'active';
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Organisation & Auth | 2 | `organisations`, `users` |
| Reference Data | 1 | `jurisdictions` (rules in JSONB) |
| Employees | 1 | Profile details in JSONB |
| Offboarding Cases | 1 | Case-specific and compliance data in JSONB |
| Tasks | 2 | Templates (task defs in JSONB), instance tasks |
| SaaS & Access | 2 | Connector config in JSONB, access details in JSONB |
| Exit & Knowledge | 2 | Responses and AI analysis in JSONB |
| Documents & Equipment | 2 | Variable fields in JSONB |
| Integration & Audit | 3 | HRIS config in JSONB, audit details in JSONB |
| Schema Registry | 1 | JSON Schema validation definitions |
| **Total** | **17** | ~45% fewer tables than normalized model |

---

## Key Design Decisions

1. **JSONB for variable, relational for stable**: Every table has relational columns for fields that are universal (status, foreign keys, timestamps) and JSONB columns for fields that vary by type, jurisdiction, or customer configuration. The boundary is explicit: if you need a foreign key or a unique constraint, it is relational; if it varies by context, it is JSONB.

2. **Connector configuration as JSONB**: Each SaaS app connector has fundamentally different configuration (SCIM endpoints vs. OAuth credentials vs. manual instructions). Rather than a connector-type-per-table approach, a single `connector_config` JSONB field with type-discriminated schemas keeps the table count low and makes adding new connector types trivial.

3. **Compliance data as a structured JSONB field**: `offboarding_cases.compliance_data` stores per-framework compliance evidence (ISO 27001, NIST, GDPR) as a structured JSON object. This allows different compliance frameworks to be tracked without schema changes, and new frameworks can be added at runtime.

4. **Exit interview responses co-located with AI analysis**: Rather than separate tables for questions, responses, and AI analysis, the `interview_data` JSONB field contains the complete interview record. This makes the interview a self-contained document that can be rendered, exported, or processed without JOINs.

5. **JSON Schema registry for validation**: Since JSONB fields are not self-documenting, the `json_schema_registry` table stores JSON Schema 2020-12 definitions for every JSONB column. Application code validates all JSONB writes against the registered schema, preventing drift.

6. **GIN indexes on frequently queried JSONB paths**: Exit interview themes, audit details, and compliance data use GIN indexes for containment queries. Expression indexes (e.g., `CREATE INDEX ON offboarding_cases ((compliance_data->'gdpr'->>'erasure_requested'))`) can be added for hot query paths.

7. **Audit entries with JSONB details**: The `audit_entries` table has fixed columns for common query filters (org, case, action, timestamp) and a JSONB `details` field for action-specific data. This avoids the "1000-column audit table" anti-pattern while keeping the audit trail fully queryable.

8. **Fewer tables, more flexibility**: 17 tables vs. 31 in the normalized model. The trade-off is explicit: referential integrity inside JSONB is the application's responsibility, not the database's. For an early-stage product iterating rapidly, this is usually the right trade-off.

9. **Jurisdiction rules as JSONB**: The `jurisdictions.rules` field stores all jurisdiction-specific offboarding rules (notice periods, final pay deadlines, required documents, GDPR config) as structured JSON. Adding support for a new jurisdiction is a data insert, not a schema migration.

10. **Equipment and documents use thin JSONB**: These tables have few relational columns (type, status, foreign keys) with a JSONB field for the long tail of attributes (serial numbers, shipping tracking, signature status). This keeps the schema lean while still supporting full-text search and reporting on the relational columns.
