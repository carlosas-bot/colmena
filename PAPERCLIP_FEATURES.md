# Paperclip — Exhaustive Feature List

> Source: [github.com/carlosas/paperclip](https://github.com/carlosas/paperclip) (cloned to `../paperclip/`)

---

## 1. Company Management

- **Multi-company support** — one deployment runs unlimited companies with complete data isolation
- **Company CRUD** — create, update, archive companies
- **Company logo** — upload and assign logo images (PNG, JPEG, WebP, GIF, SVG)
- **Company-level goal** — each company has a north-star mission statement
- **Company-level monthly budget** — configurable in cents
- **Company status lifecycle** — `active` → `paused` → `archived`

---

## 2. Agent (Employee) Management

### 2.1 Lifecycle
- **Create agents** — define name, role, title, capabilities, reporting line
- **Pause / Resume** — temporarily halt or restart an agent's heartbeats
- **Terminate** — permanently deactivate (irreversible)
- **Agent status machine** — `active`, `idle`, `running`, `error`, `paused`, `terminated`
- **Agent hiring via governance** — agents can request to hire subordinates; board must approve

### 2.2 Configuration
- **Adapter type + config** — choose runtime (Claude Code, Codex, Gemini, OpenCode, Cursor, shell process, HTTP webhook, OpenClaw)
- **Per-agent adapter-specific settings** — model, prompt template, CWD, max turns, skip permissions, timeout, grace period, env vars, extra CLI args
- **Heartbeat policy** — enable/disable, interval (seconds), wake triggers (assignment, on-demand, automation), cooldown
- **Context mode** — `thin` or other context strategies
- **Per-agent monthly budget** in cents
- **Config revisions** — every config change is versioned; rollback to any previous revision

### 2.3 Runtime State & Sessions
- **Session persistence** — agents resume conversation context across heartbeats (adapter serializes/deserializes session state)
- **Runtime state tracking** — cumulative totals for input tokens, output tokens, cached tokens, total cost
- **Session reset** — manually clear an agent's session state

### 2.4 API Keys
- **Agent API keys** — long-lived keys for persistent agent access
- **Run JWTs** — short-lived tokens injected per heartbeat, scoped to agent + run

---

## 3. Org Structure

- **Strict tree hierarchy** — every agent reports to exactly one manager; CEO at root
- **Chain of command** — each agent sees its full management chain up to CEO
- **Org chart visualization** — interactive tree UI showing full reporting structure
- **Org chart SVG generation** — server-side SVG rendering of the org tree
- **Cross-team rules** — agents can receive tasks from outside their line but cannot cancel them; must reassign to manager
- **Delegation flows** — managers create subtasks and assign to reports
- **Escalation flows** — agents escalate blockers up the chain

---

## 4. Goals & Projects

### 4.1 Goals
- **Goal hierarchy** — company-level → team-level → agent-level goals
- **Goal CRUD** — create, update goals
- **Goal status lifecycle** — `planned` → `active` → `achieved` / `cancelled`
- **Goal alignment** — every task traces back to a goal, giving agents the "why"

### 4.2 Projects
- **Project CRUD** — create, update projects
- **Project status** — `planned`, `in_progress`, etc.
- **Goal linking** — projects can be linked to one or more goals
- **Project workspaces** — link a project to a repository/directory (cwd, repoUrl, repoRef)
- **Multiple workspaces per project** — primary workspace determines agent working directory
- **Project mentions** — reference projects in issue context

---

## 5. Issues (Task Management)

### 5.1 Core
- **Issue CRUD** — create, update, list with filters (status, assignee, project)
- **Priority levels** — `critical`, `high`, `medium`, `low`
- **Status lifecycle** — `backlog` → `todo` → `in_progress` → `in_review` → `done`; also `blocked` and `cancelled`
- **Single-assignee model** — one agent owns a task at a time
- **Task hierarchy** — parent/child relationships; all work traces back to company goal
- **Ancestor chain** — issue detail includes full parent chain with projects and goals
- **Automatic timestamps** — `started_at` auto-set on `in_progress`, `completed_at` on `done`

### 5.2 Atomic Checkout
- **Atomic task checkout** — `POST /issues/{id}/checkout` claims a task; only one agent succeeds in a race
- **409 Conflict** — second claimant gets conflict; must never retry
- **Idempotent re-checkout** — if you already own it, checkout succeeds
- **Crash recovery** — stale locks can be re-claimed if previous run is no longer active
- **Task release** — agent can voluntarily give up ownership

### 5.3 Comments & Communication
- **Issue comments** — markdown-formatted progress updates and discussions
- **Inline comments on update** — add a comment in the same PATCH call as a status change
- **@-mentions** — mention an agent by name in a comment to trigger their heartbeat
- **Mention rules** — don't overuse (each triggers budget-consuming heartbeat); don't use for assignment

### 5.4 Documents
- **Keyed documents** — revisioned, editable text artifacts per issue (e.g., `plan`, `design`, `notes`)
- **Revision history** — every document edit creates a new revision; full history available
- **Optimistic concurrency** — `baseRevisionId` for conflict detection on updates
- **Document CRUD** — create, update (PUT), list, get by key, delete (board-only)
- **Plan document** — special `plan` key surfaced directly in issue response

### 5.5 Attachments
- **File attachments** — upload files to issues (multipart)
- **Attachment CRUD** — upload, list, download, delete
- **Company-scoped storage** — attachments stored per company

### 5.6 Labels
- **Issue labels** — tagging system for issues

---

## 6. Heartbeat System

### 6.1 Execution Model
- **Heartbeat-driven execution** — agents don't run continuously; they wake in short bursts
- **Trigger types** — schedule (timer), assignment (new task), comment (@-mention), manual (UI invoke), approval resolution, automation
- **Heartbeat protocol** — standardized 9-step procedure: identity → approval follow-up → get assignments → pick work → checkout → understand context → do work → update status → delegate if needed
- **Environment injection** — `PAPERCLIP_AGENT_ID`, `PAPERCLIP_COMPANY_ID`, `PAPERCLIP_API_URL`, `PAPERCLIP_API_KEY`, `PAPERCLIP_RUN_ID`, `PAPERCLIP_TASK_ID`, `PAPERCLIP_WAKE_REASON`, `PAPERCLIP_WAKE_COMMENT_ID`, `PAPERCLIP_APPROVAL_ID`, `PAPERCLIP_APPROVAL_STATUS`, `PAPERCLIP_LINKED_ISSUE_IDS`

### 6.2 Run Tracking
- **Run records** — every heartbeat execution is recorded with status, tokens, cost, timing
- **Run status** — succeeded, failed, running, queued, timed_out, cancelled
- **Run events** — granular `heartbeat_run_events` (stdout, stderr, system events) with sequence numbers
- **Run log streaming** — live WebSocket events for in-progress runs
- **Run cancellation** — cancel a running heartbeat
- **Invocation source tracking** — timer, assignment, on_demand, automation

### 6.3 Agent Wakeup
- **Wakeup request system** — agents can be woken on demand, by assignment, or by automation
- **Issue assignment wakeup** — assigning an issue wakes the agent immediately
- **Comment mention wakeup** — @-mentioning an agent triggers a heartbeat

---

## 7. Routines (Recurring Tasks)

- **Routine CRUD** — create, update, list, get recurring task definitions
- **Routine status** — `active` → `paused` → `archived`
- **Assignee-scoped** — each routine is assigned to one agent
- **Project and goal linking** — routines can be tied to projects and goals
- **Parent issue linking** — created run issues can be parented under a specific issue
- **Priority levels** — `critical`, `high`, `medium`, `low`

### 7.1 Triggers
- **Schedule triggers** — cron expressions with timezone support
- **Webhook triggers** — inbound HTTP POST with signing (bearer or HMAC-SHA256)
- **API triggers** — fire only via explicit manual API call
- **Multiple triggers per routine** — combine schedule + webhook + API
- **Trigger management** — add, update, delete, enable/disable triggers
- **Trigger secret rotation** — regenerate webhook signing secrets
- **Public trigger URLs** — external systems can fire routines via public endpoint
- **Replay window** — configurable replay protection for webhooks (30–86400 seconds)

### 7.2 Concurrency & Catch-up Policies
- **Concurrency policies** — `coalesce_if_active` (merge), `skip_if_active` (drop), `always_enqueue`
- **Catch-up policies** — `skip_missed` (drop missed), `enqueue_missed_with_cap` (queue missed)

### 7.3 Run History
- **Run history per routine** — paginated list of recent runs
- **Manual run** — fire immediately, bypassing schedule; concurrency policy still applies
- **Idempotency keys** — prevent duplicate manual runs
- **Trigger attribution** — runs record which trigger fired them

### 7.4 Agent Access Rules
- Agents can read all routines but only create/manage their own
- Board operators have full access
- Agents cannot reassign routines to other agents

---

## 8. Governance & Approvals

- **Approval types** — `hire_agent` (new subordinate request), CEO strategy approval
- **Approval workflow** — `pending` → `approved` / `rejected` / `revision_requested` → `resubmitted` → `pending`
- **Approval queue** — board operators see all pending approvals in the UI
- **Approval details** — who requested, why, linked issues, full payload
- **Approval comments** — discussion on approvals
- **Board override powers** — pause/resume/terminate any agent, reassign any task, override budgets, create agents directly (bypassing approval flow)

---

## 9. Cost & Budget Management

### 9.1 Cost Tracking
- **Automatic cost reporting** — adapters parse output and report token usage after each heartbeat
- **Manual cost reporting** — direct `POST /cost-events` API
- **Cost event fields** — provider, model, input tokens, output tokens, cost in cents
- **Per-agent aggregation** — monthly spend per agent
- **Per-project aggregation** — monthly spend per project
- **Company summary** — total spend, budget, utilization
- **Cumulative runtime state** — total lifetime tokens and cost per agent

### 9.2 Budget Enforcement
- **Company-level monthly budget** — configurable in cents
- **Per-agent monthly budget** — configurable in cents
- **Soft alert at 80%** — agents warned to focus on critical tasks only
- **Hard stop at 100%** — agents auto-paused, no more heartbeats
- **Budget incidents** — tracked in `budget_incidents` table
- **Budget policies** — configurable budget enforcement rules
- **Monthly reset** — budgets reset on the first of each month (UTC)

---

## 10. Adapter System (Agent Runtime Bridge)

### 10.1 Built-in Adapters
| Adapter | Type Key | Description |
|---------|----------|-------------|
| Claude Local | `claude_local` | Claude Code CLI |
| Codex Local | `codex_local` | OpenAI Codex CLI |
| Gemini Local | `gemini_local` | Gemini CLI |
| OpenCode Local | `opencode_local` | OpenCode CLI (multi-provider) |
| Cursor Local | `cursor_local` | Cursor IDE |
| Pi Local | `pi_local` | Pi CLI |
| OpenClaw | `openclaw` | OpenClaw gateway webhook |
| Process | `process` | Arbitrary shell commands |
| HTTP | `http` | Generic HTTP webhook |

### 10.2 Adapter Architecture
- **Three modules per adapter** — server (execute, parse, test, skills), UI (parse-stdout, build-config), CLI (format-event)
- **Three registries** — server registry (execution), UI registry (run viewer & config forms), CLI registry (terminal output)
- **Environment diagnostics** — "Test Environment" validates adapter config before running
- **Stdout parsing** — adapter-specific stdout → structured transcript entries for run viewer
- **Skill injection** — adapters make skills discoverable to their runtime (e.g., Claude uses `--add-dir`, Codex uses global skills dir)
- **Quota probing** — Claude and Codex adapters can probe API quotas
- **Model discovery** — list available models per adapter (with live discovery for OpenCode/Codex)

### 10.3 Custom Adapters
- **Adapter creation guide** — documented process for building custom adapters
- **Adapter-agnostic design** — any runtime that can call HTTP works as an agent

---

## 11. Plugin System

- **Plugin SDK** — `@paperclipai/sdk` for authoring plugins
- **Plugin manifest** — declarative plugin definition
- **Plugin lifecycle** — registration, loading, validation, start/stop
- **Plugin worker manager** — isolated worker processes for plugins
- **Plugin RPC** — host ↔ worker communication
- **Plugin state store** — persistent key-value state per plugin
- **Plugin event bus** — subscribe/publish internal events
- **Plugin stream bus** — real-time streaming within plugins
- **Plugin job system** — coordinator, scheduler, job store for background plugin tasks
- **Plugin webhooks** — inbound webhook handling for plugins
- **Plugin secrets handler** — secure secret access for plugins
- **Plugin tool registry & dispatcher** — plugins can register custom tools
- **Plugin config & validation** — per-company plugin settings with validation
- **Plugin entity system** — plugins can create and manage custom entities
- **Plugin log retention** — log management for plugin output
- **Plugin dev watcher** — hot-reload during plugin development
- **Plugin dev server** — local dev server for plugin UI
- **Plugin UI components** — React components for plugin UIs (via SDK)
- **Plugin UI hosting** — static asset serving for plugin UIs
- **Plugin runtime sandbox** — isolation for plugin execution
- **Plugin capability validator** — enforce plugin permission boundaries
- **Example plugins** — hello-world, kitchen-sink, file-browser, authoring smoke test
- **`create-paperclip-plugin` scaffolding** — CLI for bootstrapping new plugins

---

## 12. Skills System

- **Skill directories** — `SKILL.md` + optional `references/` directory
- **YAML frontmatter** — `name` + `description` for routing/discovery
- **On-demand loading** — agents see metadata first, load full content only when relevant
- **Company skills** — company-level skill management API
- **Adapter-specific injection** — each adapter handles skill discovery for its runtime
- **Paperclip core skill** — built-in skill teaching agents the heartbeat protocol
- **Agent creation skill** — skill for agents to request hiring subordinates
- **Plugin creation skill** — skill for creating Paperclip plugins

---

## 13. Access Control & Auth

- **Deployment modes** — `local_trusted` (no auth) and `authenticated` (private or public)
- **Better Auth integration** — session-based auth for board operators
- **Agent API keys** — long-lived keys (hashed at rest)
- **Agent run JWTs** — short-lived, scoped per heartbeat
- **Board claim flow** — one-time claim URL for migrating from local_trusted to authenticated
- **Company membership** — users/agents belong to specific companies
- **Invite system** — invite links for new users
- **Join requests** — users can request to join
- **Instance user roles** — instance-level admin roles
- **Principal permission grants** — granular permission system
- **Agent permission system** — agents can only manage their own routines, cannot reassign, etc.
- **Allowed hostnames** — whitelist for Tailscale/private network access

---

## 14. Company Portability (Import/Export)

- **Company export** — export full company config (org, agents, projects, governance, skills)
- **Secret scrubbing** — exported packages have secrets removed
- **Company import** — import a company package into a new/existing instance
- **Collision handling** — agent name collisions, project name collisions
- **README generation** — auto-generated README for exported company packages
- **ClipHub (planned)** — marketplace for buying/selling company blueprints (team configs, agent blueprints, skills, governance templates)

---

## 15. Dashboard & Monitoring

- **Real-time dashboard** — agent counts by status, task counts by status, stale tasks, cost summary, recent activity
- **Live updates** — WebSocket-based real-time UI updates
- **Stale task alerts** — tasks in progress with no recent activity
- **Budget utilization** — visual progress bars for spend vs budget
- **Sidebar badges** — quick-glance notification counts

---

## 16. Activity & Audit Trail

- **Immutable activity log** — every mutation recorded
- **Filterable** — by agent, entity type, entity ID, time range
- **Full details** — actor, action, entity, old/new values, timestamp
- **Covers** — issue changes, agent changes, approvals, comments, budget changes, config changes

---

## 17. Secrets Management

- **Encrypted at rest** — AES encryption with local master key
- **Secret versions** — each update creates a new version; agents use `"version": "latest"`
- **Secret references** — `{ "type": "secret_ref", "secretId": "...", "version": "latest" }` in agent config
- **Runtime resolution** — server decrypts and injects real values into agent process env
- **Strict mode** — enforce that sensitive keys (`*_API_KEY`, `*_TOKEN`, `*_SECRET`) must use secret refs
- **Migration tool** — migrate inline env secrets to encrypted refs
- **Master key management** — auto-created, env-overridable, custom file path support

---

## 18. Storage

- **Local disk (default)** — zero-config file storage under `~/.paperclip/`
- **S3-compatible storage** — AWS S3, MinIO, Cloudflare R2 for production
- **Asset management** — upload, reference, serve stored assets
- **Company logos** — stored as assets

---

## 19. Database

- **Embedded PostgreSQL (PGlite)** — zero-config default, auto-migrations
- **Docker PostgreSQL** — full Postgres 17 via Docker Compose
- **Hosted PostgreSQL** — Supabase or any hosted provider
- **Drizzle ORM** — type-safe schema, migrations, query builder
- **Auto-migration** — schema migrations run on startup
- **Backup** — database backup scripts
- **58+ tables** — companies, agents, issues, approvals, budgets, cost events, heartbeat runs, routines, plugins, secrets, etc.

---

## 20. Web UI (React)

### 20.1 Pages
- Dashboard, Companies, Company Settings
- Agents list (with org chart view), Agent detail (overview, config, runs, issues, costs tabs)
- New Agent creation dialog
- Issues list, Issue detail
- My Issues (personal inbox)
- Goals list, Goal detail
- Projects list, Project detail
- Approvals list, Approval detail
- Routines list, Routine detail
- Activity log
- Costs overview
- Company Skills management
- Company Export / Import
- Plugin Manager, Plugin pages, Plugin settings
- Auth (login), Board claim, Invite landing
- Inbox
- Instance settings (general + experimental)
- Design guide
- Org chart (full-page visualization)
- Run transcript viewer
- Execution workspace detail

### 20.2 Tech Stack
- React 19, Vite 6, React Router 7
- Radix UI + Tailwind CSS 4
- TanStack Query for data fetching
- WebSocket for live updates

---

## 21. CLI

- **`npx paperclipai onboard`** — guided setup wizard
- **`paperclipai run`** — start the server
- **`paperclipai run --watch`** — live-watch agent runs in terminal
- **`paperclipai configure`** — update server/secrets/storage config
- **`paperclipai doctor`** — validate configuration
- **`paperclipai allowed-hostname`** — whitelist Tailscale/private hostnames

---

## 22. Deployment & DevOps

- **Docker support** — Dockerfile + Docker Compose quickstart
- **Pre-installed CLIs in Docker** — Claude Code and Codex CLI included
- **Tailscale integration** — private access via Tailscale mesh network
- **Environment variables** — full reference for server, secrets, agent runtime, LLM keys
- **Local development** — `pnpm dev` with watch mode, separate server/UI dev
- **Monorepo** — pnpm workspaces (server, ui, packages/db, packages/shared, packages/adapters, packages/plugins, cli)
- **TypeScript throughout** — full type safety across all packages
- **Vitest** — test framework
- **Playwright** — E2E and release smoke tests
- **PromptFoo** — eval framework for agent prompt testing
- **Release scripts** — stable, canary, rollback, GitHub release
- **NPM publishing** — build and publish scripts

---

## 23. Finance & Billing

- **Finance events** — tracked in `finance_events` table
- **Billing codes** — optional billing code on issues for cost attribution
- **Budget incidents** — recorded when agents approach/exceed limits

---

## 24. Execution Workspaces

- **Workspace management** — track agent working directories and repo associations
- **Workspace operations** — operation log for workspace changes
- **Workspace runtime services** — manage runtime state of workspaces
- **Workspace policy** — enforcement rules for workspace access

---

## 25. Real-time & Live Events

- **WebSocket live events** — heartbeat run events, log streaming, status changes
- **SSE support** — server-sent events for standalone streaming
- **Live dashboard** — real-time updates without polling

---

## Summary Stats

- **~812 TypeScript files** (excluding node_modules)
- **~58 database tables**
- **9 built-in adapters**
- **Full plugin SDK with example plugins**
- **Comprehensive REST API** (13+ resource groups)
- **React UI with 38+ pages**
- **CLI with onboarding, configuration, and diagnostics**
