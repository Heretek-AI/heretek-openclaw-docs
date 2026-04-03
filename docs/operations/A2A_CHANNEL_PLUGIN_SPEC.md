# A2A Channel Plugin Specification

**Version:** 0.1.0 (Draft)
**Status:** Design — Not Yet Implemented
**Author:** Heretek OpenClaw Collective
**Created:** 2026-04-02

---

## 1. Overview

### Purpose

This document specifies a new OpenClaw **channel plugin** that intercepts all agent-to-agent (A2A) WebSocket messages exchanged between agents via the OpenClaw Gateway, logs them to PostgreSQL, and exposes them via a dashboard REST API.

The goal is to give the Heretek Control Dashboard a **live A2A feed** alongside the existing static communication graph (`/api/memory/graph`), enabling operators to observe deliberation flows, message routing, and inter-agent dependencies in real time.

### Scope

- Intercepts A2A WebSocket messages at the Gateway layer
- Writes structured metadata to PostgreSQL (never raw message content with secrets)
- Exposes a `GET /api/a2a/live` endpoint consumed by the dashboard's A2A page
- Requires zero changes to existing agent code

### Out of Scope

- Modifying agent SOUL.md, skills, or behavior
- Full message payload storage (security — see §6)
- Replacing the existing static graph endpoint
- Real-time WebSocket push to the dashboard (dashboard polls REST instead)

---

## 2. Gateway Architecture Context

The OpenClaw Gateway is a Go binary that:
1. Manages agent sessions (spawn/resume/kill)
2. Routes A2A messages between agents over WebSocket (`ws://localhost:18789`)
3. Exposes a JSON-RPC or REST control plane on its HTTP port

**The challenge:** The Go gateway binary does not natively support plugin hooks. A2A messages flow over an internal WebSocket router with no external tap point.

Three integration strategies are considered (§6 Open Questions).

---

## 3. Data Model

### Table: `a2a_messages`

```sql
CREATE TABLE IF NOT EXISTS a2a_messages (
  id            UUID        PRIMARY KEY DEFAULT gen_random_uuid(),
  logged_at     TIMESTAMPTZ NOT NULL DEFAULT now(),
  from_agent    TEXT        NOT NULL,
  to_agent      TEXT        NOT NULL,
  message_type  TEXT        NOT NULL,
  payload_summary JSONB,
  session_key   TEXT,
  routed_via    TEXT,
  -- internal tracking
  _gateway_ts   TIMESTAMPTZ,  -- original gateway timestamp if available
  _seq          BIGINT       -- per-session sequence number
);

CREATE INDEX IF NOT EXISTS idx_a2a_from     ON a2a_messages(from_agent);
CREATE INDEX IF NOT EXISTS idx_a2a_to       ON a2a_messages(to_agent);
CREATE INDEX IF NOT EXISTS idx_a2a_logged   ON a2a_messages(logged_at DESC);
CREATE INDEX IF NOT EXISTS idx_a2a_type     ON a2a_messages(message_type);
CREATE INDEX IF NOT EXISTS idx_a2a_session  ON a2a_messages(session_key);
```

### Field Semantics

| Field | Type | Description |
|-------|------|-------------|
| `id` | UUID | Primary key. Auto-generated. |
| `logged_at` | timestamptz | Wall-clock time the plugin logged this row. |
| `from_agent` | text | Source agent ID (e.g. `alpha`, `steward`). |
| `to_agent` | text | Destination agent ID or `*` for broadcast. |
| `message_type` | text | A2A protocol type: `message`, `status`, `proposal`, `broadcast`, `error`, etc. |
| `payload_summary` | jsonb | **Sanitized summary only** — see §6 Security. |
| `session_key` | text | OpenClaw session key (e.g. `agent:heretek:alpha`). |
| `routed_via` | text | Path indicator: `gateway_direct`, `websocket_proxy`, `plugin_hook`, etc. |
| `_gateway_ts` | timestamptz | Gateway-assigned timestamp, if extractable. |
| `_seq` | bigint | Per-session message sequence number. |

---

## 4. API Endpoints

### `GET /api/a2a/live`

Returns the last N A2A message events, most recent first.

**Query Parameters:**
| Param | Type | Default | Description |
|-------|------|---------|-------------|
| `limit` | int | 50 | Max messages to return (1–200). |
| `since` | ISO8601 | — | Return only messages after this timestamp. |

**Response:**
```json
{
  "messages": [
    {
      "id": "3f4a5b6c-...",
      "logged_at": "2026-04-02T20:45:00.123Z",
      "from_agent": "alpha",
      "to_agent": "sentinel",
      "message_type": "proposal",
      "payload_summary": {
        "type": "deliberation_proposal",
        "proposal_id": "p-001",
        "action": "implement_feature_x"
      },
      "session_key": "agent:heretek:alpha",
      "routed_via": "gateway_direct"
    }
  ],
  "total": 1,
  "oldest": "2026-04-02T20:45:00.123Z",
  "newest": "2026-04-02T20:45:00.123Z",
  "logged_at": "2026-04-02T20:45:01.000Z"
}
```

**Status codes:** `200 OK`, `500 Internal Server Error` (DB down).

---

### `GET /api/a2a/history`

Returns historical A2A messages with filtering.

**Query Parameters:**
| Param | Type | Description |
|-------|------|-------------|
| `from` | text | Filter by source agent ID. |
| `to` | text | Filter by destination agent ID. |
| `type` | text | Filter by message type. |
| `since` | ISO8601 | Return messages after this time. |
| `until` | ISO8601 | Return messages before this time. |
| `limit` | int | Max results (default 100, max 500). |
| `offset` | int | Pagination offset. |

**Response:** Same shape as `/api/a2a/live` with `offset` and `total_count` fields.

---

## 5. Dashboard Integration

The Heretek Control Dashboard's **A2A page** (`A2ACommunication.tsx`) already implements:

1. **Static graph mode** — fetches `/api/memory/graph`, displays the agent communication topology as an interactive SVG node-edge diagram.

2. **Live mode toggle** — when enabled, polls `/api/memory/graph` every 10 seconds and shows a live message feed.

Once this plugin is implemented, the live feed should switch from polling the static graph endpoint to `GET /api/a2a/live`:

```
A2ACommunication.tsx changes:
  1. useGraphData hook: when liveMode=true, fetch /api/a2a/live
  2. LiveFeed component: render A2ALiveMessage[] items instead of static graph stats
  3. Keep the graph view as a background topology reference
  4. Add a "Feed" / "Graph" / "Split" view mode toggle
```

**Fallback behavior:** If `/api/a2a/live` returns an empty array (interception not yet wired), show the current placeholder: *"No live messages yet — A2A interception is not yet wired."*

---

## 6. Security

### What is logged
- Message metadata: from, to, type, timestamp, session key
- A `payload_summary` JSONB field containing only **public structural data**

### What is NEVER logged
- API keys, tokens, or credentials
- Full message content or agent prompts
- File paths containing sensitive information
- User PII

### payload_summary Sanitization Rules

Before writing to `payload_summary`, the plugin must:

1. Accept the raw message JSON as input
2. Extract only known-safe fields (type, proposal_id, action, vote_count, etc.)
3. Strip any field whose key contains: `token`, `key`, `secret`, `password`, `credential`, `auth`, `pii`, `email`, `phone`
4. Replace stripped values with `"[REDACTED]"`
5. Discard the full message body entirely

### Access Control
- `GET /api/a2a/live` requires no authentication for dashboard use (internal network)
- For external exposure, add an `X-API-Key` header check

---

## 7. Open Questions

### Q1: Gateway Interception Approach

The core challenge is tapping into A2A WebSocket messages inside the Go gateway binary. Three approaches:

| Approach | Description | Pros | Cons |
|----------|-------------|------|------|
| **a) Gateway plugin hook** | Add a callback/hook in the gateway Go code that calls out to the health-api on each message. Requires gateway code changes + recompile. | Clean, low-latency | Requires Go development, must upstream or maintain a fork |
| **b) WebSocket proxy/mirror** | Deploy a transparent WebSocket proxy between agents and the gateway that mirrors all traffic to the health-api. Agents connect to the proxy instead of directly to the gateway. | No gateway changes needed | Extra hop, proxy becomes a SPOF, latency added |
| **c) Gateway state polling** | Poll the gateway's internal state via its control plane HTTP API every N seconds. Extract session/message stats. | Simplest to implement | Not real-time, misses individual messages, gateway must expose state endpoint |

**Recommended:** Approach (a) if gateway source access is available. Approach (b) as a near-term fallback.

### Q2: payload_summary Schema

Should the `payload_summary` JSONB schema be fixed or dynamic?

- **Fixed schema:** Define a set of known `A2AMessageSummary` fields (type, proposal_id, action, votes, etc.). Unknown message types write `{}`.
- **Dynamic schema:** Store all top-level JSON fields except banned keys. More information but harder to query.

**Recommendation:** Start with fixed schema, extend as needed.

### Q3: Data Retention

How long should `a2a_messages` rows be kept?

- Short-term (24h): Sufficient for dashboard live view
- Medium-term (7 days): Enables incident investigation
- Long-term (90 days): Compliance/audit

**Recommendation:** Default to 7-day retention with a daily cron job (`DELETE FROM a2a_messages WHERE logged_at < now() - INTERVAL '7 days'`). Make retention configurable via environment variable.

### Q4: Message Deduplication

Network retries or WebSocket reconnection may cause the same message to be logged multiple times. Consider:

- A `UNIQUE(from_agent, to_agent, message_type, _gateway_ts)` constraint with `ON CONFLICT DO NOTHING`
- A `_seq` field to detect duplicates at the application level

### Q5: Gateway Version Compatibility

If the gateway is updated, the message format or WebSocket protocol may change. The plugin should:

- Log the gateway version at startup
- Validate message structure before inserting; drop unparseable messages with a warning
- Emit a metric (`a2a_messages_dropped_total`) for observability

---

## 8. Implementation Checklist

### Phase 1 — Infrastructure (this spec)
- [x] Create SQL schema: `002_a2a_message_log.sql`
- [x] Document API endpoint shape
- [x] Document security rules

### Phase 2 — API Endpoint
- [ ] Add `GET /api/a2a/live` to `health-api.js`
- [ ] Add `GET /api/a2a/history` to `health-api.js`
- [ ] Write migration runner for `002_a2a_message_log.sql`
- [ ] Add retention cleanup cron job

### Phase 3 — Interception
- [ ] Choose interception approach (Q1)
- [ ] Implement gateway plugin OR WebSocket proxy
- [ ] Add payload sanitization
- [ ] Add deduplication logic
- [ ] Test with live triad deliberation

### Phase 4 — Dashboard
- [ ] Wire `A2ACommunication.tsx` to `GET /api/a2a/live` in live mode
- [ ] Add "Feed / Graph / Split" view toggle
- [ ] Add message type filter chips
- [ ] Add "since" auto-refresh from user scroll position

---

*Document maintained by Steward. Update this spec before implementing Phase 2.*
