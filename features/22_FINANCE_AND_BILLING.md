# Finance & Billing — finance events, billing codes, budget incidents

## Investigation Summary

Paperclip has a finance layer that sits alongside the simpler cost tracking system. While `cost_events` tracks token usage and spend per heartbeat run, `finance_events` provides a full double-entry-style accounting ledger with debits, credits, billing types, units, providers, and external invoice references. This enables tracking not just "how many tokens did this agent use" but "how much did this cost, who billed it, what pricing tier, was it estimated, and what's the net position." The billing code system on issues enables cross-team cost attribution. Budget incidents (covered in 01 and 09) bridge finance data to governance actions.

---

## 1. Data Model

### 1.1 `finance_events` Table

(Full schema covered in 09_COST_AND_BUDGET.md — key fields summarized here)

| Column | Type | Notes |
|--------|------|-------|
| `id` | UUID | PK |
| `companyId` | UUID | FK → companies |
| `agentId`, `issueId`, `projectId`, `goalId` | UUID | Optional scope FKs |
| `heartbeatRunId` | UUID | FK → heartbeat_runs |
| `costEventId` | UUID | FK → cost_events — links to the parent cost event |
| `billingCode` | TEXT | Cross-team cost attribution tag |
| `description` | TEXT | Human-readable description |
| `eventKind` | TEXT | See kinds below |
| `direction` | TEXT | `debit` or `credit` |
| `biller` | TEXT | Who bills (may differ from provider) |
| `provider` | TEXT | LLM provider |
| `executionAdapterType` | TEXT | Which adapter ran |
| `pricingTier` | TEXT | E.g., tier name or plan level |
| `region` | TEXT | Deployment region |
| `model` | TEXT | Model used |
| `quantity` | INT | Number of units |
| `unit` | TEXT | Unit type (see below) |
| `amountCents` | INT | Dollar amount in cents |
| `currency` | TEXT | `USD` (3-letter ISO code) |
| `estimated` | BOOL | Whether cost is estimated vs confirmed |
| `externalInvoiceId` | TEXT | Provider invoice reference |
| `metadataJson` | JSONB | Additional metadata |
| `occurredAt` | TIMESTAMPTZ | When the charge occurred |

### 1.2 Event Kinds (14)

```
inference_charge          — LLM API call cost
platform_fee              — Platform usage fee
credit_purchase           — Credits bought
credit_refund             — Credits refunded
credit_expiry             — Credits expired
byok_fee                  — Bring-your-own-key fee
gateway_overhead          — Gateway/proxy overhead cost
log_storage_charge        — Log storage costs
logpush_charge            — Log forwarding costs
provisioned_capacity_charge — Reserved capacity costs
training_charge           — Model training costs
custom_model_import_charge  — Custom model import
custom_model_storage_charge — Custom model hosting
manual_adjustment         — Manual correction
```

### 1.3 Finance Units (11)

```
input_token, output_token, cached_input_token, request,
credit_usd, credit_unit, model_unit_minute, model_unit_hour,
gb_month, train_token, unknown
```

### 1.4 Billing Types (6)

```
metered_api               — Pay-per-use API calls
subscription_included     — Included in subscription (cost = $0)
subscription_overage      — Usage beyond subscription limit
credits                   — Credit-based consumption
fixed                     — Fixed-price usage
unknown                   — Fallback
```

---

## 2. How Finance Events are Created

### 2.1 Automatic (Heartbeat Post-Processing)

After each heartbeat run, the heartbeat service creates a finance event alongside the cost event:

1. Parse adapter result: `provider`, `biller`, `billingType`, `costUsd`, `model`, usage tokens
2. Resolve billing scope: `issueId`, `projectId` from run context
3. Determine `biller`: adapter may report a different biller than provider (e.g., `"max_subscription"` for Claude Pro)
4. Normalize cost: `subscription_included` → $0; otherwise `round(costUsd * 100)`
5. Create `finance_events` row with full breakdown

### 2.2 Manual (API)

Board operators can create finance events directly:

```
POST /api/companies/{companyId}/finance-events
{
  eventKind: "manual_adjustment",
  direction: "credit",
  biller: "company",
  amountCents: 5000,
  currency: "USD",
  description: "Refund for failed runs",
  occurredAt: "2026-03-23T10:00:00Z"
}
```

---

## 3. Billing Codes

### 3.1 Purpose

Billing codes enable cross-team cost attribution. When an agent works on a task for a different team, the billing code on the issue tags the cost to the right cost center.

### 3.2 Where They Appear

| Entity | Field | Description |
|--------|-------|-------------|
| `issues` | `billingCode` | Optional tag on each issue |
| `cost_events` | `billingCode` | Copied from the issue when recording costs |
| `finance_events` | `billingCode` | Copied from the issue for finance tracking |

### 3.3 Heartbeat Protocol

The Paperclip skill instructs agents: "Set `billingCode` for cross-team work" when creating subtasks for other teams.

---

## 4. Finance API

| Method | Path | Description |
|--------|------|-------------|
| `POST` | `/api/companies/:companyId/finance-events` | Create finance event |
| `GET` | `/api/companies/:companyId/costs/finance-summary` | Debit/credit/net summary |
| `GET` | `/api/companies/:companyId/costs/finance-by-biller` | Per-biller breakdown |
| `GET` | `/api/companies/:companyId/costs/finance-by-kind` | Per-event-kind breakdown |
| `GET` | `/api/companies/:companyId/costs/finance-events` | List finance events (paginated) |

### 4.1 Finance Summary Response

```typescript
{
  companyId: string,
  debitCents: number,       // Total debits
  creditCents: number,      // Total credits
  netCents: number,         // debitCents - creditCents
  estimatedDebitCents: number, // Debits marked as estimated
  eventCount: number
}
```

### 4.2 By-Biller Breakdown

Groups by `biller` with:
- Debit/credit/net totals
- Estimated debit total
- Event count, distinct kind count

### 4.3 By-Kind Breakdown

Groups by `eventKind` with same aggregation fields.

---

## 5. Budget Incidents (Bridge to Governance)

Budget incidents (covered in 01_COMPANY_MANAGEMENT.md) connect finance data to governance:

1. **Soft threshold** (default 80%) → creates `budget_incident` with `thresholdType: "soft"`
2. **Hard threshold** (100%) → creates `budget_incident` with `thresholdType: "hard"` + `budget_override_required` approval
3. Board resolves: `raise_budget_and_resume` or `keep_paused`

The `budget_incidents` table tracks:
- Policy reference, scope (company/agent/project)
- Amount limit vs observed amount
- Window start/end
- Status: `open`, `resolved`, `dismissed`
- Linked approval ID

---

## 6. Biller vs Provider

A key distinction in Paperclip's finance model:

| Concept | Example | Meaning |
|---------|---------|---------|
| **Provider** | `anthropic` | Who provides the model/API |
| **Biller** | `anthropic` | Who charges for usage (same as provider for API) |
| **Biller** | `max_subscription` | Who charges (different when using Claude Pro subscription) |

This matters because:
- Claude Pro subscription usage is `subscription_included` (cost = $0) even though the provider is still `anthropic`
- The cost dashboard can show "you spent $50 on Anthropic API and $0 on your Max subscription" separately

---

## 7. Key Design Decisions

| Decision | Rationale |
|----------|-----------|
| Separate `cost_events` and `finance_events` | Cost events are simple (tokens + cents). Finance events are full accounting entries with direction, units, estimates. |
| Double-entry style (debit/credit) | Enables credits (refunds, adjustments) and net position calculation |
| `estimated` flag | Provider costs may be estimated before final invoice; this tracks accuracy |
| `externalInvoiceId` | Links Paperclip finance events to provider invoices for reconciliation |
| Billing codes on issues | Cross-team cost attribution at the task level |
| Biller ≠ Provider | Correctly handles subscription models where the biller differs from the API provider |
| 14 event kinds | Covers all observed cloud AI billing line items |
| 11 unit types | Covers token-based, time-based, storage-based, and credit-based pricing models |

---

## 8. Observations for Colmena

1. **Finance events are a production accounting feature.** For Colmena MVP, `cost_events` alone (token counts + cents) covers 95% of use cases. Add `finance_events` when enterprise customers need invoice reconciliation.

2. **Billing codes are simple but valuable** — a free-text tag on issues for cost attribution. Easy to implement, useful for any team with shared agents.

3. **The biller/provider distinction** is important for correctly handling subscription models (Claude Pro, ChatGPT Plus). Without it, subscription usage looks artificially expensive.

4. **Budget incidents** are the enforcement mechanism — they bridge cost data to governance actions (pause agent, require board approval). This is core functionality, not optional.

5. **For Colmena MVP**: track `cost_events` with `agentId`, `provider`, `model`, `costCents`, `tokens`. Implement budget policies with soft/hard thresholds. Skip `finance_events`, billing codes, and biller/provider distinction initially.

6. **The `estimated` flag** is forward-thinking — some providers don't report exact costs at API-call time. When Colmena integrates with providers that charge asynchronously, this becomes relevant.
