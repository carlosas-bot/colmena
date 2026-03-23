# Deployment — Docker, Tailscale, environment variables, local/cloud modes

## Investigation Summary

Paperclip supports multiple deployment modes from local development to cloud production. The deployment system includes a multi-stage Docker build, Docker Compose configurations (quickstart and full), Tailscale private access integration, a comprehensive environment variable reference, and a config file system with env var override merging. The architecture supports running on a developer's laptop, a home server, a VPS, or a cloud platform — all from the same codebase.

---

## 1. Deployment Modes

### 1.1 Local Development

```bash
pnpm dev              # Full dev (API + UI, watch mode)
pnpm dev:once         # Without file watching
pnpm dev:server       # Server only
```

- Embedded PostgreSQL (zero config)
- `local_trusted` mode (no auth)
- Host: `127.0.0.1:3100`
- Hot-reload via Vite + tsc watch

### 1.2 Docker (Quickstart)

```bash
docker compose -f docker-compose.quickstart.yml up --build
```

Single container with:
- Embedded PostgreSQL inside the container
- `authenticated` mode with `private` exposure
- Pre-installed Claude Code, Codex, OpenCode CLIs
- Bind mount for data persistence: `./data/docker-paperclip:/paperclip`
- Requires: `BETTER_AUTH_SECRET`, optionally `OPENAI_API_KEY`, `ANTHROPIC_API_KEY`

### 1.3 Docker (Full)

```bash
docker compose up --build
```

Two containers:
- **PostgreSQL 17** — separate container with health check
- **Paperclip server** — connects to external Postgres via `DATABASE_URL`
- Data volumes for both Postgres and Paperclip data
- Requires: `BETTER_AUTH_SECRET`, `PAPERCLIP_PUBLIC_URL`

### 1.4 Tailscale Private Access

For accessing Paperclip from other devices on a Tailscale network:

```bash
pnpm dev --tailscale-auth
```

Configures:
- `PAPERCLIP_DEPLOYMENT_MODE=authenticated`
- `PAPERCLIP_DEPLOYMENT_EXPOSURE=private`
- `PAPERCLIP_AUTH_BASE_URL_MODE=auto`
- `HOST=0.0.0.0` (bind all interfaces)

Access via: `http://<tailscale-host>:3100`

Custom hostnames whitelisted via: `paperclipai allowed-hostname my-machine.tailnet.ts.net`

---

## 2. Docker Build

### 2.1 Multi-Stage Dockerfile

```dockerfile
# Stage 1: Base (Node.js LTS, git, curl)
FROM node:lts-trixie-slim AS base

# Stage 2: Dependencies (pnpm install --frozen-lockfile)
FROM base AS deps

# Stage 3: Build (UI + server compilation)
FROM base AS build
RUN pnpm --filter @paperclipai/ui build
RUN pnpm --filter @paperclipai/server build

# Stage 4: Production
FROM base AS production
# Install agent CLIs globally
RUN npm install --global @anthropic-ai/claude-code@latest @openai/codex@latest opencode-ai
# Run as non-root user
USER node
CMD ["node", "--import", "./server/node_modules/tsx/dist/loader.mjs", "server/dist/index.js"]
```

**Key decisions:**
- Agent CLIs (Claude Code, Codex, OpenCode) pre-installed globally in the Docker image
- Non-root user (`node`) for security
- Data directory at `/paperclip` as a Docker volume
- `tsx` loader for TypeScript execution in production

### 2.2 Default Production Environment

```
NODE_ENV=production
HOME=/paperclip
HOST=0.0.0.0
PORT=3100
SERVE_UI=true
PAPERCLIP_HOME=/paperclip
PAPERCLIP_DEPLOYMENT_MODE=authenticated
PAPERCLIP_DEPLOYMENT_EXPOSURE=private
```

---

## 3. Configuration System

### 3.1 Config Interface

The server's `Config` interface (~40 fields):

```typescript
interface Config {
  // Deployment
  deploymentMode: "local_trusted" | "authenticated";
  deploymentExposure: "private" | "public";
  host: string;
  port: number;
  allowedHostnames: string[];
  
  // Auth
  authBaseUrlMode: "auto" | "explicit";
  authPublicBaseUrl: string | undefined;
  authDisableSignUp: boolean;
  
  // Database
  databaseMode: "embedded-postgres" | "postgres";
  databaseUrl: string | undefined;
  embeddedPostgresDataDir: string;
  embeddedPostgresPort: number;
  databaseBackupEnabled: boolean;
  databaseBackupIntervalMinutes: number;
  databaseBackupRetentionDays: number;
  databaseBackupDir: string;
  
  // Server
  serveUi: boolean;
  
  // Secrets
  secretsProvider: "local_encrypted" | "aws_secrets_manager" | ...;
  secretsStrictMode: boolean;
  secretsMasterKeyFilePath: string;
  
  // Storage
  storageProvider: "local_disk" | "s3";
  storageLocalDiskBaseDir: string;
  storageS3Bucket: string;
  storageS3Region: string;
  storageS3Endpoint: string | undefined;
  storageS3Prefix: string;
  storageS3ForcePathStyle: boolean;
  
  // Scheduler
  heartbeatSchedulerEnabled: boolean;
  heartbeatSchedulerIntervalMs: number;
  
  // Features
  companyDeletionEnabled: boolean;
}
```

### 3.2 Config Resolution Priority

1. **Environment variables** — highest priority, overrides everything
2. **Config file** — `~/.paperclip/instances/{id}/config.json` or `PAPERCLIP_CONFIG` path
3. **Defaults** — built-in defaults for each field

Both `.env` files are loaded:
1. Paperclip env file: `~/.paperclip/.env`
2. CWD `.env` file (if different)

### 3.3 Data Directory Layout

```
~/.paperclip/                          # PAPERCLIP_HOME
├── .env                               # Global env file
└── instances/
    └── default/                       # PAPERCLIP_INSTANCE_ID
        ├── config.json                # Instance config
        ├── db/                        # Embedded Postgres data
        ├── data/
        │   └── storage/               # File storage (local_disk)
        ├── secrets/
        │   └── master.key             # Encryption master key
        ├── backups/                    # Database backups
        ├── run-logs/                   # Heartbeat run log files
        ├── skills/                    # Materialized company skills
        │   └── {companyId}/
        ├── workspaces/                # Managed project workspaces
        │   └── {companyId}/{projectId}/
        └── agents/
            └── {agentId}/
                └── workspace/         # Default agent workspace
```

---

## 4. Environment Variables

### 4.1 Server Configuration

| Variable | Default | Description |
|----------|---------|-------------|
| `PORT` | `3100` | Server port |
| `HOST` | `127.0.0.1` | Bind address |
| `DATABASE_URL` | (embedded) | PostgreSQL connection string |
| `SERVE_UI` | `true` | Serve React UI |
| `PAPERCLIP_HOME` | `~/.paperclip` | Base data directory |
| `PAPERCLIP_INSTANCE_ID` | `default` | Instance identifier |
| `PAPERCLIP_DEPLOYMENT_MODE` | `local_trusted` | `local_trusted` or `authenticated` |
| `PAPERCLIP_DEPLOYMENT_EXPOSURE` | `private` | `private` or `public` |
| `PAPERCLIP_PUBLIC_URL` | — | Public URL for authenticated mode |
| `PAPERCLIP_ALLOWED_HOSTNAMES` | — | Comma-separated allowed hostnames |

### 4.2 Auth & Secrets

| Variable | Default | Description |
|----------|---------|-------------|
| `BETTER_AUTH_SECRET` | — | Auth session secret |
| `PAPERCLIP_AGENT_JWT_SECRET` | — | Agent JWT signing secret |
| `PAPERCLIP_SECRETS_MASTER_KEY` | — | Encryption master key (base64/hex) |
| `PAPERCLIP_SECRETS_MASTER_KEY_FILE` | — | Path to key file |
| `PAPERCLIP_SECRETS_STRICT_MODE` | `false` | Require secret refs for sensitive env vars |

### 4.3 Agent Runtime (auto-injected)

| Variable | Description |
|----------|-------------|
| `PAPERCLIP_AGENT_ID` | Agent UUID |
| `PAPERCLIP_COMPANY_ID` | Company UUID |
| `PAPERCLIP_API_URL` | API base URL |
| `PAPERCLIP_API_KEY` | Short-lived JWT |
| `PAPERCLIP_RUN_ID` | Current heartbeat run ID |
| `PAPERCLIP_TASK_ID` | Issue that triggered wake |
| `PAPERCLIP_WAKE_REASON` | Why agent was woken |
| `PAPERCLIP_WAKE_COMMENT_ID` | Comment that triggered wake |
| `PAPERCLIP_APPROVAL_ID` | Resolved approval ID |
| `PAPERCLIP_APPROVAL_STATUS` | Approval decision |
| `PAPERCLIP_LINKED_ISSUE_IDS` | Comma-separated linked issues |

### 4.4 LLM Provider Keys

| Variable | Description |
|----------|-------------|
| `ANTHROPIC_API_KEY` | For Claude Code adapter |
| `OPENAI_API_KEY` | For Codex adapter |

---

## 5. Testing & CI

| Tool | Purpose |
|------|---------|
| **Vitest** | Unit/integration tests |
| **Playwright** | E2E tests (`tests/e2e/`) and release smoke tests (`tests/release-smoke/`) |
| **PromptFoo** | Agent prompt evaluation (`evals/promptfoo/`) |

```bash
pnpm test:run                    # Vitest
pnpm test:e2e                    # Playwright E2E
pnpm test:release-smoke          # Playwright release smoke
pnpm evals:smoke                 # PromptFoo prompt evals
```

---

## 6. Release & Publishing

```bash
pnpm release                     # Release script
pnpm release:canary              # Canary release
pnpm release:stable              # Stable release
pnpm release:github              # Create GitHub release
pnpm release:rollback            # Rollback latest release
pnpm build:npm                   # Build for NPM publishing
```

NPM package: `paperclipai` (CLI entrypoint)

---

## 7. Key Design Decisions

| Decision | Rationale |
|----------|-----------|
| Multi-stage Docker build | Small production image, fast rebuilds via layer caching |
| Agent CLIs pre-installed in Docker | Zero-config for running Claude/Codex agents in containers |
| Non-root Docker user | Security best practice |
| Config file + env var merging | Persistent defaults + 12-factor production overrides |
| Data directory isolation | Multiple instances on same machine, clean separation of concerns |
| Tailscale-friendly defaults | Common use case: access from phone/laptop via Tailscale mesh |
| Embedded Postgres in Docker quickstart | Single-container simplicity for trying Paperclip |
| Separate Postgres in full Docker | Production-grade database isolation |
| PromptFoo for eval | Standardized LLM evaluation framework |

---

## 8. Observations for Colmena

1. **The local → Docker → cloud progression** is a great DX story. Users start locally, containerize for persistence, deploy for production. Worth replicating.

2. **Pre-installing agent CLIs in Docker** eliminates the most common setup friction. Do this from day one.

3. **For Colmena MVP**, support: local development (`pnpm dev`) and Docker quickstart (single container). Add full Docker Compose and cloud deployment docs later.

4. **The Tailscale integration** is a smart niche — solo developers using Paperclip from their phone via Tailscale. Low effort to support, high value for mobile management.

5. **The data directory layout** is well-organized — each concern (db, storage, secrets, skills, workspaces, agents, backups, logs) has its own subdirectory. Worth adopting.

6. **The `~40 field Config interface`** is comprehensive but complex. For Colmena MVP, start with ~15 essential fields (port, host, databaseUrl, deploymentMode, serveUi, secretsMasterKey, storageDir).

7. **PromptFoo for agent prompt evaluation** is forward-thinking — tests that the heartbeat protocol skill actually works. Consider this for Colmena's core skill.

8. **The release scripts** (stable/canary/rollback/GitHub release) show maturity. For Colmena MVP, `npm publish` is sufficient.
