# Company Portability — export/import with secret scrubbing, collision handling

## Investigation Summary

Paperclip supports exporting entire companies as self-contained packages and importing them into other instances. This enables sharing company templates (org structures, agent configs, skills, projects, issues), backing up company configurations, and the planned ClipHub marketplace. The system is one of the largest services in the codebase (~3,300 lines) and handles secret scrubbing, collision strategies, import previews, agent instructions bundling, org chart rendering, binary asset support, and two import modes (board full access vs agent-safe restricted).

---

## 1. Package Format

### 1.1 Manifest Structure

The export produces a JSON manifest + file tree:

```typescript
interface CompanyPortabilityManifest {
  version: 1;
  exportedAt: string;
  source: {
    companyId: string;
    companyName: string;
    instanceId: string;
    paperclipVersion: string;
    bundledSkillsCommit: string | null;
  };
  company: {
    path: string;           // "COMPANY.md"
    name: string;
    description: string | null;
    brandColor: string | null;
    logoPath: string | null;
    requireBoardApprovalForNewAgents: boolean;
  };
  agents: Array<{
    slug: string;
    name: string;
    path: string;           // "agents/{slug}/AGENTS.md"
    skills: string[];       // Desired skill keys
    role: string;
    title: string | null;
    icon: string | null;
    capabilities: string | null;
    reportsToSlug: string | null;
    adapterType: string;
    adapterConfig: Record<string, unknown>;  // Secrets scrubbed
    runtimeConfig: Record<string, unknown>;
    permissions: Record<string, unknown>;
    budgetMonthlyCents: number;
    metadata: Record<string, unknown> | null;
  }>;
  skills: Array<{
    key: string;
    slug: string;
    name: string;
    path: string;           // "skills/{slug}/SKILL.md"
    description: string | null;
    sourceType: string;
    sourceLocator: string | null;
    sourceRef: string | null;
    trustLevel: string | null;
    compatibility: string | null;
    metadata: Record<string, unknown> | null;
    fileInventory: Array<{ path: string; kind: string }>;
  }>;
  projects: Array<{
    slug: string;
    name: string;
    path: string;           // "projects/{slug}/PROJECT.md"
    description: string | null;
    status: string;
    goalIds: string[];
    leadAgentSlug: string | null;
    workspaces: Array<{
      name: string;
      cwd: string | null;
      repoUrl: string | null;
      repoRef: string | null;
      isPrimary: boolean;
    }>;
  }>;
  issues: Array<{
    slug: string;
    identifier: string;
    title: string;
    path: string;           // "tasks/{slug}.md"
    description: string | null;
    status: string;
    priority: string;
    assigneeAgentSlug: string | null;
    projectSlug: string | null;
    parentSlug: string | null;
    billingCode: string | null;
  }>;
  envInputs: Array<{
    key: string;
    description: string | null;
    agentSlug: string | null;
    kind: "secret" | "plain";
    requirement: "required" | "optional";
    defaultValue: string | null;
    portability: "portable" | "system_dependent";
  }>;
}
```

### 1.2 File Entries

Files can be text or binary:
```typescript
type FileEntry = string | { encoding: "base64"; data: string; contentType?: string };
```

File kinds: `company`, `agent`, `skill`, `project`, `issue`, `extension`, `readme`, `other`.

### 1.3 Generated Files

- **`README.md`** — auto-generated with company name, description, Mermaid org chart, agent table, skill list, project list
- **`COMPANY.md`** — company description as markdown
- **`agents/{slug}/AGENTS.md`** — agent instructions (from managed bundle or promptTemplate)
- **`skills/{slug}/SKILL.md`** — skill files (+ references, scripts, assets)
- **`projects/{slug}/PROJECT.md`** — project description
- **`tasks/{slug}.md`** — issue description
- **Org chart PNG** — rendered as `.assets/org-chart.png` using the server-side SVG renderer

---

## 2. Export Flow

### 2.1 What Gets Exported

Configurable via `include`:
```typescript
{
  company: boolean,    // Company metadata + logo
  agents: boolean,     // Agents + their instructions bundles
  projects: boolean,   // Projects + workspaces
  issues: boolean,     // Issues/tasks
  skills: boolean      // Company skills
}
```

Can also select specific agents, projects, issues, skills by reference.

### 2.2 Secret Scrubbing

On export, agent adapter configs are **sanitized**:
- `secret_ref` bindings → replaced with `envInputs` entries
- Inline API keys/tokens → detected by sensitive key pattern, replaced with placeholder
- The `envInputs` array in the manifest declares what secrets the importer needs to provide
- Each entry specifies: key name, description, which agent uses it, kind (secret/plain), requirement (required/optional), portability (portable/system-dependent)

### 2.3 Org Relationships via Slugs

- Each agent gets a unique slug derived from its name
- `reportsToSlug` references the manager's slug (not UUID)
- On import, slugs are resolved to UUIDs in the target company
- Orphan slugs (manager doesn't exist) → agent becomes a root node

### 2.4 Export Preview

`previewExport()` returns what would be exported without actually generating:
- File inventory with kinds and sizes
- Agent count, skill count, project count
- Warnings about skipped entities

---

## 3. Import Flow

### 3.1 Import Targets

```typescript
{ mode: "new_company", newCompanyName?: string }  // Create fresh company
{ mode: "existing_company", companyId: string }    // Merge into existing
```

### 3.2 Import Preview

`previewImport()` dry-runs the import and returns a plan:
```typescript
{
  plan: {
    companyAction: "create" | "update" | "skip",
    agentPlans: Array<{
      slug: string;
      action: "create" | "update" | "skip";
      plannedName: string;
      existingAgentId: string | null;
      reason: string | null;
    }>,
    projectPlans: [...],
    issuePlans: [...],
    skillPlans: [...]
  },
  warnings: string[],
  errors: string[]
}
```

### 3.3 Collision Strategies

```typescript
type CollisionStrategy = "rename" | "skip" | "replace";
```

| Strategy | Behavior |
|----------|----------|
| `rename` (default) | Append suffix to conflicting names (e.g., "Engineer 2") |
| `skip` | Keep existing entity, skip the import |
| `replace` | Overwrite existing entity with imported data |

### 3.4 Import Modes

| Mode | Who | Restrictions |
|------|-----|-------------|
| `board_full` | Board operators | Full access — can create, update, replace |
| `agent_safe` | CEO agents | Restricted — can only create or skip, no replace, must target own company |

### 3.5 Import Side Effects

On import:
1. Create or update company (name, description, brandColor, logo)
2. Import skills (via `companySkills.importPackageFiles`)
3. Create agents with instructions bundles, adapter configs, reporting chains
4. Resolve `reportsToSlug` → set `reportsTo` FK
5. Create projects with workspaces
6. Create issues with parent/project/assignee references
7. Copy user memberships (for agent-safe new company imports)
8. Log `company.imported` activity

---

## 4. Instructions Bundle Handling

### 4.1 Export

For each agent:
1. If managed instructions bundle exists → export all files under `agents/{slug}/`
2. If only `promptTemplate` → export as `agents/{slug}/AGENTS.md`
3. Additional bundle files (HEARTBEAT.md, SOUL.md, TOOLS.md, etc.) included

### 4.2 Import

For each agent:
1. Look for `agents/{slug}/AGENTS.md` in package files
2. Fall back to `promptTemplate` from manifest adapterConfig
3. Materialize as managed instructions bundle on the created agent
4. Clear legacy `promptTemplate` field

---

## 5. Env Inputs (Secret Placeholders)

The `envInputs` array tells the importer what secrets are needed:
```typescript
{
  key: "ANTHROPIC_API_KEY",
  description: "Anthropic API key for Claude agents",
  agentSlug: "cto",              // Which agent needs this
  kind: "secret",                // Must be encrypted
  requirement: "required",
  defaultValue: null,
  portability: "system_dependent" // Can't be copied between systems
}
```

The import UI can present these as a form for the operator to fill in before completing the import.

---

## 6. README Generation

Auto-generated `README.md` includes:
- Company name and description
- **Mermaid org chart** (flowchart TD with role labels and reporting edges)
- Agent table (name, role, adapter type, budget)
- Skills list with source badges (GitHub links, local paths)
- Projects list with descriptions
- Import instructions
- Env input table (required secrets)

---

## 7. API Surface

| Method | Path | Description |
|--------|------|-------------|
| `POST` | `/api/companies/:companyId/export` | Export company package |
| `POST` | `/api/companies/:companyId/exports/preview` | Preview what would be exported |
| `POST` | `/api/companies/:companyId/exports` | Export (CEO-accessible) |
| `POST` | `/api/companies/import/preview` | Preview import |
| `POST` | `/api/companies/import` | Import package (board only) |
| `POST` | `/api/companies/:companyId/imports/preview` | Preview import (agent-safe) |
| `POST` | `/api/companies/:companyId/imports/apply` | Apply import (agent-safe) |

---

## 8. Key Design Decisions

| Decision | Rationale |
|----------|-----------|
| Slug-based references (not UUIDs) | Packages must be portable across instances where UUIDs differ |
| Secret scrubbing with envInputs | Never export plaintext secrets; declare what's needed for import |
| Collision strategies | Different use cases: rename for merging, skip for idempotent imports, replace for updates |
| Two import modes (board/agent-safe) | Agents can import templates but shouldn't be able to overwrite existing agents |
| Instructions as files in package | Bundles with multiple files (AGENTS.md, HEARTBEAT.md, etc.) must be portable |
| Auto-generated README with Mermaid | Human-readable package overview; Mermaid renders on GitHub |
| Preview before import | No surprises — operators see exactly what will happen |
| Binary file support (base64 encoding) | Company logos and other assets can be included |
| Org chart PNG in exports | Visual preview without rendering Mermaid |

---

## 9. Observations for Colmena

1. **Company portability is the foundation for a marketplace** (ClipHub). If Colmena plans a marketplace, invest in this system early. If not, it can be deferred.

2. **The secret scrubbing + envInputs pattern is excellent** — packages are safe to share publicly; operators provide their own secrets on import.

3. **Slug-based references** are the right approach for portable packages. UUIDs would break across instances.

4. **The preview system** is essential UX — importing a company without knowing what will happen is scary. The preview + collision plan gives operators confidence.

5. **For Colmena MVP, skip portability entirely.** It's ~3,300 lines of complex code that only matters when you have multiple instances or want to share templates.

6. **When implementing later**, start with export-only (backup/sharing). Add import when there's a real use case for importing templates.

7. **The README with Mermaid org chart** is a great touch — makes packages self-documenting and renders beautifully on GitHub.

8. **Agent-safe import mode** is forward-thinking — allows CEO agents to import templates autonomously without full admin access. This is governance in action.
