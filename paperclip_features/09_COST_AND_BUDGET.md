# Cost & Budget Management — per-agent/project/company tracking, auto-pause at 100%

## Investigation Summary

Paperclip tracks every token spent by every agent and enforces budget limits to prevent runaway costs. The system has two layers: **cost tracking** (recording what was spent) and **budget enforcement** (controlling what can be spent). Cost events flow in from adapter results after each heartbeat run, get aggregated into summaries, and trigger budget policy evaluations that can auto-pause agents, projects, or entire companies. Additionally, a more granular **finance events** layer tracks billing-type-aware ledger entries. The system spans 4 database tables, 3 services, and integrates deeply with the heartbeat and budget systems (covered in 01 and 06).

---

## 1. Data Model

### 1.1 `cost_events` Table

The primary cost tracking table. One row per heartbeat run's token usage.

| Column | Type | Default | Notes |
|--------|------|---------|-------|
| `id` | UUID | random | PK |
| `companyId` | UUID | — | FK → companies |
| `agentId` | UUID | — | FK → agents (required) |
| `issueId` | UUID | NULL | FK → issues — which task incurred this cost |
| `projectId` | UUID | NULL | FK → projects — for project-level budgets |
| `goalId` | UUID | NULL | FK → goals |
| `heartbeatRunId` | UUID | NULL | FK → heartbeat_runs — which run produced this |
| `billingCode` | TEXT | NULL | Cross-team cost attribution |
| `provider` | TEXT | — | LLM provider: `anthropic`, `openai`, etc. |
| `biller` | TEXT | `"unknown"` | Who bills: may differ from provider (e.g., `"max_subscription"`) |
| `billingType` | TEXT | `"unknown"` | `metered_api`, `subscription_included`, `subscription_overage`, `credits`, `fixed`, `unknown` |
| `model` | TEXT | — | Model used: `claude-sonnet-4-20250514`, etc. |
| `inputTokens` | INT | 0 | Input tokens consumed |
| `cachedInputTokens` | INT | 0 | Cached input tokens (lower cost) |
| `outputTokens` | INT | 0 | Output tokens generated |
| `costCents` | INT | — | Dollar cost in cents |
| `occurredAt` | TIMESTAMPTZ | — | When the cost was incurred |
| `createdAt` | TIMESTAMPTZ | now | — |

**Indexes:**
- `(companyId, occurredAt)` — time-range queries
- `(companyId, agentId, occurredAt)` — per-agent breakdown
- `(companyId, provider, occurredAt)` — per-provider breakdown
- `(companyId, biller, occurredAt)` — per-biller breakdown
- `(companyId, heartbeatRunId)` — link to specific run

### 1.2 `finance_events` Table

A more granular ledger for billing-aware cost tracking.

| Column | Type | Default | Notes |
|--------|------|---------|-------|
| `id` | UUID | random | PK |
| `companyId` | UUID | — | FK → companies |
| `agentId` | UUID | NULL | FK → agents |
| `issueId` | UUID | NULL | FK → issues |
| `projectId` | UUID | NULL | FK → projects |
| `goalId` | UUID | NULL | FK → goals |
| `heartbeatRunId` | UUID | NULL | FK → heartbeat_runs |
| `costEventId` | UUID | NULL | FK → cost_events — links to the parent cost event |
| `billingCode` | TEXT | NULL | — |
| `description` | TEXT | NULL | Human-readable description |
| `eventKind` | TEXT | — | See event kinds below |
| `direction` | TEXT | `"debit"` | `debit` or `credit` |
| `biller` | TEXT | — | Who bills |
| `provider` | TEXT | NULL | LLM provider |
| `executionAdapterType` | TEXT | NULL | Adapter that ran |
| `pricingTier` | TEXT | NULL | — |
| `region` | TEXT | NULL | — |
| `model` | TEXT | NULL | — |
| `quantity` | INT | NULL | Number of units consumed |
| `unit` | TEXT | NULL | `input_token`, `output_token`, `cached_input_token`, `request`, `credit_usd`, etc. |
| `amountCents` | INT | — | Dollar amount in cents |
| `currency` | TEXT | `"USD"` | — |
| `estimated` | BOOL | false | Whether the cost is estimated vs exact |
| `externalInvoiceId` | TEXT | NULL | Provider invoice reference |
| `metadataJson` | JSONB | NULL | Additional metadata |
| `occurredAt` | TIMESTAMPTZ | — | — |
| `createdAt` | TIMESTAMPTZ | now | — |

**Finance event kinds:**
`inference_charge`, `platform_fee`, `credit_purchase`, `credit_refund`, `credit_expiry`, `byok_fee`, `gateway_overhead`, `log_storage_charge`, `logpush_charge`, `provisioned_capacity_charge`, `training_charge`, `custom_model_import_charge`, `custom_model_storage_charge`, `manual_adjustment`

### 1.3 `budget_policies` Table

(Covered in detail in 01_COMPANY_MANAGEMENT.md — summarized here)

| Column | Type | Notes |
|--------|------|-------|
| `id` | UUID | PK |
| `companyId` | UUID | FK → companies |
| `scopeType` | TEXT | `company`, `agent`, or `project` |
| `scopeId` | UUID | The scoped entity |
| `metric` | TEXT | `billed_cents` |
| `windowKind` | TEXT | `calendar_month_utc` or `lifetime` |
| `amount` | INT | Budget limit in cents |
| `warnPercent` | INT | Soft alert threshold (default 80%) |
| `hardStopEnabled` | BOOL | Auto-pause at 100% |
| `notifyEnabled` | BOOL | Create incidents at soft threshold |
| `isActive` | BOOL | — |

**Unique:** `(companyId, scopeType, scopeId, metric, windowKind)`

### 1.4 `budget_incidents` Table

(Covered in detail in 01_COMPANY_MANAGEMENT.md — summarized here)

| Column | Type | Notes |
|--------|------|-------|
| `id` | UUID | PK |
| `policyId` | UUID | FK → budget_policies |
| `thresholdType` | TEXT | `soft` or `hard` |
| `amountLimit`, `amountObserved` | INT | — |
| `status` | TEXT | `open`, `resolved`, `dismissed` |
| `approvalId` | UUID | FK → approvals (for hard stops) |

---

## 2. Cost Event Flow

### 2.1 How Costs Are Recorded

After every heartbeat run, the heartbeat service:

1. **Parses adapter result** — extracts `inputTokens`, `outputTokens`, `cachedInputTokens`, `costUsd`, `provider`, `model`, `biller`, `billingType`
2. **Normalizes usage** — if adapter reports cumulative session tokens, computes delta from previous run
3. **Converts cost** — `costCents = Math.round(costUsd * 100)`, with `subscription_included` mapped to 0
4. **Creates cost event** via `costService.createEvent()`
5. **Creates finance event** via `financeService.createEvent()` with detailed ledger entry

### 2.2 Cost Event Creation Pipeline

`costService.createEvent()`:

1. Validate agent belongs to company
2. Insert `cost_events` row
3. Compute fresh monthly spend for the agent → update `agents.spentMonthlyCents`
4. Compute fresh monthly spend for the company → update `companies.spentMonthlyCents`
5. **Trigger budget evaluation** → `budgets.evaluateCostEvent(event)`

### 2.3 Budget Evaluation (on every cost event)

`budgets.evaluateCostEvent()`:

1. Load all active budget policies for the company
2. Filter to relevant policies (company-scope matches company, agent-scope matches agent, project-scope matches project)
3. For each relevant policy:
   - Compute observed amount (sum of `cost_cents` in the current window)
   - If `notifyEnabled` and observed ≥ `warnPercent` threshold → create soft incident
   - If `hardStopEnabled` and observed ≥ 100% → resolve soft incidents, create hard incident (with `budget_override_required` approval), auto-pause the scope, cancel active work

---

## 3. API Surface

### 3.1 Cost API

| Method | Path | Description |
|--------|------|-------------|
| `POST` | `/api/companies/:companyId/cost-events` | Report cost event (typically automated by adapters) |
| `GET` | `/api/companies/:companyId/costs/summary` | Company cost summary (spend, budget, utilization %) |
| `GET` | `/api/companies/:companyId/costs/by-agent` | Per-agent cost breakdown |
| `GET` | `/api/companies/:companyId/costs/by-provider` | Per-provider breakdown (with biller, billing type, model) |
| `GET` | `/api/companies/:companyId/costs/by-biller` | Per-biller aggregation |
| `GET` | `/api/companies/:companyId/costs/by-project` | Per-project cost (with activity_log fallback for run-project linking) |
| `GET` | `/api/companies/:companyId/costs/by-agent-model` | Per-agent-per-model breakdown |
| `GET` | `/api/companies/:companyId/costs/window-spend` | Rolling window spend (5h, 24h, 7d) by provider |
| `GET` | `/api/companies/:companyId/costs/quotas` | Provider quota windows (rate limits, remaining capacity) |

### 3.2 Finance API

| Method | Path | Description |
|--------|------|-------------|
| `POST` | `/api/companies/:companyId/finance-events` | Record finance event |
| `GET` | `/api/companies/:companyId/finance/summary` | Debit/credit/net summary |
| `GET` | `/api/companies/:companyId/finance/by-biller` | Per-biller financial breakdown |
| `GET` | `/api/companies/:companyId/finance/by-kind` | Per-event-kind breakdown |
| `GET` | `/api/companies/:companyId/finance/events` | List finance events (paginated) |

### 3.3 Budget API

| Method | Path | Description |
|--------|------|-------------|
| `GET` | `/api/companies/:companyId/budgets/overview` | Full budget overview (policies, active incidents, paused counts) |
| `PUT` | `/api/companies/:companyId/budgets/policies` | Upsert budget policy |
| `GET` | `/api/companies/:companyId/budgets/policies` | List budget policies |
| `POST` | `/api/companies/:companyId/budgets/incidents/:id/resolve` | Resolve budget incident (raise budget or keep paused) |

---

## 4. Aggregation Queries

### 4.1 Summary

```sql
SELECT COALESCE(SUM(cost_cents), 0) FROM cost_events
WHERE company_id = $1 AND occurred_at >= $from AND occurred_at < $to
```

Returns: `{ spendCents, budgetCents, utilizationPercent }`

### 4.2 By Agent

Groups by `agentId` with JOINs to `agents` for name/status. Includes:
- Total cost, input/output/cached tokens
- Separate counts for `metered_api` vs `subscription_included` runs
- Subscription-specific token breakdowns

### 4.3 By Project

Complex query — cost events may not have `projectId` set directly. Uses a fallback via `activity_log`:
1. Find runs that touched issues in a project (via activity log entries)
2. Use `COALESCE(cost_events.projectId, derived_project_id)` for attribution

### 4.4 Window Spend

Rolling windows (5h, 24h, 7d) by provider — useful for monitoring spend velocity without waiting for monthly aggregation.

### 4.5 Provider Quotas

Polls registered adapters for their provider quota windows (rate limits, remaining tokens). Each adapter implements `getQuotaWindows()` with a 20-second timeout. Failures are caught and returned as error results.

---

## 5. Budget Enforcement (Summary)

Covered in depth in 01_COMPANY_MANAGEMENT.md. Key points:

| Threshold | Action |
|-----------|--------|
| `warnPercent` (default 80%) | Soft incident created. Agent warned to focus on critical tasks. |
| 100% | Hard incident created. `budget_override_required` approval. Scope auto-paused. Active work cancelled. |

**Scopes:** Company → Agent → Project (checked in order before each heartbeat invocation)

**Resolution:** Board can `raise_budget_and_resume` (increase budget, resume scope, approve linked approval) or `keep_paused` (keep paused, reject approval).

**Monthly reset:** Budget windows are `calendar_month_utc` — spend resets on the 1st of each month.

---

## 6. Billing Type Awareness

Paperclip distinguishes between billing models:

| Billing Type | `costCents` Behavior | Description |
|-------------|---------------------|-------------|
| `metered_api` | Actual cost | Pay-per-token API usage |
| `subscription_included` | Always 0 | Tokens included in subscription (e.g., Claude Pro) |
| `subscription_overage` | Actual cost | Overage beyond subscription |
| `credits` | Actual cost | Credit-based usage |
| `fixed` | Actual cost | Fixed-price usage |
| `unknown` | Actual cost | Fallback |

This matters for cost reporting — subscription-included runs show token usage but $0 cost, giving accurate per-agent activity metrics without inflating spend.

---

## 7. Key Design Decisions

| Decision | Rationale |
|----------|-----------|
| Separate `cost_events` and `finance_events` | Cost events are the simple token-based view; finance events are the full accounting ledger with debits/credits |
| `costCents` as integer (not float) | Avoids floating-point precision issues. Cents, not dollars. |
| Live spend computation (not cached) | Accuracy over performance. `spentMonthlyCents` on agents/companies is a convenience cache, recomputed on every cost event |
| Budget evaluation on every cost event | Real-time enforcement. No delay between spend and enforcement. |
| Project cost attribution via activity log fallback | Cost events may not have `projectId` set. The fallback links runs to projects through their issue activity. |
| Rolling window spend | Gives operators a velocity view without waiting for monthly reports |
| Provider quota polling | Surfaces rate limit info from adapters (Claude/Codex) without hardcoding provider APIs |
| Billing type in cost events | Enables accurate reporting for mixed billing models (API + subscription) |

---

## 8. Observations for Colmena

1. **The two-layer model (cost_events + finance_events) may be overkill for MVP.** `cost_events` alone covers 90% of use cases. `finance_events` adds accounting granularity (debits/credits, billing types, external invoice references) that's useful for production but not essential initially.

2. **Live spend recomputation on every cost event** is simple but won't scale. At high throughput, consider materialized views or periodic aggregation.

3. **The billing type awareness** is smart — it correctly handles the case where an agent uses Claude Pro (subscription-included) vs API (metered). Without it, subscription usage would either be double-counted or invisible.

4. **Project cost attribution via activity log** is a creative workaround for missing `projectId` on cost events. For Colmena, ensure `projectId` is always set on cost events to avoid this complexity.

5. **Budget enforcement on every cost event** is the right approach for preventing runaway spend. The alternative (periodic checks) leaves a window where agents can overspend.

6. **Provider quota polling** is a nice-to-have — surfaces rate limit information from Claude/Codex APIs. Not essential for MVP but useful for operators managing API usage.

7. **The `warnPercent` soft alert** gives agents a chance to self-moderate before hitting the hard stop. The Paperclip skill teaches agents to check their budget and skip low-priority tasks above 80%.

8. **For Colmena MVP**, a simplified version could: track `cost_events` with `agentId`, `provider`, `model`, `costCents`, `tokens`. Enforce a single company-level budget. Skip finance events, billing types, and provider quotas initially.
