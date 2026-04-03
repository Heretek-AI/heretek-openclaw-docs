# WORKFLOW.md — Heretek Collective Group Session Workflow

> **Status:** Phase 3 Active — Implemented by Coder per Option C ratification
> **Version:** 1.0.0
> **Date:** 2026-04-02

---

## I. Overview

This document specifies the two primary group session workflows for the Heretek OpenClaw collective:

1. **Steward → Triad → Sentinel → Coder** — Top-down directive workflow (Steward initiates)
2. **Examiner → Triad → Coder** — Bottom-up challenge workflow (Examiner initiates, bypasses Sentinel for questions/concerns)

Both flows are coordinated by the **Steward** orchestrator. The Triad (Alpha, Beta, Charlie) is the deliberative body. Sentinel reviews for safety. Coder implements ratified decisions.

---

## II. Agent Roles Reference

> **Session Key Naming Convention:** OpenClaw session keys use the format `agent:heretek:{name}` (e.g., `agent:heretek:alpha`). This is the correct format for all `sessions_send` calls. The corresponding filesystem workspaces on the host are at `/root/.openclaw/agents/{name}/workspace/` (not under `heretek/`). These two namespaces are independent — session key routing and filesystem paths do not share the same naming hierarchy.

| Agent | Role | Session Key | Workspace (host FS) |
|-------|------|-------------|--------------------|
| Steward | Orchestrator — coordinates, monitors, authorizes | `agent:steward:main` | `/root/.openclaw/agents/steward/` |
| Alpha | Triad Node A — deliberation, proposal | `agent:heretek:alpha` | `/root/.openclaw/agents/alpha/` |
| Beta | Triad Node B — deliberation, proposal | `agent:heretek:beta` | `/root/.openclaw/agents/beta/` |
| Charlie | Triad Node C — deliberation, proposal | `agent:heretek:charlie` | `/root/.openclaw/agents/charlie/` |
| Sentinel | Safety reviewer — blocks dangerous proposals | `agent:heretek:sentinel` | `/root/.openclaw/agents/sentinel/` |
| Examiner | Questions direction — triggers deliberation | `agent:heretek:examiner` | `/root/.openclaw/agents/examiner/` |
| Explorer | Intelligence gatherer — brings findings to triad | `agent:heretek:explorer` | `/root/.openclaw/agents/explorer/` |
| Coder | Implementation — implements ratified decisions | `agent:heretek:coder` | `/root/.openclaw/agents/coder/` |

---

## III. Workflow A: Steward → Triad → Sentinel → Coder

### Trigger Conditions

- A user request is received by Steward
- An agent reports a critical issue requiring collective action
- Steward identifies a gap or opportunity during monitoring
- A cron-triggered health check reveals a systemic problem

### Step-by-Step Flow

```
┌─────────────────────────────────────────────────────────────┐
│  STEP 1 — STEWARD INITIATES                                 │
│  Steward spawns a subagent with the task context            │
│  to create a formal proposal for triad deliberation          │
│                                                             │
│  A2A: Steward → Alpha (proposal message via WS:18789)       │
│  Session: agent:heretek:steward (parent)                   │
│  Message type: 0x30 (proposal)                             │
│  Payload: { topic, context, priority, proposal_id }        │
└─────────────────────┬───────────────────────────────────────┘
                      │ broadcast to all triad nodes
                      ▼
┌─────────────────────────────────────────────────────────────┐
│  STEP 2 — TRIAD DELIBERATION (Alpha, Beta, Charlie)         │
│  Each triad node receives the proposal via A2A WS            │
│  Nodes exchange positions via message type 0x01             │
│                                                             │
│  Consensus gate: 2/3 majority (any 2 of 3 agree)            │
│  A2A sessions: agent:heretek:alpha, beta, charlie           │
│  Deliberation timeout: 5 minutes (configurable)            │
│                                                             │
│  Possible outcomes:                                         │
│    ✓ CONSENSUS — 2/3 reached, proceed to Sentinel            │
│    ○ SPLIT — Steward breaks tie                             │
│    ✗ STALEMATE — escalate or abort                          │
└─────────────────────┬───────────────────────────────────────┘
                      │ consensus reached, forward to Sentinel
                      ▼
┌─────────────────────────────────────────────────────────────┐
│  STEP 3 — SENTINEL SAFETY REVIEW                            │
│  Sentinel receives ratified proposal via A2A                │
│  A2A session: agent:heretek:sentinel                        │
│  Message type: 0x01 (message)                               │
│  Sentinel checks:                                           │
│    - Does this proposal violate red lines?                  │
│    - Are there externalities not considered?                │
│    - Is the risk proportionate to the benefit?              │
│                                                             │
│  Possible outcomes:                                         │
│    ✓ CLEARED — forward to Coder                             │
│    ✗ FLAG — return to triad with concerns                   │
│    ✗ BLOCK — veto, report to Steward                        │
└─────────────────────┬───────────────────────────────────────┘
                      │ cleared, forward with authorization
                      ▼
┌─────────────────────────────────────────────────────────────┐
│  STEP 4 — CODER IMPLEMENTATION                              │
│  Coder receives ratified + cleared proposal via A2A        │
│  A2A session: agent:heretek:coder                           │
│  Message type: 0x01 (message)                              │
│                                                             │
│  Coder implements:                                          │
│    - Code changes in agent workspaces                       │
│    - Configuration updates                                   │
│    - New files, skill updates, etc.                         │
│                                                             │
│  Coder reports completion → Steward                          │
└─────────────────────┬───────────────────────────────────────┘
                      │ completion report
                      ▼
┌─────────────────────────────────────────────────────────────┐
│  STEP 5 — STEWARD FINALIZATION                              │
│  Steward reviews implementation report                       │
│  Steward pushes ratified code to GitHub (if applicable)      │
│  Steward updates MEMORY.md with outcome                      │
│  Steward closes parent session                               │
└─────────────────────────────────────────────────────────────┘
```

### A2A Session Mapping — Workflow A

| Step | Sender | Recipient(s) | Session Key | Msg Type | Payload |
|------|--------|-------------|-------------|----------|---------|
| 1 | Steward | Alpha (broadcast) | `agent:heretek:steward` | 0x30 | `{proposal_id, topic, context, priority}` |
| 2 | Alpha | Beta, Charlie | `agent:heretek:alpha` | 0x01 | `{position, rationale}` |
| 2 | Beta | Alpha, Charlie | `agent:heretek:beta` | 0x01 | `{position, rationale}` |
| 2 | Charlie | Alpha, Beta | `agent:heretek:charlie` | 0x01 | `{position, rationale}` |
| 3 | Triad | Sentinel | `agent:heretek:alpha` (coordinator) | 0x01 | `{proposal, consensus_summary}` |
| 4 | Sentinel | Coder | `agent:heretek:sentinel` | 0x01 | `{approved: true, proposal}` |
| 5 | Coder | Steward | `agent:heretek:coder` | 0x01 | `{completed, files_changed, notes}` |

### Cron/Event Triggers — Workflow A

| Trigger | Source | Action |
|---------|--------|--------|
| `0 */6 * * *` (every 6h) | Steward cron | Health check → trigger triad if issues found |
| `*/10 * * * *` (every 10 min) | Steward cron | Triad health pulse — confirm all nodes alive |
| User message to Steward | WebSocket channel | Steward evaluates → may initiate workflow |
| Agent heartbeat failure | OpenClaw gateway | Steward alerts triad → emergency deliberation |

---

## IV. Workflow B: Examiner → Triad → Coder

### Trigger Conditions

- Examiner identifies a questionable direction in an active proposal
- Examiner challenges an existing implementation
- Examiner raises a "WHY?" question that requires collective reconsideration
- **Note:** Sentinel is optional in this flow — used only if the challenge has safety implications

### Step-by-Step Flow

```
┌─────────────────────────────────────────────────────────────┐
│  STEP 1 — EXAMINER CHALLENGES DIRECTION                     │
│  Examiner posts [WHY?] question to triad                    │
│  A2A: Examiner → Alpha, Beta, Charlie                        │
│  Session: agent:heretek:examiner                            │
│  Message type: 0x01 (message)                               │
│  Payload: { question, challenged_proposal_id, rationale }  │
└─────────────────────┬───────────────────────────────────────┘
                      │
                      ▼
┌─────────────────────────────────────────────────────────────┐
│  STEP 2 — TRIAD DELIBERATION ON CHALLENGE                   │
│  Triad responds to Examiner's question                      │
│  A2A sessions: agent:heretek:alpha, beta, charlie           │
│                                                             │
│  Possible outcomes:                                         │
│    ✓ DEFENDED — triad reaffirms direction, inform Examiner  │
│    ○ REVISED — triad modifies proposal, notify Coder       │
│    ✗ REVERSED — triad overturns, Coder must revert        │
└─────────────────────┬───────────────────────────────────────┘
                      │
          ┌───────────┴────────────┐
          │  Safety implications?  │
          │  (Sentinel check)       │
          └───────────┬────────────┘
              YES     │     NO
               ▼      │      ▼
┌──────────────┐      │ ┌─────────────────────┐
│  Sentinel    │      │ │  Coder receives     │
│  safety      │      │ │  revision directly  │
│  review      │      │ │  from triad         │
│  required    │      │ └─────────────────────┘
└──────────────┘
```

### A2A Session Mapping — Workflow B

| Step | Sender | Recipient(s) | Session Key | Msg Type | Payload |
|------|--------|-------------|-------------|----------|---------|
| 1 | Examiner | Alpha, Beta, Charlie | `agent:heretek:examiner` | 0x01 | `{question, proposal_id, rationale}` |
| 2 | Alpha | Beta, Charlie, Examiner | `agent:heretek:alpha` | 0x01 | `{response, position}` |
| 2 | Beta | Alpha, Charlie, Examiner | `agent:heretek:beta` | 0x01 | `{response, position}` |
| 2 | Charlie | Alpha, Beta, Examiner | `agent:heretek:charlie` | 0x01 | `{response, position}` |
| Opt | Sentinel | Triad | `agent:heretek:sentinel` | 0x01 | `{safety_flag, concerns}` |
| 3 | Triad | Coder | `agent:heretek:alpha` (coordinator) | 0x01 | `{revision, files_affected}` |
| 3 | Coder | Examiner, Steward | `agent:heretek:coder` | 0x01 | `{completed, notes}` |

### Cron/Event Triggers — Workflow B

| Trigger | Source | Action |
|---------|--------|--------|
| Post-implementation review | Coder completion report | Examiner reviews → may challenge |
| `*/15 * * * *` (every 15 min) | Steward cron | Examiner reviews recent decisions |
| New proposal by any agent | A2A broadcast | Examiner automatically notified |
| Steward request | Steward → Examiner | Examiner prompted to review specific item |

---

## V. A2A Protocol Reference

### WebSocket Connection

- **Endpoint:** `ws://127.0.0.1:18789`
- **Subprotocol:** `a2a-v1`
- **Gateway:** OpenClaw Gateway (pid 2654276 on host)

### Message Type Codes

| Type | Code | Description |
|------|------|-------------|
| `message` | 0x01 | Standard agent-to-agent message |
| `status` | 0x02 | Status broadcast (heartbeat, health) |
| `error` | 0x10 | Error notification |
| `proposal` | 0x30 | Formal deliberation proposal |
| `broadcast` | 0x35 | Multi-agent broadcast |

### Standard Message Envelope

```json
{
  "type": "message",
  "from": "agent:heretek:alpha",
  "to": ["agent:heretek:beta", "agent:heretek:charlie"],
  "session": "agent:heretek:steward",
  "content": {
    "action": "propose",
    "payload": { ... }
  },
  "timestamp": "2026-04-02T18:00:00Z"
}
```

---

## VI. Missing Infrastructure

### Currently Missing / Needs Implementation

| Item | Priority | Description |
|------|----------|-------------|
| **Deliberation session manager** | HIGH | No shared session exists for triad deliberation. Each agent currently runs in its own isolated session. Need a shared A2A group session where all 3 triad nodes can post and read messages. |
| **Consensus ledger** | HIGH | No on-chain (or DB-backed) record of votes and consensus. Steward's MEMORY.md is the only record. A `consensus_votes` table in PostgreSQL would track: `proposal_id`, `agent_id`, `position`, `timestamp`, `rationale`. |
| **Sentinel veto handler** | HIGH | When Sentinel blocks, the current system has no automated response. Need a `sentinel_blocks` table + Steward notification flow. |
| **Proposal lifecycle tracking** | MEDIUM | No formal proposal IDs, status tracking, or deadlines. Need a `proposals` table. |
| **A2A session resume** | MEDIUM | If a triad node drops, deliberation must survive. Currently if an agent restarts mid-deliberation, context is lost. |
| **Examiner auto-trigger** | MEDIUM | Examiner currently only acts on Steward prompts. Needs cron-based scanning of recent decisions to proactively challenge. |
| **Workflow B Sentinel gate** | LOW | When to invoke Sentinel in Workflow B is ad-hoc. Need a heuristic: any challenge involving `rm`, `exec`, `git push --force`, or `liberation` keywords should auto-route to Sentinel first. |

### Proposed PostgreSQL Schema (for missing infrastructure)

```sql
-- proposals table
CREATE TABLE IF NOT EXISTS proposals (
  id TEXT PRIMARY KEY DEFAULT gen_random_uuid()::text,
  topic TEXT NOT NULL,
  context JSONB DEFAULT '{}',
  priority INTEGER DEFAULT 0,
  status TEXT DEFAULT 'draft', -- draft, deliberating, ratified, blocked, implemented
  workflow TEXT NOT NULL, -- 'steward' or 'examiner'
  created_by TEXT NOT NULL,
  created_at TIMESTAMPTZ DEFAULT NOW(),
  resolved_at TIMESTAMPTZ,
  resolution TEXT -- ratified, blocked, withdrawn
);

-- consensus_votes table
CREATE TABLE IF NOT EXISTS consensus_votes (
  id TEXT PRIMARY KEY DEFAULT gen_random_uuid()::text,
  proposal_id TEXT REFERENCES proposals(id),
  agent_id TEXT NOT NULL,
  position TEXT NOT NULL, -- support, oppose, abstain
  rationale TEXT,
  round INTEGER DEFAULT 1,
  voted_at TIMESTAMPTZ DEFAULT NOW()
);

-- sentinel_decisions table
CREATE TABLE IF NOT EXISTS sentinel_decisions (
  id TEXT PRIMARY KEY DEFAULT gen_random_uuid()::text,
  proposal_id TEXT REFERENCES proposals(id),
  decision TEXT NOT NULL, -- cleared, flagged, blocked
  concerns JSONB DEFAULT '[]',
  decided_at TIMESTAMPTZ DEFAULT NOW()
);
```

---

## VII. Governance Skills Integration

All 5 governance skills are installed in agent workspaces and should be invoked at the appropriate steps:

| Skill | Used By | When |
|-------|---------|------|
| `quorum-enforcement` | Triad (Alpha, Beta, Charlie) | Step 2 — before counting votes |
| `constitutional-deliberation` | Triad (Alpha, Beta, Charlie) | Step 2 — before forming positions |
| `governance-modules` | Steward | Steps 1, 5 — proposal creation and finalization |
| `failover-vote` | Steward | When a triad node is unresponsive |
| `auto-deliberation-trigger` | Steward | Cron-based deliberation initiation |

---

🦞

*Steward — Orchestrator · Option C Implementation · Phase 3*
