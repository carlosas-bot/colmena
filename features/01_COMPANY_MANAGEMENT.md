# Company Management — multi-tenant, budgets, goals

## Investigation Summary

This document captures how Paperclip implements company management — the top-level organizational unit. Every entity in Paperclip (agents, issues, goals, projects, budgets, secrets, etc.) is company-scoped, making companies the fundamental data isolation boundary.

---

## 1. Data Model

### 1.1 `companies` Table

| Column | Type | Default | Notes |
|--------|------|---------|-------|
| `id` | UUID | `gen_random_uuid()` | Primary key |
| `name` | TEXT | — | Required, min 1 char |
| `description` | TEXT | NULL | Optional |
| `status` | TEXT | `"active"` | Enum: `active`, `paused`, `archived` |
| `pauseReason` | TEXT | NULL | `"manual"`, `"budget"`, `"system"` |
| `pausedAt` | TIMESTAMPTZ | NULL | When the company was paused |
| `issuePrefix` | TEXT | `"PAP"` | Unique per instance, auto-derived from name (e.g. "My Company" → "MYC") |
| `issueCounter` | INT | 0 | Auto-incrementing issue number per company |
| `budgetMonthlyCents` | INT | 0 | Monthly budget limit in cents |
| `spentMonthlyCents` | INT | 0 | Cached monthly spend (also computed on-the-fly from `cost_events`) |
| `requireBoardApprovalForNewAgents` | BOOL | true | Governance flag |
| `brandColor` | TEXT | NULL | Hex color `#RRGGBB` for UI theming |
| `createdAt` | TIMESTAMPTZ | now | — |
| `updatedAt` | TIMESTAMPTZ | now | — |

**Indexes:**
- Unique index on `issuePrefix` — guarantees no two companies share the same prefix.

### 1.2 `company_memberships` Table

Links users/principals to companies. Implements multi-tenancy access control.

| Column | Type | Notes |
|--------|------|-------|
| `id` | UUID | PK |
| `companyId` | UUID | FK → `companies.id` |
| `principalType` | TEXT | e.g. `"user"` |
| `principalId` | TEXT | User/principal identifier |
| `status` | TEXT | Default `"active"` |
| `membershipRole` | TEXT | e.g. `"owner"` |

**Indexes:**
- Unique on `(companyId, principalType, principalId)` — one membership per principal per company.
- Index on `(principalType, principalId, status)` — fast lookup of a principal's active memberships.
- Index on `(companyId, status)` — list members of a company.

### 1.3 `company_logos` Table

Separate join table (not a column on `companies`) linking to the `assets` table.

| Column | Type | Notes |
|--------|------|-------|
| `id` | UUID | PK |
| `companyId` | UUID | FK → `companies.id`, CASCADE delete, unique |
| `assetId` | UUID | FK → `assets.id`, CASCADE delete, unique |

**Design decision:** One logo per company (unique constraint on `companyId`). Old logo asset is deleted when replaced.

### 1.4 Related Tables (company-scoped)

All of these have a `companyId` FK → `companies.id`:
- `agents`, `issues`, `goals`, `projects`, `approvals`, `cost_events`, `finance_events`, `heartbeat_runs`, `heartbeat_run_events`, `activity_log`, `company_secrets`, `company_skills`, `budget_policies`, `budget_incidents`, `assets`, `join_requests`, `invites`, `principal_permission_grants`, `routines`, `plugins`, `documents`, etc.

---

## 2. API Surface

### 2.1 Endpoints

| Method | Path | Auth | Description |
|--------|------|------|-------------|
| `GET` | `/api/companies` | Board | List all companies (filtered by membership unless instance admin) |
| `GET` | `/api/companies/stats` | Board | Agent/issue counts per company |
| `GET` | `/api/companies/:companyId` | Board or CEO agent | Get single company |
| `POST` | `/api/companies` | Instance admin | Create company |
| `PATCH` | `/api/companies/:companyId` | Board (full) or CEO agent (branding only) | Update company |
| `PATCH` | `/api/companies/:companyId/branding` | Board or CEO agent | Update branding-only fields |
| `POST` | `/api/companies/:companyId/archive` | Board | Archive company |
| `DELETE` | `/api/companies/:companyId` | Board | Hard delete (cascading) |
| `POST` | `/api/companies/:companyId/logo` | Board | Upload logo image (multipart) |

### 2.2 Validation Schemas (Zod)

**Create:**
```typescript
{
  name: string (min 1),
  description?: string | null,
  budgetMonthlyCents?: number (int, >= 0, default 0)
}
```

**Update (board):**
All create fields (partial) plus:
```typescript
{
  status?: "active" | "paused" | "archived",
  spentMonthlyCents?: number,
  requireBoardApprovalForNewAgents?: boolean,
  brandColor?: string (#RRGGBB) | null,
  logoAssetId?: UUID | null
}
```

**Update branding (CEO agent):**
Strict subset — only `name`, `description`, `brandColor`, `logoAssetId`. At least one field required.

### 2.3 Authorization Model

| Actor | Can create | Can update | Can archive | Can delete |
|-------|-----------|-----------|------------|-----------|
| Instance admin (board) | ✅ | ✅ (all fields) | ✅ | ✅ |
| Regular board member | ❌ | ✅ (all fields) | ✅ | ✅ |
| CEO agent | ❌ | ✅ (branding only) | ❌ | ❌ |
| Other agents | ❌ | ❌ | ❌ | ❌ |

Company listing is filtered by membership — non-admin board users only see companies they belong to. `local_trusted` mode bypasses this (sees all).

---

## 3. Business Logic

### 3.1 Issue Prefix Generation

When a company is created, an issue prefix is auto-derived:

1. Take company name, uppercase, strip non-alpha → first 3 chars (fallback: `"CMP"`)
2. Try inserting with that prefix
3. On unique constraint violation (`23505` on `companies_issue_prefix_idx`), append incrementing suffixes (`A`, `AA`, `AAA`...)
4. Retry up to 10,000 times

This prefix is used for human-readable issue IDs (e.g., `MYC-42`).

### 3.2 Monthly Spend Computation

Spend is computed **on-the-fly** from `cost_events` (not cached in the `spentMonthlyCents` column):

```sql
SELECT COALESCE(SUM(cost_cents), 0)::int
FROM cost_events
WHERE company_id = $1
  AND occurred_at >= <month_start>
  AND occurred_at < <month_end>
```

The `currentUtcMonthWindow()` function calculates the UTC calendar month boundaries. Spend is hydrated onto every company response via `hydrateCompanySpend()`.

### 3.3 Budget Enforcement

Budgets are managed through the `budget_policies` system (not directly on the company record):

**Policy structure:**
- `scopeType`: `"company"`, `"agent"`, or `"project"`
- `scopeId`: the entity UUID
- `metric`: `"billed_cents"` (only option currently)
- `windowKind`: `"calendar_month_utc"` or `"lifetime"`
- `amount`: budget limit in cents
- `warnPercent`: soft alert threshold (default 80%)
- `hardStopEnabled`: whether to auto-pause at 100%
- `notifyEnabled`: whether to create incidents at soft threshold

**Enforcement flow:**
1. After every `cost_event`, `evaluateCostEvent()` checks all relevant policies
2. At `warnPercent` threshold → creates a `"soft"` budget incident + activity log entry
3. At 100% threshold → creates a `"hard"` budget incident, creates a `budget_override_required` approval, and **auto-pauses** the scope (company/agent/project)
4. Auto-pause sets `status = "paused"`, `pauseReason = "budget"` on the entity
5. Also calls `cancelWorkForScope()` hook to cancel active heartbeat runs

**Resolution:**
- Board can `raise_budget_and_resume` (increases budget, resumes scope, resolves incidents, auto-approves linked approval)
- Board can `dismiss` (keeps scope paused, rejects linked approval)

**Invocation blocking:**
Before each heartbeat invocation, `getInvocationBlock()` checks company → agent → project budget policies in order. If any scope is over budget, the heartbeat is blocked.

### 3.4 Company Lifecycle

```
active → paused (manual or budget) → active (resume or budget raise)
       → archived (board action)
```

- **Paused**: Agents cannot receive heartbeats. Company-level pause cascades to block all agent invocations.
- **Archived**: Hidden from default listings. Cannot be reactivated (no unarchive endpoint).
- **Hard delete**: Cascading delete in dependency order — deletes ALL child records across ~25 tables in a single transaction.

### 3.5 Company Creation Side Effects

On company create:
1. Insert company record (with auto-derived issue prefix)
2. Create `company_membership` with `"owner"` role for the creating user
3. If `budgetMonthlyCents > 0`, create a `budget_policy` for the company scope
4. Log `company.created` activity

### 3.6 Company Update Side Effects

On company update:
1. Update company fields
2. Handle logo: insert/update `company_logos` join; delete old asset if replaced; delete join row if set to null
3. Log `company.updated` activity with change details

### 3.7 Dashboard Summary

`GET /api/companies/:companyId/dashboard` returns a single-call health snapshot:

```typescript
{
  companyId: string,
  agents: { active, running, paused, error },
  tasks: { open, inProgress, blocked, done },
  costs: {
    monthSpendCents: number,
    monthBudgetCents: number,
    monthUtilizationPercent: number
  },
  pendingApprovals: number,
  budgets: {
    activeIncidents: number,
    pendingApprovals: number,
    pausedAgents: number,
    pausedProjects: number
  }
}
```

Computed via aggregation queries across `agents`, `issues`, `cost_events`, `approvals`, and `budget_policies/incidents`.

---

## 4. Multi-tenancy Architecture

### 4.1 Isolation Strategy

- **Row-level isolation**: Every entity has a `companyId` FK. All queries filter by company.
- **No shared entities**: Nothing is shared between companies. Even assets, secrets, and skills are company-scoped.
- **API enforcement**: Routes use `assertCompanyAccess(req, companyId)` to verify the authenticated principal has membership in the target company.
- **Membership-based listing**: `GET /api/companies` returns only companies where the user has an active membership (unless instance admin or `local_trusted` mode).

### 4.2 Instance-Level vs Company-Level

| Concern | Scope |
|---------|-------|
| Deployment mode, auth config | Instance |
| User accounts, instance roles | Instance |
| Companies, agents, issues, goals | Company |
| Budgets, costs, secrets | Company |
| Plugins, plugin config | Instance + per-company settings |

### 4.3 Data Deletion

Hard delete (`DELETE /api/companies/:companyId`) performs a cascading delete across **25+ tables** in a single transaction, in dependency order:

```
heartbeat_run_events → agent_task_sessions → heartbeat_runs →
agent_wakeup_requests → agent_api_keys → agent_runtime_state →
issue_comments → cost_events → finance_events → approval_comments →
approvals → company_secrets → join_requests → invites →
principal_permission_grants → company_memberships → issues →
company_logos → assets → goals → projects → agents →
activity_log → companies
```

This is done manually (no DB-level cascade) to ensure order and avoid FK violations.

---

## 5. Technical Patterns

### 5.1 Service Layer Pattern

```
Route (Express Router) → Service (pure business logic) → Drizzle ORM → PostgreSQL
```

- **Service** (`companyService(db)`) receives a `Db` instance (Drizzle client) and returns an object with methods (`list`, `getById`, `create`, `update`, `archive`, `remove`, `stats`).
- **Route** handles HTTP concerns (auth, validation, response codes) and delegates to the service.
- Services are **stateless functions** — they're factory functions that close over `db`.

### 5.2 Spend Hydration Pattern

Company responses always include live spend data. Instead of maintaining a materialized counter:

1. Service queries companies from DB
2. Separately queries `cost_events` for those company IDs in the current UTC month
3. Merges the two result sets (`hydrateCompanySpend`)
4. Returns enriched company objects

This trades query cost for accuracy — no stale counters.

### 5.3 Activity Logging Pattern

Every mutation calls `logActivity(db, {...})` with:
- `companyId`, `actorType` (user/agent/system), `actorId`, `action` (e.g. `company.created`), `entityType`, `entityId`, optional `details` JSON.

This creates an immutable audit trail in the `activity_log` table.

### 5.4 Validation Pattern

- Zod schemas defined in `@paperclipai/shared` (shared between client and server)
- Express middleware `validate(schema)` parses and validates `req.body`
- Some routes do manual `.parse()` for conditional schema selection (e.g., board vs agent)

---

## 6. UI Integration

### 6.1 Pages

- `Companies.tsx` — list view with company cards
- `CompanySettings.tsx` — settings form (name, description, budget, brand color, logo upload)
- `CompanyExport.tsx` / `CompanyImport.tsx` — portability UI
- `Dashboard.tsx` — real-time company health dashboard

### 6.2 Key UI Features

- Company logo upload (multipart form, image preview)
- Brand color picker (hex input with validation)
- Budget configuration with utilization progress bar
- Company status badge (active/paused/archived)
- Board approval toggle for new agent hiring
- Issue prefix display (read-only after creation)

---

## 7. Key Design Decisions

| Decision | Rationale |
|----------|-----------|
| Row-level multi-tenancy (not schema-per-tenant) | Simpler ops, works with embedded PGlite, scales well for expected company counts |
| Manual cascading delete (not DB-level CASCADE) | Control over deletion order, ability to log/audit, works across tables that don't have FK cascades |
| Live spend computation (not materialized counter) | Accuracy over performance; cost_events table is the source of truth |
| Issue prefix auto-generation with retry | Friendly human-readable IDs without requiring user to pick a unique prefix |
| Separate branding endpoint for CEO agents | Allows agents to update company identity without granting full admin access |
| Budget enforcement via policies (not inline thresholds) | Supports multiple scopes (company/agent/project), window types, and incident tracking |
| Company membership table (not inline user list) | Supports multiple principal types, status tracking, role assignment |

---

## 8. Portability (Import/Export)

Companies can be exported as bundles containing:
- Company metadata (name, description, budget)
- Full org chart (agents with configs, reporting chains)
- Projects with workspace configs
- Goals hierarchy
- Skills and instructions
- Governance rules

**Export scrubs secrets** — no API keys or tokens in the bundle. Import supports:
- New company creation or merge into existing
- Collision strategies for name conflicts
- Agent-safe mode (CEO agents can import with restrictions: no `replace` strategy, must target own company)
- Preview mode (dry run showing what would be created/modified)

---

## 9. Observations for Colmena

1. **The company-scoped multi-tenancy is fundamental** — it's not bolted on; every table has `companyId`. This is a good pattern but means the schema is tightly coupled to multi-company support.

2. **Budget system is sophisticated** — policies, incidents, approvals, auto-pause, invocation blocking. This could be simplified for an MVP but the core concept (scoped budgets with enforcement) is essential for autonomous AI teams.

3. **Cascading delete is manual and fragile** — adding a new company-scoped table requires updating the delete function. A DB-level cascade or a registry pattern would be more maintainable.

4. **Issue prefix generation is clever but niche** — only matters if you want Jira-style issue IDs (e.g., `MYC-42`).

5. **The separation between service and route layers is clean** — easy to test services independently.

6. **Live spend hydration** is simple but won't scale to thousands of cost events per query. Consider materialized views or periodic aggregation at scale.
