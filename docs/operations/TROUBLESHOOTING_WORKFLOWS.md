# Troubleshooting Workflows

**Version:** 1.0  
**Last Updated:** 2026-04-01  
**Scope:** Scenario-Based Troubleshooting

---

## Table of Contents

1. [Agent Offline Workflow](#agent-offline-workflow)
2. [LiteLLM Gateway Failure Workflow](#litellm-gateway-failure-workflow)
3. [Triad Deliberation Deadlock Workflow](#triad-deliberation-deadlock-workflow)
4. [Database Corruption Workflow](#database-corruption-workflow)
5. [Memory Leak Detection Workflow](#memory-leak-detection-workflow)
6. [Network Partition Recovery Workflow](#network-partition-recovery-workflow)

---

## Agent Offline Workflow

**Symptoms:** Agent not responding to health checks, missing from status dashboard, WebSocket disconnection events.

### Detection Phase

```bash
# Step 1: Check agent status dashboard
./scripts/lifecycle-manager.sh dashboard

# Expected output shows agent as "offline" or "stopped"
# ╔══════════════════════════════════════════════════════╗
# ║ Agent       │ Status  │ Uptime   │ Health │ Memory  ║
# ╠─────────────┼─────────┼──────────┼────────┼─────────╣
# ║ alpha       │ offline │ -        │ ✗ -    │ -       ║
```

```bash
# Step 2: Get detailed agent status
./scripts/lifecycle-manager.sh status --agent alpha

# Expected output:
# Agent: alpha
# Status: offline
# Last Seen: 2026-04-01T14:30:00Z (1h ago)
# Exit Code: 137 (OOM kill)
# Restart Count: 3
```

```bash
# Step 3: Check Gateway for agent connection status
./scripts/gateway-pulse.sh status

# Look for WebSocket disconnection events
# Gateway: healthy | Active WS: 7/8 (alpha disconnected)
```

### Diagnosis Phase

```bash
# Step 4: Analyze agent logs for errors
./scripts/analyze-logs.sh analyze --component alpha --since "2h ago" --filter errors

# Expected output:
# Error Analysis for: alpha
# Time Range: 13:30:00 - 15:30:00
# Total Errors: 15
#
# Error Categories:
#   Resource: 8 (53%) - Memory pressure
#   Connection: 5 (33%) - Gateway timeouts
#   Application: 2 (13%) - Skill failures
```

```bash
# Step 5: Check for memory-related patterns
./scripts/analyze-logs.sh patterns --search "memory\|OOM\|heap\|killed"

# Expected output:
# Detected Patterns:
# [CRITICAL] Out of memory (3 occurrences)
#   Pattern: "Out of memory: Killed process"
#   Times: 14:15:00, 14:45:00, 15:15:00
#   Component: alpha
```

```bash
# Step 6: Inspect agent memory state before restart
./scripts/inspect-state.sh memory --agent alpha --export

# Output: memory-alpha-20260401.json
# Preserves memory state for restoration after restart
```

```bash
# Step 7: Check system resources
./scripts/diagnostics.sh component --name system

# Expected output:
# System Resources:
#   CPU: 45% (normal)
#   Memory: 92% (critical)
#   Disk: 67% (normal)
#   Network: healthy
```

### Recovery Phase

```bash
# Step 8: Attempt graceful restart
./scripts/lifecycle-manager.sh restart --agents alpha --verify-health

# Expected output:
# Restarting: alpha
# [OK] Stopped (graceful)
# [OK] Started
# [OK] Health check passed (180ms)
# Agent alpha restarted successfully
```

```bash
# Step 9: If restart fails, check for corruption
./scripts/recover.sh scan --component memory --agent alpha

# If corruption detected:
# [!] Agent Memory: corruption detected (2 files)
# Recommendation: Recovery required
```

```bash
# Step 10: Recover from backup if needed
./scripts/recover.sh recover --component memory --agent alpha --auto --validate

# Expected output:
# Recovery completed successfully
# Validation: PASSED
```

### Verification Phase

```bash
# Step 11: Verify agent is healthy
./scripts/lifecycle-manager.sh status --agent alpha

# Expected output:
# Agent: alpha
# Status: running
# Uptime: 2m
# Health: ✓ 180ms
```

```bash
# Step 12: Compare memory state after restart
./scripts/inspect-state.sh memory --agent alpha --compare

# Expected output:
# Memory Comparison:
#   Before: 1,189 entries (19.2 MB)
#   After:  1,189 entries (19.2 MB)
#   Status: preserved ✓
```

### Prevention

```bash
# Enable auto-restart monitoring
./scripts/lifecycle-manager.sh monitor --auto-restart --interval 30

# Increase container memory limit (docker-compose.yml)
# services:
#   alpha:
#     deploy:
#       resources:
#         limits:
#           memory: 2G
```

---

## LiteLLM Gateway Failure Workflow

**Symptoms:** API errors, model access failures, rate limiting, timeout errors.

### Detection Phase

```bash
# Step 1: Check LiteLLM status
./scripts/gateway-pulse.sh status --service litellm

# Expected output:
# ╔══════════════════════════════════════════════════════╗
# ║ Service   │ Port  │ Status   │ Latency │ Error Rate ║
# ╠───────────┼───────┼──────────┼─────────┼────────────╣
# ║ LiteLLM   │ 4000  │ degraded │ 8500ms  │ 15%        ║
```

```bash
# Step 2: Run detailed LiteLLM diagnostics
./scripts/diagnostics.sh component --name litellm --verbose

# Expected output:
# LiteLLM Health Check
# ────────────────────
# Process:     running (PID 5678)
# Port 4000:   listening
# API:         degraded (high latency)
# Models:      8/10 available
# Errors (1h): 47
#
# Issues:
#   - High response latency (8500ms avg)
#   - Rate limit errors from anthropic
#   - Model claude-3 unavailable
```

### Diagnosis Phase

```bash
# Step 3: Analyze LiteLLM logs
./scripts/analyze-logs.sh analyze --component litellm --since "1h ago" --filter errors

# Expected output:
# Error Analysis for: litellm
# Total Errors: 47
#
# Error Categories:
#   External: 25 (53%) - Provider API failures
#   Resource: 12 (26%) - Connection pool exhaustion
#   Configuration: 10 (21%) - Model config issues
```

```bash
# Step 4: Check for rate limiting patterns
./scripts/analyze-logs.sh patterns --search "rate limit\|429\|too many requests"

# Expected output:
# Detected Patterns:
# [HIGH] Rate limit exceeded (15 occurrences)
#   Pattern: "Rate limit exceeded for model.*"
#   Provider: anthropic
#   Times: 14:30:00 - 15:30:00
```

```bash
# Step 5: Check token usage patterns
./scripts/analyze-logs.sh patterns --search "token\|usage\|quota"

# Expected output:
# Detected Patterns:
# [MEDIUM] High token usage (8 occurrences)
#   Pattern: "Token usage:.*"
#   Peak: 45,000 tokens/min at 14:55:00
```

```bash
# Step 6: Verify model availability
curl -s http://localhost:4000/v1/models | jq '.data[].id'

# Expected output:
# "gpt-4"
# "gpt-3.5-turbo"
# "claude-3" (missing - unavailable)
```

### Recovery Phase

```bash
# Step 7: Enable auto-remediation
./scripts/gateway-pulse.sh remediate --enable --run

# Expected output:
# Auto-remediation triggered
# Actions:
#   [✓] Cleared connection pool
#   [✓] Reset model cache
#   [✓] Restarted LiteLLM service
```

```bash
# Step 8: If auto-remediation fails, manual restart
docker compose -p heretek-openclaw-core restart litellm

# Wait for startup
sleep 10

# Verify restart
./scripts/gateway-pulse.sh status --service litellm
```

```bash
# Step 9: Check LiteLLM configuration
./scripts/diagnostics.sh config --file litellm_config.yaml

# If configuration issues found:
# [!] litellm_config.yaml - invalid model configuration
# Fix: Update model provider credentials
```

### Verification Phase

```bash
# Step 10: Test model connectivity
curl -s http://localhost:4000/health | jq

# Expected output:
# {
#   "status": "healthy",
#   "models": 10,
#   "latency": "45ms"
# }
```

```bash
# Step 11: Test chat completion
curl -s -X POST http://localhost:4000/v1/chat/completions \
  -H "Content-Type: application/json" \
  -d '{"model":"gpt-4","messages":[{"role":"user","content":"test"}]}' | jq

# Expected output:
# {
#   "choices": [...],
#   "usage": {"total_tokens": 15}
# }
```

### Prevention

```yaml
# Update litellm_config.yaml with rate limiting
model_list:
  - model_name: claude-3
    models:
      - claude-3-opus-20240229
    litellm_params:
      rpm: 100  # Requests per minute limit
      tpm: 50000  # Tokens per minute limit
```

---

## Triad Deliberation Deadlock Workflow

**Symptoms:** Proposal stuck in pending/voting state, votes not progressing, consensus not reached.

### Detection Phase

```bash
# Step 1: Check triad dashboard
./scripts/triad-status.sh dashboard

# Expected output:
# ╔══════════════════════════════════════════════════════╗
# ║ Active Proposals: 2                                   ║
# ║ Pending Votes: 1                                      ║
# ║ Deadlocks: 1 ⚠                                        ║
# ╠──────────────────────────────────────────────────────╣
# ║ [!] prop-20260401-003: DEADLOCK (1-1-1)              ║
```

```bash
# Step 2: Check for deadlock explicitly
./scripts/triad-status.sh check-deadlock

# Expected output:
# ⚠ DEADLOCK DETECTED
# Proposal: prop-20260401-003
# Votes: 1 approve, 1 reject, 1 abstain
# Timeout: exceeded by 2h
# Resolution methods available:
#   - steward-tiebreak
#   - timeout-expire
#   - revote
```

### Diagnosis Phase

```bash
# Step 3: View proposal details
./scripts/triad-status.sh proposal --id prop-20260401-003

# Expected output:
# Proposal: prop-20260401-003
# Title: Update rate limits
# Type: config
# Status: deadlocked
# Created: 2026-04-01T12:00:00Z
# Timeout: 2026-04-01T13:00:00Z (exceeded by 2h)
#
# Votes:
#   steward: approve
#   alpha: reject
#   beta: abstain
```

```bash
# Step 4: Analyze triad logs for deliberation issues
./scripts/analyze-logs.sh patterns --search "deliberation\|consensus\|deadlock\|vote"

# Expected output:
# Detected Patterns:
# [HIGH] Consensus failed (5 occurrences)
#   Pattern: "Consensus failed: split vote"
#   Times: 13:00:00 - 15:00:00
#   Components: steward, alpha, beta
```

```bash
# Step 5: Inspect consensus ledger
./scripts/inspect-state.sh ledger --filter triad --since "2h ago"

# Expected output:
# Consensus Ledger (Triad entries)
# ────────────────────────────────
# [15:00:00] prop-20260401-002: APPROVED (2-1)
# [14:30:00] prop-20260401-003: DEADLOCK (1-1-1)
# [14:00:00] prop-20260401-001: APPROVED (3-0)
#
# Ledger Health: valid
```

```bash
# Step 6: Check triad agent connectivity
./scripts/lifecycle-manager.sh status --agents steward,alpha,beta

# Expected output:
# steward: running (healthy)
# alpha: running (healthy)
# beta: running (healthy)
# All triad agents connected
```

### Recovery Phase

```bash
# Step 7: Resolve deadlock with steward tie-breaker
./scripts/triad-status.sh resolve-deadlock --proposal prop-20260401-003 --method steward-tiebreak

# Expected output:
# Deadlock resolved: prop-20260401-003
# Method: steward-tiebreak
# Steward casting tie-breaking vote: approve
# Proposal status: approved
# Executing proposal...
```

```bash
# Step 8: Alternative - Force timeout expiration
./scripts/triad-status.sh resolve-deadlock --proposal prop-20260401-003 --method timeout-expire

# Expected output:
# Deadlock resolved: prop-20260401-003
# Method: timeout-expire
# Proposal status: expired
# Proposal rejected by timeout
```

```bash
# Step 9: Alternative - Trigger revote
./scripts/triad-status.sh resolve-deadlock --proposal prop-20260401-003 --method revote

# Expected output:
# Deadlock resolved: prop-20260401-003
# Method: revote
# New proposal created: prop-20260401-004
# Voting period: 1 hour
```

### Verification Phase

```bash
# Step 10: Verify proposal resolution
./scripts/triad-status.sh proposal --id prop-20260401-003

# Expected output:
# Proposal: prop-20260401-003
# Status: approved (executed)
# Resolution: steward-tiebreak
# Executed at: 2026-04-01T15:30:00Z
```

```bash
# Step 11: Verify ledger updated
./scripts/inspect-state.sh ledger --verify

# Expected output:
# Ledger Integrity Check
# ──────────────────────
# Total Entries: 148
# Checksums Verified: 148/148 ✓
# New Entry: prop-20260401-003 approved
```

### Prevention

```json
// Update governance-quorum-rules.json
{
  "deadlock_timeout": "30m",
  "tiebreak_enabled": true,
  "revote_limit": 2,
  "escalation_threshold": 3
}
```

---

## Database Corruption Workflow

**Symptoms:** Query failures, data inconsistencies, checksum mismatches, connection errors.

### Detection Phase

```bash
# Step 1: Check database health
./scripts/diagnostics.sh component --name database --integrity

# Expected output:
# Database Health Check
# ─────────────────────
# Connection:  established
# Latency:     12ms
# Tables:      24/24 accessible
# Indexes:     48/48 valid
# Constraints: all satisfied
# Integrity:   FAILED ⚠
#
# Issues:
#   - Table ledger_entries: 2 rows corrupted
#   - Checksum mismatch in entries-20260401
```

```bash
# Step 2: Run corruption scan
./scripts/recover.sh scan --component database --full

# Expected output:
# Integrity Scan Report
# ─────────────────────
# Component: database
# Scan Time: 15:30:00
#
# Results:
#   [✗] Tables: 2 corrupted
#   [✓] Indexes: valid
#   [✓] Constraints: satisfied
#
# Corruption Details:
#   - ledger_entries: rows 145-146 checksum mismatch
#   - agent_memory: row 1234 truncated
```

### Diagnosis Phase

```bash
# Step 3: Analyze database logs
./scripts/analyze-logs.sh patterns --search "database\|postgresql\|corruption\|checksum"

# Expected output:
# Detected Patterns:
# [CRITICAL] Checksum mismatch (3 occurrences)
#   Pattern: "checksum mismatch.*"
#   Times: 14:00:00, 14:30:00, 15:00:00
#   Tables: ledger_entries, agent_memory
```

```bash
# Step 4: Check for related application errors
./scripts/analyze-logs.sh correlate --event "database-error"

# Expected output:
# Correlated Event Cluster:
#   Time: 14:00:00 - 14:05:00
#   Root: Database write failure
#   Effects:
#     - Application errors (5)
#     - Agent offline events (2)
#     - Consensus failures (1)
#   Confidence: 92%
```

```bash
# Step 5: Inspect affected state
./scripts/inspect-state.sh scan --component database

# Expected output:
# State Corruption Scan
# ─────────────────────
# Component: database
#
# Results:
#   [✗] ledger_entries: 2 entries corrupted
#   [✗] agent_memory: 1 entry corrupted
#   [✓] configuration: valid
```

### Recovery Phase

```bash
# Step 6: List available database backups
./scripts/recover.sh list --component database

# Expected output:
# Available Backups (database)
# ────────────────────────────
# backup-20260401-1500│ 15:00:00 │ verified ✓
# backup-20260401-1400│ 14:00:00 │ verified ✓
# backup-20260401-1300│ 13:00:00 │ verified ✓
```

```bash
# Step 7: Preview recovery
./scripts/recover.sh preview --component database --backup backup-20260401-1400 --diff

# Expected output:
# Recovery Preview
# ────────────────
# Backup: backup-20260401-1400
# Component: database
#
# Changes:
#   - ledger_entries: restore rows 145-146
#   - agent_memory: restore row 1234
#
# Services to Stop: steward, gateway
# Estimated Downtime: 45 seconds
```

```bash
# Step 8: Execute recovery
./scripts/recover.sh recover --component database --backup backup-20260401-1400 --validate

# Expected output:
# Recovery Execution
# ──────────────────
# Stage 1: Pre-Recovery
#   [✓] Backing up current state
#   [✓] Stopping affected services
#   [✓] Verifying backup integrity
#
# Stage 2: Recovery
#   [✓] Extracting backup data
#   [✓] Restoring corrupted rows
#   [✓] Updating metadata
#
# Stage 3: Post-Recovery
#   [✓] Validating restored data
#   [✓] Restarting services
#   [✓] Verifying system health
#
# Stage 4: Verification
#   [✓] Integrity checks passed
#   [✓] Functional tests passed
#   [✓] Consistency verified
#
# Recovery completed successfully.
```

### Verification Phase

```bash
# Step 9: Verify database integrity
./scripts/diagnostics.sh component --name database --integrity

# Expected output:
# Database Integrity Check
# ────────────────────────
# Tables:      24/24 valid ✓
# Indexes:     48/48 valid ✓
# Checksums:   all matched ✓
# Integrity:   PASSED ✓
```

```bash
# Step 10: Verify application functionality
./scripts/diagnostics.sh health-score --breakdown

# Expected output:
# Health Score: 95/100 (Excellent)
# Database (20%): 20/20 ✓
```

### Prevention

```bash
# Enable automated backups
./scripts/backup.sh schedule --interval hourly --retention 24

# Configure WAL archiving
# postgresql.conf:
# wal_level = replica
# archive_mode = on
# archive_command = 'cp %p /backup/wal/%f'
```

---

## Memory Leak Detection Workflow

**Symptoms:** Increasing memory usage over time, OOM kills, performance degradation.

### Detection Phase

```bash
# Step 1: Check health score for memory issues
./scripts/diagnostics.sh health-score --breakdown

# Expected output:
# Health Score: 72/100 (Degraded)
#
# Component Breakdown:
# System (10%): 6/10 ⚠
#   Memory: 89% (warning threshold)
#   Swap: 45% used
```

```bash
# Step 2: Check agent memory usage
./scripts/lifecycle-manager.sh dashboard

# Expected output:
# ╔══════════════════════════════════════════════════════╗
# ║ Agent       │ Status  │ Uptime   │ Health │ Memory  ║
# ╠─────────────┼─────────┼──────────┼────────┼─────────╣
# ║ steward     │ running │ 2d 4h    │ ✓      │ 1.8GB ↑ ║
# ║ alpha       │ running │ 2d 4h    │ ✓      │ 1.7GB ↑ ║
# ║ beta        │ running │ 2d 4h    │ ✓      │ 1.9GB ↑ ║
```

### Diagnosis Phase

```bash
# Step 3: Inspect agent memory details
./scripts/inspect-state.sh memory --all

# Expected output:
# Agent Memory Summary
# ────────────────────
# Agent       │ STM Entries │ LTM Entries │ Total Size
# ────────────┼─────────────┼─────────────┼────────────
# steward     │ 2,456       │ 5,678       │ 1.8 GB ↑
# alpha       │ 2,234       │ 5,234       │ 1.7 GB ↑
# beta        │ 2,567       │ 5,890       │ 1.9 GB ↑
#
# Memory Growth (24h):
#   steward: +45% ⚠
#   alpha: +42% ⚠
#   beta: +48% ⚠
```

```bash
# Step 4: Analyze memory-related logs
./scripts/analyze-logs.sh patterns --search "memory\|heap\|OOM\|killed"

# Expected output:
# Detected Patterns:
# [HIGH] Memory pressure (12 occurrences)
#   Pattern: "Memory usage exceeded.*"
#   Components: steward, alpha, beta
#
# [CRITICAL] OOM kill (2 occurrences)
#   Pattern: "Out of memory: Killed process"
#   Times: 14:15:00, 15:15:00
#   Component: scout
```

```bash
# Step 5: Build memory timeline
./scripts/analyze-logs.sh timeline --filter "memory" --since "24h ago"

# Expected output:
# Memory Event Timeline (24 hours)
# ────────────────────────────────
# 14:00:00 [steward] Memory: 1.2GB
# 14:30:00 [steward] Memory: 1.4GB
# 15:00:00 [steward] Memory: 1.6GB (warning)
# 15:30:00 [steward] Memory: 1.8GB (critical)
# 15:45:00 [scout] OOM killed
# 15:46:00 [scout] Restarted
```

```bash
# Step 6: Export memory report for analysis
./scripts/inspect-state.sh memory --all --output memory-report.json

# Output: memory-report.json (detailed memory state)
```

### Recovery Phase

```bash
# Step 7: Restart affected agents
./scripts/lifecycle-manager.sh restart --agents steward,alpha,beta --rolling

# Expected output:
# Rolling restart initiated...
# [1/3] Restarting steward... OK
# [2/3] Restarting alpha... OK
# [3/3] Restarting beta... OK
# Memory cleared, agents healthy
```

```bash
# Step 8: If memory persists, check for corruption
./scripts/recover.sh scan --component memory

# If corruption detected:
# [!] Agent Memory: potential leak detected
# Recommendation: Recovery from backup
```

```bash
# Step 9: Recover memory from clean backup
./scripts/recover.sh recover --component memory --auto --validate

# Expected output:
# Recovery completed
# Memory restored to clean state
```

### Prevention

```yaml
# Update docker-compose.yml with memory limits
services:
  steward:
    deploy:
      resources:
        limits:
          memory: 2G
        reservations:
          memory: 1G
  alpha:
    deploy:
      resources:
        limits:
          memory: 2G
```

```bash
# Enable memory monitoring
./scripts/gateway-pulse.sh monitor --interval 30

# Configure memory alerts
./scripts/gateway-pulse.sh alerts --set --memory-warning 80 --memory-critical 90
```

---

## Network Partition Recovery Workflow

**Symptoms:** Multiple agents offline, WebSocket connection failures, network timeout errors.

### Detection Phase

```bash
# Step 1: Check overall system status
./scripts/diagnostics.sh full

# Expected output:
# ╔══════════════════════════════════════════════════════╗
# ║ Gateway:     [✗] degraded (connection issues)         ║
# ║ LiteLLM:     [✓] healthy                              ║
# ║ Database:    [✓] connected                            ║
# ║ Agents:      [✗] 3/8 running                          ║
# ║ System:      [✓] normal                               ║
# ╠──────────────────────────────────────────────────────╣
# ║ HEALTH SCORE: 45/100 (Critical)                      ║
```

```bash
# Step 2: Check network connectivity
./scripts/diagnostics.sh deps --network

# Expected output:
# Network Connectivity Check
# ──────────────────────────
# Gateway (18789): ✗ connection refused
# LiteLLM (4000):  ✓ connected
# Database (5432): ✓ connected
# Redis (6379):    ✗ connection refused
#
# Network Issues:
#   - Gateway port unreachable
#   - Redis port unreachable
```

### Diagnosis Phase

```bash
# Step 3: Analyze network-related logs
./scripts/analyze-logs.sh patterns --search "connection refused\|network\|partition\|timeout"

# Expected output:
# Detected Patterns:
# [CRITICAL] Connection refused (25 occurrences)
#   Pattern: "connection refused.*"
#   Components: gateway, steward, scout, artisan
#   Times: 15:00:00 - 15:30:00
#
# [HIGH] Network partition (5 occurrences)
#   Pattern: "network partition detected"
#   Components: gateway
```

```bash
# Step 4: Correlate network events
./scripts/analyze-logs.sh correlate --event "network"

# Expected output:
# Correlated Event Cluster:
#   Time: 15:00:00 - 15:05:00
#   Root: Network interface failure
#   Effects:
#     - Gateway disconnection (8)
#     - Agent offline events (5)
#     - WebSocket failures (12)
#   Confidence: 95%
```

```bash
# Step 5: Check Gateway status
./scripts/gateway-pulse.sh status

# Expected output:
# ╔══════════════════════════════════════════════════════╗
# ║ Service   │ Port  │ Status       │ Latency │ Errors ║
# ╠───────────┼───────┼──────────────┼─────────┼────────╣
# ║ Gateway   │ 18789 │ unreachable  │ -       │ 100%   ║
# ║ LiteLLM   │ 4000  │ healthy      │ 32ms    │ 0%     ║
```

### Recovery Phase

```bash
# Step 6: Check Docker network status
docker network ls
docker network inspect heretek-openclaw-core_default

# Expected output:
# Network: heretek-openclaw-core_default
# Driver: bridge
# Containers: 8 connected
# Status: healthy
```

```bash
# Step 7: Restart Gateway service
./scripts/lifecycle-manager.sh restart --agents gateway

# If Gateway is Docker-based:
docker compose -p heretek-openclaw-core restart gateway

# Wait for startup
sleep 10
```

```bash
# Step 8: Restart affected agents
./scripts/lifecycle-manager.sh restart --agents steward,alpha,beta,scout,artisan

# Expected output:
# Restarting 5 agents...
# [✓] steward restarted
# [✓] alpha restarted
# [✓] beta restarted
# [✓] scout restarted
# [✓] artisan restarted
```

```bash
# Step 9: Verify WebSocket connections
./scripts/gateway-pulse.sh status

# Expected output:
# ╔══════════════════════════════════════════════════════╗
# ║ Service   │ Port  │ Status  │ Latency │ Active WS   ║
# ╠───────────┼───────┼─────────┼─────────┼─────────────╣
# ║ Gateway   │ 18789 │ healthy │ 45ms    │ 8/8         ║
```

### Verification Phase

```bash
# Step 10: Run full system check
./scripts/diagnostics.sh full

# Expected output:
# HEALTH SCORE: 92/100 (Excellent)
# All components healthy
```

```bash
# Step 11: Verify agent connectivity
./scripts/lifecycle-manager.sh dashboard

# Expected output:
# ╔══════════════════════════════════════════════════════╗
# ║ Agents: 8/8 running                                   ║
# ║ Health: 100%                                          ║
```

### Prevention

```yaml
# Docker Compose health checks
services:
  gateway:
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:18789/health"]
      interval: 10s
      timeout: 5s
      retries: 3
      start_period: 30s
```

---

## Related Documentation

- [Skills Operations Guide](./SKILLS_OPERATIONS_GUIDE.md) - Main operations guide
- [Orchestration Skills Guide](./ORCHESTRATION_SKILLS.md) - Orchestration skill usage
- [Debugging Skills Guide](./DEBUGGING_SKILLS.md) - Debugging skill usage
- [Skill Integration Patterns](./SKILL_INTEGRATION_PATTERNS.md) - Development patterns

---

## Quick Reference

| Workflow | Primary Skill | Detection Command | Recovery Command |
|----------|---------------|-------------------|------------------|
| Agent Offline | `agent-lifecycle-manager` | `lifecycle-manager.sh dashboard` | `lifecycle-manager.sh restart` |
| LiteLLM Failure | `gateway-pulse` | `gateway-pulse.sh status` | `gateway-pulse.sh remediate` |
| Triad Deadlock | `triad-orchestrator` | `triad-status.sh check-deadlock` | `triad-status.sh resolve-deadlock` |
| Database Corruption | `corruption-recovery` | `recover.sh scan` | `recover.sh recover` |
| Memory Leak | `state-inspector` | `inspect-state.sh memory` | `recover.sh recover` |
| Network Partition | `system-diagnostics` | `diagnostics.sh deps --network` | `lifecycle-manager.sh restart` |
