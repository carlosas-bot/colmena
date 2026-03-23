# Agent Management — full lifecycle, config versioning, session persistence

## Investigation Summary

Agents are the employees of a Paperclip company. Every agent has a role, a reporting line, an adapter config (how it runs), a heartbeat policy (when it runs), and a budget. Agent management is the most complex domain in Paperclip — the service is ~500 lines, the route file is ~1200 lines, and there are 6 dedicated database tables.

---

## 1. Data Model

### 1.1 `agents` Table (core)

| Column | Type | Default | Notes |
|--------|------|---------|-------|
| `id` | UUID | random | PK |
| `companyId` | UUID | — | FK → `companies.id` |
| `name` | TEXT | — | Required, min 1 char. Unique *shortname* per company (derived via URL-key normalization) |
| `role` | TEXT | `"general"` | Enum: `ceo`, `cto`, `cmo`, `cfo`, `engineer`, `designer`, `pm`, `qa`, `devops`, `researcher`, `general` |
| `title` | TEXT | NULL | Free-text title (e.g., "VP of Engineering") |
| `icon` | TEXT | NULL | Enum of ~40 icon names (`bot`, `cpu`, `brain`, `zap`, `rocket`, etc.) |
| `status` | TEXT | `"idle"` | Enum: `active`, `paused`, `idle`, `running`, `error`, `pending_approval`, `terminated` |
| `reportsTo` | UUID | NULL | Self-referencing FK → `agents.id`. NULL = root (CEO) |
| `capabilities` | TEXT | NULL | Free-text description of what this agent can do |
| `adapterType` | TEXT | `"process"` | Enum: `process`, `http`, `claude_local`, `codex_local`, `opencode_local`, `pi_local`, `cursor`, `openclaw_gateway`, `hermes_local` |
| `adapterConfig` | JSONB | `{}` | Adapter-specific settings (model, CWD, prompt, env vars, etc.) |
| `runtimeConfig` | JSONB | `{}` | Heartbeat policy, timeout, context mode, extra args |
| `budgetMonthlyCents` | INT | 0 | Per-agent monthly budget |
| `spentMonthlyCents` | INT | 0 | Cached (also computed live from `cost_events`) |
| `pauseReason` | TEXT | NULL | `"manual"`, `"budget"`, `"system"` |
| `pausedAt` | TIMESTAMPTZ | NULL | When paused |
| `permissions` | JSONB | `{}` | e.g., `{ canCreateAgents: true }` |
| `lastHeartbeatAt` | TIMESTAMPTZ | NULL | Last heartbeat timestamp |
| `metadata` | JSONB | NULL | Arbitrary agent metadata |
| `createdAt` | TIMESTAMPTZ | now | — |
| `updatedAt` | TIMESTAMPTZ | now | — |

**Indexes:**
- `(companyId, status)` — list active agents for a company
- `(companyId, reportsTo)` — org chart queries

### 1.2 `agent_config_revisions` Table

Every configuration change is versioned for audit and rollback.

| Column | Type | Notes |
|--------|------|-------|
| `id` | UUID | PK |
| `companyId` | UUID | FK → companies |
| `agentId` | UUID | FK → agents (CASCADE delete) |
| `createdByAgentId` | UUID | FK → agents (SET NULL on delete) |
| `createdByUserId` | TEXT | — |
| `source` | TEXT | `"patch"`, `"rollback"`, `"skill-sync"`, `"instructions_path_patch"`, `"instructions_bundle_patch"`, `"instructions_bundle_file_put"` |
| `rolledBackFromRevisionId` | UUID | Set when this revision is a rollback |
| `changedKeys` | JSONB (string[]) | Which config fields changed |
| `beforeConfig` | JSONB | Full config snapshot before change |
| `afterConfig` | JSONB | Full config snapshot after change |
| `createdAt` | TIMESTAMPTZ | — |

**Tracked fields:** `name`, `role`, `title`, `reportsTo`, `capabilities`, `adapterType`, `adapterConfig`, `runtimeConfig`, `budgetMonthlyCents`, `metadata`.

**Design:** Before/after snapshots enable diff display and full rollback. Snapshots are **sanitized** (secrets redacted) before storage. Rollback to a revision with redacted values is blocked.

### 1.3 `agent_runtime_state` Table

Tracks cumulative runtime state and session persistence per agent.

| Column | Type | Notes |
|--------|------|-------|
| `agentId` | UUID | PK, FK → agents |
| `companyId` | UUID | FK → companies |
| `adapterType` | TEXT | — |
| `sessionId` | TEXT | Adapter session ID for conversation continuity |
| `stateJson` | JSONB | Arbitrary adapter-specific state |
| `lastRunId` | UUID | — |
| `lastRunStatus` | TEXT | — |
| `totalInputTokens` | BIGINT | Lifetime cumulative |
| `totalOutputTokens` | BIGINT | Lifetime cumulative |
| `totalCachedInputTokens` | BIGINT | Lifetime cumulative |
| `totalCostCents` | BIGINT | Lifetime cumulative |
| `lastError` | TEXT | — |

**Design:** Single row per agent (PK = agentId). Updated after every heartbeat run. Session persistence enables agents to resume conversations across heartbeats without re-reading everything.

### 1.4 `agent_task_sessions` Table

Separate session state **per task** (not just per agent). Allows an agent to maintain different conversation contexts for different tasks.

| Column | Type | Notes |
|--------|------|-------|
| `id` | UUID | PK |
| `companyId` | UUID | FK → companies |
| `agentId` | UUID | FK → agents |
| `adapterType` | TEXT | — |
| `taskKey` | TEXT | Issue ID or task identifier |
| `sessionParamsJson` | JSONB | Session state for this specific task |
| `sessionDisplayId` | TEXT | Human-readable session ref |
| `lastRunId` | UUID | FK → heartbeat_runs |
| `lastError` | TEXT | — |

**Unique constraint:** `(companyId, agentId, adapterType, taskKey)` — one session per agent per adapter per task.

### 1.5 `agent_api_keys` Table

Long-lived API keys for agents that need persistent access outside heartbeats.

| Column | Type | Notes |
|--------|------|-------|
| `id` | UUID | PK |
| `agentId` | UUID | FK → agents |
| `companyId` | UUID | FK → companies |
| `name` | TEXT | Key label |
| `keyHash` | TEXT | SHA-256 hash of the actual token |
| `lastUsedAt` | TIMESTAMPTZ | — |
| `revokedAt` | TIMESTAMPTZ | — |

**Design:** Tokens are `pcp_` + 24 random hex bytes. Only the hash is stored. Full value shown once at creation. Keys are auto-revoked on agent termination.

### 1.6 `agent_wakeup_requests` Table

Queue system for agent wakeup events.

| Column | Type | Notes |
|--------|------|-------|
| `id` | UUID | PK |
| `companyId`, `agentId` | UUID | FKs |
| `source` | TEXT | `"timer"`, `"assignment"`, `"on_demand"`, `"automation"` |
| `triggerDetail` | TEXT | `"manual"`, `"ping"`, `"callback"`, `"system"` |
| `reason` | TEXT | Human-readable reason |
| `payload` | JSONB | Context data for the wakeup |
| `status` | TEXT | `queued` → `deferred_issue_execution` / `claimed` / `coalesced` / `skipped` / `completed` / `failed` / `cancelled` |
| `coalescedCount` | INT | How many requests were coalesced into this one |
| `requestedByActorType` | TEXT | — |
| `requestedByActorId` | TEXT | — |
| `idempotencyKey` | TEXT | Dedup key |
| `runId` | UUID | The heartbeat run that served this wakeup |
| `requestedAt`, `claimedAt`, `finishedAt` | TIMESTAMPTZ | Lifecycle timestamps |
| `error` | TEXT | — |

---

## 2. Agent Status Machine

```
                                     ┌──────────────┐
                                     │   pending_    │
                          ┌─────────▶│   approval    │─────────┐
                          │          └──────────────┘          │
                          │            (board approves)        │ (board rejects
                          │                    │               │  or terminates)
                          │                    ▼               │
  ┌──────────┐     ┌──────┴──┐      ┌─────────┐              │
  │ (create) │────▶│   idle   │◀────▶│ running  │              │
  └──────────┘     └────┬─────┘      └────┬────┘              │
                        │                  │                    │
                        ▼                  ▼                    │
                   ┌─────────┐      ┌──────────┐              │
                   │ paused   │      │  error    │              │
                   │(manual/  │      └────┬─────┘              │
                   │ budget)  │           │                     │
                   └────┬─────┘           │                     │
                        │                  │                    │
                        ▼                  ▼                    ▼
                   ┌─────────────────────────────────────────────┐
                   │              terminated (irreversible)       │
                   └─────────────────────────────────────────────┘
```

**Key transitions:**
- **Create** → `idle` (direct creation by board) or `pending_approval` (via agent hire request)
- **idle** ↔ **running** — heartbeat starts/finishes
- **Any active state** → **paused** — manual, budget, or system pause
- **paused** → **idle** — resume (only if not budget-paused while still over budget)
- **pending_approval** → **idle** — board approves hire
- **Any state** → **terminated** — irreversible. All API keys revoked.

**Constraints:**
- Cannot resume `terminated` agents
- Cannot directly activate `pending_approval` agents (must go through approval)
- Cannot create API keys for `pending_approval` or `terminated` agents

---

## 3. API Surface

### 3.1 Core CRUD + Lifecycle

| Method | Path | Auth | Description |
|--------|------|------|-------------|
| `GET` | `/api/companies/:companyId/agents` | Company access | List agents (excludes terminated by default) |
| `GET` | `/api/agents/me` | Agent JWT | Get current agent with chain of command + access state |
| `GET` | `/api/agents/me/inbox-lite` | Agent JWT | Lightweight issue inbox for the agent |
| `GET` | `/api/agents/:id` | Company access | Get agent detail (config redacted for non-privileged agents) |
| `POST` | `/api/companies/:companyId/agents` | Board only | Create agent (direct, no approval) |
| `POST` | `/api/companies/:companyId/agent-hires` | Board or agent with `canCreateAgents` | Create agent via hire flow (may require approval) |
| `PATCH` | `/api/agents/:id` | Board or CEO/agent-creator | Update agent config |
| `POST` | `/api/agents/:id/pause` | Board | Pause agent (cancels active heartbeats) |
| `POST` | `/api/agents/:id/resume` | Board | Resume agent |
| `POST` | `/api/agents/:id/terminate` | Board | Terminate agent (irreversible, revokes keys) |
| `DELETE` | `/api/agents/:id` | Board | Hard delete (cascading delete of runs, keys, sessions, etc.) |

### 3.2 Configuration Management

| Method | Path | Description |
|--------|------|-------------|
| `GET` | `/api/agents/:id/configuration` | Get redacted agent config (for privileged users) |
| `GET` | `/api/companies/:companyId/agent-configurations` | List all agent configs (redacted) |
| `GET` | `/api/agents/:id/config-revisions` | List config revision history |
| `GET` | `/api/agents/:id/config-revisions/:revisionId` | Get specific revision |
| `POST` | `/api/agents/:id/config-revisions/:revisionId/rollback` | Rollback to a previous config |

### 3.3 Instructions Bundle

Managed instructions system (AGENTS.md files) for agents:

| Method | Path | Description |
|--------|------|-------------|
| `GET` | `/api/agents/:id/instructions-bundle` | Get instructions metadata |
| `PATCH` | `/api/agents/:id/instructions-bundle` | Update bundle config (mode, root path, entry file) |
| `GET` | `/api/agents/:id/instructions-bundle/file?path=...` | Read a specific instructions file |
| `PUT` | `/api/agents/:id/instructions-bundle/file` | Create/update instructions file |
| `DELETE` | `/api/agents/:id/instructions-bundle/file?path=...` | Delete instructions file |
| `PATCH` | `/api/agents/:id/instructions-path` | Update instructions file path in adapter config |

### 3.4 Skills

| Method | Path | Description |
|--------|------|-------------|
| `GET` | `/api/agents/:id/skills` | List skills snapshot for agent |
| `POST` | `/api/agents/:id/skills/sync` | Sync desired skills to agent |

### 3.5 Runtime & Sessions

| Method | Path | Description |
|--------|------|-------------|
| `GET` | `/api/agents/:id/runtime-state` | Get runtime state (session, tokens, errors) |
| `GET` | `/api/agents/:id/task-sessions` | List per-task sessions |
| `POST` | `/api/agents/:id/runtime-state/reset-session` | Reset session state (optionally per-task) |

### 3.6 API Keys

| Method | Path | Description |
|--------|------|-------------|
| `GET` | `/api/agents/:id/keys` | List API keys |
| `POST` | `/api/agents/:id/keys` | Create API key (full token returned once) |
| `DELETE` | `/api/agents/:id/keys/:keyId` | Revoke API key |

### 3.7 Heartbeat Invocation

| Method | Path | Description |
|--------|------|-------------|
| `POST` | `/api/agents/:id/heartbeat/invoke` | Manual heartbeat invocation |
| `POST` | `/api/agents/:id/wakeup` | Wakeup request (richer, with source/payload/idempotency) |
| `POST` | `/api/agents/:id/claude-login` | Claude-specific OAuth login flow |

### 3.8 Org Chart

| Method | Path | Description |
|--------|------|-------------|
| `GET` | `/api/companies/:companyId/org` | Full org tree JSON |
| `GET` | `/api/companies/:companyId/org.svg` | Org chart as SVG (with style parameter) |
| `GET` | `/api/companies/:companyId/org.png` | Org chart as PNG |

### 3.9 Heartbeat Runs & Logs

| Method | Path | Description |
|--------|------|-------------|
| `GET` | `/api/companies/:companyId/heartbeat-runs` | List runs (filterable by agent, with limit) |
| `GET` | `/api/companies/:companyId/live-runs` | Currently running/queued runs |
| `GET` | `/api/heartbeat-runs/:runId` | Get run detail |
| `POST` | `/api/heartbeat-runs/:runId/cancel` | Cancel a running heartbeat |
| `GET` | `/api/heartbeat-runs/:runId/events` | List run events (paginated by seq) |
| `GET` | `/api/heartbeat-runs/:runId/log` | Read raw log (offset + limitBytes) |
| `GET` | `/api/heartbeat-runs/:runId/workspace-operations` | Workspace ops for a run |
| `GET` | `/api/issues/:issueId/live-runs` | Active runs for an issue |
| `GET` | `/api/issues/:issueId/active-run` | Currently active run for an issue |

### 3.10 Adapter Utilities

| Method | Path | Description |
|--------|------|-------------|
| `GET` | `/api/companies/:companyId/adapters/:type/models` | List available models for adapter |
| `POST` | `/api/companies/:companyId/adapters/:type/test-environment` | Validate adapter config |

---

## 4. Business Logic

### 4.1 Agent Name Deduplication

Agent names must be unique per company (by normalized URL key):

1. `normalizeAgentUrlKey(name)` produces a URL-safe shortname
2. On create: `deduplicateAgentName()` appends ` 2`, ` 3`, etc. if collision detected
3. On update: `assertCompanyShortnameAvailable()` throws `409 Conflict` if new name collides

Terminated agents are excluded from collision checks.

### 4.2 Org Hierarchy Validation

**On create:**
- If `reportsTo` is set, validates the manager exists and belongs to the same company

**On update:**
- `assertNoCycle()` walks the chain of command to prevent circular reporting relationships
- Validates manager belongs to same company

**Org tree building:**
- `orgForCompany()` loads all non-terminated agents, groups by `reportsTo`, recursively builds tree starting from `null` (root nodes)

### 4.3 Config Revision System

Every config update (if `recordRevision` option is set) goes through:

1. Snapshot `beforeConfig` from existing agent (sanitized — secrets redacted)
2. Apply update
3. Snapshot `afterConfig` from updated agent
4. Diff to find `changedKeys`
5. If any keys changed, insert `agent_config_revisions` row

**Rollback** applies the `afterConfig` snapshot from the target revision as a new patch, creating a new revision with `source: "rollback"`.

### 4.4 Agent Hiring (Governance Flow)

When an agent requests to hire a subordinate:

1. Agent calls `POST /api/companies/:companyId/agent-hires` with proposed config
2. Server checks `company.requireBoardApprovalForNewAgents`
3. If approval required:
   - Agent is created with `status: "pending_approval"`
   - An `hire_agent` approval is created with the proposed config as payload
   - Source issues are linked to the approval
4. If no approval required:
   - Agent is created with `status: "idle"` and is immediately active
5. Default instructions bundle is materialized (AGENTS.md written to disk)
6. Default `tasks:assign` permission grant is applied

### 4.5 Permissions System

**Agent-level permissions:**
- `canCreateAgents` — boolean, defaults to `true` for CEO role, `false` for others
- `canAssignTasks` — derived from: CEO role, `canCreateAgents` permission, or explicit `tasks:assign` grant

**Who can update agents:**
- Board users: always
- CEO agents: always (within their company)
- Agents with `canCreateAgents`: can modify other agents
- Self: agents can read their own config
- Other agents: restricted view only (config redacted)

**Who can manage instructions path:**
- Board users: always
- Self: agents can manage their own instructions
- Ancestor managers in the chain of command

### 4.6 Adapter Config Normalization

On create/update:
1. `applyCreateDefaultsByAdapterType()` — sets default model, sandbox bypass, etc. per adapter
2. `ensureGatewayDeviceKey()` — generates Ed25519 key pair for OpenClaw gateway adapters
3. `secretsSvc.normalizeAdapterConfigForPersistence()` — converts inline secrets to encrypted `secret_ref` objects
4. `assertAdapterConfigConstraints()` — adapter-specific validation (e.g., OpenCode model availability check)
5. `syncInstructionsBundleConfigFromFilePath()` — keeps instructions bundle config in sync with file path changes

### 4.7 Default Instructions Bundle

When a new agent is created:
1. If no explicit instructions bundle configured, system materializes a default `AGENTS.md`
2. Default content is role-aware (different defaults for CEO, CTO, engineer, etc.)
3. If a `promptTemplate` was provided, it becomes the AGENTS.md content instead
4. The `promptTemplate` field is then cleared from `adapterConfig` (replaced by the materialized file)
5. Bundle is written to disk via `agentInstructionsService`

### 4.8 Session Persistence

**Agent-level sessions:**
- `agent_runtime_state` stores a single session per agent
- Session ID and state JSON are preserved across heartbeats
- Adapter serializes session after each run, restores on next

**Task-level sessions:**
- `agent_task_sessions` stores per-task session state
- Unique per `(agent, adapter, taskKey)`
- Allows an agent to maintain separate conversation contexts per task
- Task sessions record which heartbeat run last used them

**Session reset:**
- Board can reset agent session (global or per-task)
- Clears session ID and state JSON

### 4.9 Agent Deletion (Cascading)

Hard delete `DELETE /api/agents/:id`:
1. Reparent direct reports to NULL (`UPDATE agents SET reportsTo = NULL WHERE reportsTo = id`)
2. Delete in order: `heartbeat_run_events` → `agent_task_sessions` → `heartbeat_runs` → `agent_wakeup_requests` → `agent_api_keys` → `agent_runtime_state` → `agents`
3. All in a single transaction

### 4.10 Agent Shortname / URL Key

Agents can be referenced by UUID or by shortname (URL key):
- `normalizeAgentUrlKey("My Agent")` → `"my-agent"`
- Route param middleware auto-resolves shortnames to UUIDs
- If ambiguous (multiple agents with same shortname after normalization), returns `409 Conflict`

---

## 5. Heartbeat Policy (in `runtimeConfig`)

The heartbeat schedule is stored in `runtimeConfig.heartbeat`:

```typescript
{
  heartbeat: {
    enabled: boolean,      // default true
    intervalSec: number,   // default 0 (disabled)
    // Wake triggers (parsed elsewhere):
    wakeOnAssignment: boolean,   // default true
    wakeOnOnDemand: boolean,     // default true
    wakeOnAutomation: boolean,   // default true
    cooldownSec: number,         // default 10
  }
}
```

The instance-level scheduler queries all agents with `intervalSec > 0` and eligible status to determine heartbeat scheduling.

---

## 6. Adapter Config Structure

The `adapterConfig` JSONB holds runtime-specific settings. Common fields:

```typescript
{
  // Shared across adapters:
  cwd: string,                    // Working directory
  promptTemplate: string,         // Legacy — now replaced by instructions bundle
  model: string,                  // LLM model override
  env: Record<string, string | SecretRef>,  // Environment variables (may include secret refs)
  
  // Instructions bundle (managed):
  instructionsBundleMode: "managed" | "external",
  instructionsRootPath: string,
  instructionsEntryFile: string,
  instructionsFilePath: string,   // Path to AGENTS.md
  
  // Claude-specific:
  maxTurnsPerRun: number,         // default 80
  skipPermissions: boolean,       // default true
  
  // Codex-specific:
  search: boolean,
  dangerouslyBypassApprovalsAndSandbox: boolean,
  
  // Process-specific:
  command: string,
  args: string[],
  
  // HTTP-specific:
  url: string,
  method: string,
  headers: Record<string, string>,
  
  // OpenClaw gateway-specific:
  devicePrivateKeyPem: string,    // Auto-generated Ed25519 key
  disableDeviceAuth: boolean,
  
  // Skill sync preferences:
  paperclipSkillSyncDesiredSkills: string[],
}
```

---

## 7. Access State & Detail Response

`GET /api/agents/:id` and `GET /api/agents/me` return enriched detail:

```typescript
{
  // All agent fields...
  chainOfCommand: [
    { id, name, role, title },  // Direct manager
    { id, name, role, title },  // Manager's manager
    // ... up to CEO
  ],
  access: {
    canAssignTasks: boolean,
    taskAssignSource: "ceo_role" | "agent_creator" | "explicit_grant" | "none",
    membership: { ... } | null,
    grants: [{ permissionKey, ... }],
  },
  urlKey: string,  // URL-safe shortname
}
```

---

## 8. Key Design Decisions

| Decision | Rationale |
|----------|-----------|
| Self-referencing FK for `reportsTo` | Simple tree structure without separate hierarchy table. Cycle detection done in application layer |
| JSONB for `adapterConfig` and `runtimeConfig` | Adapter configs are highly polymorphic; structured JSONB is more flexible than normalized tables |
| Config revisions with full before/after snapshots | Enables complete rollback without incremental patch replay. More storage but simpler logic |
| Separate agent-level and task-level sessions | Agents may work on multiple tasks; per-task sessions prevent context contamination |
| SHA-256 hashed API keys | Standard security practice — keys can be verified without storing plaintext |
| URL key (shortname) resolution via middleware | Transparent to route handlers — shortnames work everywhere IDs work |
| Hire flow creating agent in `pending_approval` state | Agent record exists immediately (for linking), but can't run until approved |
| Default instructions bundle materialization | Agents get working AGENTS.md files automatically; no manual setup required |
| Permissions as JSONB with application-level defaults | Simple and extensible; CEO gets `canCreateAgents` by default |
| Manual cascading delete | Control over order and ability to reparent orphaned direct reports |

---

## 9. Observations for Colmena

1. **Agent config is the most complex part** — adapterConfig is a polymorphic JSONB bag that varies wildly by adapter type. The normalization pipeline (defaults → secrets → constraints → sync) is ~6 steps. Consider whether Colmena needs all this flexibility or if a more opinionated config structure would suffice.

2. **Config revision system is valuable** — being able to rollback a broken config is critical for autonomous systems. Worth keeping.

3. **Session persistence (both agent-level and task-level) is a key differentiator** — without this, agents lose all context on every heartbeat. The per-task session design is particularly clever.

4. **The hire/approval flow is governance gold** — agents requesting to hire subordinates with board approval is a powerful pattern for autonomous org growth.

5. **Instructions bundle system is sophisticated but complex** — managed AGENTS.md files, role-aware defaults, legacy prompt template migration. For Colmena MVP, a simpler approach (just a prompt/instructions field) might suffice.

6. **Wakeup request queue** with coalescing, idempotency, and multiple status transitions is production-grade but complex. Simpler approaches possible for MVP.

7. **The 9 adapter types** suggest the system was built to support many runtimes. Colmena should decide upfront which adapters to support and possibly start with fewer.

8. **Org chart SVG/PNG rendering server-side** is a nice touch for dashboards and exports.

9. **Agent shortname resolution** is a UX win — using `@EngineerLead` instead of UUIDs makes @-mentions and API calls more natural.
