# Real-time & Live Events — WebSocket, SSE, live dashboard updates

## Investigation Summary

Paperclip provides real-time updates via a company-scoped WebSocket server and an in-process event emitter. Every mutation (agent status change, heartbeat run progress, issue update, activity log entry) publishes a live event that WebSocket clients receive instantly. This powers the real-time dashboard, streaming run viewer, toast notifications, and sidebar badges — all without polling. The implementation is ~300 lines (server-side) plus ~600 lines (UI consumer).

---

## 1. Architecture

```
Service (mutation) → publishLiveEvent() → EventEmitter → WebSocket Server → Browser Client
                                              ↓
                                     Plugin Event Bus (optional)
```

### 1.1 In-Process Event Bus

```typescript
// server/src/services/live-events.ts (~55 lines)
const emitter = new EventEmitter();
emitter.setMaxListeners(0);  // Unlimited listeners (one per WebSocket client)

export function publishLiveEvent(input: {
  companyId: string;
  type: LiveEventType;
  payload?: Record<string, unknown>;
}) {
  const event = {
    id: nextEventId++,
    companyId: input.companyId,
    type: input.type,
    createdAt: new Date().toISOString(),
    payload: input.payload ?? {},
  };
  emitter.emit(input.companyId, event);  // Company-scoped
}

export function subscribeCompanyLiveEvents(companyId: string, listener: (event) => void) {
  emitter.on(companyId, listener);
  return () => emitter.off(companyId, listener);
}
```

Simple, effective, zero-dependency. Events are company-scoped — listeners only receive events for their company.

### 1.2 WebSocket Server

```typescript
// server/src/realtime/live-events-ws.ts (~250 lines)
```

Built on the `ws` library (loaded via `createRequire` to avoid ESM issues):
- **Endpoint:** `GET /api/companies/{companyId}/events/ws` (WebSocket upgrade)
- **Auth:** Bearer token (agent API key) or session cookie (board user) or `local_trusted` auto-auth
- **Company-scoped:** Each connection subscribes to one company's events
- **Ping/pong heartbeat:** 30-second interval, terminate unresponsive clients

---

## 2. Event Types

10 live event types (from `LIVE_EVENT_TYPES` constant):

| Event Type | When Published | Key Payload Fields |
|-----------|----------------|-------------------|
| `heartbeat.run.queued` | Run created/enqueued | `runId`, `agentId`, `source` |
| `heartbeat.run.status` | Run status changes | `runId`, `agentId`, `status`, `error`, `startedAt`, `finishedAt` |
| `heartbeat.run.event` | Each stdout/stderr/system event during run | `runId`, `seq`, `eventType`, `stream`, `message` |
| `heartbeat.run.log` | Log chunk (capped at 8KB) | `runId`, `chunk` |
| `agent.status` | Agent status changes | `agentId`, `status` |
| `activity.logged` | Every mutation in the activity log | `actorType`, `actorId`, `action`, `entityType`, `entityId`, `details` |
| `plugin.ui.updated` | Plugin UI state changed | `pluginId` |
| `plugin.worker.crashed` | Plugin worker crashed | `pluginId`, `error` |
| `plugin.worker.restarted` | Plugin worker restarted | `pluginId` |

---

## 3. Live Event Shape

```typescript
interface LiveEvent {
  id: number;           // Monotonically incrementing (in-process, not persisted)
  companyId: string;
  type: LiveEventType;
  createdAt: string;    // ISO 8601
  payload: Record<string, unknown>;
}
```

Events are **not persisted** — they exist only in-memory and are delivered to connected clients. If a client disconnects and reconnects, it misses events during the gap. The UI handles this by invalidating TanStack Query caches on reconnect.

---

## 4. WebSocket Authentication

### 4.1 Auth Methods

Three auth paths for WebSocket upgrade:

| Mode | Auth Method | How |
|------|-------------|-----|
| `local_trusted` | None | Auto-assigned board context |
| `authenticated` (cookie) | Better Auth session | Parse cookies from upgrade request headers |
| Bearer token | Agent API key | `Authorization: Bearer pcp_...` header or `?token=` query param |

### 4.2 Authorization Flow

```typescript
server.on("upgrade", async (req, socket, head) => {
  const companyId = parseCompanyId(url.pathname);  // /api/companies/{id}/events/ws
  const context = await authorizeUpgrade(db, req, companyId, url, opts);
  if (!context) {
    rejectUpgrade(socket, "403 Forbidden");
    return;
  }
  wss.handleUpgrade(req, socket, head, (ws) => {
    wss.emit("connection", ws, reqWithContext);
  });
});
```

Company membership is verified — agents can only subscribe to their own company's events.

### 4.3 Connection Lifecycle

```
Client → WS upgrade request → Auth check → Subscribe to company events
  ↕ ping/pong every 30s (detect dead connections)
  ← Live events streamed as JSON messages
  → Connection close / terminate
```

---

## 5. Where Events are Published

Events are published from across the codebase:

| Source | Events |
|--------|--------|
| `heartbeat.ts` | `heartbeat.run.queued`, `heartbeat.run.status`, `heartbeat.run.event`, `heartbeat.run.log` |
| `activity-log.ts` | `activity.logged` (every mutation) |
| `agents.ts` (via heartbeat) | `agent.status` |
| `plugin-worker-manager.ts` | `plugin.worker.crashed`, `plugin.worker.restarted` |
| `plugin-host-services.ts` | `plugin.ui.updated` |

### 5.1 High-Frequency Events

During an active heartbeat run, events can be very frequent:
- **`heartbeat.run.event`** — every stdout/stderr line from the agent process
- **`heartbeat.run.log`** — log chunks capped at 8KB per message

This enables the streaming run viewer in the UI.

---

## 6. UI Consumption

### 6.1 LiveUpdatesProvider (~600 lines)

React context provider that manages the WebSocket connection:

```typescript
// Connects to: /api/companies/{companyId}/events/ws
// Handles: reconnection, event dispatch, cache invalidation, toast notifications
```

### 6.2 Event Handling in UI

| Event | UI Action |
|-------|-----------|
| `heartbeat.run.queued` | Invalidate run list query, show toast |
| `heartbeat.run.status` | Invalidate run + agent queries, show toast (completed/failed) |
| `heartbeat.run.event` | Append to run event log (if viewing that run) |
| `heartbeat.run.log` | Append to streaming log viewer |
| `agent.status` | Invalidate agent list query |
| `activity.logged` | Invalidate activity + sidebar badge queries |
| `plugin.ui.updated` | Invalidate plugin queries |

### 6.3 Toast Notifications

Real-time events trigger toasts with:
- Agent name resolution (from query cache)
- Issue context (identifier, title)
- Rate limiting: max 3 toasts per 10-second window
- Suppress during reconnect (2-second suppress window)

### 6.4 Cache Invalidation

Events trigger TanStack Query cache invalidation:
```typescript
queryClient.invalidateQueries(queryKeys.heartbeats.runs(companyId));
queryClient.invalidateQueries(queryKeys.agents.list(companyId));
queryClient.invalidateQueries(queryKeys.sidebarBadges(companyId));
```

This keeps the UI fresh without polling.

---

## 7. Key Design Decisions

| Decision | Rationale |
|----------|-----------|
| In-process EventEmitter (not Redis/Kafka) | Single-process server; no need for external message broker. Simple and zero-latency. |
| Company-scoped events | Isolation — clients only receive events for their company |
| `ws` library (not Socket.IO) | Lightweight, standard WebSocket protocol, no fallback complexity |
| Non-persisted events | Events are transient signals for UI updates, not audit trail (that's `activity_log`) |
| Monotonic event ID (not UUID) | Fast, ordered, no persistence needed. Resets on server restart (acceptable). |
| Ping/pong heartbeat | Detects dead connections; browsers don't always close cleanly |
| Auth on upgrade | Verify once at connection time, not per message |
| Query token fallback | Browser WebSocket API doesn't support custom headers; `?token=` enables programmatic clients |
| Toast rate limiting | Prevents notification spam during rapid agent activity |

---

## 8. Observations for Colmena

1. **The in-process EventEmitter approach is perfect for single-server deployments.** No external dependencies, zero latency, ~55 lines of code. Adopt as-is.

2. **For multi-server deployments**, you'd need Redis Pub/Sub or similar to distribute events. But this is a later concern — single-server covers most Colmena use cases.

3. **The WebSocket auth** is well-handled — supports cookies (browser), bearer tokens (agents/scripts), and local_trusted (no auth). Worth replicating.

4. **Company-scoped events** are the right granularity — one subscription per company, not per entity. Keeps the event bus simple.

5. **For Colmena MVP**, implement: `publishLiveEvent()` + WebSocket server + `heartbeat.run.status` and `activity.logged` events. Add `heartbeat.run.event` (streaming logs) when the run viewer is built.

6. **The streaming run viewer** (powered by `heartbeat.run.event` + `heartbeat.run.log`) is one of Paperclip's most impressive UI features. It requires the real-time event system to work.

7. **Toast rate limiting** (max 3 per 10 seconds) is a small but important UX detail — without it, rapid agent activity floods the user with notifications.

8. **Non-persisted events are correct** — the activity log provides the audit trail. Live events are ephemeral UI signals. Don't over-engineer persistence here.

9. **The reconnect suppress window** (2 seconds) prevents a burst of stale toasts when the WebSocket reconnects after a brief disconnect.
