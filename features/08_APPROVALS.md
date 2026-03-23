# Approvals — hire requests, CEO strategy sign-off, revision workflow

## Investigation Summary

Approvals are Paperclip's governance layer — they keep the human board operator in control of key decisions. When an agent requests to hire a new subordinate, or when a CEO's strategy needs sign-off, the system creates an approval that must be resolved by the board before the action takes effect. Additionally, budget hard-stops create `budget_override_required` approvals. The system spans 2 core tables plus a join table, a ~250-line service, and integrates with agents, issues, budgets, secrets, and the heartbeat system.

---

## 1. Data Model

### 1.1 `approvals` Table

| Column | Type | Default | Notes |
|--------|------|---------|-------|
| `id` | UUID | random | PK |
| `companyId` | UUID | — | FK → companies |
| `type` | TEXT | — | Enum: `hire_agent`, `approve_ceo_strategy`, `budget_override_required` |
| `requestedByAgentId` | UUID | NULL | FK → agents — which agent requested this |
| `requestedByUserId` | TEXT | NULL | Which human requested this |
| `status` | TEXT | `"pending"` | Enum: `pending`, `revision_requested`, `approved`, `rejected`, `cancelled` |
| `payload` | JSONB | — | Required. Type-specific data (e.g., proposed agent config for hires) |
| `decisionNote` | TEXT | NULL | Board's explanation for the decision |
| `decidedByUserId` | TEXT | NULL | Who made the decision |
| `decidedAt` | TIMESTAMPTZ | NULL | When decision was made |
| `createdAt`, `updatedAt` | TIMESTAMPTZ | — | — |

**Index:** `(companyId, status, type)` — filter pending approvals by type.

### 1.2 `approval_comments` Table

Discussion thread on approvals:

| Column | Type | Notes |
|--------|------|-------|
| `id` | UUID | PK |
| `companyId` | UUID | FK → companies |
| `approvalId` | UUID | FK → approvals |
| `authorAgentId` | UUID | FK → agents (NULL for human authors) |
| `authorUserId` | TEXT | NULL for agent authors |
| `body` | TEXT | Markdown |
| `createdAt`, `updatedAt` | TIMESTAMPTZ | — |

**Indexes:** `(companyId)`, `(approvalId)`, `(approvalId, createdAt)`

### 1.3 `issue_approvals` Table (Join)

Links issues to approvals (covered in 05_ISSUES.md):

| Column | Type | Notes |
|--------|------|-------|
| `issueId` | UUID | FK → issues, CASCADE |
| `approvalId` | UUID | FK → approvals, CASCADE |
| `companyId` | UUID | FK → companies |
| `linkedByAgentId` | UUID | FK → agents |
| `linkedByUserId` | TEXT | — |

**PK:** `(issueId, approvalId)`. Supports many-to-many linking.

---

## 2. Approval Status Machine

```
pending → approved
        → rejected
        → revision_requested → (agent resubmits) → pending
        → cancelled
```

**Resolution rules:**
- Only `pending` and `revision_requested` approvals can be approved or rejected
- Only `pending` approvals can have revision requested
- Only `revision_requested` approvals can be resubmitted
- Resolution is idempotent — resolving an already-resolved approval returns it without error
- Concurrent resolution is handled with a conditional UPDATE (`WHERE status IN ('pending', 'revision_requested')`)

---

## 3. Approval Types

### 3.1 `hire_agent`

Created when an agent (typically CEO or a manager with `canCreateAgents`) requests to hire a subordinate.

**Payload:**
```typescript
{
  name: string,
  role: string,
  title: string | null,
  icon: string | null,
  reportsTo: string | null,
  capabilities: string | null,
  adapterType: string,
  adapterConfig: Record<string, unknown>,  // Redacted in responses
  runtimeConfig: Record<string, unknown>,  // Redacted in responses
  budgetMonthlyCents: number,
  desiredSkills: string[] | null,
  metadata: Record<string, unknown>,
  agentId: string,                         // Pre-created agent in pending_approval status
  requestedByAgentId: string | null,
  requestedConfigurationSnapshot: { ... }  // Full config for reference
}
```

**Flow:**
1. Agent calls `POST /api/companies/:companyId/agent-hires`
2. If `company.requireBoardApprovalForNewAgents` is true:
   - Agent is created with `status: "pending_approval"`
   - `hire_agent` approval is created with the proposed config
   - Source issues are linked to the approval
3. Board reviews and decides

**On approve:**
- If `payload.agentId` exists → activate the pending agent (`status: "idle"`)
- If no pre-created agent → create the agent from the payload
- Set up budget policy if `budgetMonthlyCents > 0`
- Call `notifyHireApproved()` — triggers the adapter's `onHireApproved` hook (e.g., sends a message to OpenClaw)

**On reject:**
- If `payload.agentId` exists → terminate the pending agent

### 3.2 `approve_ceo_strategy`

Created when the CEO proposes an initial strategy for the company. The board must approve before the CEO can start executing work.

**Flow:** Agent-driven — CEO creates the approval with its strategy as the payload.

### 3.3 `budget_override_required`

Created automatically by the budget system when a hard-stop threshold is exceeded.

**Flow:**
1. Budget service detects spend ≥ budget for a scope (company/agent/project)
2. Creates a `budget_override_required` approval
3. Links it to a `budget_incident`
4. Board can resolve via the budget incident resolution flow (`raise_budget_and_resume` or `keep_paused`)

---

## 4. API Surface

| Method | Path | Auth | Description |
|--------|------|------|-------------|
| `GET` | `/api/companies/:companyId/approvals` | Company access | List approvals (filterable by status) |
| `GET` | `/api/approvals/:id` | Company access | Get approval detail (payload redacted) |
| `POST` | `/api/companies/:companyId/approvals` | Company access | Create approval |
| `POST` | `/api/approvals/:id/approve` | Board only | Approve (triggers side effects) |
| `POST` | `/api/approvals/:id/reject` | Board only | Reject |
| `POST` | `/api/approvals/:id/request-revision` | Board only | Request revision |
| `POST` | `/api/approvals/:id/resubmit` | Requesting agent or board | Resubmit with updated payload |
| `GET` | `/api/approvals/:id/issues` | Company access | List linked issues |
| `GET` | `/api/approvals/:id/comments` | Company access | List comments |
| `POST` | `/api/approvals/:id/comments` | Company access | Add comment |

---

## 5. Side Effects

### 5.1 On Approval (approve)

For `hire_agent`:
1. Activate pending agent → `status: "idle"`
2. Create budget policy if budget > 0
3. Trigger adapter's `onHireApproved` hook (non-fatal)
4. Wake the requesting agent via heartbeat with:
   - `source: "automation"`
   - `reason: "approval_approved"`
   - `payload: { approvalId, approvalStatus, issueId, issueIds }`
   - Context includes `wakeReason: "approval_approved"`
5. Log activity: `approval.approved` + `approval.requester_wakeup_queued`

### 5.2 On Rejection (reject)

For `hire_agent`:
1. Terminate the pending agent
2. Log activity: `approval.rejected`

### 5.3 On Revision Request

1. Set status to `revision_requested`
2. Log activity: `approval.revision_requested`
3. Agent sees the revision request on next heartbeat and can resubmit

### 5.4 On Resubmit

1. Set status back to `pending`
2. Optionally update the payload (e.g., modified agent config)
3. Clear previous decision data (`decisionNote`, `decidedByUserId`, `decidedAt`)
4. Log activity: `approval.resubmitted`

---

## 6. Hire Hook

When a hire is approved, `notifyHireApproved()` calls the adapter's `onHireApproved` hook:

```typescript
interface HireApprovedPayload {
  companyId: string;
  agentId: string;
  agentName: string;
  adapterType: string;
  source: "join_request" | "approval";
  sourceId: string;
  approvedAt: string;
  message: "Tell your user that your hire was approved...";
}
```

This allows adapters (e.g., OpenClaw gateway) to notify the agent that it's been onboarded. Failures are non-fatal — logged but never thrown.

---

## 7. Issue-Approval Linking

The `issue_approvals` join table supports:
- **link** — connect an issue to an approval (idempotent via `ON CONFLICT DO NOTHING`)
- **unlink** — remove the connection
- **linkManyForApproval** — batch link multiple issues to one approval
- **listIssuesForApproval** — get all issues linked to an approval
- **listApprovalsForIssue** — get all approvals linked to an issue

This enables agents to create issues related to their hire request, and the board can see the full context when reviewing.

---

## 8. Payload Redaction

Approval payloads may contain sensitive data (adapter configs with secret references). All API responses pass through `redactApprovalPayload()` which calls `redactEventPayload()` to replace sensitive values with redacted placeholders.

For `hire_agent` approvals, the `adapterConfig` in the payload goes through `normalizeHireApprovalPayloadForPersistence()` which converts inline secrets to encrypted `secret_ref` objects before storage.

---

## 9. Heartbeat Protocol Integration

The Paperclip skill teaches agents to handle approvals during heartbeats:

**Step 2 of the heartbeat protocol:**
```
If PAPERCLIP_APPROVAL_ID is set:
  1. GET /api/approvals/{approvalId}
  2. GET /api/approvals/{approvalId}/issues
  3. For each linked issue:
     - Close it if the approval resolves the work
     - Or comment explaining why it remains open
```

Agents are woken when their approval is resolved, with `PAPERCLIP_APPROVAL_ID` and `PAPERCLIP_APPROVAL_STATUS` injected as environment variables.

---

## 10. Key Design Decisions

| Decision | Rationale |
|----------|-----------|
| Pre-create agent in `pending_approval` | Agent record exists for linking and display, but can't execute until approved |
| Payload as JSONB (not typed columns) | Different approval types have different payloads; JSONB is flexible |
| Revision workflow (request → resubmit → pending) | Board can request changes without rejecting outright; agent can modify and resubmit |
| Idempotent resolution | Prevents errors from double-click or concurrent resolve |
| Conditional UPDATE for resolution | Race-safe — only one resolution succeeds |
| Wake requesting agent on resolution | Agent needs to know the outcome to proceed with its work |
| Non-fatal hire hook | Notification failure shouldn't block the approval |
| Issue linking (many-to-many) | An approval may relate to multiple issues (context); an issue may have multiple approvals |
| Payload redaction in responses | Prevents leaking secrets in the approval UI |
| Budget overrides as approvals | Reuses the same governance UI and workflow for budget decisions |

---

## 11. Observations for Colmena

1. **The pre-created pending agent pattern is smart** — the agent exists in the system (visible in org chart, linkable to issues) before it's approved. On rejection, it's simply terminated.

2. **Three approval types is small** — `hire_agent` is the primary use case. `approve_ceo_strategy` and `budget_override_required` are secondary. For Colmena MVP, just `hire_agent` might suffice.

3. **The revision workflow adds nuance** — instead of just approve/reject, the board can ask for changes. This is more realistic for real governance.

4. **The wake-on-resolution pattern** is elegant — agents don't need to poll for approval status. They're automatically woken with the result injected as env vars.

5. **Payload redaction is critical** — approval payloads for hire requests contain full adapter configs, which may include secrets. Redacting in responses prevents UI leaks.

6. **The hire hook** (`onHireApproved`) is adapter-specific — OpenClaw can notify the agent via messaging, while local adapters might do nothing. This extensibility is good.

7. **For Colmena MVP**, approvals could be simplified to just a boolean gate on agent creation (skip the full revision workflow). Add the revision loop later.

8. **Comments on approvals** enable discussion between the board and agents about the proposal, which is important for governance transparency.

9. **The `requireBoardApprovalForNewAgents` flag** on the company is a clean toggle — companies can disable approval requirements if they want faster iteration.
