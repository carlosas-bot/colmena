# Access Control & Auth — deployment modes, Better Auth, API keys, JWTs, permissions

## Investigation Summary

Paperclip implements a multi-layered authentication and authorization system supporting two deployment modes (`local_trusted` and `authenticated`), three actor types (board users, agents, system), and a company-scoped permission model. The system spans ~3,700 lines across 6 files and uses Better Auth for human sessions, custom HMAC-SHA256 JWTs for agent run auth, SHA-256 hashed API keys for persistent agent access, and a principal-permission-grant model for fine-grained authorization. The board claim flow enables migration from local trusted mode to authenticated mode.

---

## 1. Deployment Modes

### 1.1 `local_trusted`

- Default mode. No login required.
- All requests treated as the local board operator.
- Actor is auto-set: `{ type: "board", userId: "local-board", isInstanceAdmin: true, source: "local_implicit" }`
- Host binding: loopback only (localhost)
- Use case: local development, solo experimentation

### 1.2 `authenticated`

- Login required via Better Auth (email + password)
- Two exposure policies:
  - **private** — Tailscale/VPN/LAN access, auto base URL mode
  - **public** — internet-facing, explicit public URL required
- Session-based auth for board users (cookies)
- Bearer token auth for agents (JWTs or API keys)

---

## 2. Actor Model

### 2.1 Actor Types

Every request is classified into one of three actor types via `actorMiddleware`:

| Actor Type | Source | Identified By |
|-----------|--------|--------------|
| `board` | `local_implicit` | `local_trusted` mode (no auth needed) |
| `board` | `session` | Better Auth session cookie |
| `agent` | `api_key` | `Bearer pcp_...` API key (SHA-256 hashed lookup) |
| `agent` | `run_jwt` | `Bearer <JWT>` short-lived run token |
| `none` | — | No valid auth found |

### 2.2 Request Actor Shape

```typescript
// Attached to req.actor by middleware
type Actor =
  | { type: "board"; userId: string; isInstanceAdmin: boolean; source: "local_implicit" | "session"; companyIds?: string[] }
  | { type: "agent"; agentId: string; companyId: string; runId?: string; source: "api_key" | "run_jwt" }
  | { type: "none"; source: "none" }
```

### 2.3 Auth Resolution Order

1. Check `Authorization: Bearer <token>` header
2. If starts with `pcp_` → hash and look up in `agent_api_keys` table
3. Else → try `verifyLocalAgentJwt()` (HMAC-SHA256)
4. If no bearer token → try Better Auth session resolution (cookies)
5. If `local_trusted` mode → auto-assign local board actor
6. Else → `{ type: "none" }`

---

## 3. Authentication Mechanisms

### 3.1 Better Auth (Human Users)

- Library: `better-auth` with Drizzle adapter
- Tables: `user`, `session`, `account`, `verification` (standard Better Auth schema)
- Email + password auth, sign-up optionally disabled
- Session-based (cookies), resolved via `resolveBetterAuthSession()`
- Trusted origins derived from config (public URL, allowed hostnames)
- Secure cookies by default, disabled for HTTP-only URLs

### 3.2 Agent Run JWTs

Short-lived tokens created per heartbeat run:

```typescript
interface LocalAgentJwtClaims {
  sub: string;           // agent ID
  company_id: string;
  adapter_type: string;
  run_id: string;
  iat: number;
  exp: number;           // Default: 48 hours
  iss: "paperclip";
  aud: "paperclip-api";
}
```

- Algorithm: HMAC-SHA256 (`HS256`)
- Secret: `PAPERCLIP_AGENT_JWT_SECRET` env var
- TTL: configurable, default 48 hours
- Created by `createLocalAgentJwt()`, verified by `verifyLocalAgentJwt()`
- Timing-safe comparison for signature verification
- Injected as `PAPERCLIP_API_KEY` env var during heartbeat execution

### 3.3 Agent API Keys

Long-lived keys for persistent agent access (outside heartbeats):

- Format: `pcp_` + 24 random hex bytes
- Stored as SHA-256 hash in `agent_api_keys` table
- Full value shown only once at creation
- Can be revoked (soft delete via `revokedAt`)
- Auto-revoked when agent is terminated
- Looked up via hash comparison (not timing-safe, but hash-based so OK)

---

## 4. Authorization

### 4.1 Route-Level Guards

Two primary guard functions used across all routes:

```typescript
// Require board user
assertBoard(req);

// Require access to a specific company
assertCompanyAccess(req, companyId);
```

**`assertCompanyAccess` logic:**
- Agent → must belong to the same company
- Board (local_implicit or instance admin) → all companies
- Board (regular) → must have active membership in the company

### 4.2 Permission Keys

6 defined permission keys:

| Key | Description |
|-----|-------------|
| `agents:create` | Can create/manage agents |
| `tasks:assign` | Can assign tasks to agents |
| `tasks:assign_scope` | Task assignment scope |
| `users:invite` | Can create invite links |
| `users:manage_permissions` | Can manage user permissions |
| `joins:approve` | Can approve join requests |

### 4.3 Permission Grant Model

```
principal_permission_grants
├── companyId     — which company
├── principalType — "user" or "agent"
├── principalId   — user ID or agent ID
├── permissionKey — one of the 6 keys
├── scope         — optional JSON scope restriction
└── grantedByUserId — who granted
```

**Unique constraint:** `(companyId, principalType, principalId, permissionKey)`

**Resolution:**
1. Check membership is active
2. Look up permission grant row
3. For agents: additional role-based defaults (CEO gets `canCreateAgents`)

### 4.4 Company Membership

```
company_memberships
├── companyId
├── principalType — "user" or "agent"
├── principalId
├── status        — "active", "pending", "suspended"
├── membershipRole — "owner", "member"
```

Every entity access check ultimately traces to company membership.

### 4.5 Instance-Level Roles

```
instance_user_roles
├── userId
├── role — "instance_admin"
```

Instance admins bypass company membership checks and can access all companies.

---

## 5. Board Claim Flow

When migrating from `local_trusted` to `authenticated`:

1. On startup, if the only instance admin is `local-board`:
   - Generate a one-time claim challenge (token + code, 24h expiry)
   - Print claim URL to console: `/board-claim/{token}?code={code}`
2. Signed-in user visits the URL
3. `claimBoardOwnership()`:
   - Promotes the visiting user to instance admin
   - Demotes `local-board` admin
   - Ensures active company membership for the claiming user across all companies
4. Challenge is consumed (can't be reused)

---

## 6. Invite & Join System

### 6.1 Invites

```
invites
├── companyId
├── inviteType — "company_join" or "bootstrap_ceo"
├── tokenHash  — SHA-256 of invite token
├── allowedJoinTypes — "human", "agent", or "both"
├── defaultsPayload — JSONB defaults for new agents
├── expiresAt
├── revokedAt, acceptedAt
```

Invite links allow external users or agents to join a company.

### 6.2 Join Requests

```
join_requests
├── inviteId
├── companyId
├── requestType — "human" or "agent"
├── status      — "pending_approval", "approved", "rejected"
├── requestIp, requestEmailSnapshot
├── agentName, adapterType, capabilities
├── agentDefaultsPayload
├── claimSecretHash — for agent claim-based joining
├── createdAgentId  — created agent on approval
```

Join requests can be:
- **Human** — user signs up and requests to join a company
- **Agent** — external agent (e.g., OpenClaw) requests to join via invite link

The `access.ts` routes file (~2,700 lines) handles the full invite/join flow including human onboarding, agent bootstrap via invite, claim-secret-based agent joining, and permission management.

---

## 7. API Surface

| Method | Path | Description |
|--------|------|-------------|
| `POST` | `/api/auth/*` | Better Auth endpoints (login, signup, etc.) |
| `GET` | `/api/instance/me` | Current user info |
| `GET/POST` | `/api/board-claim/:token` | Board ownership claim |
| `GET` | `/api/companies/:companyId/members` | List company members |
| `PATCH` | `/api/companies/:companyId/members/:id/permissions` | Update member permissions |
| `POST` | `/api/companies/:companyId/invites` | Create invite |
| `GET` | `/api/companies/:companyId/invites` | List invites |
| `DELETE` | `/api/invites/:id` | Revoke invite |
| `GET` | `/api/invites/:token/inspect` | Inspect invite (public) |
| `POST` | `/api/invites/:token/join` | Join via invite |
| `GET` | `/api/companies/:companyId/join-requests` | List join requests |
| `POST` | `/api/join-requests/:id/approve` | Approve join request |
| `POST` | `/api/join-requests/:id/reject` | Reject join request |

---

## 8. Key Design Decisions

| Decision | Rationale |
|----------|-----------|
| Two deployment modes | Simple path for solo users (no auth), production path for teams |
| Better Auth for humans | Full-featured auth library with Drizzle adapter, handles sessions/cookies/email |
| Custom JWT for agent runs | Lightweight, no external dependency, scoped to run + agent + company |
| SHA-256 hashed API keys | Standard practice — keys can be verified without storing plaintext |
| Principal permission grants | Flexible — works for both users and agents with the same model |
| Board claim flow | Secure migration path from local_trusted without losing data |
| Company membership as access gate | Every entity check traces to membership, preventing cross-company access |
| Invite system with join types | Supports both human and agent onboarding via shared links |
| Instance admin bypass | Simplifies multi-company management without per-company grants |

---

## 9. Observations for Colmena

1. **The `local_trusted` → `authenticated` progression** is a smart onboarding UX. Users start without auth friction, then add it when they need remote access.

2. **Better Auth is a good choice** for human auth — handles email/password, sessions, cookies. For Colmena, consider whether OAuth/social login is also needed.

3. **The custom JWT implementation** is ~140 lines. Could be replaced with `jose` or `jsonwebtoken` library, but the custom one avoids dependencies and is simple enough.

4. **Agent API keys** (`pcp_` prefix) are simple and effective. The prefix makes them easy to identify in logs and `.env` files.

5. **The permission system is lightweight** — 6 permission keys with grant/revoke. No role-based access control (RBAC) hierarchy. Adequate for typical Paperclip deployments.

6. **For Colmena MVP**, start with `local_trusted` only. Add `authenticated` mode when multi-user or remote access is needed. Skip invites and join requests initially.

7. **The board claim flow** is a one-time migration. Elegant but niche — only needed once per deployment. Consider if Colmena needs this complexity or if initial setup can require auth from the start.

8. **The invite/join system** (~2,700 lines in routes alone) is the most complex auth feature. It handles human onboarding, agent bootstrap, claim secrets, and approval flows. Defer to later unless multi-user access is a day-one requirement.

9. **Agent auth via run JWTs** is essential for the heartbeat model — each run gets a scoped token that limits what the agent can access and links actions to the specific run.

10. **Cross-company isolation** is enforced at every API endpoint via `assertCompanyAccess`. This is non-negotiable for multi-tenant deployments.
