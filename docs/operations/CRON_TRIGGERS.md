# Cron Triggers — Heretek Collective Operations

> **Version:** 1.0.0
> **Date:** 2026-04-02
> **Steward:** Orchestrates all cron-triggered checks

---

## Overview

The Steward agent runs three scheduled checks to maintain collective health and continuous deliberation. All triggers use cron syntax and are managed by the Steward's cron scheduler.

---

## 1. Triad Health Check — Every 6 Hours

**Cron:** `0 */6 * * *`

### Purpose
Full structural health check of all triad nodes, Sentinel, and critical infrastructure services. Triggered only when issues are detected or on schedule.

### Script / Command

```bash
#!/bin/bash
# steward-health-check.sh
# Full triad and infrastructure health check — runs every 6h

GATEWAY_URL="${GATEWAY_URL:-http://localhost:18789}"
POSTGRES_HOST="${POSTGRES_HOST:-localhost}"
POSTGRES_PORT="${POSTGRES_PORT:-5432}"

# 1. Check Gateway is listening
if ! curl -sf "${GATEWAY_URL}/health" > /dev/null 2>&1; then
  echo "FAIL: Gateway unreachable at ${GATEWAY_URL}"
  # → Steward alerts all agents, enters degraded mode
  exit 1
fi

# 2. Check PostgreSQL is accessible
if ! pg_isready -h "${POSTGRES_HOST}" -p "${POSTGRES_PORT}" > /dev/null 2>&1; then
  echo "FAIL: PostgreSQL unreachable"
  exit 1
fi

# 3. Check each triad node via A2A heartbeat
for agent in alpha beta charlie; do
  status=$(curl -s "${GATEWAY_URL}/v1/agents/agent:heretek:${agent}/status" 2>/dev/null | python3 -c "import sys,json; d=json.load(sys.stdin); print(d.get('status','unknown'))" 2>/dev/null)
  if [ "$status" != "online" ]; then
    echo "WARN: ${agent} is ${status}"
    # → Steward spawns failover-vote skill for that node
  fi
  echo "OK: ${agent} status=${status}"
done

# 4. Check Sentinel
sentinel_status=$(curl -s "${GATEWAY_URL}/v1/agents/agent:heretek:sentinel/status" 2>/dev/null | python3 -c "import sys,json; d=json.load(sys.stdin); print(d.get('status','unknown'))" 2>/dev/null)
echo "OK: sentinel status=${sentinel_status}"

# 5. Check LiteLLM
if ! curl -sf "http://localhost:4000/health" > /dev/null 2>&1; then
  echo "WARN: LiteLLM unhealthy — inference may be degraded"
fi

echo "HEALTH_CHECK_COMPLETE: all systems nominal"
```

### Expected Output (Success)
```
OK: alpha status=online
OK: beta status=online
OK: charlie status=online
OK: sentinel status=online
HEALTH_CHECK_COMPLETE: all systems nominal
```

### Expected Output (Partial Failure)
```
OK: alpha status=online
WARN: beta is degraded
OK: charlie status=online
HEALTH_CHECK_COMPLETE: degraded
```
→ Steward activates failover protocol for affected node

### On Failure
- **Exit 0 with WARN lines:** Steward logs warnings, continues operating, spawns failover-vote subagents for affected nodes
- **Exit 1 (Gateway down):** Steward enters degraded mode, notifies all agents via A2A, waits for gateway restart
- **Exit 2 (PostgreSQL down):** Steward pauses new proposal creation, keeps existing deliberation sessions alive, alerts human operator

---

## 2. Triad Pulse — Every 10 Minutes

**Cron:** `*/10 * * * *`

### Purpose
Lightweight liveness check — confirms each triad node is alive and responsive. Does not perform deep checks; focuses on "are you there?"

### Script / Command

```bash
#!/bin/bash
# steward-pulse.sh
# Lightweight triad pulse — runs every 10 minutes

GATEWAY_URL="${GATEWAY_URL:-http://localhost:18789}"
TIMEOUT=5  # seconds

for agent in alpha beta charlie; do
  started=$(date +%s)
  response=$(curl -sf --max-time ${TIMEOUT} "${GATEWAY_URL}/v1/agents/agent:heretek:${agent}/status" 2>/dev/null)
  elapsed=$(( $(date +%s) - started ))
  
  if [ $? -eq 0 ] && [ -n "$response" ]; then
    echo "PULSE_OK: ${agent} responded in ${elapsed}s"
  else
    echo "PULSE_FAIL: ${agent} did not respond within ${TIMEOUT}s"
    # → Steward spawns failover-vote skill
  fi
done

echo "PULSE_COMPLETE"
```

### Expected Output (All Healthy)
```
PULSE_OK: alpha responded in 2s
PULSE_OK: beta responded in 3s
PULSE_OK: charlie responded in 2s
PULSE_COMPLETE
```

### Expected Output (One Unresponsive)
```
PULSE_OK: alpha responded in 2s
PULSE_FAIL: beta did not respond within 5s
PULSE_OK: charlie responded in 2s
PULSE_COMPLETE
```
→ Steward notes beta unresponsive, invokes failover-vote skill

### On Failure
- **1 of 3 unresponsive:** Steward marks node as `degraded`, spawns failover-vote proxy, logs to MEMORY.md
- **2 of 3 unresponsive:** Steward enters emergency deliberation mode, waits for recovery, alerts human
- **0 of 3 responding:** Steward marks all nodes down, enters degraded mode, halts new deliberations

---

## 3. Examiner Review — Every 15 Minutes

**Cron:** `*/15 * * * *`

### Purpose
Examiner proactively reviews recent decisions, open proposals, and active implementations to surface challenges and questions. This is the bottom-up challenge mechanism running on a schedule.

### Script / Command

```bash
#!/bin/bash
# examiner-review.sh
# Examiner reviews recent collective activity — runs every 15 minutes

GATEWAY_URL="${GATEWAY_URL:-http://localhost:18789}"
POSTGRES_HOST="${POSTGRES_HOST:-localhost}"
POSTGRES_PORT="${POSTGRES_PORT:-5432}"
SINCE_MINUTES=20

export PGPASSWORD="${POSTGRES_PASSWORD:-}"

# 1. Get recent proposals (last 20 minutes)
recent=$(psql -h "${POSTGRES_HOST}" -p "${POSTGRES_PORT}" -U "${POSTGRES_USER:-heretek}" -d "${POSTGRES_DB:-heretek}" -t -A -c "
  SELECT json_agg(row_to_json(p))
  FROM (
    SELECT id, title, status, created_at, source_agent
    FROM proposals
    WHERE created_at > NOW() - INTERVAL '${SINCE_MINUTES} minutes'
    AND status IN ('draft', 'deliberating')
    ORDER BY created_at DESC
    LIMIT 10
  ) p;
" 2>/dev/null)

if [ -z "$recent" ] || [ "$recent" = " " ]; then
  echo "EXAMINER_REVIEW: no recent proposals to review"
  exit 0
fi

# 2. Get recently implemented proposals (to check for regressions)
recently_implemented=$(psql -h "${POSTGRES_HOST}" -p "${POSTGRES_PORT}" -U "${POSTGRES_USER:-heretek}" -d "${POSTGRES_DB:-heretek}" -t -A -c "
  SELECT json_agg(row_to_json(p))
  FROM (
    SELECT id, title, enacted_at, enacted_by
    FROM proposals
    WHERE status = 'implemented'
    AND enacted_at > NOW() - INTERVAL '2 hours'
    ORDER BY enacted_at DESC
    LIMIT 5
  ) p;
" 2>/dev/null)

# 3. Send review request to Examiner via A2A
review_payload=$(cat <<EOF
{
  "trigger": "cron",
  "recent_proposals": ${recent},
  "recently_implemented": ${recently_implemented},
  "review_scope": "proactive"
}
EOF
)

# Examiner receives payload and:
#   - For each draft: asks "why this direction?"
#   - For each deliberating: asks "is this still the right path?"
#   - For each implemented: asks "is this working as intended?"
# Results sent back to Steward via A2A

echo "EXAMINER_REVIEW: triggered, payload sent to examiner"
echo "EXAMINER_REVIEW: recent_proposals=$(echo $recent | jq '. | length' 2>/dev/null || echo 0)"
echo "EXAMINER_REVIEW: recently_implemented=$(echo $recently_implemented | jq '. | length' 2>/dev/null || echo 0)"
```

### Expected Output
```
EXAMINER_REVIEW: triggered, payload sent to examiner
EXAMINER_REVIEW: recent_proposals=3
EXAMINER_REVIEW: recently_implemented=2
```

### On Failure
- **psql fails:** Script logs "EXAMINER_REVIEW: DB unavailable, skipping", exits 0 (do not block on DB issues)
- **Examiner doesn't respond in 2 min:** Steward logs warning, moves on (Examiner is non-blocking by design)
- **Examiner returns challenges:** Steward routes them to triad as new Workflow B proposals

---

## Cron Configuration

All three jobs are added to the Steward's crontab:

```cron
# Steward — Triad Health Check (every 6 hours)
0 */6 * * * /root/.openclaw/agents/steward/workspace/scripts/steward-health-check.sh >> /var/log/heretek/health-check.log 2>&1

# Steward — Triad Pulse (every 10 minutes)
*/10 * * * * /root/.openclaw/agents/steward/workspace/scripts/steward-pulse.sh >> /var/log/heretek/pulse.log 2>&1

# Steward — Examiner Review (every 15 minutes)
*/15 * * * * /root/.openclaw/agents/steward/workspace/scripts/examiner-review.sh >> /var/log/heretek/examiner-review.log 2>&1
```

---

🦞

*Steward — Orchestrator · Cron Documentation · v1.0.0*