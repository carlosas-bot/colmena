# Adapters — 9 built-in agent runtime bridges (Claude, Codex, Gemini, OpenCode, Cursor, Pi, OpenClaw, Process, HTTP)

## Investigation Summary

Adapters are the bridge between Paperclip's orchestration layer and actual agent runtimes. Each adapter knows how to spawn/call a specific AI agent, capture its output, parse usage/costs, manage sessions, and inject skills. The adapter system is pluggable — 10 built-in adapters are registered, and custom adapters can be created following a documented structure. Each adapter has 3 modules (server, UI, CLI) consumed by 3 registries. The adapter contract is defined in `@paperclipai/adapter-utils` (~300 lines of types) with shared utilities for process spawning, env building, template rendering, and skill management.

---

## 1. Adapter Architecture

### 1.1 Three-Module Structure

Each adapter is a package with three independently consumable modules:

```
packages/adapters/<name>/
  src/
    index.ts              # Shared metadata (type, label, models, config docs)
    server/
      execute.ts          # Core: spawn agent, capture output, return result
      parse.ts            # Parse stdout for usage, errors, session state
      skills.ts           # Skill injection logic (optional)
      test.ts             # Environment diagnostics (pre-run validation)
      quota.ts            # Provider quota/rate-limit polling (optional)
    ui/
      parse-stdout.ts     # Stdout → TranscriptEntry[] for run viewer
      build-config.ts     # Form values → adapterConfig JSON
      index.ts            # UI exports
    cli/
      format-event.ts     # Terminal formatter for `paperclipai run --watch`
      index.ts            # CLI exports
```

### 1.2 Three Registries

| Registry | Location | Consumes | Purpose |
|----------|---------|----------|---------|
| **Server** | `server/src/adapters/registry.ts` | `execute`, `testEnvironment`, `listSkills`, `syncSkills`, `sessionCodec`, `getQuotaWindows`, `onHireApproved` | Executes agents, captures results |
| **UI** | `ui/src/adapters/registry.ts` | `parseStdout`, `buildConfig` | Renders run transcripts, provides config forms |
| **CLI** | `cli/src/adapters/registry.ts` | `formatStdoutEvent` | Terminal output for live watching |

All registries use a simple `Map<string, AdapterModule>` keyed by adapter type string.

---

## 2. Built-in Adapters

### 2.1 Local CLI Adapters (spawn child process)

| Adapter | Type Key | Runtime | Default Model | Key Features |
|---------|----------|---------|---------------|-------------|
| **Claude Code** | `claude_local` | Claude Code CLI | claude-sonnet-4-6 | Session persistence (session IDs), `--add-dir` for skills, `--print` mode, `--permission-mode`, max turns per run, quota probing, login flow |
| **Codex** | `codex_local` | OpenAI Codex CLI | gpt-5.3-codex | `--dangerously-bypass-approvals-and-sandbox`, quota probing, model discovery |
| **Gemini** | `gemini_local` | Gemini CLI | auto | Skill injection via global dir |
| **OpenCode** | `opencode_local` | OpenCode CLI | (explicit required) | Multi-provider `provider/model` format, model discovery via `opencode models`, model validation |
| **Cursor** | `cursor_local` | Cursor CLI | auto | Model discovery |
| **Pi** | `pi_local` | Pi CLI | (discovered) | Multi-provider, model discovery |
| **Hermes** | `hermes_local` | Hermes CLI | (from config) | External package (`hermes-paperclip-adapter`) |

**Common patterns for local adapters:**
- Spawned via `runChildProcess()` utility (handles timeouts, signals, exit codes)
- Session persistence via `sessionCodec` (serialize/deserialize session params)
- Environment injection via `buildPaperclipEnv()` + adapter-specific env vars
- Prompt rendering via `renderTemplate()` with agent/context variables
- Skills injected via tmpdir symlinks or global config dirs
- Stdout parsed for usage, errors, session state

### 2.2 Remote/Webhook Adapters

| Adapter | Type Key | Mechanism | Key Features |
|---------|----------|-----------|-------------|
| **OpenClaw Gateway** | `openclaw_gateway` | HTTP wake payload to OpenClaw instance | Ed25519 device key auth, no local JWT (uses gateway auth) |
| **HTTP** | `http` | Generic HTTP POST | Configurable URL, method, headers, payload template |

### 2.3 Generic Process Adapter

| Adapter | Type Key | Mechanism | Key Features |
|---------|----------|-----------|-------------|
| **Process** | `process` | Arbitrary shell command | `command` + `args`, env vars, timeout/grace period. Fallback for unknown adapter types. |

---

## 3. Adapter Contract (ServerAdapterModule)

```typescript
interface ServerAdapterModule {
  type: string;                          // Unique adapter identifier
  
  // Required
  execute(ctx: AdapterExecutionContext): Promise<AdapterExecutionResult>;
  testEnvironment(ctx: AdapterEnvironmentTestContext): Promise<AdapterEnvironmentTestResult>;
  
  // Optional capabilities
  listSkills?: (ctx: AdapterSkillContext) => Promise<AdapterSkillSnapshot>;
  syncSkills?: (ctx: AdapterSkillContext, desiredSkills: string[]) => Promise<AdapterSkillSnapshot>;
  sessionCodec?: AdapterSessionCodec;
  sessionManagement?: AdapterSessionManagement;
  onHireApproved?: (payload: HireApprovedPayload, config: Record<string, unknown>) => Promise<HireApprovedHookResult>;
  getQuotaWindows?: () => Promise<ProviderQuotaResult>;
  
  // Metadata
  models?: AdapterModel[];               // Static model list
  listModels?: () => Promise<AdapterModel[]>;  // Dynamic model discovery
  supportsLocalAgentJwt?: boolean;        // Whether to inject PAPERCLIP_API_KEY JWT
  agentConfigurationDoc?: string;         // Markdown docs for agent creation UI
}
```

### 3.1 Execution Context (Input)

```typescript
interface AdapterExecutionContext {
  runId: string;
  agent: { id, companyId, name, adapterType, adapterConfig };
  runtime: { sessionId, sessionParams, sessionDisplayId, taskKey };
  config: Record<string, unknown>;       // Resolved adapterConfig (secrets decrypted)
  context: Record<string, unknown>;      // Heartbeat context (issueId, wakeReason, etc.)
  onLog: (stream: "stdout"|"stderr", chunk: string) => Promise<void>;
  onMeta?: (meta: AdapterInvocationMeta) => Promise<void>;
  onSpawn?: (meta: { pid, startedAt }) => Promise<void>;
  authToken?: string;                    // JWT for API access
}
```

### 3.2 Execution Result (Output)

```typescript
interface AdapterExecutionResult {
  exitCode: number | null;
  signal: string | null;
  timedOut: boolean;
  errorMessage?: string | null;
  errorCode?: string | null;
  
  // Usage & cost
  usage?: { inputTokens, outputTokens, cachedInputTokens };
  provider?: string;
  biller?: string;
  model?: string;
  billingType?: "metered_api" | "subscription_included" | ...;
  costUsd?: number;
  
  // Session (for persistence)
  sessionId?: string;
  sessionParams?: Record<string, unknown>;
  sessionDisplayId?: string;
  clearSession?: boolean;
  
  // Extras
  resultJson?: Record<string, unknown>;
  runtimeServices?: AdapterRuntimeServiceReport[];
  summary?: string;
}
```

### 3.3 Session Codec

Each adapter can define how sessions are serialized/deserialized:

```typescript
interface AdapterSessionCodec {
  deserialize(raw: unknown): Record<string, unknown> | null;
  serialize(params: Record<string, unknown> | null): Record<string, unknown> | null;
  getDisplayId?: (params: Record<string, unknown> | null) => string | null;
}
```

### 3.4 Environment Test

```typescript
interface AdapterEnvironmentTestResult {
  adapterType: string;
  status: "pass" | "warn" | "fail";
  checks: Array<{
    code: string;
    level: "info" | "warn" | "error";
    message: string;
    detail?: string;
    hint?: string;
  }>;
  testedAt: string;
}
```

---

## 4. Shared Utilities (`@paperclipai/adapter-utils`)

### 4.1 Server Utilities

| Utility | Purpose |
|---------|---------|
| `buildPaperclipEnv(agent)` | Build PAPERCLIP_* env vars for injection |
| `runChildProcess(opts)` | Spawn process with timeout, signals, PID tracking |
| `renderTemplate(template, data)` | Mustache-style template rendering for prompts |
| `ensureAbsoluteDirectory(path)` | Validate and create CWD |
| `ensureCommandResolvable(cmd)` | Check command exists in PATH |
| `ensurePathInEnv(env)` | Ensure PATH is set in environment |
| `readPaperclipRuntimeSkillEntries(config)` | Load company skills for injection |
| `joinPromptSections(sections)` | Combine prompt parts |
| `redactEnvForLogs(env)` | Remove secrets from env for logging |

### 4.2 Session Compaction

| Utility | Purpose |
|---------|---------|
| `resolveSessionCompactionPolicy(adapterType, runtimeConfig)` | Parse compaction thresholds per adapter |
| `hasSessionCompactionThresholds(policy)` | Check if any thresholds are configured |
| `getAdapterSessionManagement(adapterType)` | Get adapter-specific session management config |

### 4.3 Billing

| Utility | Purpose |
|---------|---------|
| `normalizeBillingType(value)` | Normalize billing type strings |
| `resolveProviderBiller(provider, billing)` | Determine who bills for a usage event |

---

## 5. Skills Injection

Different adapters inject Paperclip skills differently:

| Adapter | Mechanism |
|---------|-----------|
| `claude_local` | Create tmpdir with `.claude/skills/` containing symlinks → pass `--add-dir <tmpdir>` to Claude Code |
| `codex_local` | Symlink to global skills directory |
| `gemini_local` | Symlink to global skills directory |
| `opencode_local` | Symlink to global skills directory |
| `cursor` | Symlink to global skills directory |
| `pi_local` | Symlink to global skills directory |
| `process` | N/A (skill content in prompt template) |
| `http` | N/A (skill content in payload template) |

Skills are loaded from:
1. Paperclip's built-in `skills/paperclip/` (heartbeat protocol)
2. Company-managed skills (`company_skills` table, materialized to disk)

---

## 6. Model Management

### 6.1 Static Models

Defined in `src/index.ts` as a static array:
```typescript
export const models = [
  { id: "claude-sonnet-4-6", label: "Claude Sonnet 4.6" },
  ...
];
```

### 6.2 Dynamic Discovery

Some adapters implement `listModels()` for live discovery:
- **Codex** — merged with OpenAI model discovery
- **OpenCode** — discovered via `opencode models` command
- **Pi** — discovered from Pi CLI
- **Cursor** — runtime discovery

If discovery returns empty, falls back to static models.

### 6.3 Model Validation

OpenCode requires explicit model selection and validates against live discovery before saving config.

---

## 7. Adapter-Specific Features

### 7.1 Claude Code (`claude_local`)

- **Session persistence** — Claude Code session IDs persist across heartbeats
- **Skills via `--add-dir`** — unique tmpdir approach with symlinks
- **Max turns per run** — configurable limit (default 80)
- **Permission bypass** — `--permission-mode bypassPermissions`
- **Login flow** — `POST /api/agents/:id/claude-login` triggers OAuth
- **Quota probing** — polls Anthropic rate limit headers
- **Unknown session handling** — detects and clears stale sessions

### 7.2 Codex (`codex_local`)

- **Sandbox bypass** — `--dangerously-bypass-approvals-and-sandbox`
- **Search mode** — optional web search capability
- **Quota probing** — polls OpenAI rate limit headers
- **Codex home** — manages `~/.codex/` configuration

### 7.3 OpenClaw Gateway (`openclaw_gateway`)

- **Device key auth** — Ed25519 key pair auto-generated per agent
- **Wake payload** — sends context to OpenClaw instance via HTTP
- **No local JWT** — uses OpenClaw's own auth

---

## 8. Creating Custom Adapters

Documented process:
1. Create package under `packages/adapters/<name>/`
2. Implement `execute()` and `testEnvironment()` in server module
3. Implement `parseStdout()` and `buildConfig()` in UI module
4. Implement `formatStdoutEvent()` in CLI module
5. Register in all 3 registries

**Security requirements:**
- Treat agent output as untrusted (parse defensively)
- Inject secrets via env vars, not prompts
- Always enforce timeout and grace period

---

## 9. Key Design Decisions

| Decision | Rationale |
|----------|-----------|
| Three-module architecture (server/UI/CLI) | Different consumers need different parts; tree-shaking keeps bundles small |
| Map-based registry (not dynamic imports) | Deterministic, type-safe, no runtime discovery overhead |
| Process adapter as fallback for unknown types | Graceful degradation — unknown adapters can still run if they accept a command |
| Shared utilities in `adapter-utils` | DRY — common patterns (process spawning, env building) shared across all adapters |
| Session codec per adapter | Different runtimes have different session formats (Claude session IDs vs OpenCode state blobs) |
| Skills via tmpdir symlinks (Claude) | Non-destructive — doesn't write to the agent's working directory |
| Adapter configuration docs as inline markdown | Surfaced in the agent creation UI for each adapter type |
| `onHireApproved` lifecycle hook | Adapters can notify their runtime when an agent is approved (e.g., OpenClaw sends a message) |

---

## 10. Observations for Colmena

1. **The three-module pattern is well-engineered but heavyweight.** For Colmena MVP, start with server-only adapters. Add UI and CLI modules when the UI needs run viewers and config forms.

2. **7 local CLI adapters** suggests the market has many coding agents. Colmena should decide which to support initially — `claude_local` and `codex_local` cover the two dominant providers. Others can be added later.

3. **The `process` adapter is the escape hatch** — any command that runs and exits can be an agent. This is valuable for non-AI agents (cron jobs, scripts, build tools).

4. **The `http` adapter enables external agents** — any service that can receive an HTTP POST can be woken as an agent. Useful for cloud-hosted agents, serverless functions, etc.

5. **Skills injection is adapter-specific** and somewhat hacky (tmpdir symlinks). For Colmena, consider a more standardized approach (env var pointing to skills directory, or prompt injection for all adapters).

6. **Session codecs** are essential for session persistence. Without them, agents would lose all conversation context every heartbeat.

7. **Model discovery** is a polish feature that improves UX — instead of typing model IDs manually, the UI can show a dropdown. Not essential for MVP.

8. **The `testEnvironment()` method** is a UX win — validates adapter config before the first heartbeat, surfacing issues (missing CLI, wrong model, no API key) early.

9. **Adapter configuration docs** as inline markdown is clever — the agent creation form can show adapter-specific help without a separate docs page.

10. **For Colmena MVP**, implement: `claude_local` (most popular), `process` (escape hatch), `http` (external agents). Add others as demand emerges.
