# Skills System — on-demand loading, per-company skills, adapter-specific injection

## Investigation Summary

Skills are reusable instruction sets that teach agents how to perform specific tasks. They are markdown-first files with YAML frontmatter, organized in directories with optional `references/`, `scripts/`, and `assets/` subdirectories. Paperclip's skills system is far more than a file format — it includes a full company-scoped registry with import from GitHub/URLs/local paths/skills.sh, per-agent skill assignment, adapter-specific injection, runtime materialization, file inventory tracking, trust levels, update detection, project workspace scanning, and a UI for managing skills. The service is **~1,800 lines** (one of the largest in the codebase) backed by a single database table.

---

## 1. Data Model

### 1.1 `company_skills` Table

| Column | Type | Default | Notes |
|--------|------|---------|-------|
| `id` | UUID | random | PK |
| `companyId` | UUID | — | FK → companies |
| `key` | TEXT | — | Canonical key (globally unique within company). Format: `owner/repo/slug` or `company/{companyId}/slug` |
| `slug` | TEXT | — | URL-safe short name |
| `name` | TEXT | — | Human-readable name (from frontmatter) |
| `description` | TEXT | NULL | Routing description (from frontmatter) |
| `markdown` | TEXT | — | Full SKILL.md content |
| `sourceType` | TEXT | `"local_path"` | Enum: `local_path`, `github`, `url`, `catalog`, `skills_sh` |
| `sourceLocator` | TEXT | NULL | URL, path, or registry reference |
| `sourceRef` | TEXT | NULL | Git commit SHA (for GitHub sources) |
| `trustLevel` | TEXT | `"markdown_only"` | `markdown_only`, `assets`, `scripts_executables` |
| `compatibility` | TEXT | `"compatible"` | — |
| `fileInventory` | JSONB | `[]` | Array of `{ path, kind }` entries |
| `metadata` | JSONB | NULL | Source metadata (owner, repo, ref, trackingRef, sourceKind, etc.) |
| `createdAt`, `updatedAt` | TIMESTAMPTZ | — | — |

**Indexes:**
- Unique on `(companyId, key)` — one skill per canonical key per company
- Index on `(companyId, name)` — search by name

---

## 2. Skill Format

### 2.1 SKILL.md Structure

```markdown
---
name: my-skill
description: >
  Short routing description. Agents read this to decide
  whether to load the full skill content.
---

# My Skill

Detailed instructions for the agent...
```

### 2.2 Directory Layout

```
skills/my-skill/
├── SKILL.md          # Main skill document (required)
├── references/       # Supporting reference docs
│   └── api.md
├── scripts/          # Executable scripts
│   └── deploy.sh
└── assets/           # Images, configs, etc.
    └── diagram.png
```

### 2.3 File Inventory

Every skill tracks its file inventory:
```typescript
{
  path: "SKILL.md",        kind: "skill"
  path: "references/api.md", kind: "reference"
  path: "scripts/deploy.sh", kind: "script"
  path: "assets/logo.png",  kind: "asset"
}
```

Kind classification: `skill` (SKILL.md), `reference` (.md in references/), `script` (shell/JS/Python), `asset` (images/PDFs), `markdown` (other .md), `other`.

### 2.4 Trust Levels

Derived from file inventory:
| Level | Meaning |
|-------|---------|
| `markdown_only` | Only SKILL.md and reference markdown files |
| `assets` | Includes non-markdown assets (images, configs) |
| `scripts_executables` | Includes executable scripts |

---

## 3. Skill Sources

### 3.1 Source Types

| Source | How it works |
|--------|-------------|
| **Local path** | Read from filesystem (`/path/to/skill/SKILL.md`) |
| **GitHub** | Fetch from GitHub repo via API (tree listing + raw content) |
| **URL** | Fetch single SKILL.md from any HTTP URL |
| **skills.sh** | Registry shorthand: `owner/repo/skill` → resolved to GitHub |
| **Catalog** | Inline files from import packages (materialized to managed directory) |
| **Bundled** | Built-in Paperclip skills (e.g., heartbeat protocol) |

### 3.2 Canonical Key Derivation

Each skill gets a canonical key based on its source:
- **Bundled**: `paperclipai/paperclip/{slug}`
- **GitHub**: `{owner}/{repo}/{slug}`
- **skills.sh**: same as GitHub after resolution
- **URL**: `url/{host}/{hash}/{slug}`
- **Local managed**: `company/{companyId}/{slug}`
- **Local external**: `local/{pathHash}/{slug}`

### 3.3 Import Sources

The `parseSkillImportSourceInput()` function handles multiple input formats:
- `owner/repo/skill` → skills.sh key-style import
- `owner/repo` → GitHub repo (all skills)
- `https://github.com/owner/repo/...` → GitHub URL
- `https://skills.sh/owner/repo/skill` → skills.sh URL
- `npx skills add owner/repo/skill --skill slug` → command-line paste
- `https://example.com/SKILL.md` → raw URL
- `/local/path/to/skills/` → local filesystem

---

## 4. Import & Update Flow

### 4.1 GitHub Import

1. Parse GitHub URL → extract owner, repo, ref, basePath
2. Resolve pinned commit SHA (for reproducibility)
3. Fetch git tree via GitHub API (`/git/trees/{ref}?recursive=1`)
4. Find all `SKILL.md` files in the tree
5. Fetch each SKILL.md content from raw GitHub
6. Parse frontmatter for name, description, slug
7. Build file inventory from tree paths
8. Upsert into `company_skills` table

### 4.2 Update Detection

For GitHub-sourced skills:
1. Read `trackingRef` from metadata (e.g., `main`)
2. Resolve latest commit SHA for that ref
3. Compare with stored `sourceRef`
4. Report: `{ hasUpdate, currentRef, latestRef }`

`installUpdate()` re-imports the skill from its source URL.

### 4.3 Project Workspace Scanning

`scanProjectWorkspaces()` discovers skills in project workspaces:

1. For each project workspace with a local CWD:
2. Check 30+ skill directory roots: `skills/`, `.claude/skills/`, `.agents/skills/`, `.pi/skills/`, etc.
3. For each discovered `SKILL.md`:
4. Import as a `local_path` skill with `sourceKind: "project_scan"` metadata
5. Handle conflicts (slug collision, key collision)
6. Return: `{ discovered, imported, updated, skipped, conflicts, warnings }`

---

## 5. Agent Skill Assignment

### 5.1 Per-Agent Desired Skills

Each agent's `adapterConfig` contains a skill preference list:
```typescript
adapterConfig.paperclipSkillSyncDesiredSkills: ["paperclipai/paperclip/paperclip", "my-company/my-skill"]
```

Written/read via `writePaperclipSkillSyncPreference()` / `readPaperclipSkillSyncPreference()`.

### 5.2 Required Skills

Bundled Paperclip skills (like the heartbeat protocol) are always marked as `required`:
```typescript
{ key: "paperclipai/paperclip/paperclip", required: true, requiredReason: "Bundled Paperclip skills are always available..." }
```

### 5.3 Skill Sync

`POST /api/agents/:id/skills/sync` syncs desired skills to an agent:
1. Resolve requested skill references to canonical keys
2. Merge with required skills
3. Write to `adapterConfig`
4. For adapters that support it, call `adapter.syncSkills()` to materialize on disk
5. Record config revision

---

## 6. Runtime Skill Materialization

For adapters that need skills on disk:

1. **Local path skills** — use source directory directly
2. **Remote skills (GitHub, URL, catalog)** — materialize to managed directory:
   ```
   ~/.paperclip/instances/default/skills/{companyId}/__runtime__/{slug}--{keyHash}/
   ```
3. Each skill file is written to the materialized directory
4. Adapters receive `paperclipRuntimeSkills` entries in their config:
   ```typescript
   { key, runtimeName, source: "/path/to/materialized/dir", required, requiredReason }
   ```

---

## 7. Adapter-Specific Injection

| Adapter | Mechanism |
|---------|-----------|
| `claude_local` | tmpdir with `.claude/skills/` symlinks → `--add-dir` flag |
| `codex_local`, `gemini_local`, `opencode_local`, `cursor`, `pi_local` | Symlinks in global skills directory |
| `process`, `http` | Not supported (skill content in prompt template) |

The runtime name includes a hash suffix to avoid collisions: `{slug}--{keyHash10}`

---

## 8. API Surface

| Method | Path | Description |
|--------|------|-------------|
| `GET` | `/api/companies/:companyId/skills` | List skills (with agent usage counts) |
| `GET` | `/api/companies/:companyId/skills/:id` | Skill detail (with agent usage breakdown) |
| `POST` | `/api/companies/:companyId/skills` | Create local skill |
| `POST` | `/api/companies/:companyId/skills/import` | Import from source (GitHub/URL/local/skills.sh) |
| `POST` | `/api/companies/:companyId/skills/scan` | Scan project workspaces for skills |
| `GET` | `/api/companies/:companyId/skills/:id/update-status` | Check for updates (GitHub only) |
| `POST` | `/api/companies/:companyId/skills/:id/install-update` | Apply update from source |
| `GET` | `/api/companies/:companyId/skills/:id/files/:path` | Read skill file content |
| `PUT` | `/api/companies/:companyId/skills/:id/files/:path` | Update skill file (local only) |
| `DELETE` | `/api/companies/:companyId/skills/:id` | Delete skill (removes from agents too) |

---

## 9. Skill Deletion Cleanup

When a skill is deleted:
1. Remove from all agents' `desiredSkills` lists that reference it
2. Delete DB row
3. Remove materialized runtime files from disk

---

## 10. Key Design Decisions

| Decision | Rationale |
|----------|-----------|
| Markdown-first with YAML frontmatter | Universal format — works with any agent runtime, no tooling required |
| Canonical key system (`owner/repo/slug`) | Prevents duplicates across imports, enables update tracking |
| Company-scoped registry | Skills are isolated per company, can be shared across agents within a company |
| Pinned commit SHA for GitHub imports | Reproducibility — same skill version across heartbeats until explicitly updated |
| Trust levels | Operators can see at a glance if a skill includes executable code |
| 30+ directory roots for project scanning | Support skills from every known coding agent convention |
| Custom YAML parser | Avoids `js-yaml` dependency; handles the simple frontmatter cases needed |
| Runtime materialization to disk | Adapters need filesystem access; remote skills are materialized on demand |
| Skill slug with hash suffix for runtime | Prevents name collisions when multiple sources produce same slug |

---

## 11. Observations for Colmena

1. **The skills system is surprisingly complex** (~1,800 lines) for what is conceptually "markdown files that teach agents." The complexity comes from: multi-source import, canonical key derivation, trust classification, file inventory, update tracking, project scanning, runtime materialization, and per-agent assignment.

2. **For Colmena MVP**, simplify to: skills as markdown files in a directory, loaded at runtime. Skip GitHub import, skills.sh, update detection, project scanning, and file inventory. Add those features when there's demand for a skill marketplace.

3. **The canonical key system** is well-designed for preventing duplicates and tracking updates. Worth adopting if Colmena supports importing skills from external sources.

4. **The 30+ directory roots for project scanning** shows how fragmented the coding agent ecosystem is — every tool has its own skills convention. Colmena could standardize on one convention.

5. **Custom YAML frontmatter parser** — Paperclip rolls its own instead of using a library. For Colmena, consider `gray-matter` or similar to save ~200 lines.

6. **The bundled Paperclip skill** (heartbeat protocol) is critical — it's what teaches agents how to behave. Colmena will need an equivalent "here's how to be an agent in our system" skill.

7. **Skill-to-agent assignment** via `adapterConfig.paperclipSkillSyncDesiredSkills` is somewhat hidden. A dedicated `agent_skills` join table might be cleaner.

8. **Runtime materialization** handles the gap between "skill stored in DB" and "adapter needs a filesystem path." It's necessary but adds disk management complexity.
