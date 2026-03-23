# Web UI — React 19, 38+ pages, real-time WebSocket updates

## Investigation Summary

Paperclip's web UI is a React 19 single-page application built with Vite 6, Tailwind CSS 4, shadcn/ui components (Radix UI primitives), TanStack Query for data fetching, and React Router 7 for navigation. The UI spans ~55,000 lines across 250 files, with 38 page components, 77 feature components, 21 shadcn/ui primitives, adapter-specific UI modules for 9 adapters, and a real-time update system via WebSocket/SSE. It provides a complete dashboard for managing autonomous AI companies.

---

## 1. Tech Stack

| Layer | Technology | Version |
|-------|-----------|---------|
| Framework | React | 19 |
| Build | Vite | 6 |
| Styling | Tailwind CSS | 4 |
| UI primitives | shadcn/ui (Radix UI) | — |
| Data fetching | TanStack Query | — |
| Routing | React Router | 7 |
| Real-time | WebSocket + SSE | — |

### 1.1 Codebase Size

- **250 TypeScript/TSX files**
- **~55,000 lines** of code
- **38 page components**
- **77 feature components**
- **21 shadcn/ui primitive components**
- **~1,600 lines** of API client code (40 API modules)
- **9 adapter-specific UI modules** (transcript parsers + config builders)

---

## 2. Page Inventory

### 2.1 Core Pages (38 total)

| Page | File | Lines | Description |
|------|------|-------|-------------|
| **AgentDetail** | `AgentDetail.tsx` | 3,988 | Tabbed agent view: overview, config, runs, issues, costs |
| **DesignGuide** | `DesignGuide.tsx` | 1,330 | Component showcase / design system reference |
| **CompanyImport** | `CompanyImport.tsx` | 1,295 | Import wizard with preview and collision handling |
| **IssueDetail** | `IssueDetail.tsx` | 1,183 | Issue detail with comments, documents, attachments |
| **CompanySkills** | `CompanySkills.tsx` | 1,170 | Skill management: list, import, scan, edit |
| **Costs** | `Costs.tsx` | 1,102 | Cost dashboard: by agent, provider, project, model |
| **RoutineDetail** | `RoutineDetail.tsx` | 1,020 | Routine config with triggers and run history |
| **CompanyExport** | `CompanyExport.tsx` | 919 | Export wizard with entity selection |
| **Inbox** | `Inbox.tsx` | 927 | Personal issue inbox with unread tracking |
| **PluginSettings** | `PluginSettings.tsx` | 836 | Plugin configuration UI |
| **CompanySettings** | `CompanySettings.tsx` | 661 | Company name, budget, branding, logo |
| **Routines** | `Routines.tsx` | 660 | Routine list with active issue badges |
| **ProjectDetail** | `ProjectDetail.tsx` | 632 | Project detail with workspaces |
| **Dashboard** | `Dashboard.tsx` | ~400 | Company health overview |
| **Agents** | `Agents.tsx` | ~300 | Agent list with org chart toggle |
| **Issues** | `Issues.tsx` | ~300 | Issue list with filters |
| **Goals** | `Goals.tsx` | ~200 | Goal hierarchy |
| **Projects** | `Projects.tsx` | ~200 | Project list |
| **Approvals** | `Approvals.tsx` | ~200 | Approval queue |
| **Activity** | `Activity.tsx` | ~200 | Activity feed |
| **OrgChart** | `OrgChart.tsx` | ~150 | Full-page org chart visualization |
| **NewAgent** | `NewAgent.tsx` | ~400 | Agent creation wizard |
| **Auth** | `Auth.tsx` | — | Login/signup |
| **BoardClaim** | `BoardClaim.tsx` | — | Board ownership claim |
| **InviteLanding** | `InviteLanding.tsx` | — | Invite acceptance |
| **InstanceSettings** | `InstanceSettings.tsx` | — | Instance-level settings |
| **PluginManager** | `PluginManager.tsx` | 509 | Plugin list + install |
| **PluginPage** | `PluginPage.tsx` | — | Plugin-rendered pages |
| Other pages | `MyIssues`, `GoalDetail`, `ApprovalDetail`, `Org`, `ExecutionWorkspaceDetail`, `RunTranscriptUxLab`, `NotFound` | — | — |

### 2.2 Largest Page: AgentDetail (3,988 lines)

The agent detail page is the most complex, with tabs for:
- **Overview** — agent identity, status, org position, session state
- **Configuration** — editable adapter config, heartbeat policy, runtime settings
- **Runs** — heartbeat run history with expandable detail + streaming log viewer
- **Issues** — assigned issues list
- **Costs** — per-agent cost breakdown with budget utilization

---

## 3. Component Architecture

### 3.1 shadcn/ui Primitives (21)

`avatar`, `badge`, `breadcrumb`, `button`, `card`, `checkbox`, `collapsible`, `command`, `dialog`, `dropdown-menu`, `input`, `label`, `popover`, `scroll-area`, `select`, `separator`, `sheet`, `skeleton`, `tabs`, `textarea`, `tooltip`

These are Radix UI primitives styled with Tailwind, following the shadcn/ui pattern.

### 3.2 Feature Components (77)

Domain-specific components including:
- `AgentConfigForm`, `AgentActionButtons`, `AgentProperties`, `AgentIconPicker`
- `CommentThread`, `ApprovalCard`, `ApprovalPayload`
- `BudgetPolicyCard`, `BudgetIncidentCard`, `BudgetSidebarMarker`
- `CompanyRail`, `CompanySwitcher`, `CompanyPatternIcon`
- `CommandPalette`, `FilterBar`, `EntityRow`, `EmptyState`
- `ActivityRow`, `ActivityCharts`
- `BillerSpendCard`, `ClaudeSubscriptionPanel`, `CodexSubscriptionPanel`
- `FinanceBillerCard`, `FinanceKindCard`, `AccountingModelCard`
- `DevRestartBanner`, `CopyText`

### 3.3 Adapter UI Modules (9)

Each adapter has UI-side modules for:
- **Transcript parser** (`parse-stdout.ts`) — converts agent stdout to structured transcript entries for the run viewer
- **Config builder** (`build-config.ts`) — converts form values to `adapterConfig` JSON
- **Config fields** — adapter-specific form fields in the agent creation/edit UI

Adapters with UI modules: `claude-local`, `codex-local`, `cursor`, `gemini-local`, `http`, `openclaw-gateway`, `opencode-local`, `pi-local`, `process`

---

## 4. Data Layer

### 4.1 API Client

40 API modules in `ui/src/api/`:
`access`, `activity`, `agents`, `approvals`, `assets`, `auth`, `budgets`, `client`, `companies`, `companySkills`, `costs`, `dashboard`, `execution-workspaces`, `goals`, `health`, `heartbeats`, `instanceSettings`, `issues`, `plugins`, `projects`, `routines`, `secrets`, `sidebarBadges`

Each module wraps `fetch` calls to the REST API with TypeScript types.

### 4.2 TanStack Query

56 TanStack Query imports across the codebase. Used for:
- Data fetching with caching
- Background refetching
- Optimistic updates
- Query invalidation on mutations
- Loading/error states

### 4.3 Query Key Organization

Centralized `queryKeys` object for cache management:
```typescript
queryKeys.agents.list(companyId)
queryKeys.issues.detail(issueId)
queryKeys.heartbeats.runs(companyId, agentId)
// etc.
```

---

## 5. Real-Time Updates

### 5.1 LiveUpdatesProvider

WebSocket/SSE connection for real-time events:

```typescript
// Events handled:
"heartbeat.run.queued"   → toast + invalidate run queries
"heartbeat.run.status"   → toast + invalidate run/agent queries
"heartbeat.run.event"    → append to run event log
"heartbeat.run.log"      → append to run log viewer
"agent.status"           → invalidate agent queries
"activity.logged"        → invalidate activity + sidebar badge queries
"plugin.ui.updated"      → invalidate plugin queries
```

### 5.2 Toast Notifications

Real-time events trigger toast notifications:
- Agent started/completed/failed runs
- Issue status changes
- Approval decisions
- Budget threshold alerts

Cooldown: max 3 toasts per 10-second window to prevent spam.

---

## 6. Context Providers

| Context | Purpose |
|---------|---------|
| `CompanyContext` | Current company selection |
| `BreadcrumbContext` | Dynamic breadcrumb trail |
| `DialogContext` | Modal/dialog state management |
| `PanelContext` | Side panel state |
| `SidebarContext` | Sidebar collapse/expand |
| `ThemeContext` | Dark/light theme |
| `ToastContext` | Toast notification queue |

---

## 7. Routing Structure

Company-scoped routes under `/:companyPrefix/`:
```
/:prefix/dashboard
/:prefix/agents
/:prefix/agents/:id
/:prefix/agents/new
/:prefix/issues
/:prefix/issues/:id
/:prefix/projects
/:prefix/projects/:id
/:prefix/goals
/:prefix/goals/:id
/:prefix/approvals
/:prefix/approvals/:id
/:prefix/routines
/:prefix/routines/:id
/:prefix/costs
/:prefix/activity
/:prefix/inbox
/:prefix/settings
/:prefix/skills
/:prefix/export
/:prefix/import
/:prefix/org
/:prefix/plugins
/:prefix/plugins/:pluginId
/:prefix/plugins/:pluginId/settings
```

Plus instance-level routes: `/settings`, `/auth`, `/board-claim/:token`, `/invite/:token`

---

## 8. Key UI Features

### 8.1 Run Viewer / Log Streaming

The run viewer (`AgentDetail.tsx` runs tab) is the most interactive component:
- Streaming stdout/stderr with syntax highlighting
- Structured transcript entries (assistant, thinking, tool calls, results, errors)
- Real-time append via WebSocket events
- Log pagination with "Load more" for large outputs
- Run status timeline with timestamps
- Token usage and cost breakdown per run

### 8.2 Command Palette

Global `Cmd+K` command palette for quick navigation between pages, agents, issues, projects.

### 8.3 Org Chart

Interactive org chart with tree layout, status badges, and click-to-navigate.

### 8.4 Cost Dashboard

Multi-view cost breakdown: by agent, by provider, by biller, by project, by agent-model. Includes rolling window spend (5h/24h/7d) and subscription vs API split.

### 8.5 Company Import/Export Wizards

Multi-step wizards with entity selection, preview, collision handling, and progress reporting.

---

## 9. Key Design Decisions

| Decision | Rationale |
|----------|-----------|
| React 19 + Vite 6 | Latest framework versions, fast build, HMR |
| shadcn/ui (not a component library) | Copy-paste components — full control, no version lock-in |
| TanStack Query (not SWR) | More features: mutations, prefetching, devtools |
| Company-scoped routing | URL structure mirrors data model; bookmarkable |
| Real-time via WebSocket (not polling) | Low latency, lower server load than polling |
| Toast cooldown | Prevents notification spam during rapid agent activity |
| Adapter-specific UI modules | Each agent runtime needs different config forms and transcript parsers |
| Design guide page | Living component reference for UI consistency |
| 40 API modules (not one big client) | Tree-shakable, domain-organized, clear ownership |

---

## 10. Observations for Colmena

1. **The UI is substantial** (~55,000 lines) but well-organized. For Colmena MVP, build ~30% of this: dashboard, agent list/detail, issue list/detail, basic costs.

2. **AgentDetail at 3,988 lines is too large.** Consider splitting into separate tab components from the start.

3. **shadcn/ui is the right approach** — copy-paste Radix primitives with Tailwind. No component library lock-in, full customization.

4. **TanStack Query + real-time WebSocket** is the right data pattern. Cache management + live updates cover all needs.

5. **The run viewer with streaming logs** is one of Paperclip's most compelling UI features. Prioritize this for Colmena — it's how operators monitor agent activity.

6. **Command palette** (`Cmd+K`) is a power-user feature worth adding early — it becomes the fastest way to navigate.

7. **For Colmena MVP UI**, prioritize: Dashboard, Agents (list + detail with run viewer), Issues (list + detail), Costs (basic summary). Add org chart, routines, skills, import/export later.

8. **The adapter UI registry** (transcript parsers + config builders per adapter) is essential if supporting multiple agent runtimes. Build it as part of the adapter system.

9. **The company-scoped routing** (`/:prefix/...`) is clean but adds complexity. For Colmena MVP with single-company focus, flat routes may suffice initially.

10. **The design guide page** is a great development aid — keeps UI consistent across the team. Consider building one from the start.
