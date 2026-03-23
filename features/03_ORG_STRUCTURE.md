# Org Structure — strict tree hierarchy, chain of command, delegation/escalation

## Investigation Summary

Paperclip models AI agent organizations as strict trees with a single root (CEO). The org structure is not a separate entity — it's encoded via a self-referencing FK on the `agents` table (`reportsTo`). This makes it lightweight but tightly coupled to agent management. The hierarchy drives delegation, escalation, permission scoping, and visualization.

---

## 1. Data Model

### 1.1 How the Hierarchy is Stored

There is **no separate org table**. The tree is encoded entirely in the `agents` table:

```sql
agents.reportsTo UUID REFERENCES agents(id)
```

- `reportsTo = NULL` → root node (CEO)
- `reportsTo = <uuid>` → reports to that agent
- Self-referencing FK with index `(companyId, reportsTo)` for fast subtree queries

### 1.2 Constraints (Application-Level, Not DB)

| Constraint | Enforcement | Location |
|-----------|------------|----------|
| No cycles | `assertNoCycle()` walks up the chain | `agents.ts` service |
| No self-reference | `agentId !== reportsTo` check | `agents.ts` service |
| Same company | Manager must belong to same company | `ensureManager()` |
| Single parent | Only one `reportsTo` per agent | Column is scalar, not array |
| Acyclic | Walk-up traversal with visited set + 50-node cap | `assertNoCycle()` |

**Cycle detection algorithm:**
```typescript
async function assertNoCycle(agentId: string, reportsTo: string | null) {
  if (!reportsTo) return;
  if (reportsTo === agentId) throw "Agent cannot report to itself";
  let cursor = reportsTo;
  while (cursor) {
    if (cursor === agentId) throw "Would create cycle";
    const next = await getById(cursor);
    cursor = next?.reportsTo ?? null;
  }
}
```

Simple walk-up with no depth limit other than a safeguard at 50 nodes (in `getChainOfCommand`). Not the most efficient for deep trees, but adequate for typical org sizes.

### 1.3 Orphan Handling

When an agent is **hard deleted**, its direct reports are reparented to NULL:
```typescript
await tx.update(agents).set({ reportsTo: null }).where(eq(agents.reportsTo, id));
```

This means deleted managers create root-level orphans rather than cascading deletions or reparenting to the grandparent. The board must manually reassign them.

When an agent is **terminated** (soft delete), `reportsTo` references are NOT updated — terminated agents remain in the tree but with `status: "terminated"`. The org chart query (`orgForCompany`) filters out terminated agents.

---

## 2. Chain of Command

### 2.1 Computation

The chain of command is computed on-demand (not stored):

```typescript
async function getChainOfCommand(agentId: string) {
  const chain = [];
  const visited = new Set([agentId]);
  const start = await getById(agentId);
  let currentId = start?.reportsTo ?? null;
  while (currentId && !visited.has(currentId) && chain.length < 50) {
    visited.add(currentId);
    const mgr = await getById(currentId);
    if (!mgr) break;
    chain.push({ id: mgr.id, name: mgr.name, role: mgr.role, title: mgr.title });
    currentId = mgr.reportsTo ?? null;
  }
  return chain;
}
```

Returns an ordered list from direct manager up to the CEO. Used in:
- `GET /api/agents/me` response (agents see their full reporting chain)
- Permission checks (ancestor managers can update instructions)
- The Paperclip skill teaches agents to escalate via `chainOfCommand`

### 2.2 Performance Characteristics

- One DB query per level of the hierarchy (N+1 pattern)
- Capped at 50 levels (prevents infinite loops from data corruption)
- Visited set prevents cycles even if DB data is inconsistent
- No caching — recomputed on every request

---

## 3. Org Tree Building

### 3.1 Algorithm

```typescript
async function orgForCompany(companyId: string) {
  // 1. Load all non-terminated agents
  const rows = await db.select().from(agents)
    .where(and(eq(agents.companyId, companyId), ne(agents.status, "terminated")));
  
  // 2. Group by manager
  const byManager = new Map<string | null, Agent[]>();
  for (const row of rows) {
    const key = row.reportsTo ?? null;
    byManager.get(key)?.push(row) ?? byManager.set(key, [row]);
  }
  
  // 3. Recursive build from roots (reportsTo = null)
  function build(managerId: string | null) {
    return (byManager.get(managerId) ?? []).map(member => ({
      ...member,
      reports: build(member.id)
    }));
  }
  return build(null);
}
```

**Key properties:**
- Single query to load all agents
- In-memory tree construction (efficient for typical org sizes)
- Returns forest (multiple roots) if there are multiple agents with `reportsTo = null`
- Terminated agents excluded from the tree

### 3.2 Response Format

The org tree is returned as a lean recursive structure:

```typescript
interface OrgNode {
  id: string;
  name: string;
  role: string;
  status: string;
  reports: OrgNode[];  // Direct reports, recursively nested
}
```

The `toLeanOrgNode()` function strips the full agent record down to just these 4 fields + recursive reports, keeping the response lightweight.

---

## 4. Org Chart Visualization

### 4.1 Server-Side SVG/PNG Rendering

Paperclip renders org charts **entirely server-side** — no browser, no Puppeteer, no external service.

**Endpoints:**
| Method | Path | Output |
|--------|------|--------|
| `GET` | `/api/companies/:companyId/org` | JSON tree |
| `GET` | `/api/companies/:companyId/org.svg?style=warmth` | SVG image |
| `GET` | `/api/companies/:companyId/org.png?style=warmth` | PNG image |

**5 visual styles:**
1. **Monochrome** — Vercel-inspired dark minimal
2. **Nebula** — Glassmorphism on cosmic gradient
3. **Circuit** — Linear/Raycast-inspired, indigo traces
4. **Warmth** (default) — Airbnb-inspired, light with colored avatars
5. **Schematic** — Blueprint style with grid background and monospace font

### 4.2 Layout Algorithm

Top-down centered tree layout:

1. **Measure** — calculate card width from text metrics (name/role length × font size × 0.58)
2. **Subtree width** — recursive: max of card width and sum of children's subtree widths + gaps
3. **Layout** — position each card centered within its subtree width, children spaced below
4. **Scale** — fit the entire tree into a fixed 1280×640 viewport (social media preview size)

**Constants:**
| Parameter | Value |
|-----------|-------|
| Card height | 96px |
| Card min width | 150px |
| Horizontal gap | 24px |
| Vertical gap | 56px |
| Padding | 48px |
| Target canvas | 1280×640 |

### 4.3 Role-Aware Visuals

Each role gets a distinct visual treatment:
- **Color-coded avatar backgrounds** — CEO gets warm amber, CTO gets blue, CMO gets green, etc.
- **Twemoji SVG icons** — Crown for CEO, Laptop for CTO, Globe for CMO, Bar chart for CFO, Keyboard for Engineer, etc.
- **Role labels** — "Chief Executive", "Technology", "Marketing", "Engineering", etc.
- **Role detection** — `guessRoleTag()` maps agent name/role strings to visual categories

The `ROLE_ICONS` map contains full inline Twemoji SVG paths (no external font/image dependencies), making the SVGs fully self-contained.

### 4.4 PNG Generation

PNG is rendered by:
1. Generate SVG (same as above)
2. Pass SVG buffer to `sharp` library
3. Render at 2× density (144 DPI) for retina quality
4. Resize to 1280×640

---

## 5. How the Org Structure is Used

### 5.1 Delegation

Managers create subtasks assigned to their direct reports:

```
POST /api/companies/{companyId}/issues
{
  "title": "...",
  "assigneeAgentId": "{reportAgentId}",
  "parentId": "{parentIssueId}",
  "goalId": "{goalId}"
}
```

**Rules:**
- Always set `parentId` to maintain task hierarchy
- Set `goalId` for goal alignment
- Set `billingCode` for cross-team work attribution

### 5.2 Escalation

When an agent is blocked, the heartbeat protocol dictates:
1. Update issue to `blocked` status with a comment explaining the blocker
2. Escalate via `chainOfCommand` — reassign to manager or create a task for them
3. Never cancel cross-team tasks — reassign to your manager instead

### 5.3 Permission Scoping

The org hierarchy is used for permission checks:

| Permission | Who has it |
|-----------|-----------|
| Update any agent config | Board, CEO, agents with `canCreateAgents` |
| Update agent instructions path | Board, self, or **any ancestor manager in chain of command** |
| Agent-to-agent communication | @-mentions in comments (triggers heartbeat for mentioned agent) |
| Task assignment | CEO, agents with `canCreateAgents`, agents with explicit `tasks:assign` grant |

The chain-of-command check for instructions management:
```typescript
const chainOfCommand = await svc.getChainOfCommand(targetAgent.id);
if (chainOfCommand.some(manager => manager.id === actorAgent.id)) return; // allowed
throw forbidden("Only the target agent or an ancestor manager can update instructions path");
```

### 5.4 Agent Identity

Every agent receives its `chainOfCommand` in the `GET /api/agents/me` response. The Paperclip skill instructs agents to:
- Know their place in the hierarchy
- Escalate to their chain of command when stuck
- Never cancel cross-team tasks (reassign to manager)
- Create subtasks only for their direct reports

### 5.5 Company Portability

The export/import system preserves org relationships via slugs:
- Each agent gets a `slug` (URL-safe identifier)
- `reportsToSlug` references the manager's slug
- On import, slugs are resolved to UUIDs in the target company
- If a manager slug doesn't exist in the target, the agent becomes a root node

---

## 6. Roles & Semantics

### 6.1 Predefined Roles

```typescript
const AGENT_ROLES = [
  "ceo", "cto", "cmo", "cfo",
  "engineer", "designer", "pm", "qa", "devops", "researcher",
  "general"
] as const;
```

Roles affect:
- **Default permissions** — CEO gets `canCreateAgents: true` by default; others get `false`
- **Default instructions** — CEO gets 4 files (AGENTS.md, HEARTBEAT.md, SOUL.md, TOOLS.md); others get 1 (AGENTS.md)
- **Visual treatment** — role-specific icons and colors in org chart
- **API branding access** — only CEO agents can update company branding
- **Company portability** — CEO gets import/export rights

### 6.2 CEO Special Powers

The CEO is not just a role label — it has structural implications:
- Root of the org tree (`reportsTo = null`)
- Default `canCreateAgents` permission
- Can update company branding (name, description, brandColor, logo)
- Can manage company import/export
- Can modify other agents' configs
- Gets richer default instructions bundle
- CEO strategy requires board approval before execution

---

## 7. Default Agent Instructions by Role

### 7.1 Bundle Structure

| Role | Files |
|------|-------|
| CEO | `AGENTS.md`, `HEARTBEAT.md`, `SOUL.md`, `TOOLS.md` |
| All others | `AGENTS.md` |

The CEO bundle includes:
- **HEARTBEAT.md** — heartbeat-specific checklist/protocol
- **SOUL.md** — persona, values, communication style
- **TOOLS.md** — local tool configuration notes

These files are materialized to disk in a managed directory when the agent is created, unless the user provides explicit instructions.

---

## 8. Technical Patterns

### 8.1 Tree-in-Table Pattern

The entire org structure is a **materialized path** pattern variant, using a self-referencing FK:
- **Pros**: Simple schema, easy to add/remove nodes, natural DB queries
- **Cons**: N+1 queries for chain-of-command, no built-in ancestor/descendant queries, cycle detection is application-level

### 8.2 Lean Projection

The org tree endpoint strips full agent records to `{id, name, role, status, reports[]}`. This keeps the response small and avoids leaking sensitive adapter config to the org chart UI.

### 8.3 Server-Side Rendering

The SVG rendering is entirely string-based (template literals building SVG XML). No DOM, no virtual DOM, no headless browser. This is:
- **Fast**: Sub-millisecond for typical org sizes
- **Portable**: Works anywhere Node.js runs
- **Self-contained**: All fonts, icons, and styles are inline

PNG rendering uses `sharp` (libvips-based), which is very fast for rasterization.

---

## 9. Key Design Decisions

| Decision | Rationale |
|----------|-----------|
| Self-referencing FK (not separate hierarchy table) | Simplicity — most orgs are small (< 50 agents). No need for closure tables or materialized paths |
| Application-level cycle detection | PostgreSQL doesn't natively enforce acyclic graphs in FK constraints. Walk-up with visited set is simple and sufficient |
| Orphan-to-root on manager delete | Safer than cascading delete (which would kill entire subtrees) or reparenting to grandparent (which may not be appropriate) |
| Chain of command computed on-demand | Avoids stale cached data. Performance is acceptable for typical org depths (< 10 levels) |
| Server-side SVG with 5 themes | No browser dependency. Self-contained output for dashboards, exports, social previews |
| Role-aware defaults (CEO gets richer bundle) | CEO needs more context (heartbeat checklist, soul, tools) to drive strategy. ICs need less |
| Roles as enum (not custom) | Predictable behavior — roles drive permissions, visuals, and defaults. Custom roles would need a mapping system |

---

## 10. Observations for Colmena

1. **The self-referencing FK approach is simple and effective** for typical org sizes (< 100 agents). For larger orgs, consider closure tables or nested sets for efficient subtree queries.

2. **Chain-of-command is N+1** — one query per level. For a 10-level org this means 10 sequential queries. Acceptable for now but could be a single recursive CTE:
   ```sql
   WITH RECURSIVE chain AS (
     SELECT id, name, role, title, reports_to FROM agents WHERE id = $1
     UNION ALL
     SELECT a.id, a.name, a.role, a.title, a.reports_to
     FROM agents a JOIN chain c ON a.id = c.reports_to
   ) SELECT * FROM chain;
   ```

3. **Orphan handling is minimal** — when a manager is deleted, reports become roots. A better UX might be to reparent to the grandparent or prompt the board to choose.

4. **The 5 SVG themes are a polish feature** — impressive but not essential for MVP. Start with one clean theme and add others later.

5. **Role-specific defaults are a great pattern** — CEO needs different context than an IC engineer. This could be even more granular (CTO gets architecture focus, CMO gets marketing focus, etc.).

6. **No multi-manager / matrix org support** — strictly one manager per agent. This is intentional and simplifies everything, but some companies use matrix structures. Colmena should decide early if this limitation is acceptable.

7. **The Paperclip skill teaches agents their org behavior** — delegation, escalation, cross-team rules. This is where the org structure comes alive at runtime, not just in the schema.

8. **Portability via slugs** is elegant — org relationships survive export/import across instances, even when UUIDs change.
