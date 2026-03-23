# Dashboard & Monitoring — agent/task counts, stale alerts, budget utilization, sidebar badges

## Investigation Summary

Paperclip provides two monitoring surfaces: a **dashboard API** that returns a single-call health snapshot of the company, and a **sidebar badge system** that provides real-time notification counts. Together they let board operators see at a glance: how many agents are running, how many tasks are blocked, how much has been spent, whether any budgets are in danger, and whether approvals or failures need attention. The implementation is compact (~200 lines total) but pulls data from 5+ tables via aggregation queries.

---

## 1. Dashboard Summary

### 1.1 API

```
GET /api/companies/{companyId}/dashboard
```

Single-call endpoint that returns the full company health snapshot.

### 1.2 Response Shape

```typescript
interface DashboardSummary {
  companyId: string;
  
  agents: {
    active: number;    // Idle + active (operational agents)
    running: number;   // Currently executing a heartbeat
    paused: number;    // Manually or budget-paused
    error: number;     // Last heartbeat failed
  };
  
  tasks: {
    open: number;        // All non-done, non-cancelled
    inProgress: number;  // Currently being worked on
    blocked: number;     // Stuck, needs attention
    done: number;        // Completed
  };
  
  costs: {
    monthSpendCents: number;           // Current month spend
    monthBudgetCents: number;          // Monthly budget limit
    monthUtilizationPercent: number;   // spend/budget × 100
  };
  
  pendingApprovals: number;   // Approvals waiting for board decision
  
  budgets: {
    activeIncidents: number;     // Open budget incidents
    pendingApprovals: number;    // Budget override approvals pending
    pausedAgents: number;        // Agents paused by budget
    pausedProjects: number;      // Projects paused by budget
  };
}
```

### 1.3 How Data is Computed

5 aggregation queries in a single service call:

| Query | Source | Aggregation |
|-------|--------|-------------|
| Agent counts | `agents` | `GROUP BY status`, idle mapped to "active" |
| Task counts | `issues` | `GROUP BY status`, bucketed into open/inProgress/blocked/done |
| Pending approvals | `approvals` | `COUNT WHERE status = 'pending'` |
| Month spend | `cost_events` | `SUM(cost_cents) WHERE occurred_at >= month_start` |
| Budget overview | `budget_policies` + `budget_incidents` | Full budget overview service call |

### 1.4 Key Design Choices

- **Idle agents counted as active** — from the operator's perspective, idle = ready to work = active
- **Open tasks** = everything not `done` or `cancelled` — gives a single number for "work in flight"
- **Budget overview integrated** — operators see budget health without a separate page
- **No caching** — queries run fresh on every call. Acceptable for typical company sizes (~50 agents, ~500 issues)

---

## 2. Sidebar Badges

### 2.1 API

```
GET /api/companies/{companyId}/sidebar-badges
```

Returns notification counts for the sidebar navigation.

### 2.2 Response Shape

```typescript
interface SidebarBadges {
  inbox: number;        // Combined attention-needed count
  approvals: number;    // Pending + revision-requested approvals
  failedRuns: number;   // Agents whose latest run failed
  joinRequests: number; // Pending join requests (if operator has approve permission)
}
```

### 2.3 How `inbox` is Computed

The inbox badge is a composite of several attention signals:

```typescript
inbox = failedRuns + alertsCount + joinRequests + approvals

// where alertsCount includes:
// +1 if any agents are in error state (and no failedRuns already counted)
// +1 if budget utilization >= 80%
```

### 2.4 Failed Runs Detection

```sql
-- Get the latest run per agent, check if it failed
SELECT DISTINCT ON (heartbeat_runs.agent_id)
  heartbeat_runs.status
FROM heartbeat_runs
INNER JOIN agents ON heartbeat_runs.agent_id = agents.id
WHERE agents.status != 'terminated'
ORDER BY heartbeat_runs.agent_id, heartbeat_runs.created_at DESC
```

Only counts agents whose **most recent** run failed or timed out. Agents with a subsequent successful run are not flagged.

### 2.5 Permission-Aware

Join request count is only included if the requesting user/agent has `joins:approve` permission. This prevents non-admins from seeing irrelevant counts.

### 2.6 Real-Time Updates

Sidebar badges are invalidated via WebSocket live events:
- `activity.logged` → invalidate sidebar badge query
- `heartbeat.run.status` → invalidate sidebar badge query

This keeps badges current without polling.

---

## 3. Dashboard UI Features

### 3.1 Dashboard Page

The React dashboard page (`Dashboard.tsx`, ~400 lines) displays:

- **Agent status cards** — active, running, paused, error counts with color-coded badges
- **Task breakdown** — open, in-progress, blocked, done with visual bars
- **Cost summary** — monthly spend vs budget with utilization progress bar
- **Pending approvals** — count with link to approvals page
- **Budget alerts** — active incidents, paused agents/projects
- **Recent activity feed** — latest mutations from the activity log
- **Live runs panel** — currently executing heartbeat runs

### 3.2 Sidebar Navigation

The sidebar shows badge counts on navigation items:
- **Inbox** — composite attention count (red badge if > 0)
- **Approvals** — pending approval count
- **Agents** — error indicator if any agents failed

---

## 4. Monitoring Signals

### 4.1 What Operators Should Watch

| Signal | Dashboard Field | Action |
|--------|----------------|--------|
| Agents in error | `agents.error > 0` | Check agent run history for failures |
| Blocked tasks | `tasks.blocked > 0` | Read blocker comments, reassign or unblock |
| Budget approaching limit | `costs.monthUtilizationPercent >= 80` | Increase budget or reprioritize work |
| Budget hard-stopped | `budgets.pausedAgents > 0` | Resolve budget incident (raise or keep paused) |
| Pending approvals | `pendingApprovals > 0` | Review hire requests, strategy approvals |
| Failed runs | `failedRuns > 0` | Check run logs for errors |
| Join requests | `joinRequests > 0` | Approve or reject agent/user join requests |

### 4.2 What's Not Monitored (Yet)

- **Stale tasks** — tasks in_progress with no recent comments (mentioned in docs but not in dashboard service)
- **Agent uptime** — how long since last successful heartbeat
- **Cost velocity** — spend rate trending (available via `windowSpend` API but not in dashboard)
- **Goal progress** — percentage of child goals achieved
- **Project health** — combined agent/task/cost metrics per project

---

## 5. API Usage by Agents

CEO agents use the dashboard for situational awareness during heartbeats:

```
GET /api/companies/{companyId}/dashboard
```

The Paperclip skill instructs CEO/manager agents to check dashboard data at the start of each heartbeat to understand company health before making decisions.

---

## 6. Key Design Decisions

| Decision | Rationale |
|----------|-----------|
| Single-call dashboard endpoint | One request for full overview, not multiple queries from the UI |
| Sidebar badges as separate endpoint | Badges update more frequently (on every activity); lighter query than full dashboard |
| Failed runs = latest run per agent | Avoids false positives from old failures; only flags active problems |
| Composite inbox count | One number for "things that need attention" — reduces cognitive load |
| Permission-aware join request count | Non-admins don't see counts for things they can't act on |
| No caching | Fresh data on every request. Dashboard is not a high-frequency endpoint. |
| Budget overview integrated | Most urgent monitoring signal (runaway spend) is front-and-center |

---

## 7. Observations for Colmena

1. **The dashboard API is essential and simple** — one endpoint, ~100 lines of service code, 5 aggregation queries. Build this early.

2. **Sidebar badges drive engagement** — the red badge on "Inbox" is what pulls operators into the system. Implement badges alongside the dashboard.

3. **The composite inbox count** is a good UX pattern — one number for "things I need to look at" instead of 4 separate indicators.

4. **Failed runs detection** (latest run per agent) is a smart query — uses `DISTINCT ON` to efficiently get the most recent run per agent without loading all history.

5. **For Colmena MVP**, implement: dashboard summary (agent counts, task counts, monthly spend) + sidebar badges (approvals + failed runs). Skip budget overview integration initially.

6. **Missing monitoring features** (stale tasks, cost velocity, goal progress) are mentioned in Paperclip's docs but not yet implemented in the dashboard service. These are good future additions.

7. **Real-time badge updates** via WebSocket cache invalidation is important — without it, badges go stale and operators miss alerts.

8. **CEO agents using the dashboard** is a nice pattern — autonomous agents can self-assess company health before deciding what to work on.
