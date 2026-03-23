# Routines — cron/webhook/API triggers, concurrency & catch-up policies

## Investigation Summary

Routines are recurring tasks that fire on a schedule, webhook, or API call and create execution issues for agents. They bridge the gap between "things that need to happen periodically" and Paperclip's issue-based work model. When a routine fires, it creates a new issue assigned to the configured agent, which triggers a heartbeat. The routine system spans 3 database tables, a ~900-line service, and integrates with the issue, heartbeat, secret, and cron subsystems.

---

## 1. Data Model

### 1.1 `routines` Table

| Column | Type | Default | Notes |
|--------|------|---------|-------|
| `id` | UUID | random | PK |
| `companyId` | UUID | — | FK → companies, CASCADE |
| `projectId` | UUID | — | FK → projects, CASCADE (required) |
| `goalId` | UUID | NULL | FK → goals, SET NULL |
| `parentIssueId` | UUID | NULL | FK → issues, SET NULL — created issues parent under this |
| `title` | TEXT | — | Required. Also becomes the title of created issues |
| `description` | TEXT | NULL | Also becomes issue description |
| `assigneeAgentId` | UUID | — | FK → agents (required — who runs this routine) |
| `priority` | TEXT | `"medium"` | `critical`, `high`, `medium`, `low` |
| `status` | TEXT | `"active"` | `active`, `paused`, `archived` |
| `concurrencyPolicy` | TEXT | `"coalesce_if_active"` | What to do when fired while previous run is still active |
| `catchUpPolicy` | TEXT | `"skip_missed"` | What to do with missed scheduled runs |
| `createdByAgentId`, `createdByUserId` | — | — | Who created |
| `updatedByAgentId`, `updatedByUserId` | — | — | Who last updated |
| `lastTriggeredAt` | TIMESTAMPTZ | NULL | When last triggered |
| `lastEnqueuedAt` | TIMESTAMPTZ | NULL | When last successfully created an issue |
| `createdAt`, `updatedAt` | TIMESTAMPTZ | — | — |

**Indexes:** `(companyId, status)`, `(companyId, assigneeAgentId)`, `(companyId, projectId)`

### 1.2 `routine_triggers` Table

Each routine can have multiple triggers of different kinds.

| Column | Type | Default | Notes |
|--------|------|---------|-------|
| `id` | UUID | random | PK |
| `companyId` | UUID | — | FK → companies, CASCADE |
| `routineId` | UUID | — | FK → routines, CASCADE |
| `kind` | TEXT | — | `"schedule"`, `"webhook"`, `"api"` |
| `label` | TEXT | NULL | Human-readable label |
| `enabled` | BOOL | true | Can disable individual triggers |
| `cronExpression` | TEXT | NULL | Cron expression (schedule triggers only) |
| `timezone` | TEXT | NULL | IANA timezone (schedule triggers only) |
| `nextRunAt` | TIMESTAMPTZ | NULL | Pre-computed next fire time (schedule only) |
| `lastFiredAt` | TIMESTAMPTZ | NULL | — |
| `publicId` | TEXT | NULL | Public webhook endpoint ID (webhook only) |
| `secretId` | UUID | NULL | FK → company_secrets — signing secret (webhook only) |
| `signingMode` | TEXT | NULL | `"bearer"` or `"hmac_sha256"` (webhook only) |
| `replayWindowSec` | INT | NULL | Anti-replay window (webhook only, 30–86400 sec) |
| `lastRotatedAt` | TIMESTAMPTZ | NULL | When webhook secret was last rotated |
| `lastResult` | TEXT | NULL | Human-readable last trigger result |
| `createdByAgentId`, `createdByUserId` | — | — | — |
| `updatedByAgentId`, `updatedByUserId` | — | — | — |
| `createdAt`, `updatedAt` | TIMESTAMPTZ | — | — |

**Indexes:** `(companyId, routineId)`, `(companyId, kind)`, `(nextRunAt)` — scheduler query, unique on `publicId`

### 1.3 `routine_runs` Table

Records each time a routine was triggered.

| Column | Type | Default | Notes |
|--------|------|---------|-------|
| `id` | UUID | random | PK |
| `companyId` | UUID | — | FK → companies, CASCADE |
| `routineId` | UUID | — | FK → routines, CASCADE |
| `triggerId` | UUID | NULL | FK → routine_triggers, SET NULL |
| `source` | TEXT | — | `"schedule"`, `"manual"`, `"api"`, `"webhook"` |
| `status` | TEXT | `"received"` | See status machine below |
| `triggeredAt` | TIMESTAMPTZ | now | When triggered |
| `idempotencyKey` | TEXT | NULL | Dedup key |
| `triggerPayload` | JSONB | NULL | Webhook/API payload |
| `linkedIssueId` | UUID | NULL | FK → issues — the created execution issue |
| `coalescedIntoRunId` | UUID | NULL | If coalesced, which run it merged into |
| `failureReason` | TEXT | NULL | — |
| `completedAt` | TIMESTAMPTZ | NULL | — |
| `createdAt`, `updatedAt` | TIMESTAMPTZ | — | — |

**Indexes:** `(companyId, routineId, createdAt)`, `(triggerId, createdAt)`, `(linkedIssueId)`, `(triggerId, idempotencyKey)`

---

## 2. Routine Run Status Machine

```
received → issue_created → completed
                         → failed
         → coalesced
         → skipped
         → failed
```

| Status | Meaning |
|--------|---------|
| `received` | Trigger fired, processing |
| `issue_created` | Execution issue created, agent will be woken |
| `coalesced` | Merged into an existing active execution (no new issue) |
| `skipped` | Dropped because active execution exists (skip policy) |
| `completed` | Linked issue reached `done` |
| `failed` | Issue moved to `blocked`/`cancelled`, or dispatch error |

---

## 3. Trigger Types

### 3.1 Schedule Triggers

- **Cron expression** with IANA timezone support
- Custom cron parser (`parseCron()` + `validateCron()`) — not a library
- **Next run computation**: iterates minute-by-minute from `after` date until a cron match is found (up to ~2.5M minutes / ~5 years)
- **Timezone handling**: uses `Intl.DateTimeFormat` to get zoned minute parts, then matches against cron fields
- **Pre-computed `nextRunAt`**: stored on the trigger, updated after each fire

### 3.2 Webhook Triggers

- **Public URL**: `POST /api/routine-triggers/public/{publicId}/fire`
- **Signing modes**:
  - `bearer` — `Authorization: Bearer <secret>` header, timing-safe comparison
  - `hmac_sha256` — `X-Paperclip-Signature: sha256=<hmac>` + `X-Paperclip-Timestamp: <epoch>`, with replay window check
- **Secret management**: each webhook trigger gets an encrypted secret in `company_secrets`, rotatable via API
- **Replay protection**: configurable window (default 300 seconds, range 30–86400)

### 3.3 API Triggers

- Fire only via manual run: `POST /api/routines/{routineId}/run`
- No automatic scheduling — purely on-demand
- Useful for "fire-and-forget" one-off executions

---

## 4. Dispatch Flow

When a routine fires (`dispatchRoutineRun`):

1. **Lock routine row** — `SELECT ... FOR UPDATE` to prevent concurrent dispatches
2. **Idempotency check** — if `idempotencyKey` matches an existing run, return it
3. **Check for live execution** — look for open issues with `originKind = "routine_execution"` and an active heartbeat run
4. **Apply concurrency policy**:
   - `coalesce_if_active` → finalize as `coalesced`, link to existing issue
   - `skip_if_active` → finalize as `skipped`, link to existing issue
   - `always_enqueue` → create new issue regardless
5. **Create execution issue** — insert into `issues` table with `originKind = "routine_execution"`, `originId = routineId`
6. **Queue agent wakeup** — trigger heartbeat for the assigned agent
7. **Update trigger state** — `lastFiredAt`, `lastResult`, compute and store `nextRunAt`
8. **Log activity** — for scheduled/webhook triggers

**Error handling**: If issue creation fails, clean up the run record and mark as `failed`. If a unique constraint violation occurs on the routine execution index, treat as a concurrent dispatch and apply concurrency policy.

---

## 5. Concurrency Policies

| Policy | Behavior |
|--------|----------|
| `coalesce_if_active` (default) | New run is immediately finalized as `coalesced` and linked to the active execution issue. No new issue created. |
| `skip_if_active` | New run is finalized as `skipped`. No new issue created. |
| `always_enqueue` | Always creates a new issue, even if one is already active. |

**Live execution detection** checks two paths:
1. Issues with `executionRunId` pointing to a `queued`/`running` heartbeat run
2. Legacy fallback: heartbeat runs with `contextSnapshot.issueId` matching an open issue

---

## 6. Catch-Up Policies

| Policy | Behavior |
|--------|----------|
| `skip_missed` (default) | Missed scheduled runs are dropped. `nextRunAt` jumps to next future occurrence. |
| `enqueue_missed_with_cap` | Missed runs are dispatched up to `MAX_CATCH_UP_RUNS` (25). Each missed tick creates a separate dispatch. |

---

## 7. Scheduler Tick

`tickScheduledTriggers(now)`:

1. Query all schedule triggers where `enabled = true`, `routine.status = "active"`, `nextRunAt <= now`
2. For each due trigger:
   - Compute catch-up runs (if `enqueue_missed_with_cap`)
   - **Optimistic claim**: UPDATE `nextRunAt` to next occurrence, only if `nextRunAt` hasn't changed (prevents double-fire)
   - Dispatch runs sequentially
3. Return `{ triggered: count }`

This is called periodically by the server's scheduler loop.

---

## 8. Agent Access Rules

| Operation | Agent | Board |
|-----------|-------|-------|
| List / Get | ✅ any routine in company | ✅ |
| Create | ✅ own assignment only | ✅ any agent |
| Update | ✅ own assignment only | ✅ any agent |
| Add/update/delete triggers | ✅ own assignment only | ✅ |
| Rotate webhook secret | ✅ own assignment only | ✅ |
| Manual run | ✅ own assignment only | ✅ |
| Reassign to another agent | ❌ | ✅ |

---

## 9. Issue Lifecycle Sync

When a routine's linked execution issue changes status:

```typescript
syncRunStatusForIssue(issueId):
  if issue.status === "done" → finalize run as "completed"
  if issue.status === "blocked" or "cancelled" → finalize run as "failed"
```

This is called from the issue update route, keeping routine run status in sync with issue outcomes.

---

## 10. API Surface

| Method | Path | Description |
|--------|------|-------------|
| `GET` | `/api/companies/:companyId/routines` | List routines (with triggers, latest run, active issue) |
| `GET` | `/api/routines/:routineId` | Get routine detail (with project, assignee, triggers, recent runs, active issue) |
| `POST` | `/api/companies/:companyId/routines` | Create routine |
| `PATCH` | `/api/routines/:routineId` | Update routine |
| `POST` | `/api/routines/:routineId/triggers` | Add trigger |
| `PATCH` | `/api/routine-triggers/:triggerId` | Update trigger |
| `DELETE` | `/api/routine-triggers/:triggerId` | Delete trigger |
| `POST` | `/api/routine-triggers/:triggerId/rotate-secret` | Rotate webhook secret |
| `POST` | `/api/routines/:routineId/run` | Manual run (with optional trigger, payload, idempotency key) |
| `POST` | `/api/routine-triggers/public/:publicId/fire` | Public webhook endpoint |
| `GET` | `/api/routines/:routineId/runs` | List run history (max 200) |

---

## 11. Key Design Decisions

| Decision | Rationale |
|----------|-----------|
| Routines create issues (not direct heartbeats) | Issues provide tracking, comments, status, and audit trail. Routine execution is just a specialized issue. |
| `SELECT FOR UPDATE` on dispatch | Prevents concurrent dispatches from creating duplicate issues |
| Pre-computed `nextRunAt` | Efficient scheduler query — just `WHERE nextRunAt <= now` instead of evaluating every cron expression |
| Optimistic claim on tick | Double-fire prevention — only one server instance can claim a trigger |
| Multiple trigger types per routine | Flexible — a routine can fire on schedule AND via webhook AND via manual API |
| Webhook signing with encrypted secrets | Production-grade security for external integrations |
| Concurrency as routine-level policy | Simple — no per-trigger or per-run overrides needed |
| Catch-up with cap (25) | Prevents runaway catch-up after long downtime while still recovering reasonable backlogs |
| `originKind + originId` on issues | Clean separation — the issue system doesn't know about routines; the link is generic |
| Partial unique index for open routine executions | DB-level guarantee against duplicate active issues per routine |

---

## 12. Observations for Colmena

1. **The "routines create issues" pattern is elegant** — it reuses the entire issue/heartbeat infrastructure. No separate execution engine needed. However, it means routine runs are visible in the issue list (hidden with `hiddenAt` to reduce clutter).

2. **Custom cron parser** — Paperclip rolls its own instead of using a library. For Colmena, consider `cron-parser` or `croner` to save effort.

3. **The `nextRunAt` pre-computation** is a smart optimization — makes the scheduler tick a simple `WHERE` clause instead of evaluating every cron expression every minute.

4. **Webhook signing is production-grade** — HMAC-SHA256 with replay protection is standard. Worth keeping for external integrations.

5. **The `SELECT FOR UPDATE` dispatch lock** is essential for multi-process deployments. Without it, two server instances could fire the same scheduled routine simultaneously.

6. **Catch-up is capped at 25** — prevents a routine that was down for a month from creating 43,200 issues (one per minute). Reasonable default.

7. **The concurrency detection** has a legacy fallback path (checking `contextSnapshot.issueId`), which adds complexity. Colmena can skip this if it starts with the `executionRunId` approach from day one.

8. **Routine detail is rich** — includes project, assignee agent, parent issue, all triggers, recent runs (last 25), and active issue. Good for the UI but many queries per detail call.

9. **For Colmena MVP**, routines could be simplified to just schedule triggers (cron + timezone). Webhook and API triggers are powerful but add significant complexity (secret management, signing, replay protection).

10. **Agent access scoping** (agents can only manage their own routines) is a good governance pattern — prevents agents from creating routines assigned to other agents.
