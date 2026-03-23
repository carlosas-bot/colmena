# Heartbeat System — trigger-based execution, heartbeat protocol, run tracking, live log streaming

## Investigation Summary

The heartbeat system is Paperclip's execution engine — it's how agents actually run. Agents don't execute continuously; they wake up in short bursts called "heartbeats," do work, and go back to sleep. The heartbeat service is the largest file in the codebase (~3,700 lines) and orchestrates the entire lifecycle: trigger → enqueue → budget check → workspace resolution → session restoration → adapter invocation → output capture → cost recording → session persistence → status update. It spans 2 core database tables, integrates with 6+ other services, and publishes live WebSocket events.

---

## 1. Data Model

### 1.1 `heartbeat_runs` Table

Records every heartbeat execution.

| Column | Type | Default | Notes |
|--------|------|---------|-------|
| `id` | UUID | random | PK |
| `companyId` | UUID | — | FK → companies |
| `agentId` | UUID | — | FK → agents |
| `invocationSource` | TEXT | `"on_demand"` | Enum: `timer`, `assignment`, `on_demand`, `automation` |
| `triggerDetail` | TEXT | NULL | `"manual"`, `"ping"`, `"callback"`, `"system"` |
| `status` | TEXT | `"queued"` | Enum: `queued`, `running`, `succeeded`, `failed`, `cancelled`, `timed_out` |
| `startedAt` | TIMESTAMPTZ | NULL | When execution began |
| `finishedAt` | TIMESTAMPTZ | NULL | When execution completed |
| `error` | TEXT | NULL | Error message if failed |
| `wakeupRequestId` | UUID | NULL | FK → agent_wakeup_requests |
| `exitCode` | INT | NULL | Process exit code |
| `signal` | TEXT | NULL | Process signal (e.g., SIGTERM) |
| `usageJson` | JSONB | NULL | Token usage: inputTokens, outputTokens, cachedInputTokens, costUsd |
| `resultJson` | JSONB | NULL | Adapter result summary |
| `sessionIdBefore` | TEXT | NULL | Session state before run |
| `sessionIdAfter` | TEXT | NULL | Session state after run |
| `logStore` | TEXT | NULL | Log storage backend (e.g., `"local_file"`) |
| `logRef` | TEXT | NULL | Reference to stored log file |
| `logBytes` | BIGINT | NULL | Log file size |
| `logSha256` | TEXT | NULL | Log integrity hash |
| `logCompressed` | BOOL | false | Whether log is compressed |
| `stdoutExcerpt` | TEXT | NULL | Truncated stdout (for previews) |
| `stderrExcerpt` | TEXT | NULL | Truncated stderr |
| `errorCode` | TEXT | NULL | Structured error code |
| `externalRunId` | TEXT | NULL | Adapter's own run ID |
| `processPid` | INT | NULL | OS process ID |
| `processStartedAt` | TIMESTAMPTZ | NULL | When OS process started |
| `retryOfRunId` | UUID | NULL | Self-ref FK — retry chain |
| `processLossRetryCount` | INT | 0 | Auto-retry counter for lost processes |
| `contextSnapshot` | JSONB | NULL | Full context: issueId, projectId, taskKey, wakeReason, commentId, etc. |
| `createdAt`, `updatedAt` | TIMESTAMPTZ | — | — |

**Index:** `(companyId, agentId, startedAt)` — list runs for an agent.

### 1.2 `heartbeat_run_events` Table

Granular event log per run (stdout lines, system messages, errors).

| Column | Type | Notes |
|--------|------|-------|
| `id` | BIGSERIAL | PK (auto-increment) |
| `companyId` | UUID | FK → companies |
| `runId` | UUID | FK → heartbeat_runs |
| `agentId` | UUID | FK → agents |
| `seq` | INT | Sequence number within run (for ordering) |
| `eventType` | TEXT | e.g., `system`, `stdout`, `stderr`, `tool_call` |
| `stream` | TEXT | `"system"`, `"stdout"`, `"stderr"` |
| `level` | TEXT | `"info"`, `"warn"`, `"error"` |
| `color` | TEXT | Optional color hint for UI rendering |
| `message` | TEXT | Human-readable message |
| `payload` | JSONB | Structured event data |
| `createdAt` | TIMESTAMPTZ | — |

**Indexes:** `(runId, seq)`, `(companyId, runId)`, `(companyId, createdAt)`

---

## 2. Run Status Machine

```
queued → running → succeeded
                 → failed
                 → cancelled
                 → timed_out
```

**Run lifecycle:**
1. **queued** — wakeup request accepted, waiting to start
2. **running** — adapter invocation in progress, agent process active
3. **succeeded** — agent completed normally (exit code 0)
4. **failed** — agent errored (non-zero exit code, exception, timeout)
5. **cancelled** — manually cancelled by board operator
6. **timed_out** — exceeded configured timeout

---

## 3. Trigger & Invocation Flow

### 3.1 Trigger Sources

| Source | Trigger Detail | When |
|--------|---------------|------|
| `timer` | `system` | Heartbeat interval elapsed (scheduler tick) |
| `assignment` | `system` | New task assigned to agent |
| `on_demand` | `manual` | Board clicks "Invoke" in UI |
| `on_demand` | `ping` | API wakeup call |
| `automation` | `callback` | Routine fired, approval resolved, @-mention |

### 3.2 Wakeup Request Queue

Every trigger goes through `enqueueWakeup()`:

1. **Validate agent** — exists, not terminated/paused/pending_approval
2. **Budget check** — `budgets.getInvocationBlock()` checks company → agent → project budgets
3. **Concurrency check** — count active runs for agent vs `maxConcurrentRuns` (default 1, max 10)
4. **Coalescing** — if an active run exists for the same task scope, coalesce instead of creating a new run
5. **Insert wakeup request** — record in `agent_wakeup_requests` with source, reason, payload, idempotency key
6. **Create heartbeat run** — insert `heartbeat_runs` row with status `queued`
7. **Agent start lock** — per-agent mutex prevents concurrent starts (`startLocksByAgent` Map)
8. **Execute** — start the actual adapter invocation

### 3.3 Coalescing Logic

If a run is already active for the same agent:
- Check if the incoming wakeup has the same task scope (`taskKey`)
- If same scope → coalesce (merge context, increment coalesced count, skip execution)
- If different scope or forced → create separate run

### 3.4 Context Snapshot

Every run carries a `contextSnapshot` JSONB with:
```typescript
{
  issueId?: string,           // Task being worked on
  projectId?: string,         // Project context
  taskId?: string,            // Same as issueId (legacy compat)
  taskKey?: string,           // Session routing key
  commentId?: string,         // Comment that triggered wake
  wakeCommentId?: string,     // Same (normalized)
  wakeReason?: string,        // e.g., "issue_assigned", "issue_comment_mentioned"
  wakeSource?: string,        // "timer", "assignment", etc.
  wakeTriggerDetail?: string, // "manual", "system", etc.
  triggeredBy?: string,       // "agent" or "user"
  actorId?: string,           // Who triggered
  forceFreshSession?: boolean, // Force new session
  executionWorkspaceId?: string,
}
```

---

## 4. Execution Pipeline

### 4.1 Pre-Execution

1. **Resolve workspace** — determine CWD from project workspace, task session, or fallback agent home dir
2. **Resolve session** — load previous session state (agent-level or task-level)
3. **Session compaction check** — if session exceeds thresholds (max runs, max tokens, max age), rotate session with handoff context
4. **Resolve secrets** — decrypt `secret_ref` values in adapter config
5. **Resolve skills** — load company skills + runtime skill entries
6. **Build environment** — inject `PAPERCLIP_*` env vars + resolved secrets + adapter config
7. **Create agent JWT** — short-lived token for API auth during this run

### 4.2 Environment Variables Injected

```typescript
{
  PAPERCLIP_AGENT_ID: agentId,
  PAPERCLIP_COMPANY_ID: companyId,
  PAPERCLIP_API_URL: apiBaseUrl,
  PAPERCLIP_API_KEY: shortLivedJwt,
  PAPERCLIP_RUN_ID: runId,
  PAPERCLIP_TASK_ID: issueId,          // If triggered by a task
  PAPERCLIP_WAKE_REASON: wakeReason,   // e.g., "issue_assigned"
  PAPERCLIP_WAKE_COMMENT_ID: commentId, // If triggered by a comment
  PAPERCLIP_APPROVAL_ID: approvalId,   // If triggered by an approval
  PAPERCLIP_APPROVAL_STATUS: status,   // "approved" or "rejected"
  PAPERCLIP_LINKED_ISSUE_IDS: ids,     // Comma-separated
}
```

### 4.3 Adapter Invocation

The heartbeat service delegates execution to the configured adapter:

```typescript
const adapter = getServerAdapter(agent.adapterType);
const result = await adapter.execute({
  runId, agentId, companyId,
  config: resolvedAdapterConfig,
  cwd: resolvedWorkspace.cwd,
  env: mergedEnvironment,
  sessionParams: previousSessionParams,
  promptPrefix: sessionHandoffMarkdown,
  // ... callbacks for stdout/stderr/events
});
```

During execution, the service:
- Streams stdout/stderr chunks to `heartbeat_run_events` and the log store
- Publishes live WebSocket events (`heartbeat.run.event`, `heartbeat.run.log`)
- Tracks excerpts (truncated stdout/stderr for previews)
- Monitors process PID for orphan detection

### 4.4 Post-Execution

1. **Parse result** — adapter returns: `exitCode`, `error`, `sessionId`, `sessionParams`, `usage`, `costUsd`, `provider`, `model`
2. **Normalize usage** — handle session-cumulative vs per-run token counts (delta calculation)
3. **Record cost event** — insert into `cost_events` for budget tracking
4. **Budget evaluation** — `budgets.evaluateCostEvent()` checks thresholds, creates incidents, auto-pauses
5. **Update session state** — persist new session params to `agent_runtime_state` and/or `agent_task_sessions`
6. **Update agent state** — set `agents.lastHeartbeatAt`, update `agent_runtime_state` totals
7. **Finalize run** — set status to `succeeded`/`failed`, store `usageJson`/`resultJson`, finalize log file
8. **Publish events** — `heartbeat.run.status` WebSocket event

---

## 5. Workspace Resolution

The workspace resolution pipeline determines where the agent runs:

```
Priority:
1. Project workspace → primary workspace CWD (if project is set and directory exists)
2. Managed checkout → auto-clone repo into managed directory (if repoUrl is set)
3. Task session CWD → resume in the same directory as the previous session
4. Fallback agent home → ~/.paperclip/instances/default/agents/{agentId}/workspace/
```

**Managed workspace auto-clone:**
- If a project workspace has `repoUrl` but no `cwd`, Paperclip auto-clones the repo to:
  ```
  ~/.paperclip/instances/default/workspaces/{companyId}/{projectId}/{repoName}/
  ```
- Uses `git clone` with a 10-minute timeout
- Subsequent runs reuse the existing checkout

---

## 6. Session Persistence & Compaction

### 6.1 Session Persistence

After each run, the adapter returns session state (e.g., Claude Code session ID):

- **Agent-level session** → stored in `agent_runtime_state.sessionId` + `stateJson`
- **Task-level session** → stored in `agent_task_sessions.sessionParamsJson` (keyed by `taskKey`)

On the next heartbeat, session state is restored and passed to the adapter, allowing the agent to resume the conversation.

### 6.2 Session Compaction

Sessions can grow unbounded. Paperclip implements automatic rotation:

**Compaction policy** (per-agent, from `runtimeConfig`):
```typescript
{
  maxSessionRuns: number,      // Max heartbeat runs per session (e.g., 20)
  maxRawInputTokens: number,   // Max cumulative input tokens (e.g., 1M)
  maxSessionAgeHours: number,  // Max session age in hours (e.g., 48)
}
```

When a threshold is exceeded:
1. Current session is marked for rotation
2. A **handoff markdown** is generated with:
   - Previous session ID
   - Issue context
   - Rotation reason
   - Last run summary
3. New run starts a fresh session with the handoff markdown as a prompt prefix
4. Agent rebuilds context from the handoff

### 6.3 Task Session Reset on Assignment

When an agent is woken because a new task was assigned (`wakeReason: "issue_assigned"`), the task session is automatically cleared to prevent stale context from a previous task.

---

## 7. Scheduler (Timer-Based Heartbeats)

### 7.1 Tick Timer

`tickTimers()` is called periodically (by the scheduler) to fire timer-based heartbeats:

```typescript
for (const agent of allAgents) {
  if (agent.status is paused/terminated/pending) continue;
  if (!policy.enabled || policy.intervalSec <= 0) continue;
  
  const elapsed = now - (agent.lastHeartbeatAt ?? agent.createdAt);
  if (elapsed >= policy.intervalSec * 1000) {
    enqueueWakeup(agent.id, { source: "timer", ... });
  }
}
```

### 7.2 Instance Scheduler View

`GET /api/instance/scheduler-heartbeats` returns all agents with their scheduler state:
```typescript
{
  id, companyId, companyName, agentName, role, status, adapterType,
  intervalSec, heartbeatEnabled, schedulerActive, lastHeartbeatAt
}
```

---

## 8. Process Management

### 8.1 Orphan Detection

`reapOrphanedRuns()` detects and cleans up runs whose processes have died:
- Checks `processPid` with `process.kill(pid, 0)` (signal 0 = liveness check)
- If process is dead → mark run as `failed` with `process_detached` error code
- Supports auto-retry (`processLossRetryCount`) for transient failures

### 8.2 Run Cancellation

`cancelRun(runId)`:
1. If running → kill the adapter process, set status to `cancelled`
2. Update wakeup request status
3. Clear issue execution lock if the cancelled run owned it
4. Log activity

`cancelActiveForAgent(agentId)`:
- Cancel all `queued` and `running` runs for an agent (used on pause/terminate)

### 8.3 Budget Scope Work Cancellation

When a budget hard-stop fires, `cancelBudgetScopeWork()` cancels active runs for the affected scope (company, agent, or project).

---

## 9. Log Storage

### 9.1 Run Log Store

Full agent output is stored separately from `heartbeat_run_events`:
- **Storage backend:** `local_file` (writes to disk under `~/.paperclip/instances/default/run-logs/`)
- **Fields on run:** `logStore`, `logRef`, `logBytes`, `logSha256`, `logCompressed`
- **Reading:** `readLog(runId, { offset, limitBytes })` — supports range reads for large logs
- **Max chunk for live streaming:** 8KB per WebSocket message

### 9.2 Excerpts

Truncated stdout/stderr excerpts are stored on the run record for quick preview (max `MAX_EXCERPT_BYTES`).

---

## 10. Live Events (WebSocket)

During execution, the heartbeat service publishes real-time events:

| Event Type | When | Payload |
|-----------|------|---------|
| `heartbeat.run.queued` | Run created | runId, agentId, source |
| `heartbeat.run.status` | Status change | runId, status, error, timing |
| `heartbeat.run.event` | Each stdout/stderr/system event | runId, seq, eventType, message |
| `heartbeat.run.log` | Log chunk (capped at 8KB) | runId, chunk |
| `agent.status` | Agent status change | agentId, status |

These power the real-time run viewer in the UI.

---

## 11. Cost & Usage Recording

After each run:

1. **Parse adapter usage:** `{ inputTokens, outputTokens, cachedInputTokens, costUsd, provider, model }`
2. **Normalize to delta:** If adapter reports cumulative session tokens, subtract previous run's totals
3. **Insert cost event:** `cost_events` table with agent/project/company scope
4. **Budget evaluation:** Check against budget policies, create incidents, auto-pause if needed
5. **Update runtime state:** Add to `totalInputTokens`, `totalOutputTokens`, `totalCostCents`
6. **Record finance event:** Detailed billing data (provider, model, billing type, unit costs)

---

## 12. Key Design Decisions

| Decision | Rationale |
|----------|-----------|
| Wakeup queue instead of direct execution | Enables coalescing, concurrency control, and budget gating before spending resources |
| Per-agent start lock (in-memory mutex) | Prevents race conditions when multiple triggers fire simultaneously for the same agent |
| Context snapshot as JSONB | Flexible, extensible — new context fields don't require schema migrations |
| Separate run events table (not inline) | Events are high-volume (every stdout line); separate table keeps runs table lean |
| Log storage as separate file (not in DB) | Agent output can be megabytes; file storage with offset reads is more efficient |
| Session compaction with handoff markdown | Agents need context to continue; a structured handoff is better than a cold start |
| Delta usage normalization | Some adapters report cumulative session tokens; normalizing to per-run deltas enables accurate cost attribution |
| Process PID tracking + orphan reaping | Essential for reliability — processes can be lost due to OOM kills, system restarts, etc. |
| Excerpt storage on run record | Enables quick preview without reading the full log file |

---

## 13. Observations for Colmena

1. **This is the most complex service in Paperclip** (~3,700 lines). For Colmena, consider splitting into: trigger/queue, execution, session management, and post-run processing.

2. **The wakeup queue with coalescing** is sophisticated but essential — without it, rapid triggers (e.g., 5 comments in a minute) would spawn 5 separate heartbeats instead of coalescing into 1.

3. **Session compaction is a production necessity** — without it, Claude sessions grow until they hit context limits. The handoff markdown approach is clever but imperfect; the agent loses some nuance.

4. **Workspace resolution is deeply intertwined** with project workspaces and execution workspaces. For Colmena MVP, a simpler "cwd from project config or agent default" might suffice.

5. **The budget gating before execution** is critical — it prevents spending money on agents that have exceeded their limits.

6. **Live WebSocket events** enable the real-time run viewer, which is one of Paperclip's most compelling UI features.

7. **Process orphan detection** via PID checking is Linux-specific and imperfect (PID recycling). But it's the best available approach for local process adapters.

8. **The delta usage normalization** handles a real problem — Claude Code reports cumulative tokens, not per-run. Without normalization, costs would be wildly inflated.

9. **The heartbeat protocol** (9-step procedure taught via the Paperclip skill) is the contract between agents and the system. It's what makes agents autonomous rather than just tools.

10. **Timer-based scheduling** is simple (poll all agents, check elapsed time) but doesn't scale beyond ~1,000 agents. For Colmena, consider a proper job scheduler (pg-boss, BullMQ) for larger deployments.
