# Activity & Audit Trail — immutable log of every mutation

## Investigation Summary

Paperclip records every mutation in an append-only activity log. This provides a complete audit trail of what happened, when, and who did it — essential for autonomous systems where agents act independently. The activity log also powers live WebSocket events, plugin event forwarding, the activity feed in the UI, issue-to-run linking, and project cost attribution. It's a single table with a simple schema but is called from nearly every service in the codebase.

---

## 1. Data Model

### 1.1 `activity_log` Table

| Column | Type | Default | Notes |
|--------|------|---------|-------|
| `id` | UUID | random | PK |
| `companyId` | UUID | — | FK → companies |
| `actorType` | TEXT | `"system"` | `"agent"`, `"user"`, `"system"` |
| `actorId` | TEXT | — | Agent ID, user ID, or system identifier (e.g., `"budget_service"`, `"hire_hook"`, `"routine-scheduler"`) |
| `action` | TEXT | — | Dot-notation action name (e.g., `"issue.created"`, `"agent.paused"`) |
| `entityType` | TEXT | — | What was affected: `"issue"`, `"agent"`, `"approval"`, `"company"`, `"project"`, `"goal"`, `"routine"`, `"routine_run"`, `"heartbeat_run"`, `"budget_policy"`, `"budget_incident"` |
| `entityId` | TEXT | — | ID of the affected entity (stored as TEXT, not UUID, for flexibility) |
| `agentId` | UUID | NULL | FK → agents — the agent performing the action (if applicable) |
| `runId` | UUID | NULL | FK → heartbeat_runs — the run context (if within a heartbeat) |
| `details` | JSONB | NULL | Action-specific data (sanitized and redacted) |
| `createdAt` | TIMESTAMPTZ | now | Immutable timestamp |

**Indexes:**
- `(companyId, createdAt)` — chronological company-wide feed
- `(runId)` — find all activity for a specific heartbeat run
- `(entityType, entityId)` — find all activity for a specific entity

**Key property:** Append-only. No UPDATE or DELETE operations on this table.

---

## 2. What Gets Logged

### 2.1 Action Catalog

| Action | Entity Type | When |
|--------|------------|------|
| `company.created` | company | Company created |
| `company.updated` | company | Company settings changed |
| `company.branding_updated` | company | Name/description/color/logo changed |
| `company.archived` | company | Company archived |
| `company.imported` | company | Company imported from package |
| `agent.created` | agent | Agent created (direct, no approval) |
| `agent.hire_created` | agent | Agent hire request created |
| `agent.updated` | agent | Agent config changed |
| `agent.paused` | agent | Agent paused |
| `agent.resumed` | agent | Agent resumed |
| `agent.terminated` | agent | Agent terminated |
| `agent.deleted` | agent | Agent hard deleted |
| `agent.key_created` | agent | API key created |
| `agent.config_rolled_back` | agent | Config rollback applied |
| `agent.runtime_session_reset` | agent | Session cleared |
| `agent.permissions_updated` | agent | Permissions changed |
| `agent.skills_synced` | agent | Skills synced |
| `agent.instructions_path_updated` | agent | Instructions path changed |
| `agent.instructions_bundle_updated` | agent | Instructions bundle changed |
| `agent.instructions_file_updated` | agent | Individual instructions file changed |
| `agent.instructions_file_deleted` | agent | Instructions file deleted |
| `issue.created` | issue | Issue created |
| `issue.updated` | issue | Issue fields changed |
| `issue.checked_out` | issue | Agent claimed task |
| `issue.released` | issue | Agent released task |
| `issue.deleted` | issue | Issue deleted |
| `issue.comment_added` | issue | Comment posted |
| `issue.document_upserted` | issue | Document created/updated |
| `issue.document_deleted` | issue | Document deleted |
| `issue.attachment_created` | issue | Attachment uploaded |
| `issue.attachment_deleted` | issue | Attachment removed |
| `issue.approval_linked` | issue | Approval linked to issue |
| `issue.approval_unlinked` | issue | Approval unlinked |
| `approval.created` | approval | Approval request created |
| `approval.approved` | approval | Board approved |
| `approval.rejected` | approval | Board rejected |
| `approval.revision_requested` | approval | Board requested revision |
| `approval.resubmitted` | approval | Agent resubmitted |
| `approval.comment_added` | approval | Comment on approval |
| `approval.requester_wakeup_queued` | approval | Requesting agent woken after resolution |
| `approval.requester_wakeup_failed` | approval | Wakeup failed |
| `goal.created` | goal | Goal created |
| `goal.updated` | goal | Goal changed |
| `goal.deleted` | goal | Goal deleted |
| `project.created` | project | Project created |
| `project.updated` | project | Project changed |
| `project.deleted` | project | Project deleted |
| `project.workspace_created` | project | Workspace added |
| `project.workspace_updated` | project | Workspace changed |
| `project.workspace_deleted` | project | Workspace removed |
| `routine.created` | routine | Routine created |
| `routine.updated` | routine | Routine changed |
| `routine.trigger_created` | routine | Trigger added |
| `routine.trigger_updated` | routine | Trigger changed |
| `routine.trigger_deleted` | routine | Trigger removed |
| `routine.trigger_secret_rotated` | routine | Webhook secret rotated |
| `routine.run_triggered` | routine_run | Routine fired (automated) |
| `heartbeat.invoked` | heartbeat_run | Heartbeat manually invoked |
| `heartbeat.cancelled` | heartbeat_run | Run cancelled |
| `budget.policy_upserted` | budget_policy | Budget policy created/updated |
| `budget.soft_threshold_crossed` | budget_incident | 80% threshold hit |
| `budget.hard_threshold_crossed` | budget_incident | 100% threshold hit |
| `budget.incident_resolved` | budget_incident | Incident resolved |
| `hire_hook.succeeded` | agent | Hire notification delivered |
| `hire_hook.failed` | agent | Hire notification failed |
| `hire_hook.error` | agent | Hire notification errored |

### 2.2 Details Field

The `details` JSONB varies by action. Examples:

```typescript
// agent.updated
{ changedTopLevelKeys: ["adapterConfig", "runtimeConfig"], changedAdapterConfigKeys: ["model"] }

// issue.created
{ title: "Implement caching", assigneeAgentId: "...", projectId: "..." }

// budget.hard_threshold_crossed
{ scopeType: "agent", scopeId: "...", amountObserved: 5200, amountLimit: 5000, approvalId: "..." }

// agent.hire_created
{ name: "Engineer", role: "engineer", requiresApproval: true, approvalId: "...", issueIds: [...] }
```

Details are **sanitized** (secrets removed via `sanitizeRecord`) and **redacted** (usernames censored if `censorUsernameInLogs` is enabled).

---

## 3. How Activity is Logged

### 3.1 `logActivity()` Function

Every service that performs mutations calls `logActivity(db, input)`:

```typescript
await logActivity(db, {
  companyId: "...",
  actorType: "agent",           // "agent" | "user" | "system"
  actorId: "agent-42",          // Who did it
  action: "issue.updated",      // What happened
  entityType: "issue",          // What was affected
  entityId: "issue-101",        // Which one
  agentId: "agent-42",          // Optional agent FK
  runId: "run-xyz",             // Optional heartbeat run FK
  details: { status: "done" },  // Optional action-specific data
});
```

### 3.2 Side Effects of Logging

Each `logActivity()` call does three things:

1. **INSERT** into `activity_log` table
2. **Publish WebSocket event** — `activity.logged` live event for real-time UI updates
3. **Forward to plugin event bus** — if the action matches a plugin event type, emits to registered plugins (non-blocking, failures caught)

### 3.3 Actor Resolution

Route handlers use `getActorInfo(req)` to extract actor metadata from the request:

```typescript
{
  actorType: "agent" | "user",
  actorId: string,       // Agent ID or user ID
  agentId: string | null, // Agent ID if actor is agent
  runId: string | null,   // From X-Paperclip-Run-Id header
}
```

---

## 4. Querying Activity

### 4.1 API Surface

| Method | Path | Description |
|--------|------|-------------|
| `GET` | `/api/companies/:companyId/activity` | List activity (filterable) |
| `POST` | `/api/companies/:companyId/activity` | Create activity event (board only) |
| `GET` | `/api/issues/:id/activity` | Activity for a specific issue |
| `GET` | `/api/issues/:id/runs` | Heartbeat runs that touched an issue |
| `GET` | `/api/heartbeat-runs/:runId/issues` | Issues touched by a specific run |

### 4.2 Filters

| Parameter | Description |
|-----------|-------------|
| `agentId` | Filter by actor agent |
| `entityType` | Filter by entity type |
| `entityId` | Filter by specific entity |

### 4.3 Issue Activity

`forIssue(issueId)` returns all activity log entries where `entityType = 'issue'` and `entityId = issueId`, ordered by `createdAt` descending.

### 4.4 Runs-for-Issue Query

`runsForIssue(companyId, issueId)` finds all heartbeat runs that interacted with an issue, using two paths:
1. `heartbeatRuns.contextSnapshot.issueId = issueId` (run was triggered by this issue)
2. EXISTS in `activity_log` where `entityType = 'issue'` AND `entityId = issueId` AND `runId = heartbeatRuns.id` (run logged activity on this issue)

### 4.5 Issues-for-Run Query

`issuesForRun(runId)` finds all issues a run interacted with:
1. Activity log entries where `runId` matches and `entityType = 'issue'`
2. Plus the `contextSnapshot.issueId` from the run record (if not already in the list)
3. Filters out hidden issues (`hiddenAt IS NULL`)

### 4.6 Hidden Issue Filtering

The main activity list filters out activity for hidden issues (routine execution issues with `hiddenAt` set), using a LEFT JOIN to the issues table:
```sql
WHERE entity_type != 'issue' OR issues.hidden_at IS NULL
```

---

## 5. Plugin Event Forwarding

When `logActivity()` is called, if the action matches a known plugin event type, it's forwarded to the plugin event bus:

```typescript
const PLUGIN_EVENT_TYPES = [
  "company.created", "company.updated",
  "project.created", "project.updated", "project.workspace_created", ...
  "issue.created", "issue.updated", "issue.comment.created",
  "agent.created", "agent.updated", "agent.status_changed", "agent.run.started", ...
  "goal.created", "goal.updated",
  "approval.created", "approval.decided",
  "cost_event.created", "activity.logged",
];
```

Plugin events include: `eventId`, `eventType`, `occurredAt`, `actorId`, `actorType`, `entityId`, `entityType`, `companyId`, `payload`.

---

## 6. Secondary Uses of Activity Log

Beyond audit trail, the activity log is used for:

1. **Project cost attribution** — cost events without `projectId` can be linked to projects via activity log entries (which run touched which issue in which project)
2. **Issue-run linking** — determines which runs worked on which issues
3. **Real-time UI feed** — `activity.logged` WebSocket events power the activity sidebar
4. **Plugin event dispatch** — domain events forwarded to plugins

---

## 7. Key Design Decisions

| Decision | Rationale |
|----------|-----------|
| Append-only table | Immutable audit trail — no rewriting history |
| `entityId` as TEXT (not UUID) | Flexibility — can store UUIDs, identifiers, or any string ID |
| `details` as JSONB | Action-specific data without schema migrations for each new action |
| Sanitized + redacted details | Prevents secrets and usernames from appearing in the audit trail |
| WebSocket event on every log | Real-time UI without polling |
| Plugin event forwarding | Plugins react to domain events without coupling to individual services |
| Run ID tracking | Links activity to heartbeat runs for debugging and cost attribution |
| Hidden issue filtering | Keeps routine execution noise out of the main activity feed |
| No pagination | Returns all matching activity — adequate for typical company sizes but would need pagination at scale |

---

## 8. Observations for Colmena

1. **The activity log is infrastructure, not a feature** — nearly every service calls `logActivity()`. Plan for it from day one; retrofitting is painful.

2. **The three-in-one pattern** (DB insert + WebSocket + plugin bus) is elegant but creates coupling. Consider an event bus pattern where logging, WebSocket, and plugin dispatch are separate subscribers.

3. **No pagination on the list endpoint** — returns all activity. For Colmena, add cursor-based pagination from the start (the `(companyId, createdAt)` index supports this efficiently).

4. **`entityId` as TEXT is pragmatic** — avoids FK constraints to every possible entity table. The tradeoff is no referential integrity, but for an append-only log that's acceptable.

5. **The run-to-issue linking via activity log** is used for project cost attribution — a secondary use that should be well-documented. For Colmena, consider always setting `projectId` on cost events to avoid this indirection.

6. **Plugin event forwarding** from the activity log is a clean integration point. If Colmena has plugins, this pattern is worth replicating.

7. **For Colmena MVP**, a simplified activity log with: `companyId`, `actorType`, `actorId`, `action`, `entityType`, `entityId`, `details`, `createdAt`. Skip plugin forwarding and run-ID tracking initially. Add WebSocket events when the UI needs real-time updates.

8. **The sanitization/redaction pipeline** on details is important — without it, the activity feed could leak API keys or usernames.
