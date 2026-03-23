# Plugin System — SDK, workers, state store, event bus, webhooks, custom tools, UI components

## Investigation Summary

Paperclip has a full-featured plugin system that allows extending the platform with custom integrations, UI components, scheduled jobs, webhooks, agent tools, and event handlers. Plugins run as isolated worker processes communicating with the host via JSON-RPC 2.0 over stdio. The system is the most code-intensive feature in Paperclip — **~10,800 lines** across 21 server-side service files, plus the SDK package, protocol layer, UI runtime, and 4 example plugins. The plugin system has its own capability model, lifecycle management, state storage, job scheduling, and UI extension points.

---

## 1. Architecture

### 1.1 Process Model

```
┌─────────────────────────────────┐
│  Paperclip Host (Express)       │
│  ┌───────────────────────────┐  │
│  │  Plugin Registry          │  │
│  │  Plugin Loader            │  │
│  │  Plugin Worker Manager    │  │
│  │  Plugin Host Services     │  │
│  └───────────┬───────────────┘  │
│              │ JSON-RPC 2.0     │
│              │ (stdio)          │
│  ┌───────────┴───────────────┐  │
│  │  Worker Process 1         │  │
│  │  (plugin-linear-sync)     │  │
│  └───────────────────────────┘  │
│  ┌───────────────────────────┐  │
│  │  Worker Process 2         │  │
│  │  (plugin-github-pr)       │  │
│  └───────────────────────────┘  │
└─────────────────────────────────┘
```

- Each plugin runs in its own **child process** (isolation)
- Communication via **JSON-RPC 2.0** over **stdin/stdout**
- Host manages worker lifecycle: start, health check, config change, shutdown, restart, crash recovery

### 1.2 Host-Side Components (21 service files, ~10,800 lines)

| Service | Lines | Purpose |
|---------|-------|---------|
| `plugin-loader` | 1,954 | Load, validate, install, upgrade plugins from filesystem/npm |
| `plugin-worker-manager` | 1,342 | Spawn/monitor/restart worker processes |
| `plugin-host-services` | 1,131 | Implement all host→worker API calls (data reads, state, secrets) |
| `plugin-lifecycle` | 821 | Plugin start/stop/enable/disable state machine |
| `plugin-job-scheduler` | 752 | Cron-based job scheduling for plugins |
| `plugin-registry` | 682 | Plugin metadata storage and lookup |
| `plugin-job-store` | 465 | Persistent job run storage |
| `plugin-capability-validator` | 449 | Enforce capability permissions per RPC call |
| `plugin-tool-registry` | 449 | Register and manage agent tools from plugins |
| `plugin-tool-dispatcher` | 448 | Route tool calls from agents to plugin workers |
| `plugin-event-bus` | 412 | Forward domain events to subscribed plugins |
| `plugin-dev-watcher` | 339 | Hot-reload during plugin development |
| `plugin-job-coordinator` | 260 | Coordinate job execution across workers |
| `plugin-state-store` | 237 | Persistent key-value state per plugin |
| `plugin-runtime-sandbox` | 221 | Process-level isolation and resource limits |
| `plugin-secrets-handler` | 354 | Resolve secret references for plugins |
| `plugin-manifest-validator` | 163 | Validate plugin manifest against schema |
| `plugin-config-validator` | 54 | Validate plugin config against JSON schema |
| `plugin-stream-bus` | 81 | Real-time streaming within plugins |
| `plugin-log-retention` | 86 | Log management and cleanup |
| `plugin-host-service-cleanup` | 59 | Cleanup on plugin uninstall |

---

## 2. Plugin Definition (SDK)

### 2.1 `definePlugin()`

Plugin authors use the SDK to define their plugin:

```typescript
import { definePlugin, runWorker } from "@paperclipai/plugin-sdk";

export default definePlugin({
  async setup(ctx) {
    // Register event handlers
    ctx.events.on("issue.created", async (event) => { ... });
    
    // Register job handlers
    ctx.jobs.register("full-sync", async (job) => { ... });
    
    // Register data providers (for UI)
    ctx.data.register("sync-health", async (params) => { ... });
    
    // Register actions (for UI)
    ctx.actions.register("resync", async (params) => { ... });
    
    // Register agent tools
    ctx.tools.register("search-linear", async (input, runCtx) => {
      return { content: "..." };
    });
  },
  
  async onHealth() {
    return { status: "ok", message: "Plugin ready" };
  },
  
  async onConfigChanged(newConfig) { ... },
  async onShutdown() { ... },
  async onValidateConfig(config) { ... },
  async onWebhook(input) { ... },
});
```

### 2.2 Plugin Context (`ctx`)

The `PluginContext` provides these API surfaces:

| Client | Capability | Description |
|--------|-----------|-------------|
| `ctx.config` | — | Read resolved plugin configuration |
| `ctx.events` | `events.subscribe`, `events.emit` | Subscribe to and emit domain events |
| `ctx.jobs` | `jobs.schedule` | Register scheduled job handlers |
| `ctx.state` | `plugin.state.read`, `plugin.state.write` | Scoped key-value state storage |
| `ctx.entities` | — | CRUD for plugin-owned entity records |
| `ctx.secrets` | `secrets.read-ref` | Resolve secret references |
| `ctx.http` | `http.outbound` | Make outbound HTTP requests |
| `ctx.activity` | `activity.log.write` | Write activity log entries |
| `ctx.data` | — | Register data providers for UI components |
| `ctx.actions` | — | Register action handlers for UI components |
| `ctx.tools` | `agent.tools.register` | Register agent-callable tools |
| `ctx.launchers` | — | Register UI launchers at runtime |
| `ctx.logger` | — | Structured logging |
| `ctx.issues` | `issues.read`, `issues.create`, `issues.update` | Issue CRUD |
| `ctx.agents` | `agents.read`, `agents.pause`, `agents.invoke` | Agent operations |
| `ctx.projects` | `projects.read`, `project.workspaces.read` | Project and workspace access |
| `ctx.goals` | `goals.read`, `goals.create`, `goals.update` | Goal operations |
| `ctx.companies` | `companies.read` | Company data access |

---

## 3. Capability Model

Plugins declare required capabilities in their manifest. The host enforces them at runtime.

### 3.1 Capability Categories (42 total)

**Data Read:** `companies.read`, `projects.read`, `project.workspaces.read`, `issues.read`, `issue.comments.read`, `issue.documents.read`, `agents.read`, `goals.read`, `activity.read`, `costs.read`

**Data Write:** `issues.create`, `issues.update`, `issue.comments.create`, `issue.documents.write`, `agents.pause`, `agents.resume`, `agents.invoke`, `goals.create`, `goals.update`, `agent.sessions.*`, `activity.log.write`, `metrics.write`

**Plugin State:** `plugin.state.read`, `plugin.state.write`

**Runtime:** `events.subscribe`, `events.emit`, `jobs.schedule`, `webhooks.receive`, `http.outbound`, `secrets.read-ref`

**Agent Tools:** `agent.tools.register`

**UI:** `instance.settings.register`, `ui.sidebar.register`, `ui.page.register`, `ui.detailTab.register`, `ui.dashboardWidget.register`, `ui.commentAnnotation.register`, `ui.action.register`

### 3.2 Capability Escalation Detection

On plugin upgrade, if the new manifest declares capabilities not present in the old one, the host flags it as a **capability escalation** requiring operator approval.

---

## 4. Plugin State

### 4.1 State Store

Scoped key-value storage with 5-part composite key:
```
(pluginId, scopeKind, scopeId, namespace, stateKey) → JSON value
```

**Scope kinds:** `instance`, `company`, `project`, `project_workspace`, `agent`, `issue`, `goal`, `run`

Stored in `plugin_state` table (see DB schema files). Isolated per plugin — plugin A cannot access plugin B's state.

### 4.2 Entity Store

Plugins can create custom entity records:
```typescript
await ctx.entities.upsert({
  entityType: "linear-issue",
  scopeKind: "issue",
  scopeId: issueId,
  externalId: linearIssueId,
  title: "Linear issue title",
  status: "in_progress",
  data: { ... }
});
```

Stored in `plugin_entities` table. Queryable by type, scope, external ID.

---

## 5. Event System

### 5.1 Domain Events

25+ event types forwarded from the activity log:
```
company.created/updated, project.created/updated, project.workspace_*,
issue.created/updated, issue.comment.created, agent.created/updated/status_changed,
agent.run.started/finished/failed/cancelled, goal.created/updated,
approval.created/decided, cost_event.created, activity.logged
```

### 5.2 Plugin-to-Plugin Events

Plugins can emit custom events namespaced by plugin ID:
```typescript
ctx.events.emit("sync-done", companyId, { count: 42 });
// Other plugins subscribe via: ctx.events.on("plugin.acme.linear.sync-done", ...)
```

### 5.3 Server-Side Filtering

Event subscriptions support server-side filters (projectId, companyId, agentId) so filtered events never cross the process boundary.

---

## 6. Job Scheduling

Plugins declare jobs in their manifest with cron schedules:
```typescript
ctx.jobs.register("full-sync", async (job) => {
  // job.jobKey, job.runId, job.trigger, job.scheduledAt
});
```

Host-side: `plugin-job-scheduler` evaluates cron expressions, `plugin-job-coordinator` manages execution, `plugin-job-store` persists run history.

Job statuses: `active`, `paused`, `failed`. Run statuses: `pending`, `queued`, `running`, `succeeded`, `failed`, `cancelled`. Triggers: `schedule`, `manual`, `retry`.

---

## 7. Webhooks

Plugins receive inbound webhooks at:
```
POST /api/plugins/:pluginId/webhooks/:endpointKey
```

The host delivers to the worker's `onWebhook()` handler with headers, raw body, parsed body, and request ID. Plugin is responsible for signature verification using `ctx.secrets.resolve()`.

Webhook delivery statuses: `pending`, `success`, `failed`.

---

## 8. Agent Tools

Plugins can register tools that agents can invoke during heartbeats:

```typescript
ctx.tools.register("search-linear", async (input, runCtx) => {
  // runCtx: { agentId, runId, companyId, projectId }
  return { content: "Search results...", data: { ... } };
});
```

Host-side: `plugin-tool-registry` manages declarations, `plugin-tool-dispatcher` routes calls from agents to workers.

---

## 9. UI Extension Points

### 9.1 Slot Types (13)

`page`, `detailTab`, `taskDetailView`, `dashboardWidget`, `sidebar`, `sidebarPanel`, `projectSidebarItem`, `globalToolbarButton`, `toolbarButton`, `contextMenuItem`, `commentAnnotation`, `commentContextMenuItem`, `settingsPage`

### 9.2 Plugin UI Runtime

- Plugin UI components are React components bundled separately
- Served as static assets via `plugin-ui-static` route
- Communicate with worker via host bridge (RPC through `usePluginData` / `usePluginAction` hooks)
- SDK provides React hooks and components: `usePluginData`, `usePluginAction`, `PluginFrame`

### 9.3 Launcher System

Plugins can register "launchers" — UI affordances (buttons, menu items) that trigger actions:
- Placement zones map to slot types
- Actions: `navigate`, `openModal`, `openDrawer`, `openPopover`, `performAction`, `deepLink`
- Render environments: `hostInline`, `hostOverlay`, `hostRoute`, `external`, `iframe`
- Size bounds: `inline`, `compact`, `default`, `wide`, `full`

---

## 10. Plugin Lifecycle

```
installed → ready | error
ready → disabled | error | upgrade_pending | uninstalled
disabled → ready | uninstalled
error → ready | uninstalled
upgrade_pending → ready | error | uninstalled
uninstalled → installed (reinstall)
```

**Lifecycle operations:**
- **Install** — load manifest, validate, create DB records, spawn worker
- **Enable/Disable** — toggle without uninstalling
- **Upgrade** — detect capability escalation, apply new manifest
- **Uninstall** — stop worker, clean up state/jobs/webhooks/entities

---

## 11. Worker Management

- **Spawn** — fork child process with `NODE_OPTIONS`, `PAPERCLIP_PLUGIN_*` env vars
- **Health probes** — periodic `health` RPC calls
- **Crash recovery** — auto-restart on unexpected exit (with backoff)
- **Graceful shutdown** — `shutdown` RPC with 10s timeout → SIGTERM → SIGKILL
- **Hot-reload** — `plugin-dev-watcher` watches filesystem changes during development
- **Dev server** — local dev server for plugin UI development

---

## 12. Example Plugins

| Plugin | Purpose |
|--------|---------|
| `plugin-hello-world-example` | Minimal reference (setup + health) |
| `plugin-kitchen-sink-example` | Full demo of all features (events, jobs, state, tools, webhooks, UI, launchers, processes, filesystem) |
| `plugin-file-browser-example` | File browser UI component |
| `plugin-authoring-smoke-example` | Vitest-based authoring smoke test |

---

## 13. Key Design Decisions

| Decision | Rationale |
|----------|-----------|
| Isolated child processes | Security and stability — plugin crash doesn't take down the host |
| JSON-RPC 2.0 over stdio | Standard protocol, language-agnostic, easy to debug |
| Capability-based security | Fine-grained permissions — plugins only access what they need |
| Scoped state store | Plugin isolation + flexible scoping (instance → company → project → issue → run) |
| Manifest-declared capabilities | Explicit permissions visible to operators before enabling a plugin |
| Capability escalation detection | Prevents silent permission expansion on upgrade |
| Worker health probes | Detect stuck/unresponsive workers without polling all state |
| 42 capabilities in 6 categories | Comprehensive but granular — plugins can request minimal permissions |
| UI slots with launcher system | Plugins can extend the UI at predefined extension points without forking the host UI |

---

## 14. Observations for Colmena

1. **The plugin system is a platform within a platform** — ~10,800 lines of server code alone. This is a **significant investment** and should only be built if Colmena aims to be an extensible platform with third-party integrations.

2. **For Colmena MVP, skip the plugin system entirely.** Build integrations as first-class features instead. Add a plugin system later when the core is stable and there's demand for third-party extensions.

3. **If a plugin system is needed later**, the JSON-RPC 2.0 over stdio pattern is well-proven. The capability model is a good security foundation. But consider whether plugins need full process isolation or if a simpler in-process hook system would suffice.

4. **The event bus pattern** (domain events forwarded to subscribers) is valuable independently of the plugin system. Consider implementing it as a standalone feature for webhooks and real-time updates.

5. **Agent tools from plugins** is a powerful extensibility mechanism — allows third-party integrations (Linear, GitHub, Jira) to expose tools agents can use during heartbeats.

6. **The UI extension slot system** is impressive but complex. For Colmena, a simpler approach (iframe-based plugin UIs at predefined mount points) might suffice initially.

7. **State scoping** (5-part key: plugin + scopeKind + scopeId + namespace + stateKey) is well-designed for multi-tenant isolation. Worth studying even if not implementing plugins immediately.

8. **The `create-paperclip-plugin` scaffolding tool** and dev server/watcher show maturity — the plugin authoring DX is polished.

9. **Capability escalation detection** on upgrade is a thoughtful security feature — prevents a plugin update from silently gaining new permissions.

10. **The kitchen-sink example** is an excellent reference — demonstrates every feature in one plugin. Worth maintaining an equivalent if Colmena builds a plugin system.
