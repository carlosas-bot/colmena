# CLI — onboarding, configuration, diagnostics

## Investigation Summary

Paperclip provides a CLI (`paperclipai`) built with Commander.js for instance setup, configuration, diagnostics, and agent management. The CLI spans ~11,800 lines across 30+ files and includes an interactive onboarding wizard, a diagnostic doctor tool, configuration management, database backup, agent/issue/project CRUD commands, heartbeat run watching, git worktree management, and adapter-specific terminal formatters. It uses `@clack/prompts` for beautiful interactive prompts and `picocolors` for terminal styling.

---

## 1. Command Inventory

### 1.1 Setup & Configuration

| Command | Description |
|---------|-------------|
| `paperclipai onboard [--yes]` | Interactive first-run setup wizard (database, server, auth, storage, secrets, LLM) |
| `paperclipai configure --section <s>` | Update config sections: `llm`, `database`, `logging`, `server`, `storage`, `secrets` |
| `paperclipai doctor [--repair]` | Run diagnostic checks on setup; optionally auto-repair |
| `paperclipai env` | Print environment variables for deployment |
| `paperclipai allowed-hostname <host>` | Whitelist hostname for authenticated/private mode |
| `paperclipai auth bootstrap-ceo` | Generate first admin invite URL |

### 1.2 Runtime

| Command | Description |
|---------|-------------|
| `paperclipai run [--watch]` | Start the API server (optionally with live agent run watching in terminal) |
| `paperclipai run --agent <id>` | Watch a specific agent's runs in terminal |
| `paperclipai db:backup` | Create database backup (pg_dump-based) |

### 1.3 Client Commands (API interaction)

| Command | Description |
|---------|-------------|
| `paperclipai context set` | Set active company/agent context |
| `paperclipai context show` | Show current context |
| `paperclipai company list\|create\|...` | Company CRUD |
| `paperclipai agent list\|create\|invoke\|local-cli\|...` | Agent management + heartbeat invocation |
| `paperclipai issue list\|create\|checkout\|update\|...` | Issue management |
| `paperclipai approval list\|approve\|reject\|...` | Approval management |
| `paperclipai activity list` | Activity log viewer |
| `paperclipai dashboard` | Terminal dashboard summary |
| `paperclipai plugin list\|enable\|disable\|...` | Plugin management |
| `paperclipai worktree ...` | Git worktree management for agent workspaces |

---

## 2. Onboarding Wizard

The `onboard` command (~490 lines) is the entry point for new installations:

### 2.1 Modes

- **Quickstart** (`--yes`) — accept all defaults, start immediately
- **Advanced** — interactive prompts for each configuration section

### 2.2 Configuration Sections

1. **Database** — embedded (default) or external PostgreSQL URL
2. **Server** — host, port, serve UI toggle
3. **Auth** — deployment mode (`local_trusted` / `authenticated`), exposure, base URL
4. **Storage** — local disk (default) or S3-compatible
5. **Secrets** — provider, strict mode, master key path
6. **LLM** — API keys for Anthropic, OpenAI (prompted but stored as env vars)
7. **Logging** — log directory, level

### 2.3 Side Effects

On completion:
- Writes config to `~/.paperclip/instances/{id}/config.json`
- Ensures agent JWT secret exists
- Ensures local secrets master key file exists
- Optionally starts the server immediately
- Optionally generates bootstrap CEO invite

### 2.4 Environment Variable Detection

The wizard detects 25+ env vars and uses them as defaults if present:
`PAPERCLIP_PUBLIC_URL`, `DATABASE_URL`, `HOST`, `PORT`, `SERVE_UI`, `PAPERCLIP_DEPLOYMENT_MODE`, `BETTER_AUTH_URL`, `PAPERCLIP_STORAGE_*`, `PAPERCLIP_SECRETS_*`, etc.

---

## 3. Doctor (Diagnostics)

The `doctor` command (~200 lines) runs diagnostic checks:

### 3.1 Checks

| Check | What it validates |
|-------|------------------|
| `configCheck` | Config file exists and is parseable |
| `databaseCheck` | Database is reachable and migrations are current |
| `portCheck` | Configured port is available |
| `llmCheck` | LLM API keys are set (ANTHROPIC_API_KEY, OPENAI_API_KEY) |
| `logCheck` | Log directory exists and is writable |
| `secretsCheck` | Master key file exists, secrets provider is functional |
| `storageCheck` | Storage directory exists (local) or bucket is accessible (S3) |
| `agentJwtSecretCheck` | JWT secret is configured |
| `deploymentAuthCheck` | Auth config is consistent with deployment mode |

### 3.2 Output Format

Each check returns:
```typescript
{
  name: string;
  status: "pass" | "warn" | "fail";
  message: string;
  canRepair: boolean;
  repairHint?: string;
}
```

Visual output: `✓` (green), `!` (yellow), `✗` (red) per check.

### 3.3 Auto-Repair

With `--repair`, the doctor can fix some issues automatically:
- Create missing directories
- Generate missing key files
- Apply pending database migrations

---

## 4. Run Command

### 4.1 Server Start

`paperclipai run` starts the Express server with:
- Embedded or external database
- Auto-migration on startup
- UI static file serving (if enabled)
- Startup banner with instance info

### 4.2 Watch Mode

`paperclipai run --watch` or `paperclipai run --agent <id>`:
- Connects to the WebSocket live event stream
- Pretty-prints agent heartbeat activity in the terminal
- Uses adapter-specific formatters (Claude, Codex, etc.) for structured output
- Color-coded: green for success, red for errors, yellow for warnings

---

## 5. Client Commands

### 5.1 Context Management

```bash
paperclipai context set --company <id> --agent <id> --api-url <url> --api-key <key>
paperclipai context show
```

Stores active context in `~/.paperclip/cli-context.json`.

### 5.2 Agent Local CLI

```bash
paperclipai agent local-cli <agent-id-or-shortname> --company-id <id>
```

Installs Paperclip skills for Claude/Codex and prints the required `PAPERCLIP_*` environment variables for that agent identity. Enables running agents locally outside heartbeat runs.

### 5.3 Company Commands (~570 lines)

CRUD + export/import:
```bash
paperclipai company list
paperclipai company create --name "My Company"
paperclipai company export <companyId> [--include agents,skills] [--output ./package]
paperclipai company import <source> [--target new|existing:<id>] [--collision rename|skip|replace]
```

### 5.4 Worktree Commands (~2,460 lines)

Git worktree management for agent workspaces — the largest CLI module:
```bash
paperclipai worktree create --project <id> --branch <name>
paperclipai worktree list --project <id>
paperclipai worktree merge --project <id> --branch <name>
paperclipai worktree remove --project <id> --branch <name>
```

Includes merge history tracking (~710 lines) for complex multi-agent workspace workflows.

---

## 6. Tech Stack

| Library | Purpose |
|---------|---------|
| `commander` | CLI framework (commands, options, args) |
| `@clack/prompts` | Interactive prompts (select, confirm, text, spinner) |
| `picocolors` | Terminal color styling |
| `postgres` | Database client (for doctor checks) |
| `@paperclipai/shared` | Shared types and constants |

---

## 7. Configuration System

### 7.1 Config File

`~/.paperclip/instances/{instanceId}/config.json`:
```json
{
  "database": { "url": "...", "embeddedPort": 5433, "backupEnabled": true },
  "server": { "host": "127.0.0.1", "port": 3100, "serveUi": true },
  "auth": { "deploymentMode": "local_trusted" },
  "storage": { "provider": "local_disk", "localBaseDir": "..." },
  "secrets": { "provider": "local_encrypted", "strictMode": false },
  "logging": { "dir": "...", "level": "info" }
}
```

### 7.2 Data Directory Isolation

`--data-dir` flag isolates all state from the default `~/.paperclip`:
- Database data
- Config file
- Secrets master key
- Storage directory
- Log files

Enables running multiple instances on the same machine.

---

## 8. Key Design Decisions

| Decision | Rationale |
|----------|-----------|
| Commander.js (not oclif/yargs) | Lightweight, zero-config, good TypeScript support |
| @clack/prompts | Beautiful interactive prompts with spinner, select, confirm — better UX than inquirer |
| Separate client commands | CLI can interact with a running Paperclip instance via API (useful for automation) |
| Doctor with auto-repair | Reduces support burden — common issues can be fixed automatically |
| Config file + env var merge | Config file for persistent settings, env vars for overrides (12-factor) |
| Data dir isolation | Multiple instances on same machine without conflicts |
| Adapter-specific terminal formatters | Claude/Codex output looks different; formatters provide clean terminal output per adapter |

---

## 9. Observations for Colmena

1. **The onboarding wizard is essential UX** — it takes users from zero to running in one command. Build this early.

2. **The doctor command is a support lifesaver** — diagnoses common issues (missing keys, unreachable DB, port conflicts) and can auto-repair. Worth building from day one.

3. **For Colmena MVP CLI**, implement: `onboard`, `run`, `doctor`, `configure`. Skip client commands (agent/issue/approval CLI) — the web UI covers those use cases.

4. **@clack/prompts** provides excellent interactive UX (spinners, selects, multi-step flows). Much better than raw readline or inquirer.

5. **The worktree management** (~2,460 lines) is specialized for multi-agent git workflows. Defer unless Colmena has a strong git worktree story.

6. **The `--data-dir` isolation** is great for development (run multiple instances) and testing. Worth adding early.

7. **Agent local CLI** (`agent local-cli`) is a power-user feature — lets developers run agents locally with Paperclip identity. Useful for debugging.

8. **The config file + env var merge pattern** follows 12-factor best practices. Config file for defaults, env vars for production overrides.

9. **Watch mode** (`run --watch`) is the terminal equivalent of the run viewer UI. Useful for operators who prefer terminal over browser.
