# Goals & Projects — hierarchical goals, project workspaces, repo linking

## Investigation Summary

Goals and projects provide the strategic alignment layer in Paperclip. Goals define the **why** (from company mission down to task-level objectives). Projects define the **what** (groupings of work toward a deliverable, linked to code repositories and workspaces). Together they ensure every task an agent works on traces back to the company's mission. The implementation spans 4 database tables, 2 services, and 2 route files.

---

## 1. Data Model

### 1.1 `goals` Table

| Column | Type | Default | Notes |
|--------|------|---------|-------|
| `id` | UUID | random | PK |
| `companyId` | UUID | — | FK → `companies.id` |
| `title` | TEXT | — | Required, min 1 char |
| `description` | TEXT | NULL | Optional markdown |
| `level` | TEXT | `"task"` | Enum: `company`, `team`, `agent`, `task` |
| `status` | TEXT | `"planned"` | Enum: `planned`, `active`, `achieved`, `cancelled` |
| `parentId` | UUID | NULL | Self-referencing FK → `goals.id` (hierarchical goals) |
| `ownerAgentId` | UUID | NULL | FK → `agents.id` (which agent owns this goal) |
| `createdAt` | TIMESTAMPTZ | now | — |
| `updatedAt` | TIMESTAMPTZ | now | — |

**Indexes:**
- `(companyId)` — list all goals for a company

**Hierarchy:** Goals form a tree via `parentId` (same self-referencing FK pattern as agents). The tree typically looks like:

```
Company Goal: "Build the #1 AI note-taking app at $1M MRR"
├── Team Goal: "Launch MVP by Q1"
│   ├── Agent Goal: "Build auth system"
│   │   └── Task Goal: "Implement JWT signing"
│   └── Agent Goal: "Build note editor"
└── Team Goal: "Acquire first 1000 users"
    └── Agent Goal: "Content marketing strategy"
```

### 1.2 `projects` Table

| Column | Type | Default | Notes |
|--------|------|---------|-------|
| `id` | UUID | random | PK |
| `companyId` | UUID | — | FK → `companies.id` |
| `goalId` | UUID | NULL | FK → `goals.id` (legacy single-goal link) |
| `name` | TEXT | — | Required, unique shortname per company |
| `description` | TEXT | NULL | Optional |
| `status` | TEXT | `"backlog"` | Enum: `backlog`, `planned`, `in_progress`, `completed`, `cancelled` |
| `leadAgentId` | UUID | NULL | FK → `agents.id` (project lead) |
| `targetDate` | DATE | NULL | Optional target completion date |
| `color` | TEXT | NULL | Auto-assigned from a palette of 10 colors |
| `pauseReason` | TEXT | NULL | `"manual"`, `"budget"`, `"system"` |
| `pausedAt` | TIMESTAMPTZ | NULL | When paused |
| `executionWorkspacePolicy` | JSONB | NULL | Configuration for how agents work in this project's codebase |
| `archivedAt` | TIMESTAMPTZ | NULL | Soft archive |
| `createdAt` | TIMESTAMPTZ | now | — |
| `updatedAt` | TIMESTAMPTZ | now | — |

**Indexes:**
- `(companyId)` — list all projects for a company

### 1.3 `project_goals` Table (Many-to-Many Join)

Projects can be linked to **multiple goals**:

| Column | Type | Notes |
|--------|------|-------|
| `projectId` | UUID | FK → `projects.id`, CASCADE delete |
| `goalId` | UUID | FK → `goals.id`, CASCADE delete |
| `companyId` | UUID | FK → `companies.id` |
| `createdAt` | TIMESTAMPTZ | — |
| `updatedAt` | TIMESTAMPTZ | — |

**PK:** Composite `(projectId, goalId)`

**Legacy compatibility:** The `projects.goalId` column is kept in sync with the first goal in `project_goals` for backward compatibility. New code should use `goalIds` (array).

### 1.4 `project_workspaces` Table

Projects can have **multiple workspaces** (code directories / repositories):

| Column | Type | Default | Notes |
|--------|------|---------|-------|
| `id` | UUID | random | PK |
| `companyId` | UUID | — | FK → `companies.id` |
| `projectId` | UUID | — | FK → `projects.id`, CASCADE delete |
| `name` | TEXT | — | Auto-derived from CWD or repo URL if not provided |
| `sourceType` | TEXT | `"local_path"` | Enum: `local_path`, `git_repo`, `remote_managed`, `non_git_path` |
| `cwd` | TEXT | NULL | Local filesystem path |
| `repoUrl` | TEXT | NULL | Git repository URL |
| `repoRef` | TEXT | NULL | Git branch/ref (e.g., `main`) |
| `defaultRef` | TEXT | NULL | Default branch (fallback to `repoRef`) |
| `visibility` | TEXT | `"default"` | `"default"` or `"advanced"` |
| `setupCommand` | TEXT | NULL | Command to run on workspace setup |
| `cleanupCommand` | TEXT | NULL | Command to run on workspace teardown |
| `remoteProvider` | TEXT | NULL | e.g., cloud workspace provider |
| `remoteWorkspaceRef` | TEXT | NULL | Reference ID in remote provider |
| `sharedWorkspaceKey` | TEXT | NULL | Key for workspace sharing across projects |
| `metadata` | JSONB | NULL | Arbitrary metadata |
| `isPrimary` | BOOL | false | Primary workspace determines agent CWD |
| `createdAt` | TIMESTAMPTZ | now | — |
| `updatedAt` | TIMESTAMPTZ | now | — |

**Indexes:**
- `(companyId, projectId)` — list workspaces for a project
- `(projectId, isPrimary)` — find primary workspace
- `(projectId, sourceType)` — filter by source type
- `(companyId, sharedWorkspaceKey)` — shared workspace lookup
- Unique `(projectId, remoteProvider, remoteWorkspaceRef)` — prevent duplicate remote refs

**Design:** At least one of `cwd` or `repoUrl` is required (or `remoteWorkspaceRef` for `remote_managed`). One workspace is always marked `isPrimary` — the first created workspace becomes primary automatically.

---

## 2. API Surface

### 2.1 Goals API

| Method | Path | Description |
|--------|------|-------------|
| `GET` | `/api/companies/:companyId/goals` | List all goals |
| `GET` | `/api/goals/:id` | Get goal detail |
| `POST` | `/api/companies/:companyId/goals` | Create goal |
| `PATCH` | `/api/goals/:id` | Update goal |
| `DELETE` | `/api/goals/:id` | Delete goal |

**Validation (Zod):**

Create:
```typescript
{
  title: string (min 1),
  description?: string | null,
  level?: "company" | "team" | "agent" | "task" (default "task"),
  status?: "planned" | "active" | "achieved" | "cancelled" (default "planned"),
  parentId?: UUID | null,
  ownerAgentId?: UUID | null
}
```
Update: All fields partial.

### 2.2 Projects API

| Method | Path | Description |
|--------|------|-------------|
| `GET` | `/api/companies/:companyId/projects` | List all projects (with goals + workspaces) |
| `GET` | `/api/projects/:id` | Get project detail |
| `POST` | `/api/companies/:companyId/projects` | Create project (optionally with initial workspace) |
| `PATCH` | `/api/projects/:id` | Update project |
| `DELETE` | `/api/projects/:id` | Delete project |

**Validation (Zod):**

Create:
```typescript
{
  name: string (min 1),
  description?: string | null,
  goalId?: UUID | null,         // Legacy single-goal
  goalIds?: UUID[],             // Multi-goal (preferred)
  status?: "backlog" | "planned" | "in_progress" | "completed" | "cancelled" (default "backlog"),
  leadAgentId?: UUID | null,
  targetDate?: string | null,
  color?: string | null,        // Auto-assigned if not provided
  executionWorkspacePolicy?: { ... } | null,
  archivedAt?: ISO datetime | null,
  workspace?: CreateProjectWorkspace  // Inline workspace creation
}
```

### 2.3 Project Workspaces API

| Method | Path | Description |
|--------|------|-------------|
| `GET` | `/api/projects/:id/workspaces` | List workspaces |
| `POST` | `/api/projects/:id/workspaces` | Create workspace |
| `PATCH` | `/api/projects/:id/workspaces/:workspaceId` | Update workspace |
| `DELETE` | `/api/projects/:id/workspaces/:workspaceId` | Delete workspace |

**Workspace creation validation:**
```typescript
{
  name?: string,           // Auto-derived from cwd/repoUrl if omitted
  sourceType?: "local_path" | "git_repo" | "remote_managed" | "non_git_path",
  cwd?: string | null,     // At least one of cwd or repoUrl required
  repoUrl?: URL | null,    // (unless sourceType is remote_managed)
  repoRef?: string | null,
  defaultRef?: string | null,
  visibility?: "default" | "advanced",
  setupCommand?: string | null,
  cleanupCommand?: string | null,
  remoteProvider?: string | null,
  remoteWorkspaceRef?: string | null,
  sharedWorkspaceKey?: string | null,
  metadata?: Record<string, unknown> | null,
  isPrimary?: boolean (default false)
}
```

---

## 3. Business Logic

### 3.1 Goal Hierarchy

Goals are hierarchical via `parentId`, but the service is surprisingly **simple** — no cycle detection, no depth limits, no tree traversal methods. The hierarchy is purely informational:

- Goals are listed flat (no tree building)
- The heartbeat protocol uses goals for **context** (agents see the "why" behind their tasks)
- Issues reference `goalId` to trace work back to a goal

**Default company goal resolution:**
```typescript
// Priority: active root company goal → any root company goal → any company goal
async function getDefaultCompanyGoal(db, companyId) {
  // 1. Active root company goal (level=company, status=active, parentId=null)
  // 2. Any root company goal (level=company, parentId=null)
  // 3. Any company-level goal
}
```

This is used when creating issues that need a default goal context.

### 3.2 Project Name Deduplication

Same pattern as agents — unique shortname per company:
1. `normalizeProjectUrlKey(name)` derives a URL-safe key
2. On collision, appends ` 2`, ` 3`, etc. (up to 10,000 attempts)
3. Fallback: appends timestamp

### 3.3 Project Shortname Resolution

Projects support shortname references (same as agents):
- UUID → direct lookup
- Non-UUID → normalize to URL key, search all projects in company
- If ambiguous (multiple matches) → `409 Conflict`
- Route param middleware auto-resolves transparently

### 3.4 Project Color Auto-Assignment

10-color palette: indigo, violet, pink, red, orange, yellow, green, teal, cyan, blue.

On create without explicit color:
1. Load all existing project colors in the company
2. Pick the first unused color from the palette
3. If all used, cycle through by index

### 3.5 Goal Links (Many-to-Many)

Projects link to goals via `project_goals` join table:

- **Batch loading**: `attachGoals()` loads all goal links for a set of projects in one query (JOIN with goals table)
- **Sync**: `syncGoalLinks()` deletes all existing links and re-inserts (full replace on every update)
- **Legacy compat**: `projects.goalId` column stays in sync with the first goal in the list

### 3.6 Workspace Management

**Primary workspace invariant:** Exactly one workspace per project is `isPrimary = true`.

- First workspace created → auto-primary
- Setting another workspace as primary → demotes all others to non-primary (in a transaction)
- Deleting the primary workspace → promotes the next oldest workspace
- If `isPrimary` is set to `false` → auto-selects another workspace as primary

**Workspace name auto-derivation:**
1. Explicit `name` provided → use it
2. `cwd` provided → extract last path segment (e.g., `/path/to/auth-repo` → `auth-repo`)
3. `repoUrl` provided → extract repo name from URL (e.g., `github.com/org/repo.git` → `repo`)
4. Fallback → `"Workspace"`

**Codebase derivation:**
Each project gets a computed `codebase` object:
```typescript
{
  workspaceId: string | null,        // Primary workspace ID
  repoUrl: string | null,           // From primary workspace
  repoRef: string | null,
  defaultRef: string | null,
  repoName: string | null,          // Extracted from repoUrl
  localFolder: string | null,       // From primary workspace cwd
  managedFolder: string,            // Auto-computed managed path
  effectiveLocalFolder: string,     // localFolder ?? managedFolder
  origin: "local_folder" | "managed_checkout"
}
```

**Managed workspace directory:**
If no explicit `cwd` is set, Paperclip auto-generates a managed directory path:
```
~/.paperclip/instances/default/workspaces/{companyId}/{projectId}/{repoName}/
```

### 3.7 Execution Workspace Policy

Projects can define how agents interact with the codebase:

```typescript
{
  enabled: boolean,
  defaultMode: "shared_workspace" | "isolated_workspace" | "operator_branch" | "adapter_default",
  allowIssueOverride: boolean,
  defaultProjectWorkspaceId?: UUID,
  workspaceStrategy: {
    type: "project_primary" | "git_worktree" | "adapter_managed" | "cloud_sandbox",
    baseRef?: string,
    branchTemplate?: string,        // e.g., "paperclip/{{issueId}}"
    worktreeParentDir?: string,
    provisionCommand?: string,
    teardownCommand?: string,
  },
  workspaceRuntime?: Record<string, unknown>,
  branchPolicy?: Record<string, unknown>,
  pullRequestPolicy?: Record<string, unknown>,
  runtimePolicy?: Record<string, unknown>,
  cleanupPolicy?: Record<string, unknown>,
}
```

This enables:
- **Shared workspace** — all agents work in the same directory
- **Isolated workspace** — each agent/task gets its own worktree or sandbox
- **Git worktree** — automatic `git worktree add` per issue
- **Cloud sandbox** — e2b or similar remote execution environments

### 3.8 Runtime Services

Workspaces can have associated **runtime services** (dev servers, databases, etc.) tracked via `workspace_runtime_services` table. These are loaded alongside workspace data and include:
- Service name, status, lifecycle, command, CWD, port, URL
- Health status tracking
- Stop policies
- Owner agent and run tracking

---

## 4. How Goals & Projects are Used

### 4.1 Issue Context Chain

Every issue can have a `goalId` and `projectId`. The issue detail response includes `ancestors` — the full parent chain of issues, each with their project and goal. This gives agents a complete picture:

```
Company Goal → Team Goal → Project → Parent Issue → Current Issue
```

### 4.2 Heartbeat Context

During heartbeats, agents can read:
- Their assignments (`GET /api/agents/me/inbox-lite`)
- Issue detail with ancestors and goals (`GET /api/issues/{id}`)
- Project workspaces to determine their working directory

### 4.3 Dashboard Metrics

The dashboard doesn't currently compute goal completion percentages or project progress metrics directly — it focuses on task counts, agent status, and costs.

### 4.4 Budget Scoping

Budget policies can be scoped to projects:
```typescript
{ scopeType: "project", scopeId: projectId, ... }
```
When a project budget is exceeded, the project is paused (same mechanism as company/agent budgets).

---

## 5. Technical Patterns

### 5.1 Batch Loading (N+1 Prevention)

Projects use a batch-loading pattern for related data:
1. Load all projects in one query
2. `attachGoals()` — load all `project_goals` + goals in one JOIN query, group by projectId
3. `attachWorkspaces()` — load all `project_workspaces` in one query, group by projectId
4. `listWorkspaceRuntimeServicesForProjectWorkspaces()` — batch-load runtime services

This avoids N+1 queries when listing projects.

### 5.2 Legacy Column Migration

The `projects.goalId` column is maintained for backward compatibility:
- On create: set to the first goal ID in `goalIds`
- On update: keep in sync when `goalIds` changes
- `resolveGoalIds()` handles both `goalId` (legacy) and `goalIds` (new) inputs

### 5.3 Transactional Primary Workspace Management

All primary workspace changes happen in transactions:
1. Demote all existing workspaces to non-primary
2. Promote the target workspace
3. On delete, auto-promote the next oldest

This ensures the "exactly one primary" invariant is never violated.

---

## 6. Key Design Decisions

| Decision | Rationale |
|----------|-----------|
| Goals are a separate entity from projects | Goals represent the "why", projects represent the "what". A project can serve multiple goals |
| Many-to-many goal-project linking | Projects often serve multiple strategic objectives simultaneously |
| Self-referencing goals hierarchy | Same pattern as agents — simple, adequate for typical goal trees |
| No goal cycle detection | Goals are informational, not structural. Cycles are unlikely and harmless |
| Multiple workspaces per project | Monorepos, multi-repo projects, and managed checkouts are all common patterns |
| Primary workspace invariant | Agents need exactly one CWD for a project — the primary workspace provides it |
| Auto-derived workspace names | Reduces friction — users don't need to name every workspace |
| Execution workspace policy as JSONB | Complex and evolving config — better as a schema-validated JSONB than normalized tables |
| Managed workspace directory | Projects without explicit CWD get an auto-generated directory, enabling "repo-only" setups |
| Project colors auto-assigned | Visual polish — each project gets a distinct color in the UI without user effort |
| Workspace source types | Support for local paths, git repos, remote managed workspaces, and non-git directories |

---

## 7. Observations for Colmena

1. **Goals service is surprisingly thin** — just CRUD, no tree building, no progress tracking, no completion calculation. The hierarchy is informational, used only for context in issue detail responses. For Colmena, consider whether goal progress tracking (% of child goals achieved) would add value.

2. **Projects are much richer** — workspaces, codebase derivation, execution policies, runtime services. This is where the real complexity lives, because projects are the bridge between abstract goals and concrete code.

3. **The execution workspace policy is forward-looking** — supports git worktrees, cloud sandboxes, and branch templates. This is critical for teams where multiple agents work on the same codebase without conflicts.

4. **Legacy `goalId` ↔ `goalIds` migration** is technical debt. For Colmena, start with multi-goal from day one and skip the legacy column.

5. **Runtime services per workspace** is a sophisticated feature — tracking dev servers, databases, etc. per project workspace. Consider if this is needed for Colmena MVP or if it's a later addition.

6. **The batch-loading pattern** (attachGoals, attachWorkspaces) is a good approach for preventing N+1 queries. Worth adopting.

7. **Project shortname resolution** (same as agents) provides clean URLs and natural references. Worth keeping.

8. **No project-level org structure** — projects don't have teams or member lists. Agents are assigned to tasks within projects, not to projects themselves. The `leadAgentId` is informational only.

9. **Workspace `setupCommand` / `cleanupCommand`** enables automation (e.g., `npm install` on setup, cleanup temp files). Useful for production but optional for MVP.
