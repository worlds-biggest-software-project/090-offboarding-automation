# Offboarding Automation - Development Plan

> Project: Offboarding Automation (Candidate #90)
> Created: 2026-05-25
> Status: Phased development plan

---

## Technology Decisions

### Data Model: Hybrid Relational + JSONB (Suggestion 3) with Dual-Write Audit Ledger (from Suggestion 4)

**Rationale:** The project combines elements from Data Model Suggestion 3 (Hybrid Relational + JSONB) and Data Model Suggestion 4 (Workflow-Engine / Dual-Write Audit). The pure normalised approach (Suggestion 1, 31 tables) imposes excessive migration burden during early development. Full event sourcing (Suggestion 2) introduces eventual-consistency complexity that is unnecessary for an MVP. The hybrid JSONB model (Suggestion 3, 17 tables) provides the fastest iteration path, while the dual-write audit ledger from Suggestion 4 delivers the tamper-proof compliance evidence that SOC 2, ISO 27001, and NIST auditors require -- without the complexity of CQRS projections.

The state machine definitions from Suggestion 4 are adopted for the offboarding case lifecycle and revocation job queue, but scoped narrowly: only the offboarding case and revocation jobs use the workflow engine pattern. Individual tasks use a simpler status column, avoiding the overhead of making every entity a workflow instance.

**Final schema target: ~20 tables.**

### Language & Framework

| Component | Technology | Rationale |
|-----------|-----------|-----------|
| **Backend API** | TypeScript + Node.js (Fastify) | Fastify offers superior performance over Express, native TypeScript support, and built-in JSON Schema validation for request/response -- directly supporting JSONB schema enforcement. The TypeScript ecosystem has the broadest SaaS SDK coverage (Okta, Google, Microsoft Graph, Slack, GitHub, AWS). |
| **Database** | PostgreSQL 16 | JSONB with GIN indexes, `SELECT FOR UPDATE SKIP LOCKED` for the job queue, range partitioning for the audit ledger, row-level security for multi-tenancy. No external queue dependency needed. |
| **ORM / Query Builder** | Drizzle ORM | Type-safe SQL with explicit queries (no magic). Supports JSONB columns, migrations, and PostgreSQL-specific features. Lighter than Prisma; more type-safe than Knex. |
| **Frontend** | React 19 + Next.js 15 (App Router) | Server Components for dashboard views, Server Actions for form submissions, built-in API routes for webhook endpoints. shadcn/ui for the component library. |
| **Authentication** | NextAuth.js v5 (Auth.js) | Supports OIDC/SAML for enterprise SSO (Okta, Entra ID, Google Workspace). Session management aligns with the platform's identity-first architecture. |
| **Background Jobs** | pg-boss (PostgreSQL-backed) | Job queue built on PostgreSQL `SKIP LOCKED`. No Redis or external queue infrastructure. Handles retries, timeouts, and scheduling -- directly supporting the revocation job queue and scheduled SLA checks. |
| **AI Integration** | Anthropic Claude API (via @anthropic-ai/sdk) | Exit interview synthesis, knowledge transfer brief generation, SaaS access discovery analysis. Prompt caching for repeated schema analysis patterns. |
| **File Storage** | S3-compatible (AWS S3 / MinIO for self-hosted) | Separation documents, export archives, audit evidence packages. MinIO provides the self-hosted option required by the README's deployment model. |
| **Encryption** | Node.js `crypto` (AES-256-GCM) for secrets at rest | SCIM tokens, OAuth credentials, API keys stored encrypted in JSONB connector configs. Encryption key managed via environment variable or KMS. |
| **Testing** | Vitest (unit/integration) + Playwright (E2E) | Vitest is the standard for modern TypeScript projects. Playwright for end-to-end offboarding workflow testing across the UI. |
| **API Documentation** | OpenAPI 3.1 (auto-generated from Fastify schemas) | Fastify natively produces OpenAPI specs from route schemas. Webhook schemas documented using OpenAPI 3.1 `webhooks` element, as recommended by the standards research. |
| **Containerisation** | Docker + Docker Compose | Single `docker compose up` for local development (API, PostgreSQL, MinIO). Production deployment via container orchestration (Kubernetes manifests provided). |

### Project Structure

```
offboarding-automation/
  apps/
    web/                         # Next.js 15 frontend
      src/
        app/                     # App Router pages
          (auth)/                # Login, SSO callback
          (dashboard)/           # Main application
            cases/               # Offboarding case list + detail
            tasks/               # Task board (my tasks, department view)
            access/              # SaaS access revocation dashboard
            compliance/          # Audit log, compliance reports
            exit-interviews/     # Exit survey management + AI insights
            knowledge/           # Knowledge transfer briefs
            settings/            # Org settings, connectors, templates
          api/                   # Next.js API routes (webhook receivers)
        components/              # shadcn/ui components
        lib/                     # Client-side utilities
    api/                         # Fastify backend API
      src/
        routes/                  # API route handlers
        services/                # Business logic layer
        connectors/              # SaaS connector implementations
          scim/                  # Generic SCIM 2.0 client
          google-workspace/
          slack/
          github/
          microsoft365/
          aws-iam/
          jira/
          notion/
        workflows/               # State machine engine + definitions
        ai/                      # AI service (Claude API integration)
        jobs/                    # pg-boss job handlers (revocation, scheduled)
        db/                      # Drizzle schema, migrations, queries
        webhooks/                # Inbound webhook processors (HRIS events)
        crypto/                  # Encryption utilities for secrets at rest
        validators/              # JSON Schema validation for JSONB fields
  packages/
    shared/                      # Shared types, constants, utilities
    connector-sdk/               # SDK for building custom SaaS connectors
  docker/
    docker-compose.yml
    Dockerfile.api
    Dockerfile.web
  docs/
    openapi.yaml                 # Generated OpenAPI 3.1 spec
    connector-guide.md           # How to build a custom connector
```

---

## Phase Dependency Graph

```
Phase 1: Foundation
    |
    v
Phase 2: Offboarding Case Engine ──────────────────────┐
    |                                                    |
    v                                                    v
Phase 3: SaaS Connectors & Revocation     Phase 4: Checklist & Task Management
    |                                          |
    └──────────────┬───────────────────────────┘
                   |
                   v
Phase 5: HRIS Webhooks & Integration
                   |
                   v
Phase 6: Documents, Equipment & Compliance Reporting
                   |
                   v
Phase 7: Exit Interviews & AI Synthesis
                   |
                   v
Phase 8: Knowledge Transfer & AI Discovery
                   |
                   v
Phase 9: Multi-Jurisdiction & GDPR
                   |
                   v
Phase 10: Analytics, Alumni & Polish
```

**Legend:** Arrows indicate hard dependencies. Phase 3 and Phase 4 can proceed in parallel after Phase 2 is complete. All subsequent phases depend on the merge point after Phases 3+4.

---

## Phase 1: Foundation

**Goal:** Establish the project skeleton, database schema, authentication, and deployment infrastructure. At the end of this phase, a developer can clone the repo, run `docker compose up`, and see a working login screen with multi-tenant organisation support.

**Definition of Done:** A developer can register an account, create an organisation, invite a team member, log in via email/password and SSO stub, and see an empty dashboard. All tables exist in PostgreSQL. CI runs tests green. Docker Compose starts all services.

### Task 1.1: Repository & Tooling Setup

**What:** Initialise the monorepo with apps/web (Next.js 15), apps/api (Fastify), and packages/shared. Configure TypeScript, ESLint, Prettier, Vitest, and Docker Compose with PostgreSQL 16 and MinIO.

**Design:**

```
# Root package.json (pnpm workspaces)
{
  "private": true,
  "workspaces": ["apps/*", "packages/*"]
}

# docker/docker-compose.yml
services:
  postgres:
    image: postgres:16-alpine
    environment:
      POSTGRES_DB: offboarding
      POSTGRES_USER: offboarding
      POSTGRES_PASSWORD: dev_password
    ports: ["5432:5432"]
    volumes: ["pgdata:/var/lib/postgresql/data"]

  minio:
    image: minio/minio:latest
    command: server /data --console-address ":9001"
    environment:
      MINIO_ROOT_USER: minioadmin
      MINIO_ROOT_PASSWORD: minioadmin
    ports: ["9000:9000", "9001:9001"]

  api:
    build: { context: .., dockerfile: docker/Dockerfile.api }
    depends_on: [postgres]
    environment:
      DATABASE_URL: postgres://offboarding:dev_password@postgres:5432/offboarding
    ports: ["3001:3001"]

  web:
    build: { context: .., dockerfile: docker/Dockerfile.web }
    depends_on: [api]
    ports: ["3000:3000"]
```

**Testing:**
- `pnpm install` succeeds with zero errors
- `pnpm lint` passes across all workspaces
- `docker compose up` starts all 4 services; `curl http://localhost:3001/health` returns `200`
- `pnpm test` runs Vitest suite (currently 0 tests, but the runner works)

### Task 1.2: Database Schema & Migrations (Core Tables)

**What:** Create Drizzle ORM schema definitions and migration files for the core tables: `organisations`, `users`, `jurisdictions`, `employees`. Include the `audit_ledger` (append-only) from Phase 1 so all subsequent phases can write audit entries.

**Design:**

```typescript
// apps/api/src/db/schema/organisations.ts
import { pgTable, uuid, varchar, jsonb, timestamp, boolean } from 'drizzle-orm/pg-core';

export const organisations = pgTable('organisations', {
  id: uuid('id').primaryKey().defaultRandom(),
  name: varchar('name', { length: 255 }).notNull(),
  slug: varchar('slug', { length: 100 }).notNull().unique(),
  planTier: varchar('plan_tier', { length: 50 }).notNull().default('free'),
  settings: jsonb('settings').notNull().default({}),
  createdAt: timestamp('created_at', { withTimezone: true }).notNull().defaultNow(),
  updatedAt: timestamp('updated_at', { withTimezone: true }).notNull().defaultNow(),
});

// apps/api/src/db/schema/audit-ledger.ts
export const auditLedger = pgTable('audit_ledger', {
  id: bigint('id', { mode: 'number' }).primaryKey().generatedAlwaysAsIdentity(),
  organisationId: uuid('organisation_id').notNull(),
  offboardingCaseId: uuid('offboarding_case_id'),
  eventType: varchar('event_type', { length: 100 }).notNull(),
  fromState: varchar('from_state', { length: 50 }),
  toState: varchar('to_state', { length: 50 }),
  triggerEvent: varchar('trigger_event', { length: 100 }),
  actorId: uuid('actor_id'),
  actorType: varchar('actor_type', { length: 50 }).notNull().default('user'),
  details: jsonb('details').notNull().default({}),
  ipAddress: varchar('ip_address', { length: 45 }),
  idempotencyKey: varchar('idempotency_key', { length: 255 }),
  createdAt: timestamp('created_at', { withTimezone: true }).notNull().defaultNow(),
});
```

**Testing:**
- `pnpm drizzle-kit push` applies migrations without error
- `SELECT * FROM organisations` returns empty result set
- `INSERT INTO audit_ledger (...)` succeeds; `UPDATE audit_ledger` fails (trigger enforces append-only)
- Migration rollback (`pnpm drizzle-kit drop`) cleanly removes all tables
- Unit test: Drizzle schema types compile correctly

### Task 1.3: Authentication & Authorisation

**What:** Implement NextAuth.js v5 with email/password credentials provider (for development) and a stub OIDC provider (Okta/Entra ID). Implement role-based access control (admin, hr_manager, it_admin, finance, facilities, member). Session tokens stored in the database.

**Design:**

```typescript
// apps/web/src/app/api/auth/[...nextauth]/route.ts
import NextAuth from 'next-auth';
import Credentials from 'next-auth/providers/credentials';
import Okta from 'next-auth/providers/okta';

export const { handlers, auth, signIn, signOut } = NextAuth({
  providers: [
    Credentials({
      credentials: { email: {}, password: {} },
      authorize: async (credentials) => {
        // Verify against users table; bcrypt compare
      },
    }),
    Okta({
      clientId: process.env.OKTA_CLIENT_ID,
      clientSecret: process.env.OKTA_CLIENT_SECRET,
      issuer: process.env.OKTA_ISSUER,
    }),
  ],
  callbacks: {
    jwt({ token, user }) {
      if (user) {
        token.organisationId = user.organisationId;
        token.role = user.role;
      }
      return token;
    },
    session({ session, token }) {
      session.user.organisationId = token.organisationId;
      session.user.role = token.role;
      return session;
    },
  },
});

// apps/api/src/middleware/authorise.ts
type Permission = 'cases.create' | 'cases.approve' | 'access.revoke' |
                  'reports.compliance' | 'settings.manage' | 'templates.edit';

const ROLE_PERMISSIONS: Record<string, Permission[]> = {
  admin: ['cases.create', 'cases.approve', 'access.revoke', 'reports.compliance', 'settings.manage', 'templates.edit'],
  hr_manager: ['cases.create', 'cases.approve', 'reports.compliance', 'templates.edit'],
  it_admin: ['access.revoke', 'reports.compliance'],
  finance: ['reports.compliance'],
  member: [],
};

export function requirePermission(permission: Permission) {
  return async (request: FastifyRequest, reply: FastifyReply) => {
    const userRole = request.user.role;
    if (!ROLE_PERMISSIONS[userRole]?.includes(permission)) {
      return reply.status(403).send({ error: 'Insufficient permissions' });
    }
  };
}
```

**Testing:**
- E2E: Register new user -> redirected to dashboard
- E2E: Login with valid credentials -> session cookie set, dashboard loads
- E2E: Login with invalid credentials -> error message displayed
- Unit: `requirePermission('cases.approve')` returns 403 for `member` role
- Unit: `requirePermission('cases.approve')` returns 200 for `hr_manager` role
- Integration: SSO stub login creates user record with `sso_provider` and `sso_external_id` populated

### Task 1.4: Multi-Tenant Organisation Management

**What:** Implement organisation CRUD, member invitation (via email), and the organisation switcher in the UI. All API queries are scoped to the authenticated user's `organisation_id`.

**Design:**

```typescript
// apps/api/src/services/organisation-service.ts
export class OrganisationService {
  async create(name: string, creatorUserId: string): Promise<Organisation> {
    return db.transaction(async (tx) => {
      const org = await tx.insert(organisations).values({
        name,
        slug: slugify(name),
      }).returning();

      await tx.insert(orgMembers).values({
        organisationId: org[0].id,
        userId: creatorUserId,
        role: 'admin',
      });

      await tx.insert(auditLedger).values({
        organisationId: org[0].id,
        eventType: 'organisation.created',
        actorId: creatorUserId,
        actorType: 'user',
        details: { name },
      });

      return org[0];
    });
  }

  async inviteMember(orgId: string, email: string, role: string, inviterId: string): Promise<void> {
    // Generate invitation token, send email, create pending membership
  }
}
```

**Testing:**
- Integration: Create organisation -> verify `organisations` row, `org_members` row with `admin` role, and `audit_ledger` entry
- Integration: Invite member -> verify invitation email sent (mocked SMTP), pending membership created
- Integration: Accept invitation -> verify `org_members` row created, user can access org resources
- Unit: API queries for org A cannot return data from org B (tenant isolation)
- E2E: Organisation settings page renders, name can be updated

### Task 1.5: Seed Data & Jurisdiction Reference Data

**What:** Populate the `jurisdictions` table with initial reference data for the top 15 jurisdictions (US federal, US-CA, US-NY, US-TX, UK, DE, FR, NL, IE, AU, CA-ON, CA-BC, SG, JP, IN). Each row includes jurisdiction-specific offboarding rules as JSONB.

**Design:**

```typescript
// apps/api/src/db/seeds/jurisdictions.ts
export const jurisdictionSeeds = [
  {
    countryCode: 'US',
    subdivisionCode: 'CA',
    name: 'California, United States',
    rules: {
      notice_period_days: 0,
      final_pay_deadline: 'same_day_if_terminated_72h_if_resigned',
      final_pay_deadline_days_terminated: 0,
      final_pay_deadline_days_resigned: 3,
      warn_act_threshold: 75,
      gdpr_applicable: false,
      data_retention_years: 4,
      required_documents: ['final_pay_stub', 'cobra_notice', 'edd_pamphlet'],
      required_offboarding_steps: ['final_pay_calculation', 'cobra_notification', 'edd_reporting'],
    },
  },
  {
    countryCode: 'DE',
    subdivisionCode: null,
    name: 'Germany',
    rules: {
      notice_period_days: 28,
      final_pay_deadline_days: 30,
      gdpr_applicable: true,
      gdpr_erasure_deadline_days: 30,
      data_retention_years: 6,
      works_council_notification_required: true,
      required_documents: ['arbeitszeugnis', 'lohnsteuerbescheinigung'],
      required_offboarding_steps: ['works_council_notification', 'final_pay_calculation', 'tax_certificate', 'reference_letter'],
    },
  },
  // ... 13 more jurisdictions
];
```

**Testing:**
- Seed script runs idempotently (re-running does not create duplicates)
- `SELECT COUNT(*) FROM jurisdictions` returns 15 after seeding
- Query `SELECT rules->>'gdpr_applicable' FROM jurisdictions WHERE country_code = 'DE'` returns `true`
- Unit test: jurisdiction rules conform to the registered JSON Schema

---

## Phase 2: Offboarding Case Engine

**Goal:** Implement the core offboarding case lifecycle with state machine transitions, case creation, approval workflow, and the dual-write audit pattern. At the end of this phase, an HR manager can create an offboarding case for a departing employee, submit it for approval, approve it, and track it through the state machine.

**Definition of Done:** An offboarding case can be created, submitted, approved, and moved through the full state machine lifecycle. Every state transition is recorded in the audit ledger. The case detail page shows current status, timeline, and audit history.

### Task 2.1: State Machine Engine

**What:** Implement a generic state machine engine that validates transitions against definitions stored in the database, enforces guards, and performs the dual-write (update mutable state + append to audit ledger) in a single transaction.

**Design:**

```typescript
// apps/api/src/workflows/state-machine.ts
export interface StateMachineDefinition {
  name: string;
  version: number;
  initialState: string;
  states: Record<string, { type: 'initial' | 'intermediate' | 'active' | 'terminal' }>;
  transitions: Array<{
    from: string | string[];
    to: string;
    event: string;
    guards?: string[];
    sideEffects?: string[];
  }>;
}

export class StateMachineEngine {
  constructor(
    private db: DrizzleDB,
    private guardRegistry: Map<string, GuardFn>,
    private sideEffectRegistry: Map<string, SideEffectFn>,
  ) {}

  async transition(
    instanceId: string,
    event: string,
    context: { actorId: string; actorType: string; details: Record<string, unknown>; orgId: string; caseId?: string },
  ): Promise<{ success: boolean; newState: string }> {
    return this.db.transaction(async (tx) => {
      // 1. Load current state with FOR UPDATE lock
      const instance = await tx.query.workflowInstances.findFirst({
        where: eq(workflowInstances.id, instanceId),
        for: 'update',
      });
      if (!instance) throw new Error('Instance not found');

      // 2. Load state machine definition
      const definition = await this.getDefinition(tx, instance.stateMachineName, instance.stateMachineVersion);

      // 3. Find valid transition
      const transition = definition.transitions.find(t => {
        const fromStates = Array.isArray(t.from) ? t.from : [t.from];
        return fromStates.includes(instance.currentState) && t.event === event;
      });
      if (!transition) throw new InvalidTransitionError(instance.currentState, event);

      // 4. Evaluate guards
      for (const guardName of transition.guards ?? []) {
        const guard = this.guardRegistry.get(guardName);
        if (guard && !(await guard(instance, context))) {
          throw new GuardFailedError(guardName);
        }
      }

      // 5. Dual-write: update mutable state + append audit ledger
      await tx.update(workflowInstances)
        .set({ currentState: transition.to, updatedAt: new Date() })
        .where(and(
          eq(workflowInstances.id, instanceId),
          eq(workflowInstances.currentState, instance.currentState), // optimistic concurrency
        ));

      await tx.insert(auditLedger).values({
        organisationId: context.orgId,
        workflowInstanceId: instanceId,
        offboardingCaseId: context.caseId ?? instanceId,
        eventType: `state_transition.${instance.stateMachineName}`,
        fromState: instance.currentState,
        toState: transition.to,
        triggerEvent: event,
        actorId: context.actorId,
        actorType: context.actorType,
        details: context.details,
      });

      // 6. Execute side effects (outside the main transaction for non-critical actions)
      for (const effectName of transition.sideEffects ?? []) {
        const effect = this.sideEffectRegistry.get(effectName);
        if (effect) await effect(instance, transition.to, context);
      }

      return { success: true, newState: transition.to };
    });
  }
}
```

**Testing:**
- Unit: Valid transition `draft -> pending_approval` via `submit` event succeeds
- Unit: Invalid transition `draft -> completed` via `complete` event throws `InvalidTransitionError`
- Unit: Transition with failing guard throws `GuardFailedError`, state unchanged
- Integration: Dual-write produces both `workflow_instances` update and `audit_ledger` insert in the same transaction
- Integration: Concurrent transitions on the same instance -- one succeeds, the other fails gracefully (optimistic concurrency)
- Unit: State machine definition loading from database with version pinning

### Task 2.2: Offboarding Case CRUD & State Transitions

**What:** Implement the offboarding case API: create case (linked to employee and workflow instance), submit for approval, approve/reject, cancel. Case status derives from the workflow instance's `current_state`.

**Design:**

```typescript
// apps/api/src/routes/cases.ts
app.post('/api/v1/cases', {
  schema: {
    body: {
      type: 'object',
      required: ['employeeId', 'departureReason', 'departureDate'],
      properties: {
        employeeId: { type: 'string', format: 'uuid' },
        departureReason: { type: 'string', enum: ['voluntary', 'involuntary', 'redundancy', 'end_of_contract', 'retirement'] },
        departureDate: { type: 'string', format: 'date' },
        noticeDate: { type: 'string', format: 'date' },
        priority: { type: 'string', enum: ['urgent', 'high', 'normal', 'low'], default: 'normal' },
        notes: { type: 'string' },
      },
    },
  },
  preHandler: [requirePermission('cases.create')],
}, async (request, reply) => {
  const caseId = await caseService.create({
    ...request.body,
    organisationId: request.user.organisationId,
    initiatedBy: request.user.id,
  });
  return reply.status(201).send({ id: caseId });
});

app.post('/api/v1/cases/:id/submit', {
  preHandler: [requirePermission('cases.create')],
}, async (request, reply) => {
  await stateMachine.transition(request.params.id, 'submit', {
    actorId: request.user.id,
    actorType: 'user',
    orgId: request.user.organisationId,
    details: {},
  });
  return reply.send({ status: 'pending_approval' });
});

app.post('/api/v1/cases/:id/approve', {
  preHandler: [requirePermission('cases.approve')],
}, async (request, reply) => {
  await stateMachine.transition(request.params.id, 'approve', {
    actorId: request.user.id,
    actorType: 'user',
    orgId: request.user.organisationId,
    details: { approvedBy: request.user.name },
  });
  return reply.send({ status: 'approved' });
});
```

**Testing:**
- Integration: `POST /api/v1/cases` creates both `workflow_instances` and `offboarding_cases` rows
- Integration: Full lifecycle `create -> submit -> approve -> start` with correct state at each step
- Integration: Approving without `cases.approve` permission returns 403
- Integration: Submitting a case already in `pending_approval` returns 409 (invalid transition)
- Integration: Cancelling an `in_progress` case fails (not a valid transition from `in_progress`)
- Integration: Every state transition creates an `audit_ledger` entry with correct `from_state`, `to_state`, and actor info

### Task 2.3: Employee Management (CRUD)

**What:** Implement employee CRUD API scoped to the organisation. Employees are the subjects of offboarding -- not platform users. Support manual entry and pre-population from HRIS sync (Phase 5).

**Design:**

```typescript
// apps/api/src/routes/employees.ts
app.get('/api/v1/employees', {
  schema: {
    querystring: {
      type: 'object',
      properties: {
        status: { type: 'string', enum: ['active', 'offboarding', 'offboarded', 'archived'] },
        department: { type: 'string' },
        search: { type: 'string' },
        page: { type: 'integer', minimum: 1, default: 1 },
        limit: { type: 'integer', minimum: 1, maximum: 100, default: 25 },
      },
    },
  },
}, async (request, reply) => {
  const { status, department, search, page, limit } = request.query;
  const result = await employeeService.list({
    organisationId: request.user.organisationId,
    status, department, search, page, limit,
  });
  return reply.send(result);
});
```

**Testing:**
- Integration: Create employee -> verify row in database with correct `organisation_id`
- Integration: List employees filters correctly by status, department, and search (first_name, last_name, email)
- Integration: Employee from org A not visible from org B
- Unit: Employee profile JSONB validated against registered schema
- Integration: Update employee status to `offboarding` when a case is created for them

### Task 2.4: Case Dashboard UI

**What:** Build the offboarding cases list view and case detail page. List view shows all cases with status badges, departure dates, and progress indicators. Detail page shows the case timeline (from audit ledger), current status, employee info, and action buttons (submit, approve, cancel).

**Design:**

```tsx
// apps/web/src/app/(dashboard)/cases/page.tsx
export default async function CasesPage({ searchParams }: { searchParams: { status?: string } }) {
  const cases = await fetchCases(searchParams);
  return (
    <div>
      <PageHeader title="Offboarding Cases" action={<CreateCaseButton />} />
      <StatusFilter current={searchParams.status} />
      <CasesTable cases={cases} />
    </div>
  );
}

// apps/web/src/app/(dashboard)/cases/[id]/page.tsx
export default async function CaseDetailPage({ params }: { params: { id: string } }) {
  const caseData = await fetchCase(params.id);
  const timeline = await fetchCaseTimeline(params.id);
  return (
    <div className="grid grid-cols-3 gap-6">
      <div className="col-span-2">
        <CaseHeader case={caseData} />
        <CaseStatusBanner status={caseData.status} />
        <CaseActions case={caseData} />
        <CaseTimeline entries={timeline} />
      </div>
      <div>
        <EmployeeCard employee={caseData.employee} />
        <CaseMetadata case={caseData} />
      </div>
    </div>
  );
}
```

**Testing:**
- E2E: Navigate to /cases, verify table renders with mock data
- E2E: Click "New Case" -> fill form -> submit -> case appears in list with `draft` status
- E2E: Click on case -> detail page shows employee info, status, and timeline
- E2E: Click "Submit for Approval" -> status changes to `pending_approval`; timeline entry appears
- E2E: Approve case -> status changes to `approved`; approve button disappears
- E2E: Filter by status -> only matching cases displayed

---

## Phase 3: SaaS Connectors & Access Revocation

**Goal:** Implement the SaaS application registry, the generic SCIM 2.0 connector, connectors for the 7 MVP apps (Google Workspace, Microsoft 365, Slack, GitHub, Jira, AWS IAM, Notion), the employee access inventory, and the revocation job queue with retry logic.

**Definition of Done:** An IT admin can register SaaS applications, view an employee's discovered app access, and trigger automated access revocation. SCIM-based apps are revoked via SCIM PATCH `active=false`. Non-SCIM apps use custom API calls. All revocation attempts are logged in the audit ledger with response codes and timing.

### Task 3.1: SaaS Application Registry

**What:** Implement CRUD for `saas_applications` with JSONB `connector_config` supporting SCIM, OAuth API, and manual connector types. Include encrypted credential storage.

**Design:**

```typescript
// apps/api/src/services/saas-app-service.ts
export class SaaSAppService {
  async register(orgId: string, app: {
    name: string;
    slug: string;
    category: string;
    riskLevel: 'critical' | 'high' | 'medium' | 'low';
    connectorConfig: SCIMConfig | OAuthAPIConfig | ManualConfig;
  }): Promise<SaaSApplication> {
    // Encrypt sensitive fields in connectorConfig before storage
    const encryptedConfig = await this.encryptConnectorSecrets(app.connectorConfig);
    // Validate connectorConfig against the appropriate JSON Schema based on type
    await this.validateConnectorConfig(app.connectorConfig);
    return db.insert(saasApplications).values({
      organisationId: orgId,
      ...app,
      connectorConfig: encryptedConfig,
    }).returning();
  }

  async testConnection(appId: string): Promise<{ success: boolean; error?: string }> {
    const app = await this.getById(appId);
    const connector = ConnectorFactory.create(app.connectorConfig);
    return connector.testConnection();
  }
}
```

**Testing:**
- Integration: Register SCIM app -> verify connector_config stored with encrypted token
- Integration: Register manual app -> verify config stored with instructions
- Integration: Test connection against mock SCIM endpoint -> returns success
- Integration: Test connection against unreachable endpoint -> returns failure with error message
- Unit: Connector config validation rejects invalid SCIM config (missing base_url)
- Integration: List apps for org, sorted by risk level

### Task 3.2: Generic SCIM 2.0 Client

**What:** Implement a standards-compliant SCIM 2.0 client (RFC 7643/7644) that supports: `GET /Users` (list/search), `PATCH /Users/{id}` (deactivate: `active=false`), and `DELETE /Users/{id}` (hard delete). Handle SCIM error responses, rate limiting, and pagination.

**Design:**

```typescript
// apps/api/src/connectors/scim/scim-client.ts
export class SCIMClient {
  constructor(
    private baseUrl: string,
    private authToken: string,
    private options?: { timeoutMs?: number; retryOn429?: boolean },
  ) {}

  async deactivateUser(userId: string): Promise<SCIMResponse> {
    const response = await fetch(`${this.baseUrl}/Users/${userId}`, {
      method: 'PATCH',
      headers: {
        'Authorization': `Bearer ${this.authToken}`,
        'Content-Type': 'application/scim+json',
      },
      body: JSON.stringify({
        schemas: ['urn:ietf:params:scim:api:messages:2.0:PatchOp'],
        Operations: [{ op: 'replace', value: { active: false } }],
      }),
      signal: AbortSignal.timeout(this.options?.timeoutMs ?? 30000),
    });
    return this.parseResponse(response);
  }

  async deleteUser(userId: string): Promise<SCIMResponse> {
    const response = await fetch(`${this.baseUrl}/Users/${userId}`, {
      method: 'DELETE',
      headers: { 'Authorization': `Bearer ${this.authToken}` },
      signal: AbortSignal.timeout(this.options?.timeoutMs ?? 30000),
    });
    return this.parseResponse(response);
  }

  async findUserByEmail(email: string): Promise<SCIMUser | null> {
    const response = await fetch(
      `${this.baseUrl}/Users?filter=userName eq "${email}"`,
      { headers: { 'Authorization': `Bearer ${this.authToken}` } },
    );
    const result = await this.parseResponse(response);
    return result.Resources?.[0] ?? null;
  }
}
```

**Testing:**
- Unit: `deactivateUser` sends correct SCIM PATCH payload with `active: false`
- Unit: `deleteUser` sends DELETE with correct headers
- Unit: `findUserByEmail` URL-encodes the filter parameter
- Integration (mock server): Deactivate user -> 200 response parsed correctly
- Integration (mock server): Deactivate user -> 429 rate limit -> retried after backoff
- Integration (mock server): Deactivate user -> 500 error -> throws with parsed SCIM error detail
- Unit: Timeout after configured `timeoutMs`

### Task 3.3: App-Specific Connectors (7 MVP Apps)

**What:** Implement connector adapters for Google Workspace, Microsoft 365, Slack, GitHub Enterprise, Jira/Confluence, AWS IAM Identity Center, and Notion. Each connector extends a common interface and handles app-specific deprovisioning sequencing (e.g., Google: suspend -> transfer Drive -> delete).

**Design:**

```typescript
// apps/api/src/connectors/connector-interface.ts
export interface SaaSConnector {
  testConnection(): Promise<{ success: boolean; error?: string }>;
  findUser(email: string): Promise<ExternalUser | null>;
  deactivateUser(externalUserId: string): Promise<RevocationResult>;
  deleteUser(externalUserId: string): Promise<RevocationResult>;
  listUserAccess(externalUserId: string): Promise<AccessInfo>;
}

// apps/api/src/connectors/google-workspace/google-workspace-connector.ts
export class GoogleWorkspaceConnector implements SaaSConnector {
  async deactivateUser(userId: string): Promise<RevocationResult> {
    // Step 1: Suspend user (PATCH suspended: true)
    await this.adminClient.users.patch({ userKey: userId, requestBody: { suspended: true } });
    // Step 2: Transfer Drive ownership (if configured)
    if (this.config.transferDriveBeforeDelete) {
      await this.transferDriveFiles(userId, this.config.driveTransferRecipient);
    }
    return { success: true, method: 'google_admin_api', steps: ['suspended', 'drive_transferred'] };
  }
}

// apps/api/src/connectors/connector-factory.ts
export class ConnectorFactory {
  static create(config: ConnectorConfig): SaaSConnector {
    switch (config.type) {
      case 'scim': return new GenericSCIMConnector(config);
      case 'google_workspace': return new GoogleWorkspaceConnector(config);
      case 'microsoft_365': return new Microsoft365Connector(config);
      case 'slack': return new SlackConnector(config);
      case 'github': return new GitHubConnector(config);
      case 'jira': return new JiraConnector(config);
      case 'aws_iam': return new AWSIAMConnector(config);
      case 'notion': return new NotionConnector(config);
      case 'manual': return new ManualConnector(config);
      default: throw new Error(`Unknown connector type: ${config.type}`);
    }
  }
}
```

**Testing:**
- Unit (per connector): Mock API calls verify correct endpoint, method, headers, and body for each app
- Integration (mock server per app): Google Workspace suspend + Drive transfer sequence completes
- Integration (mock server): Slack SCIM deactivate preserves message history
- Integration (mock server): GitHub SCIM soft-deprovision (suspend) vs hard-deprovision (delete)
- Integration (mock server): AWS IAM sequence: remove group -> remove assignment -> delete user
- Unit: ConnectorFactory returns correct connector class for each config type
- Unit: Manual connector returns instructions text and admin URL

### Task 3.4: Revocation Job Queue

**What:** Implement the revocation job queue using pg-boss backed by PostgreSQL. When an offboarding case transitions to `in_progress`, revocation jobs are created for every app the employee has access to. Workers dequeue and execute revocation jobs with retry logic, exponential backoff, and dead-letter handling.

**Design:**

```typescript
// apps/api/src/jobs/revocation-worker.ts
import PgBoss from 'pg-boss';

export class RevocationWorker {
  constructor(private boss: PgBoss, private connectorFactory: ConnectorFactory) {}

  async start() {
    await this.boss.work('revoke-access', { teamConcurrency: 10 }, async (job) => {
      const { appId, employeeEmail, externalUserId, revocationMethod, caseId, orgId } = job.data;
      const app = await saasAppService.getById(appId);
      const connector = this.connectorFactory.create(app.connectorConfig);

      const startTime = Date.now();
      try {
        const result = await connector.deactivateUser(externalUserId);
        const durationMs = Date.now() - startTime;

        // Update access record
        await employeeAccessService.markRevoked(job.data.accessRecordId, revocationMethod);

        // Audit ledger entry
        await auditService.log({
          orgId,
          caseId,
          eventType: 'access.revoked',
          actorType: 'system',
          details: { appName: app.name, revocationMethod, durationMs, responseCode: result.statusCode },
        });
      } catch (error) {
        await auditService.log({
          orgId,
          caseId,
          eventType: 'access.revocation_failed',
          actorType: 'system',
          details: { appName: app.name, error: error.message, attempt: job.retrycount + 1 },
        });
        throw error; // pg-boss handles retry based on retryLimit and retryBackoff
      }
    });
  }
}

// Job creation when case starts
export async function createRevocationJobs(caseId: string, employeeId: string, orgId: string) {
  const accessRecords = await employeeAccessService.listActiveByEmployee(employeeId);
  for (const access of accessRecords) {
    await boss.send('revoke-access', {
      accessRecordId: access.id,
      appId: access.saasApplicationId,
      employeeEmail: access.employeeEmail,
      externalUserId: access.externalUserId,
      revocationMethod: access.app.connectorConfig.deprovisioning?.method ?? 'manual',
      caseId,
      orgId,
    }, {
      retryLimit: 3,
      retryBackoff: true, // exponential backoff
      expireInMinutes: 60,
      priority: access.app.riskLevel === 'critical' ? 1 : 5,
    });
  }
}
```

**Testing:**
- Integration: Case transition to `in_progress` creates one revocation job per app access record
- Integration: Worker dequeues job, calls connector, marks access as revoked, writes audit entry
- Integration: Failed revocation -> job retried up to 3 times with exponential backoff
- Integration: All retries exhausted -> job moves to dead letter queue; audit entry with `revocation_failed`
- Integration: Critical-risk apps processed before medium-risk apps (priority ordering)
- Integration: When all revocation jobs for a case complete, case transitions to `access_revoked`
- Unit: Job payload schema validation

### Task 3.5: Access Revocation Dashboard UI

**What:** Build the access revocation dashboard showing per-case app access inventory with real-time status (pending, in_progress, revoked, failed). Include manual revocation confirmation for manual connector apps.

**Testing:**
- E2E: Navigate to case -> access tab -> see list of apps with status badges
- E2E: SCIM app shows "Revoked" with green badge and timestamp
- E2E: Failed revocation shows red badge with error message and "Retry" button
- E2E: Manual app shows instructions panel with "Mark as Done" button
- E2E: Click "Retry" on failed revocation -> job requeued, status changes to "Pending"

---

## Phase 4: Checklist & Task Management

**Goal:** Implement configurable checklist templates, automatic task generation from templates when a case starts, task assignment across departments, and the task board UI.

**Definition of Done:** An HR admin can create checklist templates with conditions (by department, employment type, jurisdiction). When an offboarding case transitions to `in_progress`, tasks are automatically generated from the matching template. Department members see their assigned tasks and can complete or skip them.

### Task 4.1: Checklist Template Engine

**What:** Implement CRUD for checklist templates with condition-based matching (department, employment type, jurisdiction, seniority). Templates contain task definitions with due date offsets, responsible department, and automation flags.

**Design:**

```typescript
// apps/api/src/services/checklist-template-service.ts
export class ChecklistTemplateService {
  async findMatchingTemplate(orgId: string, employee: Employee): Promise<ChecklistTemplate> {
    const templates = await db.select().from(checklistTemplates)
      .where(and(
        eq(checklistTemplates.organisationId, orgId),
        eq(checklistTemplates.isActive, true),
      ))
      .orderBy(desc(checklistTemplates.isDefault));

    for (const template of templates) {
      if (this.evaluateConditions(template.conditions, employee)) {
        return template;
      }
    }
    // Fall back to default template
    return templates.find(t => t.isDefault) ?? throw new Error('No matching template found');
  }

  private evaluateConditions(
    conditions: { match_all: Array<{ field: string; operator: string; value: string | string[] }> },
    employee: Employee,
  ): boolean {
    return conditions.match_all.every(condition => {
      const employeeValue = this.getFieldValue(employee, condition.field);
      switch (condition.operator) {
        case 'equals': return employeeValue === condition.value;
        case 'in': return (condition.value as string[]).includes(employeeValue);
        case 'not_equals': return employeeValue !== condition.value;
        default: return false;
      }
    });
  }
}
```

**Testing:**
- Integration: Create template with conditions `department=Engineering AND employment_type=full_time`
- Integration: Matching engine selects correct template for Engineering full-time employee
- Integration: Matching engine falls back to default template when no conditions match
- Unit: Condition evaluator handles equals, in, not_equals operators correctly
- Integration: Template CRUD respects organisation scoping

### Task 4.2: Automatic Task Generation

**What:** When a case transitions to `in_progress`, generate concrete `offboarding_tasks` rows from the matching template's task definitions. Calculate due dates based on the departure date and each task's `due_offset_days`.

**Design:**

```typescript
// apps/api/src/services/task-generation-service.ts
export class TaskGenerationService {
  async generateTasksForCase(caseId: string): Promise<void> {
    const offboardingCase = await caseService.getById(caseId);
    const employee = await employeeService.getById(offboardingCase.employeeId);
    const template = await templateService.findMatchingTemplate(offboardingCase.organisationId, employee);

    const taskDefinitions = template.tasks as TaskDefinition[];
    const tasks = taskDefinitions.map(def => ({
      offboardingCaseId: caseId,
      title: def.title,
      description: def.description,
      responsibleDept: def.dept,
      dueDate: addDays(offboardingCase.departureDate, def.due_offset_days),
      sortOrder: def.sort_order,
      taskData: {
        templateTaskId: def.id,
        automationAction: def.automatable ? def.action : null,
        automationConfig: def.automation_config ?? null,
      },
    }));

    await db.insert(offboardingTasks).values(tasks);
    await auditService.log({
      orgId: offboardingCase.organisationId,
      caseId,
      eventType: 'tasks.generated',
      actorType: 'system',
      details: { templateId: template.id, taskCount: tasks.length },
    });
  }
}
```

**Testing:**
- Integration: Case start generates correct number of tasks from the matching template
- Integration: Due dates calculated correctly (departure date + offset; negative offsets = days before)
- Integration: Automated tasks have `taskData.automationAction` populated
- Integration: Task generation logged in audit ledger
- Unit: `addDays` handles negative offsets and weekend boundaries correctly

### Task 4.3: Task Board UI & Task Completion

**What:** Build the task board showing tasks grouped by department (HR, IT, Finance, Facilities, Legal) and by status. Users see tasks assigned to their department. Tasks can be completed, skipped (with reason), or marked as failed.

**Design:**

```tsx
// apps/web/src/app/(dashboard)/tasks/page.tsx
export default async function TaskBoardPage() {
  const tasks = await fetchMyTasks();
  const grouped = groupBy(tasks, 'responsibleDept');
  return (
    <div>
      <PageHeader title="My Tasks" />
      <TaskStatusSummary tasks={tasks} />
      {Object.entries(grouped).map(([dept, deptTasks]) => (
        <TaskDepartmentGroup key={dept} department={dept} tasks={deptTasks} />
      ))}
    </div>
  );
}
```

**Testing:**
- E2E: Task board shows tasks grouped by department
- E2E: Complete a task -> status changes, completed timestamp recorded
- E2E: Skip a task -> modal asks for reason, status changes to skipped
- E2E: Task due date highlighting (overdue = red, due today = yellow)
- E2E: Filter tasks by status (pending, in_progress, completed, overdue)
- Integration: Completing all tasks for a case triggers case state evaluation

---

## Phase 5: HRIS Webhooks & Integration

**Goal:** Implement inbound webhook receivers for BambooHR, Workday, and Rippling departure events. Support Merge.dev as a unified HRIS adapter. When a departure event arrives, automatically create an offboarding case.

**Definition of Done:** A termination event from BambooHR, Workday, or Rippling (or via Merge.dev) creates an offboarding case in `draft` status, with the employee record pre-populated from the HRIS data. Webhooks are authenticated, deduplicated, and logged.

### Task 5.1: Webhook Receiver Framework

**What:** Build a generic webhook receiver that handles signature verification, payload parsing, deduplication (SHA-256 hash), and dispatching to provider-specific handlers.

**Design:**

```typescript
// apps/api/src/webhooks/webhook-receiver.ts
export class WebhookReceiver {
  async receive(provider: string, rawBody: Buffer, headers: Record<string, string>): Promise<void> {
    // 1. Look up HRIS connection by provider
    const connection = await hrisConnectionService.getByProvider(provider);

    // 2. Verify webhook signature
    const handler = WebhookHandlerFactory.create(provider);
    if (!handler.verifySignature(rawBody, headers, connection.connectionConfig.webhook.secret)) {
      throw new WebhookAuthError('Invalid webhook signature');
    }

    // 3. Parse payload
    const payload = JSON.parse(rawBody.toString());

    // 4. Deduplicate via SHA-256 hash
    const payloadHash = createHash('sha256').update(rawBody).digest('hex');
    const existing = await db.select().from(webhookEvents).where(
      and(eq(webhookEvents.source, provider), eq(webhookEvents.payloadHash, payloadHash))
    );
    if (existing.length > 0) return; // Already processed

    // 5. Store raw event
    const eventId = await db.insert(webhookEvents).values({
      hrisConnectionId: connection.id,
      source: provider,
      eventType: handler.extractEventType(payload),
      payload,
      payloadHash,
      processingStatus: 'received',
    }).returning();

    // 6. Process asynchronously
    await boss.send('process-hris-webhook', { eventId: eventId[0].id });
  }
}
```

**Testing:**
- Integration: Valid BambooHR webhook with correct HMAC signature -> event stored, processing queued
- Integration: Invalid signature -> 401 response, no event stored
- Integration: Duplicate payload (same hash) -> 200 response, no new event stored
- Integration: Malformed JSON -> 400 response
- Unit: SHA-256 deduplication produces consistent hashes for identical payloads

### Task 5.2: Provider-Specific Handlers (BambooHR, Workday, Rippling, Merge.dev)

**What:** Implement webhook handlers for each HRIS provider that extract the termination event fields (employee ID, email, departure date, departure reason) and create or update the employee record and offboarding case.

**Design:**

```typescript
// apps/api/src/webhooks/handlers/bamboohr-handler.ts
export class BambooHRWebhookHandler implements HRISWebhookHandler {
  verifySignature(body: Buffer, headers: Record<string, string>, secret: string): boolean {
    // BambooHR uses webhook secret for HMAC-SHA256
    const expected = createHmac('sha256', secret).update(body).digest('hex');
    return timingSafeEqual(Buffer.from(headers['x-bamboohr-signature'] ?? ''), Buffer.from(expected));
  }

  extractEventType(payload: any): string {
    return payload.type; // 'employmentStatusChange'
  }

  async processTermination(payload: any, orgId: string): Promise<string> {
    const employeeData = {
      hrisSource: 'bamboohr',
      hrisExternalId: payload.employees[0].id,
      email: payload.employees[0].fields.workEmail,
      firstName: payload.employees[0].fields.firstName,
      lastName: payload.employees[0].fields.lastName,
      department: payload.employees[0].fields.department,
      jobTitle: payload.employees[0].fields.jobTitle,
    };

    // Upsert employee
    const employee = await employeeService.upsertFromHRIS(orgId, employeeData);

    // Create offboarding case
    const caseId = await caseService.createFromHRIS({
      organisationId: orgId,
      employeeId: employee.id,
      departureDate: payload.employees[0].fields.terminationDate,
      departureReason: this.mapDepartureReason(payload.employees[0].fields.employmentStatus),
    });

    return caseId;
  }
}
```

**Testing:**
- Integration: BambooHR termination webhook -> employee upserted, case created in `draft` status
- Integration: Workday termination webhook -> correct field mapping from SOAP/REST payload
- Integration: Rippling `employee.terminated` webhook -> case created with correct departure date
- Integration: Merge.dev unified webhook -> employee status change creates case regardless of underlying HRIS
- Integration: Employee already exists -> upsert updates fields, no duplicate created
- Unit: Departure reason mapping (BambooHR status codes -> internal reason codes)

### Task 5.3: HRIS Connection Setup UI

**What:** Build the settings page for configuring HRIS connections: provider selection, API credentials, webhook URL display (with copy button), and connection testing.

**Testing:**
- E2E: Navigate to Settings -> Integrations -> Add HRIS Connection
- E2E: Select BambooHR -> enter API key -> test connection -> success indicator
- E2E: Webhook URL displayed with copy-to-clipboard button
- E2E: Connection status shows last sync time and error state if applicable

---

## Phase 6: Documents, Equipment & Compliance Reporting

**Goal:** Implement document collection with electronic signature, equipment return tracking, and compliance evidence reporting (SOC 2, ISO 27001, NIST). At the end of this phase, the platform produces exportable audit evidence.

**Definition of Done:** HR can upload separation documents, send them for signature, and track signature status. Equipment assigned to the employee is listed with return tracking. The compliance reporting page generates evidence packages for SOC 2 CC6.2/CC6.3, ISO 27001 A.5.18, and NIST AC-2.

### Task 6.1: Document Management & Signature Workflow

**What:** Implement document upload (to S3/MinIO), document type categorisation (separation agreement, NDA reminder, final acknowledgement), and a simple signature workflow (sent -> viewed -> signed). Documents linked to offboarding cases.

**Design:**

```typescript
// apps/api/src/services/document-service.ts
export class DocumentService {
  async upload(caseId: string, file: MultipartFile, documentType: string): Promise<Document> {
    const key = `documents/${caseId}/${randomUUID()}-${file.filename}`;
    await s3Client.putObject({ Bucket: 'offboarding-docs', Key: key, Body: file.file });

    return db.insert(documents).values({
      offboardingCaseId: caseId,
      documentType,
      title: file.filename,
      filePath: key,
      mimeType: file.mimetype,
      status: 'pending',
      docData: { fileSize: file.file.bytesRead, requiresSignature: documentType !== 'equipment_receipt' },
    }).returning();
  }

  async sendForSignature(docId: string, employeeEmail: string): Promise<void> {
    const doc = await this.getById(docId);
    const signatureUrl = await this.generateSignatureLink(doc);
    await emailService.send(employeeEmail, 'document-signature-request', { signatureUrl, documentTitle: doc.title });
    await db.update(documents).set({ status: 'sent', docData: { ...doc.docData, sentAt: new Date() } }).where(eq(documents.id, docId));
  }
}
```

**Testing:**
- Integration: Upload PDF -> stored in S3, document row created with correct metadata
- Integration: Send for signature -> email sent (mocked), status updated to `sent`
- Integration: Employee signs -> status updated to `signed`, timestamp recorded
- Integration: Expired document (past deadline) -> status marked as `expired`
- E2E: Document tab on case detail page shows document list with status badges and action buttons

### Task 6.2: Equipment Return Tracking

**What:** Implement equipment inventory linked to employees, return request workflow, and shipping label generation stub. When a case starts, equipment assigned to the departing employee is flagged for return.

**Testing:**
- Integration: Case start -> equipment assigned to employee flagged as `return_pending`
- Integration: Mark equipment as returned -> status updated, condition recorded
- E2E: Equipment tab on case detail shows assigned items with return status
- E2E: "Mark Returned" button updates status and records condition assessment

### Task 6.3: Compliance Evidence Reports

**What:** Build compliance reporting endpoints and UI that generate audit evidence packages from the audit ledger and access revocation records. Support SOC 2 (CC6.2/CC6.3), ISO 27001 (A.5.18), and NIST SP 800-53 (AC-2, PS-4).

**Design:**

```typescript
// apps/api/src/services/compliance-report-service.ts
export class ComplianceReportService {
  async generateSOC2Report(orgId: string, dateRange: { from: Date; to: Date }): Promise<SOC2Report> {
    const revocations = await db.select({
      employeeName: sql`e.first_name || ' ' || e.last_name`,
      employeeEmail: employees.email,
      departureDate: offboardingCases.departureDate,
      appName: saasApplications.name,
      revokedAt: employeeAppAccess.revokedAt,
      caseCreatedAt: offboardingCases.createdAt,
    })
    .from(employeeAppAccess)
    .innerJoin(offboardingCases, eq(employeeAppAccess.offboardingCaseId, offboardingCases.id))
    .innerJoin(employees, eq(offboardingCases.employeeId, employees.id))
    .innerJoin(saasApplications, eq(employeeAppAccess.saasApplicationId, saasApplications.id))
    .where(and(
      eq(offboardingCases.organisationId, orgId),
      between(offboardingCases.departureDate, dateRange.from, dateRange.to),
    ));

    return {
      framework: 'SOC 2 Type II',
      controls: ['CC6.2', 'CC6.3'],
      period: dateRange,
      totalDepartures: new Set(revocations.map(r => r.employeeEmail)).size,
      totalRevocations: revocations.length,
      averageTimeToRevokeHours: this.calculateAverageRevocationTime(revocations),
      slaBreaches: revocations.filter(r => this.isSlaBreach(r)).length,
      evidence: revocations,
    };
  }
}
```

**Testing:**
- Integration: SOC 2 report includes all revocations in date range with correct timing calculations
- Integration: ISO 27001 report maps to controls A.5.18 and A.6.5
- Integration: NIST report maps to AC-2 and PS-4 with SLA compliance percentages
- Integration: Empty date range returns zero results, not an error
- E2E: Compliance page shows framework selector, date range picker, and generated report
- E2E: Export report as CSV downloads correctly formatted file

---

## Phase 7: Exit Interviews & AI Synthesis

**Goal:** Implement exit survey creation, delivery, collection, and AI-powered theme synthesis using Claude. At the end of this phase, HR leaders can see aggregate themes across exit interviews and identify systemic retention risks.

**Definition of Done:** HR can configure exit survey templates, send surveys to departing employees, collect responses, and view AI-generated summaries per interview and cohort-level theme analysis across all interviews.

### Task 7.1: Exit Survey Template Management

**What:** Implement CRUD for exit survey templates with question definitions (free text, rating 1-5, multiple choice, yes/no). Include a default template with industry-standard exit interview questions.

**Testing:**
- Integration: Create template with mixed question types
- Integration: Default template seeded on organisation creation
- Integration: Template CRUD respects org scoping
- Unit: Question schema validation rejects invalid types

### Task 7.2: Survey Delivery & Response Collection

**What:** When an offboarding case reaches a configurable state, automatically send the exit survey to the departing employee via email with a unique, time-limited link. Collect and store responses in the `exit_interviews.interview_data` JSONB field.

**Design:**

```typescript
// apps/api/src/services/exit-interview-service.ts
export class ExitInterviewService {
  async sendSurvey(caseId: string): Promise<void> {
    const offboardingCase = await caseService.getById(caseId);
    const template = await exitSurveyTemplateService.getDefault(offboardingCase.organisationId);
    const employee = await employeeService.getById(offboardingCase.employeeId);

    const interview = await db.insert(exitInterviews).values({
      offboardingCaseId: caseId,
      status: 'sent',
      sentAt: new Date(),
      interviewData: { templateId: template.id, templateName: template.name, responses: [] },
    }).returning();

    const surveyUrl = await this.generateSurveyLink(interview[0].id, employee.email);
    await emailService.send(employee.personalEmail ?? employee.email, 'exit-survey', {
      surveyUrl,
      employeeName: employee.firstName,
      departureDate: offboardingCase.departureDate,
    });
  }

  async submitResponses(interviewId: string, responses: SurveyResponse[]): Promise<void> {
    const interview = await this.getById(interviewId);
    await db.update(exitInterviews).set({
      status: 'completed',
      completedAt: new Date(),
      interviewData: { ...interview.interviewData, responses },
    }).where(eq(exitInterviews.id, interviewId));

    // Trigger AI analysis
    await boss.send('analyse-exit-interview', { interviewId });
  }
}
```

**Testing:**
- Integration: Send survey -> email delivered (mocked), interview record created with `sent` status
- Integration: Submit responses -> stored in JSONB, status updated to `completed`, AI job queued
- Integration: Survey link expired after configured TTL -> returns 410 Gone
- E2E: Employee survey page renders questions from template, accepts responses, submits

### Task 7.3: AI-Powered Exit Interview Analysis

**What:** After an exit interview is completed, use Claude to generate a structured summary, extract themes, calculate sentiment score, and identify retention risk factors. Store results in the interview's `interview_data.ai_analysis` field.

**Design:**

```typescript
// apps/api/src/ai/exit-interview-analyser.ts
import Anthropic from '@anthropic-ai/sdk';

export class ExitInterviewAnalyser {
  private client = new Anthropic();

  async analyse(interviewId: string): Promise<void> {
    const interview = await exitInterviewService.getById(interviewId);
    const responses = interview.interviewData.responses;

    const message = await this.client.messages.create({
      model: 'claude-sonnet-4-20250514',
      max_tokens: 2000,
      system: `You are an HR analytics assistant. Analyse exit interview responses and produce structured insights.
Return JSON with: summary (2-3 sentences), themes (array of theme codes), sentiment_score (-1.0 to 1.0),
retention_risk_factors (array), recommended_actions (array of actionable suggestions).
Theme codes: career_growth, compensation, manager_feedback, work_life_balance, company_culture,
team_dynamics, role_fit, recognition, tools_processes, leadership, remote_work.`,
      messages: [{
        role: 'user',
        content: `Analyse these exit interview responses:\n\n${responses.map(r =>
          `Q: ${r.question}\nA: ${r.type === 'rating_1_5' ? `${r.response}/5` : r.response}`
        ).join('\n\n')}`,
      }],
    });

    const analysis = JSON.parse(message.content[0].text);
    await db.update(exitInterviews).set({
      interviewData: { ...interview.interviewData, ai_analysis: analysis },
    }).where(eq(exitInterviews.id, interviewId));
  }
}
```

**Testing:**
- Integration (mocked Claude): Analysis produces valid JSON with all required fields
- Integration (mocked Claude): Themes extracted match the defined theme code vocabulary
- Integration (mocked Claude): Sentiment score is between -1.0 and 1.0
- Unit: Prompt construction includes all response types (free text, ratings, multiple choice)
- Integration: AI analysis stored in correct JSONB path and queryable via GIN index

### Task 7.4: Cohort-Level Theme Dashboard

**What:** Build a dashboard that aggregates AI-extracted themes across all exit interviews, filterable by department, time period, and departure reason. Show theme frequency, average sentiment, and trend over time.

**Testing:**
- E2E: Dashboard shows bar chart of theme frequency across all interviews
- E2E: Filter by department -> themes update to show department-specific patterns
- E2E: Trend chart shows theme frequency over time (monthly buckets)
- Integration: JSONB aggregation query returns correct theme counts
- E2E: Click on a theme -> drill down to individual interviews mentioning that theme

---

## Phase 8: Knowledge Transfer & AI Discovery

**Goal:** Implement AI-driven knowledge transfer brief generation and SaaS access discovery. At the end of this phase, the platform proactively identifies what knowledge a departing employee holds and discovers app access IT may not know about.

**Definition of Done:** When an offboarding case starts, AI generates a knowledge transfer brief based on available data (project assignments, document authorship patterns, communication metadata). Shadow IT discovery identifies apps not in the official SaaS registry.

### Task 8.1: Knowledge Transfer Brief Generation

**What:** Use Claude to generate a structured knowledge transfer brief for each departing employee based on available organisational data (department, job title, tenure, projects, direct reports). The brief identifies knowledge areas, suggests successors, and generates a checklist of items to hand over.

**Design:**

```typescript
// apps/api/src/ai/knowledge-transfer-generator.ts
export class KnowledgeTransferGenerator {
  async generate(caseId: string): Promise<void> {
    const offboardingCase = await caseService.getById(caseId);
    const employee = await employeeService.getById(offboardingCase.employeeId);
    const accessRecords = await employeeAccessService.listByEmployee(employee.id);

    const message = await this.client.messages.create({
      model: 'claude-sonnet-4-20250514',
      max_tokens: 3000,
      system: `You are a knowledge management assistant. Given an employee's profile and app access,
generate a structured knowledge transfer brief with categories: projects, processes, relationships,
credentials/access, documentation. Each item should have a title, description, priority (critical/high/medium/low),
and suggested handover approach.`,
      messages: [{
        role: 'user',
        content: `Employee: ${employee.firstName} ${employee.lastName}
Title: ${employee.jobTitle}
Department: ${employee.department}
Tenure: ${this.calculateTenure(employee.startDate)}
Direct Reports: ${employee.profile.direct_reports_count ?? 0}
App Access: ${accessRecords.map(a => `${a.appName} (${a.accessLevel})`).join(', ')}

Generate a knowledge transfer brief for this departing employee.`,
      }],
    });

    const brief = JSON.parse(message.content[0].text);
    await db.insert(knowledgeTransfers).values({
      offboardingCaseId: caseId,
      status: 'draft',
      transferData: { aiBrief: brief.summary, items: brief.items },
    });
  }
}
```

**Testing:**
- Integration (mocked Claude): Knowledge transfer brief generated with categorised items
- Integration: Brief stored in `knowledge_transfers` with correct case linkage
- E2E: Knowledge transfer tab on case detail shows AI-generated brief
- E2E: Manager can add items manually, assign successors, mark items as completed
- Integration: Brief status tracks progress (draft -> sent_to_employee -> in_progress -> completed)

### Task 8.2: Shadow IT / SaaS Access Discovery

**What:** Implement SSO log analysis to discover apps an employee has accessed that are not in the official SaaS registry. Mark discovered access as `is_shadow_it = true` and include it in the revocation workflow.

**Design:**

```typescript
// apps/api/src/services/access-discovery-service.ts
export class AccessDiscoveryService {
  async discoverFromSSOLogs(employeeId: string, orgId: string): Promise<DiscoveredAccess[]> {
    // Query IdP (Okta, Entra ID) for employee's app access grants
    const idpConnection = await idpConnectionService.getForOrg(orgId);
    const idpConnector = IDPConnectorFactory.create(idpConnection);
    const appGrants = await idpConnector.listUserAppGrants(employeeId);

    const knownApps = await saasAppService.listByOrg(orgId);
    const knownSlugs = new Set(knownApps.map(a => a.slug));

    const shadowIT = appGrants.filter(grant => !knownSlugs.has(slugify(grant.appName)));

    for (const discovery of shadowIT) {
      await db.insert(employeeAppAccess).values({
        employeeId,
        saasApplicationId: null, // Unknown app
        externalUserId: discovery.userId,
        accessLevel: discovery.accessLevel,
        discoveryMethod: 'sso_log',
        isShadowIt: true,
        status: 'active',
        accessData: {
          discoveredAppName: discovery.appName,
          discoveryDetails: { source: 'sso_log', discoveredAt: new Date(), grantType: discovery.grantType },
        },
      });
    }

    return shadowIT;
  }
}
```

**Testing:**
- Integration (mocked IdP): Discover 3 apps not in registry -> marked as shadow IT
- Integration: Known apps not flagged as shadow IT
- Integration: Shadow IT apps appear in the access revocation dashboard with "Shadow IT" badge
- Integration: Shadow IT discovery results logged in audit ledger
- E2E: Case detail access tab shows shadow IT discoveries with warning styling

---

## Phase 9: Multi-Jurisdiction & GDPR

**Goal:** Implement jurisdiction-aware offboarding workflows and GDPR data deletion handling. Offboarding checklists adapt based on the employee's jurisdiction, and GDPR erasure requests are tracked with deadlines.

**Definition of Done:** An employee in Germany triggers additional checklist items (works council notification, Arbeitszeugnis). GDPR erasure requests are tracked with a 30-day deadline. Data retention policies auto-purge personal data after the configured retention window.

### Task 9.1: Jurisdiction-Aware Checklist Adaptation

**What:** When tasks are generated for a case, inject jurisdiction-specific tasks from the `jurisdictions.rules.required_offboarding_steps` field. These tasks are added alongside the template-generated tasks.

**Design:**

```typescript
// apps/api/src/services/jurisdiction-task-service.ts
export class JurisdictionTaskService {
  async injectJurisdictionTasks(caseId: string, jurisdictionId: string): Promise<void> {
    const jurisdiction = await jurisdictionService.getById(jurisdictionId);
    const requiredSteps = jurisdiction.rules.required_offboarding_steps ?? [];

    const jurisdictionTasks = requiredSteps.map((step: string) => ({
      offboardingCaseId: caseId,
      title: this.stepTitleMap[step] ?? step,
      description: this.stepDescriptionMap[step] ?? null,
      responsibleDept: this.stepDeptMap[step] ?? 'hr',
      dueDate: this.calculateDueDate(step, jurisdiction.rules),
      taskData: { source: 'jurisdiction', jurisdictionCode: `${jurisdiction.countryCode}-${jurisdiction.subdivisionCode ?? ''}`, step },
    }));

    if (jurisdictionTasks.length > 0) {
      await db.insert(offboardingTasks).values(jurisdictionTasks);
    }
  }
}
```

**Testing:**
- Integration: German employee -> tasks include works council notification, Arbeitszeugnis, Lohnsteuerbescheinigung
- Integration: California employee -> tasks include same-day final pay, COBRA notice, EDD pamphlet
- Integration: Jurisdiction with no special steps -> no extra tasks injected
- Integration: Jurisdiction tasks have correct due dates based on jurisdiction rules
- Unit: Step-to-title mapping covers all defined steps across seeded jurisdictions

### Task 9.2: GDPR Data Deletion Workflow

**What:** Implement GDPR Article 17 erasure request handling. Track requests with deadlines (30 days). Distinguish between data that must be retained (payroll for tax law) and data eligible for deletion. Auto-purge personal data after the jurisdiction-specific retention window.

**Design:**

```typescript
// apps/api/src/services/gdpr-service.ts
export class GDPRService {
  async createErasureRequest(caseId: string, employeeId: string): Promise<void> {
    const offboardingCase = await caseService.getById(caseId);
    const jurisdiction = await jurisdictionService.getByEmployee(employeeId);

    if (!jurisdiction.rules.gdpr_applicable) {
      throw new Error('GDPR not applicable for this jurisdiction');
    }

    const deadline = addDays(new Date(), jurisdiction.rules.gdpr_erasure_deadline_days ?? 30);

    // Update case compliance data
    await db.update(offboardingCases).set({
      complianceData: sql`
        jsonb_set(compliance_data, '{gdpr}',
          '${JSON.stringify({
            erasure_requested: true,
            erasure_deadline: deadline.toISOString(),
            data_categories_to_delete: ['personal_contact', 'exit_survey_responses', 'knowledge_transfer_notes'],
            data_categories_retained: ['payroll_records', 'audit_log'],
            retention_legal_basis: `${jurisdiction.name} tax law (${jurisdiction.rules.data_retention_years} years)`,
          })}'::jsonb
        )`,
    }).where(eq(offboardingCases.id, caseId));

    // Schedule deadline check
    await boss.send('gdpr-erasure-deadline-check', { caseId }, {
      startAfter: deadline,
    });
  }

  async executeErasure(caseId: string): Promise<void> {
    // Delete personal data from employees table (replace with anonymised data)
    // Delete exit interview response text
    // Retain audit ledger entries (redact PII fields)
    // Retain payroll-related data per retention policy
  }
}
```

**Testing:**
- Integration: Erasure request created for EU employee -> deadline set to 30 days
- Integration: Erasure request rejected for non-EU employee
- Integration: Execute erasure -> personal fields anonymised, survey text deleted, audit entries retained
- Integration: Payroll data retained with legal basis documented
- Integration: Scheduled job fires at deadline, logs warning if erasure incomplete
- Unit: Data category classification (deletable vs. retained) for each data type

### Task 9.3: Data Retention Policy Enforcement

**What:** Implement a scheduled background job that checks data retention policies and auto-purges expired personal data. Configurable per-organisation and per-jurisdiction.

**Testing:**
- Integration: Data older than retention window -> purged on scheduled run
- Integration: Data within retention window -> not purged
- Integration: Organisation-specific retention override respected over jurisdiction default
- Integration: Purge operation logged in audit ledger with categories deleted

---

## Phase 10: Analytics, Alumni & Polish

**Goal:** Build the analytics dashboard, optional alumni portal, and polish the platform for production readiness. This phase covers operational metrics, performance optimisation, and the features from the "nice-to-have" backlog.

**Definition of Done:** The analytics dashboard shows offboarding completion rate, average time-to-full-deprovisioning, SLA breach trends, and security risk metrics. Licence reclamation tracking quantifies cost savings. The platform is production-ready with rate limiting, error monitoring, and deployment documentation.

### Task 10.1: Analytics Dashboard

**What:** Build the operational analytics dashboard with key metrics: total offboardings, average time to complete, average time to full access revocation, SLA compliance rate, task completion rate by department, and top apps by revocation time.

**Design:**

```typescript
// apps/api/src/services/analytics-service.ts
export class AnalyticsService {
  async getDashboardMetrics(orgId: string, period: { from: Date; to: Date }): Promise<DashboardMetrics> {
    const [caseMetrics, revocationMetrics, taskMetrics] = await Promise.all([
      this.getCaseMetrics(orgId, period),
      this.getRevocationMetrics(orgId, period),
      this.getTaskMetrics(orgId, period),
    ]);
    return { caseMetrics, revocationMetrics, taskMetrics };
  }

  private async getRevocationMetrics(orgId: string, period: DateRange) {
    return db.execute(sql`
      SELECT
        COUNT(DISTINCT oc.id) AS total_cases,
        AVG(EXTRACT(EPOCH FROM (eaa.revoked_at - oc.created_at)) / 3600) AS avg_hours_to_revoke,
        COUNT(CASE WHEN EXTRACT(EPOCH FROM (eaa.revoked_at - oc.created_at)) > 86400 THEN 1 END) AS sla_breaches,
        sa.name AS app_name,
        AVG(EXTRACT(EPOCH FROM (eaa.revoked_at - oc.created_at)) / 3600) AS avg_app_revocation_hours
      FROM employee_app_access eaa
      JOIN offboarding_cases oc ON eaa.offboarding_case_id = oc.id
      JOIN saas_applications sa ON eaa.saas_application_id = sa.id
      WHERE oc.organisation_id = ${orgId}
        AND oc.departure_date BETWEEN ${period.from} AND ${period.to}
      GROUP BY sa.name
      ORDER BY avg_app_revocation_hours DESC
    `);
  }
}
```

**Testing:**
- Integration: Metrics calculated correctly from test data (known cases with known timing)
- Integration: SLA breach correctly identified (revocation > 24h after case creation)
- E2E: Dashboard renders charts for completion rate, revocation timing, and department performance
- E2E: Date range picker updates all metrics when changed
- Integration: Empty period returns zero metrics, not errors

### Task 10.2: Licence Reclamation Tracking

**What:** Track the cost savings from reclaimed SaaS licences when employees are offboarded. Each SaaS application can have a per-seat cost configured. When access is revoked, the reclaimed value is calculated and displayed.

**Testing:**
- Integration: Revoking access to app with $15/seat/month cost -> $15 reclamation recorded
- Integration: Monthly savings dashboard sums all reclamations in the period
- E2E: Case detail shows "Licence savings" section with per-app costs
- E2E: Analytics page shows cumulative savings chart

### Task 10.3: Alumni Record (Optional)

**What:** After offboarding is complete, offer the option to maintain a lightweight alumni record for potential rehire or contractor engagement. The alumni record retains only the employee's name, personal email (if provided), departure date, and department -- no access or HR data.

**Testing:**
- Integration: Opt-in to alumni program creates record with minimal fields only
- Integration: Alumni record does not reference audit, access, or HR data
- E2E: Alumni directory page shows former employees with basic info and last contact date

### Task 10.4: Production Readiness

**What:** Implement rate limiting (per-org), structured logging, error monitoring integration (Sentry), health check endpoints, Kubernetes deployment manifests, and comprehensive deployment documentation.

**Testing:**
- Integration: Rate limiter blocks requests exceeding org quota
- Integration: Health check endpoint returns database connectivity status
- Unit: Structured log output includes correlation IDs, org IDs, and timing
- Manual: Docker Compose deployment from clean checkout succeeds
- Manual: Kubernetes manifests deploy correctly to a test cluster

---

## Phase Dependency Summary

| Phase | Depends On | Can Run In Parallel With |
|-------|-----------|--------------------------|
| Phase 1: Foundation | None | -- |
| Phase 2: Offboarding Case Engine | Phase 1 | -- |
| Phase 3: SaaS Connectors & Revocation | Phase 2 | Phase 4 |
| Phase 4: Checklist & Task Management | Phase 2 | Phase 3 |
| Phase 5: HRIS Webhooks & Integration | Phases 3+4 | -- |
| Phase 6: Documents, Equipment & Compliance | Phase 5 | -- |
| Phase 7: Exit Interviews & AI Synthesis | Phase 6 | -- |
| Phase 8: Knowledge Transfer & AI Discovery | Phase 7 | -- |
| Phase 9: Multi-Jurisdiction & GDPR | Phase 8 | -- |
| Phase 10: Analytics, Alumni & Polish | Phase 9 | -- |

**Parallelism note:** Phases 3 and 4 are the only phases that can be developed in parallel, as they share Phase 2 as a dependency but have no dependency on each other. Phase 5 requires both to be complete because HRIS webhook processing creates both access revocation jobs (Phase 3) and tasks (Phase 4).

---

## Global Definition of Done (All Phases)

A phase is considered complete when:

1. All task implementations are merged to the main branch
2. All unit and integration tests pass in CI
3. E2E tests pass for all UI features introduced in the phase
4. API routes have OpenAPI 3.1 schema documentation (auto-generated from Fastify)
5. Audit ledger entries are written for all state-changing operations
6. Multi-tenant isolation verified (org A cannot see org B data)
7. No critical or high-severity security issues in dependency audit
8. Docker Compose deployment tested from a clean checkout
