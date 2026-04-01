# Orchestration Skills Guide

**Version:** 1.0  
**Last Updated:** 2026-04-01  
**Scope:** Agent Lifecycle Manager, Triad Orchestrator, Gateway Pulse

---

## Table of Contents

1. [Agent Lifecycle Manager](#agent-lifecycle-manager)
2. [Triad Orchestrator](#triad-orchestrator)
3. [Gateway Pulse](#gateway-pulse)
4. [Prometheus Metrics Integration](#prometheus-metrics-integration)
5. [Alert Configuration](#alert-configuration)

---

## Agent Lifecycle Manager

**Skill Path:** [`heretek-openclaw-core/.roo/skills/agent-lifecycle-manager/SKILL.md`](../../../heretek-openclaw-core/.roo/skills/agent-lifecycle-manager/SKILL.md)

### Overview

The Agent Lifecycle Manager provides unified management for all OpenClaw agent processes. It supports batch operations, rolling restarts, health-based auto-restart, and dependency-aware startup sequences.

### Architecture

```
┌─────────────────────────────────────────────────────────┐
│                  Agent Lifecycle Manager                 │
├─────────────────────────────────────────────────────────┤
│  ┌─────────────────┐  ┌─────────────────┐               │
│  │ agent-controller│  │ health-monitor  │               │
│  │     .js         │  │     .js         │               │
│  └─────────────────┘  └─────────────────┘               │
│  ┌─────────────────┐                                    │
│  │  index.js       │ ← Main orchestration               │
│  └─────────────────┘                                    │
└─────────────────────────────────────────────────────────┘
```

### Usage Examples

#### Starting Agents

```bash
# Start all agents in dependency order
./scripts/lifecycle-manager.sh start-all

# Output:
# Starting agents in dependency order...
# [OK] gateway (infrastructure)
# [OK] litellm (infrastructure)
# [OK] steward (triad)
# [OK] alpha (triad)
# [OK] beta (triad)
# [OK] scout (worker)
# [OK] artisan (worker)
# [OK] guardian (worker)
# All agents started successfully.
```

```bash
# Start specific agents with health verification
./scripts/lifecycle-manager.sh start --agents steward,alpha --verify-health

# Output:
# Starting agents: steward, alpha
# [OK] steward started, health check passed (200ms)
# [OK] alpha started, health check passed (180ms)
```

#### Stopping Agents

```bash
# Stop all agents in reverse dependency order
./scripts/lifecycle-manager.sh stop-all

# Output:
# Stopping agents in reverse dependency order...
# [OK] guardian stopped
# [OK] artisan stopped
# [OK] scout stopped
# [OK] beta stopped
# [OK] alpha stopped
# [OK] steward stopped
# [OK] litellm stopped
# [OK] gateway stopped
```

#### Rolling Restart

```bash
# Perform rolling restart with 5-second delay between agents
./scripts/lifecycle-manager.sh rolling-restart --delay 5

# Output:
# Starting rolling restart (delay: 5s)...
# [1/8] Restarting gateway... OK (health: 200ms)
# [2/8] Restarting litellm... OK (health: 150ms)
# [3/8] Restarting steward... OK (health: 180ms)
# ...
# Rolling restart completed. Downtime: 0s
```

#### Status Dashboard

```bash
# View full status dashboard
./scripts/lifecycle-manager.sh dashboard

# Output:
# ╔══════════════════════════════════════════════════════╗
# ║           AGENT LIFECYCLE DASHBOARD                   ║
# ╠══════════════════════════════════════════════════════╣
# ║ Agent       │ Status  │ Uptime   │ Health │ Memory  ║
# ╠─────────────┼─────────┼──────────┼────────┼─────────╣
# ║ gateway     │ running │ 2d 4h    │ ✓ 45ms │ 128MB   ║
# ║ litellm     │ running │ 2d 4h    │ ✓ 32ms │ 256MB   ║
# ║ steward     │ running │ 1d 12h   │ ✓ 89ms │ 512MB   ║
# ║ alpha       │ running │ 1d 12h   │ ✓ 76ms │ 498MB   ║
# ║ beta        │ running │ 1d 12h   │ ✓ 82ms │ 501MB   ║
# ║ scout       │ stopped │ -        │ ✗ -    │ -       ║
# ║ artisan     │ running │ 12h      │ ✓ 95ms │ 445MB   ║
# ║ guardian    │ running │ 1d       │ ✓ 88ms │ 467MB   ║
# ╚══════════════════════════════════════════════════════╝
# Summary: 7/8 running, Health: 95%
```

#### Health Monitoring with Auto-Restart

```bash
# Start monitoring with auto-restart enabled
./scripts/lifecycle-manager.sh monitor --auto-restart --interval 30

# Output:
# Starting health monitor (interval: 30s, auto-restart: enabled)
# Monitor PID: 12345
# Press Ctrl+C to stop monitoring
```

### Dependency Order

Agents are started in this order:

| Order | Agent | Type | Dependencies |
|-------|-------|------|--------------|
| 1 | gateway | Infrastructure | None |
| 2 | litellm | Infrastructure | gateway |
| 3 | steward | Triad | gateway, litellm |
| 4 | alpha | Triad | gateway, litellm |
| 5 | beta | Triad | gateway, litellm |
| 6 | scout | Worker | triad agents |
| 7 | artisan | Worker | triad agents |
| 8 | guardian | Worker | triad agents |

---

## Triad Orchestrator

**Skill Path:** [`heretek-openclaw-core/.roo/skills/triad-orchestrator/SKILL.md`](../../../heretek-openclaw-core/.roo/skills/triad-orchestrator/SKILL.md)

### Overview

The Triad Orchestrator manages governance deliberation workflows, including proposal lifecycle tracking, vote collection, deadlock detection, and consensus ledger synchronization.

### Architecture

```
┌─────────────────────────────────────────────────────────┐
│                   Triad Orchestrator                     │
├─────────────────────────────────────────────────────────┤
│  ┌─────────────────┐  ┌─────────────────┐               │
│  │proposal-tracker │  │ vote-collector  │               │
│  │     .js         │  │     .js         │               │
│  └─────────────────┘  └─────────────────┘               │
│  ┌─────────────────┐  ┌─────────────────┐               │
│  │deadlock-detector│  │  index.js       │               │
│  │     .js         │  │                 │               │
│  └─────────────────┘  └─────────────────┘               │
└─────────────────────────────────────────────────────────┘
```

### Proposal Lifecycle Workflow

```
┌─────────┐    ┌─────────┐    ┌─────────┐    ┌─────────┐
│  draft  │ →  │ pending │ →  │ voting  │ →  │approved │
└─────────┘    └─────────┘    └─────────┘    └─────────┘
                                    │              │
                                    ↓              ↓
                              ┌─────────┐    ┌─────────┐
                              │deadlocked│   │executed │
                              └─────────┘    └─────────┘
```

### Usage Examples

#### Creating Proposals

```bash
# Create a new configuration proposal
./scripts/triad-status.sh propose --title "Update agent configuration" --type config

# Output:
# Proposal created: prop-20260401-001
# Title: Update agent configuration
# Type: config
# Status: draft
# Created by: steward
# Quorum required: 2/3
```

#### Viewing Proposals

```bash
# View all pending proposals
./scripts/triad-status.sh proposals --status pending

# Output:
# ╔══════════════════════════════════════════════════════╗
# ║ PENDING PROPOSALS                                     ║
# ╠══════════════════════════════════════════════════════╣
# ║ ID                  │ Title            │ Votes      ║
# ╠─────────────────────┼──────────────────┼────────────╣
# ║ prop-20260401-001   │ Update config    │ 1/3        ║
# ║ prop-20260401-002   │ Deploy new agent │ 2/3 ✓      ║
# ╚══════════════════════════════════════════════════════╝
```

#### Submitting Votes

```bash
# Submit approval vote
./scripts/triad-status.sh vote --proposal prop-20260401-001 --vote approve --voter alpha

# Output:
# Vote recorded: prop-20260401-001
# Voter: alpha
# Vote: approve
# Current tally: 2 approve, 0 reject, 0 abstain
# Quorum status: 2/3 reached ✓
```

#### Tabulating Results

```bash
# Tabulate vote results
./scripts/triad-status.sh tabulate --proposal prop-20260401-001

# Output:
# ╔══════════════════════════════════════════════════════╗
# ║ VOTE TALLY: prop-20260401-001                         ║
# ╠══════════════════════════════════════════════════════╣
# ║ Approve:  2 (alpha, beta)                            ║
# ║ Reject:   1 (gamma)                                  ║
# ║ Abstain:  0                                          ║
# ╠──────────────────────────────────────────────────────╣
# ║ Result: APPROVED (quorum reached)                    ║
# ║ Executing proposal...                                ║
# ╚══════════════════════════════════════════════════════╝
```

#### Deadlock Detection and Resolution

```bash
# Check for deadlocks
./scripts/triad-status.sh check-deadlock

# Output (deadlock detected):
# ⚠ DEADLOCK DETECTED
# Proposal: prop-20260401-003
# Votes: 1 approve, 1 reject, 1 abstain
# Timeout: exceeded by 2h
# Resolution methods available:
#   - steward-tiebreak
#   - timeout-expire
#   - revote
```

```bash
# Resolve deadlock with steward tie-breaker
./scripts/triad-status.sh resolve-deadlock --proposal prop-20260401-003 --method steward-tiebreak

# Output:
# Deadlock resolved: prop-20260401-003
# Method: steward-tiebreak
# Steward casting tie-breaking vote: approve
# Proposal status: approved
```

#### Consensus Ledger Operations

```bash
# Sync ledger state
./scripts/triad-status.sh sync-ledger

# Output:
# Ledger synchronized
# Entries: 147
# Last entry: prop-20260401-001 executed
# Checksum: a3f5b2c1...
```

```bash
# Verify ledger integrity
./scripts/triad-status.sh verify-ledger

# Output:
# Ledger integrity check: PASSED
# Entries verified: 147
# Checksums matched: 147/147
```

### Triad Dashboard

```bash
# View complete triad state
./scripts/triad-status.sh dashboard

# Output:
# ╔══════════════════════════════════════════════════════╗
# ║              TRIAD DELIBERATION DASHBOARD             ║
# ╠══════════════════════════════════════════════════════╣
# ║ Members: steward(✓), alpha(✓), beta(✓), gamma(✓)     ║
# ║ Quorum: 2/3 required                                 ║
# ╠──────────────────────────────────────────────────────╣
# ║ Active Proposals: 2                                  ║
# ║ Pending Votes: 1                                     ║
# ║ Deadlocks: 0                                         ║
# ╠──────────────────────────────────────────────────────╣
# ║ Recent Decisions:                                    ║
# ║   [✓] prop-20260401-001: Update config (approved)   ║
# ║   [✓] prop-20260401-002: Deploy agent (approved)    ║
# ║   [✗] prop-20260331-005: Rate limit (rejected)      ║
# ╚══════════════════════════════════════════════════════╝
```

---

## Gateway Pulse

**Skill Path:** [`heretek-openclaw-core/.roo/skills/gateway-pulse/SKILL.md`](../../../heretek-openclaw-core/.roo/skills/gateway-pulse/SKILL.md)

### Overview

Gateway Pulse provides continuous monitoring for the OpenClaw Gateway (port 18789) and LiteLLM (port 4000). It includes real-time health dashboards, alert thresholds, auto-remediation, and Prometheus metrics export.

### Architecture

```
┌─────────────────────────────────────────────────────────┐
│                     Gateway Pulse                        │
├─────────────────────────────────────────────────────────┤
│  ┌─────────────────┐  ┌─────────────────┐               │
│  │gateway-monitor  │  │litellm-monitor  │               │
│  │     .js         │  │     .js         │               │
│  └─────────────────┘  └─────────────────┘               │
│  ┌─────────────────┐  ┌─────────────────┐               │
│  │  alert-manager  │  │  index.js       │               │
│  │     .js         │  │                 │               │
│  └─────────────────┘  └─────────────────┘               │
└─────────────────────────────────────────────────────────┘
```

### Usage Examples

#### Status Checks

```bash
# Check all service statuses
./scripts/gateway-pulse.sh status

# Output:
# ╔══════════════════════════════════════════════════════╗
# ║ SERVICE STATUS                                        ║
# ╠══════════════════════════════════════════════════════╣
# ║ Service   │ Port  │ Status  │ Latency │ Error Rate ║
# ╠───────────┼───────┼─────────┼─────────┼────────────╣
# ║ Gateway   │ 18789 │ healthy │ 45ms    │ 0.02%      ║
# ║ LiteLLM   │ 4000  │ healthy │ 32ms    │ 0.01%      ║
# ╚══════════════════════════════════════════════════════╝
```

#### Continuous Monitoring

```bash
# Start monitoring with 15-second interval
./scripts/gateway-pulse.sh monitor --interval 15

# Output:
# Starting continuous monitoring...
# Interval: 15s
# Target services: gateway, litellm
# Press Ctrl+C to stop
#
# [15:30:00] Gateway: healthy (42ms)
# [15:30:00] LiteLLM: healthy (28ms)
# [15:30:15] Gateway: healthy (48ms)
# [15:30:15] LiteLLM: healthy (35ms)
```

#### Auto-Remediation

```bash
# Enable auto-remediation
./scripts/gateway-pulse.sh remediate --enable

# Output:
# Auto-remediation enabled
# Actions configured:
#   - Restart Gateway on critical failure
#   - Restart LiteLLM on critical failure
#   - Clear WebSocket cache on connection exhaustion
#   - Reset connection pools on timeout
```

### Alert Configuration and Thresholds

#### Setting Alert Thresholds

```bash
# Configure latency thresholds
./scripts/gateway-pulse.sh alerts --set --warning 5000 --critical 10000

# Output:
# Alert thresholds updated:
#   Latency Warning: 5000ms
#   Latency Critical: 10000ms
#   Error Rate Warning: 5%
#   Error Rate Critical: 20%
```

#### Default Alert Thresholds

| Metric | Warning | Critical |
|--------|---------|----------|
| Latency | 5000ms | 10000ms |
| Error Rate | 5% | 20% |
| Connection Failures | 3/min | 10/min |
| WebSocket Errors | 5/min | 20/min |

#### Viewing Alert History

```bash
# View alert history
./scripts/gateway-pulse.sh alerts --history

# Output:
# ╔══════════════════════════════════════════════════════╗
# ║ ALERT HISTORY (Last 24h)                              ║
# ╠══════════════════════════════════════════════════════╣
# ║ Time      │ Service │ Level   │ Metric     │ Value  ║
# ╠───────────┼─────────┼─────────┼────────────┼────────╣
# ║ 14:23:15  │ Gateway │ warning │ latency    │ 5.2s   ║
# ║ 12:45:00  │ LiteLLM │ critical│ error_rate │ 22%    ║
# ║ 10:12:33  │ Gateway │ warning │ connections│ 85%    ║
# ╚══════════════════════════════════════════════════════╝
```

### Health Dashboard

```bash
# View real-time health dashboard
./scripts/gateway-pulse.sh dashboard --watch

# Output:
# ╔══════════════════════════════════════════════════════╗
# ║            GATEWAY PULSE DASHBOARD                     ║
# ╠══════════════════════════════════════════════════════╣
# ║ Gateway (18789): ████████████████████ healthy 45ms   ║
# ║ LiteLLM (4000):  ████████████████████ healthy 32ms   ║
# ╠──────────────────────────────────────────────────────╣
# ║ Uptime: Gateway 2d 4h | LiteLLM 2d 4h               ║
# ║ Requests/min: 1240 | Tokens/min: 45,230             ║
# ║ Active WS: 8 | Errors (1h): 3                        ║
# ╚══════════════════════════════════════════════════════╝
```

---

## Prometheus Metrics Integration

### Exporting Metrics

```bash
# Export metrics in Prometheus format
./scripts/gateway-pulse.sh metrics --export

# Output:
# # HELP gateway_health Gateway health status
# # TYPE gateway_health gauge
# gateway_health{service="gateway"} 1
# gateway_health{service="litellm"} 1
#
# # HELP gateway_latency Gateway response latency (ms)
# # TYPE gateway_latency gauge
# gateway_latency{service="gateway"} 45
# gateway_latency{service="litellm"} 32
#
# # HELP gateway_requests_total Total requests processed
# # TYPE gateway_requests_total counter
# gateway_requests_total{service="gateway"} 1240567
# gateway_requests_total{service="litellm"} 987654
```

### Starting Metrics Server

```bash
# Start Prometheus metrics server
./scripts/gateway-pulse.sh metrics --serve --port 9090

# Output:
# Metrics server started
# Endpoint: http://localhost:9090/metrics
# Scrape interval: 15s
```

### Prometheus Configuration

Add to `prometheus.yml`:

```yaml
scrape_configs:
  - job_name: 'openclaw-gateway'
    static_configs:
      - targets: ['localhost:9090']
    scrape_interval: 15s
    metrics_path: '/metrics'
```

### Available Metrics

| Metric | Type | Description |
|--------|------|-------------|
| `gateway_health` | Gauge | Service health status (0/1) |
| `gateway_latency` | Gauge | Response latency in ms |
| `gateway_requests_total` | Counter | Total requests processed |
| `gateway_errors_total` | Counter | Total errors |
| `gateway_active_connections` | Gauge | Active WebSocket connections |
| `gateway_tokens_total` | Counter | Total tokens processed |
| `litellm_models_available` | Gauge | Available model count |

---

## Related Documentation

- [Skills Operations Guide](./SKILLS_OPERATIONS_GUIDE.md) - Main operations guide
- [Debugging Skills Guide](./DEBUGGING_SKILLS.md) - Debugging skill usage
- [Skill Integration Patterns](./SKILL_INTEGRATION_PATTERNS.md) - Development patterns
- [Troubleshooting Workflows](./TROUBLESHOOTING_WORKFLOWS.md) - Scenario-based workflows

---

## Troubleshooting Tips

### Agent Fails to Start

1. Check agent logs: `docker logs <agent-container>`
2. Verify configuration: `./scripts/diagnostics.sh config --all`
3. Check dependencies: `./scripts/lifecycle-manager.sh deps --agent <agent-id>`

### Rolling Restart Stuck

1. Check current agent: `./scripts/lifecycle-manager.sh status`
2. Verify health endpoint: `curl http://localhost:18789/health`
3. Force continue: `./scripts/lifecycle-manager.sh rolling-restart --force`

### Proposal Stuck in Pending

1. Check votes: `./scripts/triad-status.sh votes --proposal <id>`
2. Verify connectivity: `./scripts/triad-status.sh status`
3. Force timeout: `./scripts/triad-status.sh force-timeout --proposal <id>`

### Gateway Not Responding

1. Check process: `docker ps | grep gateway`
2. Check logs: `docker logs heretek-openclaw-core-gateway-1`
3. Restart: `./scripts/lifecycle-manager.sh restart --agents gateway`
