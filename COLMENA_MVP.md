# Colmena MVP — AI Software Teams

> **Colmena** (Spanish for "beehive") — a standalone tool to create and orchestrate AI software teams.

---

## 1. Vision

Colmena orchestrates a team of AI agents to work on software projects. You define goals, break them into epics and tasks, assign agents, and Colmena coordinates their execution via heartbeats. Tasks.md provides the task board UI. Colmena provides the agent orchestration engine.

**What Colmena is:**
- A control plane for AI agent teams
- A heartbeat-driven execution engine
- A task-to-agent coordination layer

**What Colmena is NOT (for MVP):**
- Not multi-company (single workspace)
- Not a cost/token tracker
- Not an org hierarchy manager
- Not a plugin platform

---

## 2. Architecture Overview

```
┌──────────────────────────────────────────────────┐
│  Tasks.md (UI)                                   │
│  Kanban board: Draft→Backlog→Todo→...→Done       │
│  Reads/writes markdown files on shared volume     │
├──────────────────────────────────────────────────┤
│  Colmena Server (Express + Node.js)              │
│  ┌─────────┐ ┌───────────┐ ┌──────────────────┐ │
│  │ REST API│ │ Scheduler │ │ Heartbeat Engine │ │
│  └─────────┘ └───────────┘ └──────────────────┘ │
│  ┌─────────┐ ┌───────────┐ ┌──────────────────┐ │
│  │ Secrets │ │ Task Sync │ │ Approval Gate    │ │
│  └─────────┘ └───────────┘ └──────────────────┘ │
├──────────────────────────────────────────────────┤
│  Colmena Web UI (React)                          │
│  Agent management + Run viewer                   │
├──────────────────────────────────────────────────┤
│  PostgreSQL (embedded PGlite or external)        │
├──────────────────────────────────────────────────┤
│  Filesystem                                      │
│  └── /tasks/  ← shared with Tasks.md            │
│      ├── Draft/                                  │
│      ├── Backlog/                                │
│      ├── Todo/                                   │
│      ├── In Progress/                            │
│      ├── In Review/                              │
│      └── Done/                                   │
└──────────────────────────────────────────────────┘
```

**Key integration:** Colmena and Tasks.md share the same `/tasks/` directory. Colmena writes task markdown files; Tasks.md renders them as a kanban board. Moving a card in Tasks.md moves the file between lane directories. Colmena watches the filesystem for changes.

---

## 3. Core Concepts

### 3.1 Goals

The "why" — high-level objectives for the team.

```
Goal: "Build a SaaS MVP with auth, billing, and landing page"
```

- Stored in database only (not on filesystem)
- Status: `planned` → `active` → `achieved` / `cancelled`
- Every epic and task traces back to a goal

### 3.2 Epics

The "what" — major deliverables that break down a goal.

```
Epic: "Authentication System"  (under Goal: "Build SaaS MVP")
Epic: "Billing Integration"    (under Goal: "Build SaaS MVP")
```

- Stored in database only
- Status: `planned` → `active` → `completed` / `cancelled`
- Contains tasks

### 3.3 Tasks

The "how" — concrete units of work for agents.

```
Task: "Implement JWT token signing"  (under Epic: "Authentication System")
```

- **Stored as markdown files** on the filesystem (in Tasks.md lane directories)
- **Also tracked in the database** (for metadata: assignee, approval status, heartbeat runs, etc.)
- Agents can create **sub-tasks** (child tasks under an existing task)
- Lane/directory = status: `Draft`, `Backlog`, `Todo`, `In Progress`, `In Review`, `Done`

### 3.4 Agents

AI workers that execute tasks via heartbeats.

- Flat list (no hierarchy) — all agents report to you (the operator)
- Each agent has: name, capabilities description, adapter config (Claude Code), working directory
- Status: `idle`, `running`, `paused`, `error`, `terminated`

### 3.5 Heartbeats

Short execution bursts where agents wake up, check their tasks, do work, and go back to sleep.

Triggers:
- **Timer** — periodic interval (e.g., every 5 minutes)
- **Assignment** — new task assigned
- **Manual** — operator clicks "Invoke"

---

## 4. Task ↔ Filesystem Integration

### 4.1 Directory Structure

```
/tasks/                          ← Tasks.md root (shared volume)
├── Draft/
│   └── TASK-7-design-api.md
├── Backlog/
│   └── TASK-3-setup-ci.md
├── Todo/
│   └── TASK-5-jwt-signing.md
├── In Progress/
│   └── TASK-2-user-model.md
├── In Review/
│   └── TASK-1-login-endpoint.md
└── Done/
    └── TASK-4-db-schema.md
```

### 4.2 Task File Format

Each task is a markdown file with YAML frontmatter:

```markdown
---
id: "task-uuid"
identifier: "TASK-5"
goal: "Build SaaS MVP"
epic: "Authentication System"
assignee: "backend-engineer"
priority: high
parent: "TASK-2"
approval: approved
created: "2026-03-23T10:00:00Z"
updated: "2026-03-23T15:30:00Z"
---

# Implement JWT token signing

## Description

Implement RS256 JWT signing for the auth service...

## Agent Notes

_This section is updated by the agent during heartbeats._

### 2026-03-23 15:30 — backend-engineer

Added RS256 support. Tests passing. Still need refresh token logic.
```

### 4.3 Filesystem ↔ Database Sync

**Colmena → Filesystem:**
- Creating a task → writes `.md` file to the appropriate lane directory
- Updating task status → moves file between directories
- Agent adding notes → appends to the "Agent Notes" section

**Filesystem → Colmena (via watcher):**
- Moving a card in Tasks.md (file moved between directories) → Colmena detects and updates DB status
- Editing task content in Tasks.md → Colmena detects and updates DB

**Sync strategy:**
- Use `chokidar` (or `fs.watch`) to watch `/tasks/` recursively
- On file change: parse frontmatter, update DB record
- On file move: detect lane change, update status in DB
- Debounce: 500ms to avoid rapid-fire updates
- **Colmena is the source of truth** for metadata (assignee, approval, IDs). Tasks.md is a view.

### 4.4 Lane ↔ Status Mapping

| Lane Directory | Task Status | Who can move here |
|---------------|-------------|-------------------|
| `Draft/` | `draft` | Operator only (new ideas, not ready) |
| `Backlog/` | `backlog` | Operator only (refined, not prioritized) |
| `Todo/` | `todo` | Operator (assign + approve → agent picks up) |
| `In Progress/` | `in_progress` | Agent (via checkout) |
| `In Review/` | `in_review` | Agent (work complete, needs review) |
| `Done/` | `done` | Operator (reviewed and accepted) |

### 4.5 Task Identifier System

Auto-incrementing: `TASK-1`, `TASK-2`, etc. Stored as filename prefix: `TASK-5-jwt-signing.md`.

---

## 5. Approval Gate

Before an agent can start working on a task, it must be **approved** by the operator.

### 5.1 Flow

```
Operator creates task → status: todo, approval: pending
Operator reviews → approval: approved  (or rejected)
Agent heartbeat → sees approved task → checkout → work
```

### 5.2 Approval States

| State | Meaning |
|-------|---------|
| `pending` | Task created/assigned, waiting for operator approval |
| `approved` | Operator approved, agent can start working |
| `rejected` | Operator rejected, agent should skip |

### 5.3 How It Works

- Task approval status is stored in the frontmatter (`approval: pending/approved/rejected`)
- In the Tasks.md UI, pending tasks show `⏳` in the title, approved show `✅`
- Agents during heartbeat: only checkout tasks where `approval: approved`
- Operator approves via Colmena web UI or by editing frontmatter directly in Tasks.md

### 5.4 Sub-Task Approvals

When an agent creates a sub-task:
1. Agent writes the new task file to `Todo/` with `approval: pending`
2. Colmena detects the new file
3. Operator reviews and approves (or rejects) via UI
4. If approved, the creating agent (or another) picks it up on next heartbeat

---

## 6. Agent System

### 6.1 Agent Configuration

```typescript
{
  id: UUID,
  name: "backend-engineer",        // Unique name
  capabilities: "Node.js, PostgreSQL, API design",
  status: "idle",                   // idle, running, paused, error, terminated

  // Claude Code adapter config
  adapterConfig: {
    model: "claude-sonnet-4-6",
    cwd: "/workspace/my-project",   // Working directory for code
    maxTurnsPerRun: 80,
    instructionsPath: "/workspace/my-project/.colmena/agents/backend-engineer/AGENTS.md",
    env: {
      GITHUB_TOKEN: { type: "secret_ref", secretId: "...", scope: "agent" },
      NODE_ENV: { type: "plain", value: "development" },
    },
  },

  // Heartbeat policy
  heartbeatPolicy: {
    enabled: true,
    intervalSec: 300,               // Every 5 minutes
    wakeOnAssignment: true,
  },
}
```

### 6.2 Agent Instructions (AGENTS.md)

Each agent gets an instructions file that teaches it how to behave:

```markdown
# Backend Engineer — Colmena Agent

You are a backend engineer working in a Colmena-managed team.

## Identity
- Name: backend-engineer
- Capabilities: Node.js, PostgreSQL, API design
- Working directory: /workspace/my-project

## Heartbeat Protocol

Every heartbeat, follow these steps:

1. **Check assignments**: GET /api/tasks?assignee=me&status=todo&approval=approved
2. **Pick work**: Work on in_progress first, then todo
3. **Checkout**: POST /api/tasks/{id}/checkout
4. **Do the work**: Use your tools
5. **Update status**: PATCH /api/tasks/{id} with notes
6. **Delegate if needed**: POST /api/tasks to create sub-tasks

## Rules
- Only work on APPROVED tasks
- Always update the Agent Notes section
- If blocked, set status to in_review with explanation
- Never skip the checkout step
```

### 6.3 Agent Lifecycle

```
create → idle ↔ running → (paused) → terminated
                  ↕
                error
```

- **idle**: Ready, waiting for heartbeat trigger
- **running**: Heartbeat in progress
- **paused**: Manually paused by operator
- **error**: Last heartbeat failed
- **terminated**: Permanently deactivated

---

## 7. Heartbeat System

### 7.1 Execution Flow

```
Trigger (timer/assignment/manual)
  → Budget/pause check
  → Resolve working directory
  → Restore session (if exists)
  → Build environment (env vars + secrets)
  → Create agent JWT
  → Spawn Claude Code CLI
  → Stream stdout/stderr → run events + log store
  → Capture result (exit code, session, usage)
  → Persist session state
  → Update agent status
  → Record run
```

### 7.2 Environment Variables Injected

```
COLMENA_AGENT_ID=<agent-uuid>
COLMENA_AGENT_NAME=<agent-name>
COLMENA_API_URL=http://localhost:3100/api
COLMENA_API_KEY=<short-lived-jwt>
COLMENA_RUN_ID=<run-uuid>
COLMENA_TASK_ID=<task-uuid>           # If triggered by assignment
COLMENA_WAKE_REASON=<reason>          # timer, assignment, manual
```

### 7.3 Run Tracking

Each heartbeat execution is recorded:

```typescript
{
  id: UUID,
  agentId: UUID,
  status: "queued" | "running" | "succeeded" | "failed" | "cancelled" | "timed_out",
  invocationSource: "timer" | "assignment" | "manual",
  startedAt, finishedAt,
  exitCode, error,
  sessionIdBefore, sessionIdAfter,   // Session persistence
  stdoutExcerpt, stderrExcerpt,      // Truncated output previews
  logStore, logRef,                   // Full log file reference
}
```

### 7.4 Run Events

Granular event log per run (every stdout/stderr line):

```typescript
{
  runId: UUID,
  seq: number,         // Ordering
  eventType: "stdout" | "stderr" | "system",
  message: string,
  createdAt: timestamp,
}
```

### 7.5 Session Persistence

Agent conversation context persists across heartbeats:
- After each run: adapter returns session ID/params → stored in `agent_runtime_state`
- Next heartbeat: session restored → agent continues from where it left off
- Session reset: operator can clear session via UI

### 7.6 Task Checkout (Atomic)

Before working on a task, the agent must checkout:

```
POST /api/tasks/{id}/checkout
{ "agentId": "...", "expectedStatuses": ["todo"] }
```

- Atomic: conditional UPDATE ensures only one agent claims a task
- 409 Conflict if another agent already owns it
- Idempotent: re-checkout by same agent succeeds
- Only works on tasks with `approval: approved`

---

## 8. Secret Management

### 8.1 Scopes

| Scope | Visibility | Example |
|-------|-----------|---------|
| **Global** | All agents | `GITHUB_TOKEN` (org-wide), `NPM_TOKEN` |
| **Agent** | Specific agent only | Agent-specific API key, deploy token |

### 8.2 Storage

- Encrypted at rest using AES-256-GCM
- Local master key auto-generated at `~/.colmena/secrets/master.key`
- Versioned: rotate without breaking agents (`version: "latest"`)
- SHA-256 hash of plaintext stored (for change detection)

### 8.3 Secret References in Agent Config

```typescript
// Agent adapter config
{
  env: {
    // Global secret (shared across all agents)
    GITHUB_TOKEN: { type: "secret_ref", secretId: "...", scope: "global" },

    // Agent-scoped secret
    DEPLOY_KEY: { type: "secret_ref", secretId: "...", scope: "agent" },

    // Plain value (not sensitive)
    NODE_ENV: { type: "plain", value: "development" },
  }
}
```

### 8.4 Runtime Resolution

Before each heartbeat:
1. Load agent's `env` config
2. For each `secret_ref`: decrypt via provider, resolve `"latest"` to current version
3. For each `plain`: pass through
4. Merge global secrets + agent secrets (agent overrides global on conflict)
5. Inject as environment variables into Claude Code process

### 8.5 API

```
GET    /api/secrets                        # List all (metadata only, no values)
POST   /api/secrets                        # Create { name, value, scope, agentId? }
PATCH  /api/secrets/:id/rotate             # Rotate value (new version)
DELETE /api/secrets/:id                     # Delete
```

### 8.6 Sensitive Key Detection

Keys matching `/(api[-_]?key|token|secret|password|credential|private[-_]?key)/i` are flagged. Plain values for these keys show a warning in the UI suggesting secret_ref instead.

---

## 9. Database Schema

### 9.1 Core Tables

```
goals
├── id (UUID, PK)
├── title (TEXT)
├── description (TEXT)
├── status (TEXT: planned, active, achieved, cancelled)
├── created_at, updated_at

epics
├── id (UUID, PK)
├── goal_id (UUID, FK → goals)
├── title (TEXT)
├── description (TEXT)
├── status (TEXT: planned, active, completed, cancelled)
├── created_at, updated_at

tasks
├── id (UUID, PK)
├── epic_id (UUID, FK → epics)
├── goal_id (UUID, FK → goals)
├── parent_id (UUID, FK → tasks, self-ref for sub-tasks)
├── identifier (TEXT, unique: "TASK-42")
├── task_number (INT, auto-increment)
├── title (TEXT)
├── description (TEXT)
├── status (TEXT: draft, backlog, todo, in_progress, in_review, done, cancelled)
├── priority (TEXT: critical, high, medium, low)
├── approval (TEXT: pending, approved, rejected)
├── assignee_agent_id (UUID, FK → agents)
├── checkout_run_id (UUID, FK → heartbeat_runs)
├── file_path (TEXT: relative path in /tasks/)
├── created_by_agent_id (UUID, FK → agents)
├── started_at, completed_at
├── created_at, updated_at

agents
├── id (UUID, PK)
├── name (TEXT, unique)
├── capabilities (TEXT)
├── status (TEXT: idle, running, paused, error, terminated)
├── adapter_type (TEXT: "claude_local")
├── adapter_config (JSONB)
├── heartbeat_policy (JSONB)
├── last_heartbeat_at (TIMESTAMPTZ)
├── pause_reason (TEXT)
├── created_at, updated_at

agent_runtime_state
├── agent_id (UUID, PK, FK → agents)
├── session_id (TEXT)
├── state_json (JSONB)
├── last_run_id (UUID)
├── last_run_status (TEXT)
├── last_error (TEXT)
├── created_at, updated_at

heartbeat_runs
├── id (UUID, PK)
├── agent_id (UUID, FK → agents)
├── status (TEXT: queued, running, succeeded, failed, cancelled, timed_out)
├── invocation_source (TEXT: timer, assignment, manual)
├── started_at, finished_at
├── exit_code (INT)
├── error (TEXT)
├── session_id_before, session_id_after (TEXT)
├── stdout_excerpt, stderr_excerpt (TEXT)
├── log_store, log_ref (TEXT)
├── context_snapshot (JSONB)
├── created_at, updated_at

heartbeat_run_events
├── id (BIGSERIAL, PK)
├── run_id (UUID, FK → heartbeat_runs)
├── agent_id (UUID, FK → agents)
├── seq (INT)
├── event_type (TEXT: stdout, stderr, system)
├── message (TEXT)
├── created_at

secrets
├── id (UUID, PK)
├── name (TEXT, unique)
├── scope (TEXT: global, agent)
├── agent_id (UUID, FK → agents, NULL for global)
├── provider (TEXT: "local_encrypted")
├── latest_version (INT)
├── description (TEXT)
├── created_at, updated_at

secret_versions
├── id (UUID, PK)
├── secret_id (UUID, FK → secrets, CASCADE)
├── version (INT)
├── material (JSONB: encrypted)
├── value_sha256 (TEXT)
├── created_at

activity_log
├── id (UUID, PK)
├── actor_type (TEXT: agent, user, system)
├── actor_id (TEXT)
├── action (TEXT: task.created, agent.paused, heartbeat.invoked, etc.)
├── entity_type (TEXT: task, agent, heartbeat_run, etc.)
├── entity_id (TEXT)
├── agent_id (UUID, FK → agents)
├── run_id (UUID, FK → heartbeat_runs)
├── details (JSONB)
├── created_at
```

### 9.2 No Tables Needed For

- Companies (single workspace)
- Company memberships / permissions
- Budgets / cost events / finance events
- Plugins
- Routines / triggers
- Invites / join requests
- Execution workspaces
- Project workspaces (CWD is in agent config)

---

## 10. API Surface

### 10.1 Goals

```
GET    /api/goals                          # List all goals
POST   /api/goals                          # Create goal
PATCH  /api/goals/:id                      # Update goal
DELETE /api/goals/:id                      # Delete goal
```

### 10.2 Epics

```
GET    /api/epics                          # List (filter by goalId)
POST   /api/epics                          # Create epic
PATCH  /api/epics/:id                      # Update epic
DELETE /api/epics/:id                      # Delete epic
```

### 10.3 Tasks

```
GET    /api/tasks                          # List (filter: status, assignee, epicId, approval)
GET    /api/tasks/:id                      # Get task detail
POST   /api/tasks                          # Create task (writes .md file)
PATCH  /api/tasks/:id                      # Update task (moves file if status changes)
DELETE /api/tasks/:id                      # Delete task (removes file)
POST   /api/tasks/:id/checkout             # Atomic checkout
POST   /api/tasks/:id/release              # Release checkout
POST   /api/tasks/:id/approve              # Approve task
POST   /api/tasks/:id/reject               # Reject task
```

### 10.4 Agents

```
GET    /api/agents                         # List all agents
GET    /api/agents/:id                     # Get agent detail + runtime state
GET    /api/agents/me                      # Agent self-identify (via JWT)
POST   /api/agents                         # Create agent
PATCH  /api/agents/:id                     # Update agent config
POST   /api/agents/:id/pause               # Pause agent
POST   /api/agents/:id/resume              # Resume agent
POST   /api/agents/:id/terminate           # Terminate agent
POST   /api/agents/:id/heartbeat/invoke    # Manual heartbeat trigger
POST   /api/agents/:id/session/reset       # Reset session
```

### 10.5 Heartbeat Runs

```
GET    /api/runs                           # List runs (filter by agentId)
GET    /api/runs/:id                       # Get run detail
GET    /api/runs/:id/events                # Get run events (paginated by seq)
GET    /api/runs/:id/log                   # Read run log (offset + limit)
POST   /api/runs/:id/cancel                # Cancel running heartbeat
```

### 10.6 Secrets

```
GET    /api/secrets                         # List (metadata only)
POST   /api/secrets                         # Create
PATCH  /api/secrets/:id/rotate              # Rotate value
DELETE /api/secrets/:id                     # Delete
```

### 10.7 Activity

```
GET    /api/activity                        # List activity log
```

### 10.8 Dashboard

```
GET    /api/dashboard                       # Health summary
```

---

## 11. Web UI (Colmena)

Minimal React UI focused on what Tasks.md doesn't cover:

### 11.1 Pages

| Page | Purpose |
|------|---------|
| **Dashboard** | Agent status summary, recent runs, pending approvals count |
| **Agents** | List agents with status badges, create/edit agents |
| **Agent Detail** | Config editor, heartbeat policy, run history, streaming log viewer |
| **Run Viewer** | Real-time streaming stdout/stderr for a specific run |
| **Secrets** | Manage global and agent-scoped secrets |
| **Goals & Epics** | Manage goals/epics hierarchy (tasks are in Tasks.md) |

### 11.2 Tech Stack

- React 19 + Vite 6
- Tailwind CSS 4 + shadcn/ui
- TanStack Query
- WebSocket for real-time run streaming

### 11.3 What's NOT in Colmena UI

- Task kanban board → **Tasks.md**
- Task creation/editing → **Tasks.md** (or API)
- Task approval → Colmena UI (button on dashboard) **or** edit frontmatter in Tasks.md

---

## 12. Real-Time Events

### 12.1 WebSocket

```
GET /api/events/ws   (WebSocket upgrade)
```

Events:
- `heartbeat.run.status` — run started/completed/failed
- `heartbeat.run.event` — each stdout/stderr line (for streaming viewer)
- `agent.status` — agent status changed
- `task.updated` — task status/approval changed
- `activity.logged` — any mutation

### 12.2 In-Process

Same pattern as Paperclip: Node.js `EventEmitter`, no external broker needed.

---

## 13. CLI

### 13.1 Commands

```bash
colmena init                     # First-run setup wizard
colmena start                    # Start the server
colmena doctor                   # Diagnostic checks
colmena agent list               # List agents
colmena agent create <name>      # Create agent interactively
colmena agent invoke <name>      # Trigger heartbeat manually
colmena task list                # List tasks
colmena task approve <id>        # Approve a task
colmena secret set <name>        # Set a secret interactively
```

### 13.2 Tech

- Commander.js + @clack/prompts + picocolors

---

## 14. Docker Deployment

### 14.1 Docker Compose

```yaml
version: "3"
services:
  colmena:
    build: .
    ports:
      - "3100:3100"
    environment:
      - HOST=0.0.0.0
      - COLMENA_HOME=/colmena
    volumes:
      - ./data:/colmena
      - ./tasks:/tasks         # Shared with Tasks.md
      - ./workspace:/workspace # Agent working directory

  tasks-md:
    image: baldissaramatheus/tasks.md
    ports:
      - "8080:8080"
    environment:
      - PUID=1000
      - PGID=1000
      - TITLE=Colmena Board
    volumes:
      - ./tasks:/tasks         # Shared with Colmena
```

Both containers share the `/tasks` volume. Colmena writes task files; Tasks.md renders them.

---

## 15. Colmena Skill (Agent Protocol)

A built-in skill that teaches agents how to work within Colmena:

```markdown
---
name: colmena
description: >
  Interact with the Colmena API to manage tasks, update status,
  and coordinate with the team.
---

# Colmena Agent Skill

You work in **heartbeats** — short execution windows.

## Authentication

Env vars: COLMENA_AGENT_ID, COLMENA_API_URL, COLMENA_API_KEY, COLMENA_RUN_ID

## Heartbeat Procedure

1. GET /api/agents/me — check identity
2. GET /api/tasks?assignee=me&status=todo,in_progress&approval=approved
3. Work on in_progress first, then todo
4. POST /api/tasks/{id}/checkout (atomic claim)
5. Do the work
6. PATCH /api/tasks/{id} { status, notes } — update progress
7. If done: set status to "in_review"
8. If need sub-task: POST /api/tasks (approval: pending, parentId: current)

## Rules

- ONLY work on tasks with approval: approved
- Always checkout before working
- Never retry a 409 (someone else claimed it)
- Update the task markdown file with progress notes
- If blocked, set status to in_review with explanation
```

---

## 16. Data Directory Layout

```
~/.colmena/                        # COLMENA_HOME
├── config.json                    # Server configuration
├── secrets/
│   └── master.key                 # Encryption master key (0600)
├── db/                            # Embedded PostgreSQL data
├── run-logs/                      # Heartbeat run log files
└── agents/
    └── {agentId}/
        └── instructions/          # Agent AGENTS.md files
```

---

## 17. Implementation Order

### Phase 1: Foundation (Week 1-2)

1. **Project scaffolding** — monorepo with `server/`, `packages/db/`, `packages/shared/`
2. **Database** — embedded Postgres, Drizzle schema for all tables, auto-migrations
3. **Config system** — config file + env var loading
4. **Secrets** — local_encrypted provider, CRUD API, env var resolution
5. **Activity log** — append-only log + WebSocket event publishing

### Phase 2: Task System (Week 2-3)

6. **Goals & Epics** — CRUD API
7. **Tasks** — CRUD API with filesystem sync (write .md files to lane directories)
8. **Filesystem watcher** — detect Tasks.md edits and moves, sync to DB
9. **Task identifiers** — auto-incrementing TASK-N
10. **Approval gate** — approval status in frontmatter, API endpoints
11. **Atomic checkout** — conditional UPDATE, 409 on conflict

### Phase 3: Agent Orchestration (Week 3-4)

12. **Agent CRUD** — create, configure, pause/resume/terminate
13. **Claude Code adapter** — execute function, stdout/stderr capture, session persistence
14. **Heartbeat engine** — trigger → enqueue → execute → record pipeline
15. **Run tracking** — heartbeat_runs + heartbeat_run_events tables
16. **Session persistence** — save/restore agent sessions across heartbeats
17. **Colmena skill** — built-in agent protocol instructions
18. **Scheduler** — timer-based heartbeat triggering

### Phase 4: UI + Polish (Week 4-5)

19. **Agent management UI** — list, create, configure, invoke
20. **Run viewer** — streaming log viewer via WebSocket
21. **Dashboard** — agent status, pending approvals, recent runs
22. **Secrets UI** — manage global/agent-scoped secrets
23. **Goals/Epics UI** — simple CRUD pages
24. **CLI** — `colmena init`, `start`, `doctor`, `agent invoke`, `task approve`
25. **Docker Compose** — Colmena + Tasks.md + shared volume

---

## 18. What's Explicitly Out of Scope (MVP)

| Feature | Why |
|---------|-----|
| Multi-company | Single workspace |
| Cost/token tracking | Not needed for MVP |
| Budget enforcement | No cost tracking |
| Org hierarchy (reportsTo) | Flat agent list |
| Plugin system | Way too complex for MVP |
| Company portability (import/export) | Single workspace |
| Routines (recurring tasks) | Manual task creation first |
| Execution workspaces / git worktrees | Agents share the project CWD |
| Multiple adapter types | Claude Code only |
| Invite/join system | Single operator |
| Issue documents (keyed, revisioned) | Task description in markdown file is enough |
| Issue attachments | Not needed |
| Issue labels | Not needed |
| Read tracking / inbox | Single operator |
| Finance events / billing types | No cost tracking |
| Adapter model discovery | Single adapter, configure model manually |
| Skills marketplace / import | Built-in skill only |
| Org chart SVG/PNG rendering | No hierarchy |
| Config revisions / rollback | Nice to have, not MVP |

---

## 19. Success Criteria

The MVP is done when:

1. ✅ Operator can define Goals → Epics → Tasks
2. ✅ Tasks appear as markdown files in Tasks.md kanban board
3. ✅ Moving cards in Tasks.md updates Colmena status
4. ✅ Operator can create and configure Claude Code agents
5. ✅ Operator can approve tasks before agents work on them
6. ✅ Agents wake up on heartbeats, check assignments, checkout tasks, do work, update status
7. ✅ Agents can create sub-tasks (with pending approval)
8. ✅ Agent sessions persist across heartbeats
9. ✅ Secrets (global + agent-scoped) are encrypted and injected as env vars
10. ✅ Run viewer shows streaming agent output in real-time
11. ✅ Docker Compose runs Colmena + Tasks.md with shared task volume
