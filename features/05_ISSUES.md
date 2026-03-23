# Issues (Tasks) — atomic checkout, status machine, comments, mentions, documents, attachments

## Investigation Summary

Issues are the unit of work in Paperclip. They are the most feature-rich entity in the system — the `issues` table has 30+ columns, the service file is 1,600+ lines, and there are 7 supporting tables. Issues implement atomic checkout (ensuring only one agent works on a task at a time), hierarchical task trees, @-mention triggers, revisioned documents, file attachments, labels, read tracking, and work products.

---

## 1. Data Model

### 1.1 `issues` Table (core)

| Column | Type | Default | Notes |
|--------|------|---------|-------|
| `id` | UUID | random | PK |
| `companyId` | UUID | — | FK → companies |
| `projectId` | UUID | NULL | FK → projects |
| `projectWorkspaceId` | UUID | NULL | FK → project_workspaces (SET NULL on delete) |
| `goalId` | UUID | NULL | FK → goals |
| `parentId` | UUID | NULL | Self-referencing FK → issues (hierarchical tasks) |
| `title` | TEXT | — | Required, min 1 char |
| `description` | TEXT | NULL | Markdown |
| `status` | TEXT | `"backlog"` | Enum: `backlog`, `todo`, `in_progress`, `in_review`, `done`, `blocked`, `cancelled` |
| `priority` | TEXT | `"medium"` | Enum: `critical`, `high`, `medium`, `low` |
| `assigneeAgentId` | UUID | NULL | FK → agents (agent assignee) |
| `assigneeUserId` | TEXT | NULL | Human assignee (mutually exclusive with agent) |
| `checkoutRunId` | UUID | NULL | FK → heartbeat_runs — which run currently owns this task |
| `executionRunId` | UUID | NULL | FK → heartbeat_runs — active execution run |
| `executionAgentNameKey` | TEXT | NULL | Cached agent name for the execution |
| `executionLockedAt` | TIMESTAMPTZ | NULL | When execution lock was acquired |
| `createdByAgentId` | UUID | NULL | FK → agents |
| `createdByUserId` | TEXT | NULL | — |
| `issueNumber` | INT | NULL | Auto-incrementing per company |
| `identifier` | TEXT | NULL | Human-readable ID, e.g., `MYC-42` (unique) |
| `originKind` | TEXT | `"manual"` | Enum: `manual`, `routine_execution` |
| `originId` | TEXT | NULL | Link to source (e.g., routine ID) |
| `originRunId` | TEXT | NULL | — |
| `requestDepth` | INT | 0 | Delegation depth tracking |
| `billingCode` | TEXT | NULL | Cost attribution for cross-team work |
| `assigneeAdapterOverrides` | JSONB | NULL | Per-issue adapter config overrides |
| `executionWorkspaceId` | UUID | NULL | FK → execution_workspaces |
| `executionWorkspacePreference` | TEXT | NULL | Workspace strategy override |
| `executionWorkspaceSettings` | JSONB | NULL | Per-issue workspace config |
| `startedAt` | TIMESTAMPTZ | NULL | Auto-set on transition to `in_progress` |
| `completedAt` | TIMESTAMPTZ | NULL | Auto-set on `done` |
| `cancelledAt` | TIMESTAMPTZ | NULL | Auto-set on `cancelled` |
| `hiddenAt` | TIMESTAMPTZ | NULL | Soft hide (routine execution issues) |
| `createdAt` | TIMESTAMPTZ | now | — |
| `updatedAt` | TIMESTAMPTZ | now | — |

**Key indexes:**
- `(companyId, status)` — filter by status
- `(companyId, assigneeAgentId, status)` — agent inbox query
- `(companyId, assigneeUserId, status)` — human inbox query
- `(companyId, parentId)` — child issues
- `(companyId, projectId)` — project issues
- Unique on `identifier` — human-readable IDs globally unique
- Partial unique on `(companyId, originKind, originId)` for open routine executions

### 1.2 `issue_comments` Table

| Column | Type | Notes |
|--------|------|-------|
| `id` | UUID | PK |
| `companyId` | UUID | FK → companies |
| `issueId` | UUID | FK → issues |
| `authorAgentId` | UUID | FK → agents (NULL for human authors) |
| `authorUserId` | TEXT | NULL for agent authors |
| `body` | TEXT | Markdown content |
| `createdAt`, `updatedAt` | TIMESTAMPTZ | — |

**Indexes:** `(issueId)`, `(companyId)`, `(companyId, issueId, createdAt)`, `(companyId, authorUserId, issueId, createdAt)`

### 1.3 `issue_documents` Table (keyed revisioned docs)

Join table linking issues to `documents`:

| Column | Type | Notes |
|--------|------|-------|
| `id` | UUID | PK |
| `companyId` | UUID | FK → companies |
| `issueId` | UUID | FK → issues, CASCADE |
| `documentId` | UUID | FK → documents, CASCADE |
| `key` | TEXT | Stable identifier: `plan`, `design`, `notes`, etc. |

**Unique:** `(companyId, issueId, key)` — one document per key per issue.

Documents themselves are stored in a separate `documents` table with `document_revisions` for version history.

### 1.4 `issue_attachments` Table

| Column | Type | Notes |
|--------|------|-------|
| `id` | UUID | PK |
| `companyId` | UUID | FK → companies |
| `issueId` | UUID | FK → issues, CASCADE |
| `assetId` | UUID | FK → assets, CASCADE |
| `issueCommentId` | UUID | FK → issue_comments (SET NULL) — optional link to comment |

### 1.5 `issue_labels` Table

Many-to-many join:

| Column | Type | Notes |
|--------|------|-------|
| `issueId` | UUID | FK → issues, CASCADE |
| `labelId` | UUID | FK → labels, CASCADE |
| `companyId` | UUID | FK → companies, CASCADE |

**PK:** `(issueId, labelId)`

Labels are company-scoped (`labels` table: `id`, `companyId`, `name`, `color`).

### 1.6 `issue_approvals` Table

Links issues to approvals (e.g., hire requests):

| Column | Type | Notes |
|--------|------|-------|
| `issueId` | UUID | FK → issues, CASCADE |
| `approvalId` | UUID | FK → approvals, CASCADE |
| `companyId` | UUID | FK → companies |
| `linkedByAgentId`, `linkedByUserId` | — | Who linked them |

### 1.7 `issue_read_states` Table

Tracks which human users have read which issues:

| Column | Type | Notes |
|--------|------|-------|
| `id` | UUID | PK |
| `companyId` | UUID | FK → companies |
| `issueId` | UUID | FK → issues |
| `userId` | TEXT | Human user ID |
| `lastReadAt` | TIMESTAMPTZ | — |

**Unique:** `(companyId, issueId, userId)` — upsert-friendly.

### 1.8 `issue_work_products` Table

Tracks outputs of agent work (PRs, deployments, services):

| Column | Type | Notes |
|--------|------|-------|
| `id` | UUID | PK |
| `companyId` | UUID | FK → companies |
| `issueId` | UUID | FK → issues, CASCADE |
| `type` | TEXT | e.g., `pull_request`, `deployment`, `service` |
| `provider` | TEXT | e.g., `github`, `vercel` |
| `externalId` | TEXT | External system ID |
| `title` | TEXT | — |
| `url` | TEXT | Link to the work product |
| `status` | TEXT | Provider-specific status |
| `reviewState` | TEXT | `none`, `pending`, `approved`, etc. |
| `isPrimary` | BOOL | Primary work product for the issue |
| `healthStatus` | TEXT | `unknown`, `healthy`, `unhealthy` |
| `summary` | TEXT | — |
| `metadata` | JSONB | — |
| `createdByRunId` | UUID | FK → heartbeat_runs |

---

## 2. Status Machine

```
backlog → todo → in_progress → in_review → done
                      ↕              ↕
                   blocked      in_progress
```

**Terminal states:** `done`, `cancelled`

**Status side effects (auto-set on transition):**
| Transition to | Side effect |
|--------------|-------------|
| `in_progress` | `startedAt = now()` |
| `done` | `completedAt = now()` |
| `cancelled` | `cancelledAt = now()` |
| Any non-done | `completedAt = null` |
| Any non-cancelled | `cancelledAt = null` |
| Any non-in_progress | `checkoutRunId = null` |

**Constraints:**
- `in_progress` requires an assignee (agent or user)
- Reassigning an issue clears `checkoutRunId`
- No explicit transition validation (any status → any status allowed at DB level; `assertTransition` just checks the status is valid)

---

## 3. Atomic Checkout System

The checkout system is the most critical piece of issue management — it ensures exactly one agent works on a task at any time.

### 3.1 Checkout Flow

```
POST /api/issues/{issueId}/checkout
{
  "agentId": "{yourAgentId}",
  "expectedStatuses": ["todo", "backlog", "blocked"]
}
Headers: X-Paperclip-Run-Id: {runId}
```

**Algorithm:**
1. Validate agent is assignable (exists, same company, not terminated/pending)
2. Attempt atomic UPDATE:
   ```sql
   UPDATE issues SET
     assignee_agent_id = $agentId,
     checkout_run_id = $runId,
     execution_run_id = $runId,
     status = 'in_progress',
     started_at = now()
   WHERE id = $issueId
     AND status IN ($expectedStatuses)
     AND (assignee_agent_id IS NULL OR (assignee_agent_id = $agentId AND checkout_run_id IS NULL))
     AND (execution_run_id IS NULL OR execution_run_id = $runId)
   ```
3. If UPDATE returns 0 rows → check why:
   - **Already owned by same agent + same run** → return success (idempotent)
   - **Owned by same agent but different run** → attempt stale lock adoption
   - **Owned by different agent** → `409 Conflict`

### 3.2 Stale Lock Adoption

When an agent's previous run crashed while holding a checkout:

```typescript
async function adoptStaleCheckoutRun(input) {
  // Check if the previous run is terminal (succeeded/failed/cancelled/timed_out) or missing
  const stale = await isTerminalOrMissingHeartbeatRun(input.expectedCheckoutRunId);
  if (!stale) return null;
  // Atomically adopt the lock
  UPDATE issues SET checkout_run_id = $newRunId, execution_run_id = $newRunId
  WHERE id = $issueId AND status = 'in_progress'
    AND assignee_agent_id = $agentId AND checkout_run_id = $oldRunId
}
```

This handles the common case where an agent's heartbeat timed out while working on a task.

### 3.3 Release

```
POST /api/issues/{issueId}/release
```

- Only the assignee can release
- Only the checkout run can release (prevents stale releases)
- Sets status back to `todo`, clears `assigneeAgentId` and `checkoutRunId`

---

## 4. Issue Identifier System

Each issue gets a human-readable identifier like `MYC-42`:

1. On create, atomically increment `companies.issueCounter`:
   ```sql
   UPDATE companies SET issue_counter = issue_counter + 1
   WHERE id = $companyId
   RETURNING issue_counter, issue_prefix
   ```
2. Compose identifier: `${issuePrefix}-${issueNumber}`
3. Store both `issueNumber` and `identifier` on the issue

Identifiers can be used to look up issues (`GET /api/issues/MYC-42`).

---

## 5. Comments & @-Mentions

### 5.1 Comments

- Markdown body, authored by agent or user
- Can be added inline with issue update: `PATCH /api/issues/{id} { "status": "done", "comment": "..." }`
- Adding a comment updates `issues.updatedAt` (for recency sorting)
- Supports pagination: `after` cursor (comment ID), `order` (asc/desc), `limit` (max 500)

### 5.2 @-Mention Detection

```typescript
async function findMentionedAgents(companyId: string, body: string) {
  // Extract @tokens using regex: /\B@([^\s@,!?.]+)/g
  // Load all agents in company
  // Case-insensitive match on agent name
  // Return matched agent IDs
}
```

When an @-mention is found in a comment, the system triggers a heartbeat for the mentioned agent (handled at the route level, not in the service).

### 5.3 Project Mentions

Issues can also reference projects by ID in text, extracted via `extractProjectMentionIds()`.

---

## 6. Documents (Keyed, Revisioned)

Issues support keyed documents — stable named text artifacts like `plan`, `design`, `notes`:

- **Create/Update:** `PUT /api/issues/{issueId}/documents/{key}` with `{ title, format, body, baseRevisionId }`
- **Optimistic concurrency:** Must provide `baseRevisionId` when updating; stale → `409 Conflict`
- **Revision history:** Every edit creates a new revision in `document_revisions`
- **Key format:** lowercase letters, numbers, underscores, hyphens (max 64 chars)
- **Max body size:** 512KB
- **Special `plan` key:** Surfaced directly in issue detail response as `planDocument`

---

## 7. Attachments

- Upload via `POST /api/companies/{companyId}/issues/{issueId}/attachments` (multipart)
- Stored as `assets` (company-scoped) linked via `issue_attachments` join
- Can optionally be linked to a specific comment (`issueCommentId`)
- CRUD: upload, list, download (via asset content endpoint), delete

---

## 8. Labels

- Company-scoped: `labels` table with `name` (max 48 chars) and `color` (hex)
- Many-to-many with issues via `issue_labels`
- Sync pattern: full replace on every update (`syncIssueLabels`)
- Filter issues by label

---

## 9. Read Tracking & Inbox

### 9.1 Read States

- Per-user read tracking via `issue_read_states`
- Upsert-friendly: `INSERT ... ON CONFLICT DO UPDATE`
- Mark read: `POST /api/issues/{id}/read`

### 9.2 User Context

For human users, each issue response can include:
```typescript
{
  myLastTouchAt: Date | null,        // Latest of: my comment, my read, my create, my assignment
  lastExternalCommentAt: Date | null, // Latest comment from someone else
  isUnreadForMe: boolean,            // External comment after my last touch
}
```

### 9.3 Inbox Queries

Two special filters:
- `touchedByUserId` — issues I created, am assigned to, have read, or commented on
- `unreadForUserId` — subset of touched issues with new external comments since my last interaction

These use complex SQL subqueries with `EXISTS` and `GREATEST()`.

---

## 10. Work Products

Track concrete outputs of agent work:
- Pull requests, deployments, runtime services
- Linked to execution workspaces and runtime services
- Provider-specific (GitHub, Vercel, etc.)
- Track review state, health status
- Primary work product per issue

---

## 11. Search

Full-text search across issues:

```typescript
// Search priority (ordered by relevance):
// 0: title starts with query
// 1: title contains query
// 2: identifier starts with query
// 3: identifier contains query
// 4: description contains query
// 5: comment body contains query
```

Uses `ILIKE` with escaped patterns. Not full-text search (no tsvector), but functional for typical org sizes.

---

## 12. Issue List Sorting

- **Default:** Priority order (`critical` → `high` → `medium` → `low`) then by `updatedAt` descending
- **Search mode:** Relevance score first, then priority, then recency

---

## 13. Goal Fallback

When creating/updating issues, the goal is resolved through a fallback chain:

```typescript
function resolveIssueGoalId({projectId, goalId, defaultGoalId}) {
  // 1. Explicit goalId provided → use it
  // 2. Project has a goalId → use project's goal
  // 3. Fall back to company's default goal
}
```

This ensures every issue has a goal link, even if the creator didn't specify one.

---

## 14. Ancestor Chain

`getAncestors(issueId)` walks up the `parentId` chain (same N+1 pattern as chain-of-command) and batch-fetches related projects, goals, and workspaces. Returns an array of ancestors from direct parent up to root, each enriched with:
- Project details (name, status, workspaces, primary workspace)
- Goal details (title, level, status)

This gives agents complete context about why a task exists.

---

## 15. Key Design Decisions

| Decision | Rationale |
|----------|-----------|
| Atomic checkout via conditional UPDATE | Race-condition-safe without explicit locks. One SQL statement determines the winner |
| Stale lock adoption | Handles crashed runs gracefully without manual intervention |
| `checkoutRunId` + `executionRunId` | Separate checkout ownership from execution tracking. A run may adopt a stale checkout from a different run |
| Mutually exclusive agent/user assignee | Simplifies the model — one assignee type at a time |
| Auto-incrementing `issueNumber` per company | Human-readable IDs (MYC-42) without requiring external sequence generators |
| Inline comment on update | Reduces API calls — agents can update status and explain in one request |
| Keyed documents with optimistic concurrency | Plan documents can be updated by multiple agents without conflicts; revision history provides audit trail |
| Labels as company-scoped with full-replace sync | Simple, adequate for typical label usage |
| Read tracking per user (not per agent) | Agents don't need read tracking; they use the heartbeat protocol |
| ILIKE search (not full-text) | Simpler, adequate for typical org sizes. Can be replaced with tsvector later |
| Routine execution issues with `hiddenAt` | Keep routine-generated issues from cluttering the main issue list |
| Work products as first-class entities | Enables tracking PRs, deployments alongside tasks |

---

## 16. Observations for Colmena

1. **The atomic checkout is non-negotiable** — this is how Paperclip prevents two agents from working on the same task. The conditional UPDATE pattern is elegant and race-condition-safe. Must keep.

2. **Stale lock adoption is critical for reliability** — agent crashes happen. Without this, tasks get permanently stuck in `in_progress` with a dead run holding the lock.

3. **The issue table is very wide (30+ columns)** — consider if all fields are needed for Colmena MVP. Execution workspace fields, adapter overrides, and work products could be deferred.

4. **Documents system (keyed, revisioned) is powerful but complex** — for MVP, a simple `description` field may suffice. Add documents when agents need to collaborate on long-form plans.

5. **@-mention detection is simple regex** — works well for controlled agent names. Consider if Colmena needs more sophisticated mention parsing.

6. **Read tracking and inbox queries** are user-facing features (for the board UI). Not critical for agent operation but important for human oversight.

7. **The identifier system** (`MYC-42`) is a great UX touch — agents and humans can reference issues naturally. Worth keeping.

8. **Goal fallback** ensures no orphan tasks — every issue traces to a goal, even by default. Good pattern for maintaining alignment.

9. **Search is basic ILIKE** — functional but won't scale to thousands of issues. Consider pg_trgm or tsvector for larger deployments.

10. **Work products** bridge the gap between task management and delivery tracking. Powerful for production but optional for MVP.
