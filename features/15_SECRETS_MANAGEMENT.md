# Secrets Management — encrypted at rest, versioned, strict mode, runtime resolution

## Investigation Summary

Paperclip encrypts secrets at rest using a local master key and provides a versioned secrets API for managing API keys, tokens, and other sensitive values. Secrets are stored as encrypted `secret_ref` objects in agent adapter configs, resolved to plaintext only at runtime when injected into agent process environments. The system spans 2 database tables, a ~380-line service, a provider registry with 4 providers (1 real + 3 stubs), and a ~130-line encryption module.

---

## 1. Data Model

### 1.1 `company_secrets` Table

| Column | Type | Default | Notes |
|--------|------|---------|-------|
| `id` | UUID | random | PK |
| `companyId` | UUID | — | FK → companies |
| `name` | TEXT | — | Unique per company. Human-readable label. |
| `provider` | TEXT | `"local_encrypted"` | Encryption provider: `local_encrypted`, `aws_secrets_manager`, `gcp_secret_manager`, `vault` |
| `externalRef` | TEXT | NULL | External system reference (for cloud providers) |
| `latestVersion` | INT | 1 | Current version number |
| `description` | TEXT | NULL | — |
| `createdByAgentId` | UUID | NULL | FK → agents |
| `createdByUserId` | TEXT | NULL | — |
| `createdAt`, `updatedAt` | TIMESTAMPTZ | — | — |

**Indexes:**
- `(companyId)` — list secrets for a company
- `(companyId, provider)` — filter by provider
- Unique `(companyId, name)` — no duplicate names per company

### 1.2 `company_secret_versions` Table

| Column | Type | Notes |
|--------|------|-------|
| `id` | UUID | PK |
| `secretId` | UUID | FK → company_secrets, CASCADE |
| `version` | INT | Sequential version number |
| `material` | JSONB | Encrypted material (provider-specific) |
| `valueSha256` | TEXT | SHA-256 hash of the plaintext value |
| `createdByAgentId` | UUID | FK → agents |
| `createdByUserId` | TEXT | — |
| `createdAt` | TIMESTAMPTZ | — |
| `revokedAt` | TIMESTAMPTZ | NULL |

**Indexes:**
- `(secretId, createdAt)` — version history
- `(valueSha256)` — deduplicate by value
- Unique `(secretId, version)` — one row per version

---

## 2. Encryption

### 2.1 Local Encrypted Provider

Default provider: `local_encrypted`. Uses AES-256-GCM symmetric encryption.

**Master key management:**
1. Check `PAPERCLIP_SECRETS_MASTER_KEY` env var (base64, hex, or raw 32-byte string)
2. Else check key file at `PAPERCLIP_SECRETS_MASTER_KEY_FILE` (or default `data/secrets/master.key`)
3. If file doesn't exist → auto-generate 32 random bytes, write to file with `0o600` permissions

**Encryption scheme (`local_encrypted_v1`):**
```typescript
{
  scheme: "local_encrypted_v1",
  iv: string,         // 12-byte random IV, base64
  tag: string,         // GCM auth tag, base64
  ciphertext: string   // AES-256-GCM encrypted value, base64
}
```

**Process:**
- Encrypt: `randomBytes(12)` → `createCipheriv("aes-256-gcm", masterKey, iv)` → `cipher.update(value) + cipher.final()` → `cipher.getAuthTag()`
- Decrypt: `createDecipheriv("aes-256-gcm", masterKey, iv)` → `decipher.setAuthTag(tag)` → `decipher.update(ciphertext) + decipher.final()`

### 2.2 Provider Registry

| Provider | Status | Description |
|----------|--------|-------------|
| `local_encrypted` | ✅ Implemented | AES-256-GCM with local master key |
| `aws_secrets_manager` | Stub | Placeholder for AWS SM integration |
| `gcp_secret_manager` | Stub | Placeholder for GCP SM integration |
| `vault` | Stub | Placeholder for HashiCorp Vault integration |

Stubs throw `unprocessable("not yet implemented")`.

---

## 3. Secret References in Agent Config

### 3.1 Env Binding Format

Agent environment variables support two binding types:

```typescript
// Plain value (stored as-is)
{ "SOME_VAR": { type: "plain", value: "hello" } }
// or shorthand:
{ "SOME_VAR": "hello" }

// Secret reference (stored encrypted, resolved at runtime)
{
  "ANTHROPIC_API_KEY": {
    type: "secret_ref",
    secretId: "8f884973-...",
    version: "latest"          // or specific version number
  }
}
```

### 3.2 Normalization Pipeline

On agent create/update, adapter config goes through `normalizeAdapterConfigForPersistence()`:

1. Parse `adapterConfig.env` as record
2. For each key:
   - Validate key format (`^[A-Za-z_][A-Za-z0-9_]*$`)
   - If `secret_ref` → validate secret exists in the same company
   - If `plain` + redacted sentinel (`***REDACTED***`) → reject (prevents saving redacted UI values)
   - If strict mode + sensitive key pattern (`*API_KEY`, `*TOKEN`, `*SECRET`, etc.) + non-empty plain value → reject
3. Return normalized env config

### 3.3 Runtime Resolution

Before each heartbeat, `resolveAdapterConfigForRuntime()`:

1. Walk `adapterConfig.env` entries
2. For `plain` bindings → pass through as-is
3. For `secret_ref` bindings → `resolveSecretValue(companyId, secretId, version)`:
   - Look up secret in DB
   - Validate company ownership
   - Load version row (latest or specific)
   - Call provider's `resolveVersion()` → decrypt
4. Return resolved config with all secrets as plaintext strings
5. Track which keys were secret-resolved (for log redaction)

---

## 4. Strict Mode

When `PAPERCLIP_SECRETS_STRICT_MODE=true`:

- Sensitive env keys (matching `/(api[-_]?key|access[-_]?token|auth|secret|password|credential|jwt|private[-_]?key|cookie|connectionstring)/i`) **must** use `secret_ref` bindings
- Plain values for sensitive keys are rejected at persistence time
- Recommended for any deployment beyond `local_trusted`

---

## 5. Secret Versioning

### 5.1 Create

1. Create `company_secrets` row with `latestVersion: 1`
2. Encrypt value → create `company_secret_versions` row with `version: 1`

### 5.2 Rotate

1. Increment `latestVersion` on the secret
2. Encrypt new value → create new version row
3. Previous versions remain (for audit) but agents using `version: "latest"` auto-get the new value

### 5.3 Resolution

- `version: "latest"` → resolve to `secret.latestVersion`, then look up that version row
- `version: N` → look up specific version row directly

---

## 6. API Surface

| Method | Path | Description |
|--------|------|-------------|
| `GET` | `/api/companies/:companyId/secrets` | List secrets (metadata only, no values) |
| `GET` | `/api/secrets/:id` | Get secret metadata |
| `POST` | `/api/companies/:companyId/secrets` | Create secret (value encrypted on creation) |
| `PATCH` | `/api/secrets/:id` | Update name/description/externalRef |
| `PATCH` | `/api/secrets/:id/rotate` | Rotate value (new version) |
| `DELETE` | `/api/secrets/:id` | Delete secret and all versions |
| `GET` | `/api/instance/secret-providers` | List available providers |

**Important:** Secret values are **never** returned in API responses. Only metadata (name, provider, version, description) is exposed.

---

## 7. Integration Points

### 7.1 Agent Config Normalization

Called during:
- Agent creation (`POST /api/companies/:companyId/agents`)
- Agent update (`PATCH /api/agents/:id`)
- Agent hire approval payload normalization
- Routine webhook secret creation

### 7.2 Runtime Resolution

Called during:
- Heartbeat execution (before adapter invocation)
- Adapter environment test
- Skill sync (for adapters that need secrets to access remote skills)
- Plugin secret resolution

### 7.3 Migration Tool

`pnpm secrets:migrate-inline-env` converts existing inline API keys in agent configs to encrypted `secret_ref` bindings:
- Dry run by default
- `--apply` flag to actually migrate
- Creates `company_secrets` entries and replaces inline values with `secret_ref`

---

## 8. Key Design Decisions

| Decision | Rationale |
|----------|-----------|
| AES-256-GCM encryption | Industry standard authenticated encryption. GCM provides both confidentiality and integrity. |
| Auto-generated master key | Zero-config for local development. File permissions (0600) provide basic protection. |
| Versioned secrets | Rotation without breaking agents. Old versions retained for audit. `"latest"` as default. |
| Secret refs in adapter config | Secrets are stored once, referenced by ID. No plaintext in agent config rows. |
| Sensitive key pattern matching | Auto-detect keys that should be encrypted (API_KEY, TOKEN, SECRET, etc.) |
| Strict mode as opt-in | Doesn't break existing setups. Recommended for production but not required. |
| Provider registry with stubs | Future extensibility to AWS/GCP/Vault without changing the core API |
| Redacted sentinel rejection | Prevents accidentally saving `***REDACTED***` from UI copy-paste |
| SHA-256 hash of plaintext | Enables deduplication and change detection without storing plaintext |

---

## 9. Observations for Colmena

1. **The local encrypted provider is simple and effective** — ~130 lines for AES-256-GCM with auto-key management. Worth adopting as-is.

2. **The versioning model is production-grade** — secrets can be rotated without coordinating with all agents. `"latest"` auto-resolves.

3. **Strict mode** is a good security practice — forces proper secret management for sensitive env vars.

4. **For Colmena MVP**, implement: local_encrypted provider, single-version secrets (skip versioning), secret_ref in env bindings, runtime resolution. Add versioning and strict mode later.

5. **The migration tool** (`migrate-inline-env`) is a useful onboarding feature — converts existing plaintext env vars to encrypted refs. Consider building this from day one.

6. **Cloud provider stubs** (AWS, GCP, Vault) show the extensibility intent but are not implemented. For Colmena, decide if cloud secret providers are needed or if local encryption suffices.

7. **The sensitive key pattern** is comprehensive but regex-based. Could produce false positives (e.g., `AUTH_MODE` is not sensitive). Acceptable for strict mode warnings.

8. **Secret values never leave the server** except as resolved env vars in agent processes. The API never returns plaintext. UI shows `***REDACTED***`. This is correct security practice.
