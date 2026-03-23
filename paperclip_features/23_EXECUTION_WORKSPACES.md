# Execution Workspaces — workspace management, operation log, runtime services, policy enforcement

## Investigation Summary

Execution workspaces are an advanced feature that manages **isolated working environments** for agent heartbeat runs. While project workspaces (covered in 04) define where code lives, execution workspaces define **how agents interact with that code** — shared vs isolated, git worktrees vs cloud sandboxes, with automatic provisioning, cleanup, and runtime service management (dev servers, databases). This spans 3 database tables and ~2,100 lines of service code, gated behind an experimental feature flag.

---

## 1. Data Model

### 1.1 `execution_workspaces` Table

Represents an isolated working environment for a specific task/agent.

| Column | Type | Notes |
|--------|------|-------|
| `id` | UUID | PK |
| `companyId` | UUID | FK → companies |
| `projectId` | UUID | FK → projects, CASCADE |
| `projectWorkspaceId` | UUID | FK → project_workspaces, SET NULL |
| `sourceIssueId` | UUID | FK → issues, SET NULL — which issue this was created for |
| `mode` | TEXT | `shared_workspace`, `isolated_workspace`, `operator_branch`, `adapter_default` |
| `strategyType` | TEXT | `project_primary`, `git_worktree`, `adapter_managed`, `cloud_sandbox` |
| `name` | TEXT | Human-readable name |
| `status` | TEXT | `active`, `idle`, `in_review`, `closed`, `cleanup_pending` |
| `cwd` | TEXT | Filesystem path to the workspace |
| `repoUrl` | TEXT | Git repository URL |
| `baseRef` | TEXT | Git base branch (e.g., `main`) |
| `branchName` | TEXT | Git branch for this workspace |
| `providerType` | TEXT | `local_fs`, `cloud_sandbox`, etc. |
| `providerRef` | TEXT | External provider reference |
| `derivedFromExecutionWorkspaceId` | UUID | Self-ref — workspace lineage |
| `lastUsedAt` | TIMESTAMPTZ | — |
| `openedAt` | TIMESTAMPTZ | When created |
| `closedAt` | TIMESTAMPTZ | When closed |
| `cleanupEligibleAt` | TIMESTAMPTZ | When cleanup can start |
| `cleanupReason` | TEXT | — |
| `metadata` | JSONB | — |

### 1.2 `workspace_operations` Table

Tracks commands run during workspace lifecycle (setup, provision, cleanup).

| Column | Type | Notes |
|--------|------|-------|
| `id` | UUID | PK |
| `companyId` | UUID | FK → companies |
| `executionWorkspaceId` | UUID | FK → execution_workspaces, SET NULL |
| `heartbeatRunId` | UUID | FK → heartbeat_runs, SET NULL |
| `phase` | TEXT | `provision`, `setup`, `cleanup`, `teardown` |
| `command` | TEXT | Shell command executed |
| `cwd` | TEXT | Working directory |
| `status` | TEXT | `running`, `succeeded`, `failed` |
| `exitCode` | INT | — |
| `logStore`, `logRef`, `logBytes`, `logSha256`, `logCompressed` | — | Log storage (same pattern as heartbeat runs) |
| `stdoutExcerpt`, `stderrExcerpt` | TEXT | Truncated output for previews |
| `metadata` | JSONB | — |
| `startedAt`, `finishedAt` | TIMESTAMPTZ | — |

### 1.3 `workspace_runtime_services` Table

Tracks services (dev servers, databases) running within workspaces.

| Column | Type | Notes |
|--------|------|-------|
| `id` | UUID | PK |
| `companyId` | UUID | FK → companies |
| `projectId`, `projectWorkspaceId`, `executionWorkspaceId`, `issueId` | UUID | Scope FKs |
| `scopeType` | TEXT | `project_workspace`, `execution_workspace`, `run`, `agent` |
| `scopeId` | TEXT | — |
| `serviceName` | TEXT | e.g., `dev-server`, `postgres`, `redis` |
| `status` | TEXT | `starting`, `running`, `stopped`, `failed` |
| `lifecycle` | TEXT | `shared` (persists) or `ephemeral` (per-run) |
| `reuseKey` | TEXT | For shared services: reuse across runs |
| `command` | TEXT | Command to start the service |
| `cwd` | TEXT | Working directory |
| `port` | INT | Listening port |
| `url` | TEXT | Service URL |
| `provider` | TEXT | `local_process`, `docker`, `cloud` |
| `providerRef` | TEXT | — |
| `ownerAgentId` | UUID | FK → agents |
| `startedByRunId` | UUID | FK → heartbeat_runs |
| `stopPolicy` | JSONB | When/how to stop the service |
| `healthStatus` | TEXT | `unknown`, `healthy`, `unhealthy` |

---

## 2. Execution Workspace Modes

| Mode | Description |
|------|-------------|
| `shared_workspace` | All agents work in the same project workspace directory. Simple, risk of conflicts. |
| `isolated_workspace` | Each task gets its own workspace (via git worktree or copy). No conflicts. |
| `operator_branch` | Board operator controls which branch agents work on. |
| `adapter_default` | Let the adapter decide (e.g., Codex sandbox). |

## 3. Strategy Types

| Strategy | How workspace is created |
|----------|------------------------|
| `project_primary` | Use the project's primary workspace directory as-is |
| `git_worktree` | `git worktree add` — separate checkout for each task branch |
| `adapter_managed` | Adapter handles workspace isolation (e.g., Codex sandbox) |
| `cloud_sandbox` | Remote sandbox environment (e.g., e2b) |

---

## 4. Workspace Policy (Project-Level)

Each project can define an `executionWorkspacePolicy`:

```typescript
{
  enabled: boolean,
  defaultMode: "shared_workspace" | "isolated_workspace" | "operator_branch" | "adapter_default",
  allowIssueOverride: boolean,      // Issues can override the project default
  defaultProjectWorkspaceId: UUID,   // Which workspace to use
  workspaceStrategy: {
    type: "project_primary" | "git_worktree" | "adapter_managed" | "cloud_sandbox",
    baseRef: string,                 // e.g., "main"
    branchTemplate: string,          // e.g., "paperclip/{{issueId}}"
    worktreeParentDir: string,
    provisionCommand: string,
    teardownCommand: string,
  },
  workspaceRuntime: { ... },         // Runtime service config
  branchPolicy: { ... },
  pullRequestPolicy: { ... },
  runtimePolicy: { ... },
  cleanupPolicy: { ... },
}
```

Issues can also have per-issue `executionWorkspaceSettings` that override the project policy.

---

## 5. Lifecycle

### 5.1 Provisioning (during heartbeat)

1. Heartbeat service resolves workspace for run (see 06_HEARTBEAT_SYSTEM.md)
2. If `isolated_workspace` mode → `realizeExecutionWorkspace()`:
   - Create `execution_workspaces` row
   - Run `git worktree add` or provision command
   - Log operation in `workspace_operations`
3. If runtime services needed → `ensureRuntimeServicesForRun()`:
   - Start dev servers, databases as configured
   - Track in `workspace_runtime_services`

### 5.2 During Execution

- Agent works in the execution workspace CWD
- Runtime services available at configured ports/URLs
- Adapters may report runtime services they discovered/created

### 5.3 Post-Execution

- `releaseRuntimeServicesForRun()` — stop ephemeral services
- `persistAdapterManagedRuntimeServices()` — save services the adapter reported
- `cleanupExecutionWorkspaceArtifacts()` — cleanup based on policy

### 5.4 Status Lifecycle

```
active → idle → in_review → closed → cleanup_pending → (deleted)
```

---

## 6. Runtime Services

The `workspace-runtime.ts` service (~1,564 lines) manages long-running services:

- **Shared services** — persist across runs (e.g., dev server stays running)
- **Ephemeral services** — start/stop per run
- **Reuse key** — shared services matched by key to avoid duplicates
- **Health tracking** — `unknown`, `healthy`, `unhealthy`
- **Stop policy** — configurable auto-stop rules (idle timeout, run completion, etc.)
- **Provider types** — `local_process`, `docker`, `cloud`

---

## 7. Feature Flag

Execution workspaces are gated behind an **experimental** flag:

```typescript
const isolatedWorkspacesEnabled = (await instanceSettings.getExperimental()).enableIsolatedWorkspaces;
```

When disabled:
- `executionWorkspaceId`, `executionWorkspacePreference`, `executionWorkspaceSettings` are stripped from issue create/update
- Workspace realization is skipped during heartbeats
- Policy is gated: `gateProjectExecutionWorkspacePolicy(policy, enabled)` returns null if disabled

---

## 8. API Surface

| Method | Path | Description |
|--------|------|-------------|
| `GET` | `/api/companies/:companyId/execution-workspaces` | List execution workspaces (filterable) |
| `GET` | `/api/execution-workspaces/:id` | Get execution workspace detail |
| `GET` | `/api/heartbeat-runs/:runId/workspace-operations` | Workspace operations for a run |
| `GET` | `/api/workspace-operations/:id/log` | Read operation log |

---

## 9. Key Design Decisions

| Decision | Rationale |
|----------|-----------|
| Experimental feature flag | Complex feature; gate it until stable |
| Separate from project workspaces | Project workspaces = where code lives. Execution workspaces = how agents work in it. |
| Git worktree as primary isolation | Lightweight, fast, native git. Better than full clones. |
| Branch template (`paperclip/{{issueId}}`) | Predictable branch naming for agent-created branches |
| Runtime service tracking | Agents may need dev servers, DBs. System tracks them for cleanup and reuse. |
| Operation log (same pattern as heartbeat runs) | Consistent log storage for all process outputs |
| Workspace lineage (`derivedFromExecutionWorkspaceId`) | Track which workspace was cloned/forked from which |

---

## 10. Observations for Colmena

1. **This is an advanced feature that most deployments won't need immediately.** It solves the real problem of multiple agents working on the same codebase concurrently, but adds significant complexity.

2. **For Colmena MVP, skip execution workspaces entirely.** Use the simpler model: agents work in the project's primary workspace directory (shared). Add isolation when concurrent agent conflicts become a problem.

3. **Git worktrees are the right isolation primitive** for local deployments. They're fast, lightweight, and agents can work on separate branches without full clones.

4. **Runtime service management** (dev servers, databases per workspace) is production-grade but complex (~1,564 lines). Consider deferring until Colmena supports agents that need local services.

5. **The workspace policy system** is well-designed for flexibility (project-level defaults + per-issue overrides) but adds configuration burden. Start with a single mode ("shared workspace") and add policy options later.

6. **The experimental feature flag pattern** is smart — ship the code but don't expose it until it's stable. Worth adopting for any complex feature in Colmena.

7. **Branch templates** (`paperclip/{{issueId}}`) are a nice touch — agents create predictable branch names that map to issues. Easy to review and merge.
