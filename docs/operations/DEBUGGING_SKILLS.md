# Debugging Skills Guide

**Version:** 1.0  
**Last Updated:** 2026-04-01  
**Scope:** System Diagnostics, Log Analyzer, State Inspector, Corruption Recovery

---

## Table of Contents

1. [System Diagnostics](#system-diagnostics)
2. [Log Analyzer](#log-analyzer)
3. [State Inspector](#state-inspector)
4. [Corruption Recovery](#corruption-recovery)
5. [Common Debugging Scenarios](#common-debugging-scenarios)

---

## System Diagnostics

**Skill Path:** [`heretek-openclaw-core/.roo/skills/system-diagnostics/SKILL.md`](../../../heretek-openclaw-core/.roo/skills/system-diagnostics/SKILL.md)

### Overview

System Diagnostics provides comprehensive one-command health checks across all OpenClaw components. It includes configuration validation, dependency verification, log aggregation, and health score calculation.

### Architecture

```
┌─────────────────────────────────────────────────────────┐
│                  System Diagnostics                      │
├─────────────────────────────────────────────────────────┤
│  ┌─────────────────┐  ┌─────────────────┐               │
│  │config-validator │  │dependency-checker               │
│  │     .js         │  │     .js         │               │
│  └─────────────────┘  └─────────────────┘               │
│  ┌─────────────────┐  ┌─────────────────┐               │
│  │ health-scorer   │  │  index.js       │               │
│  │     .js         │  │                 │               │
│  └─────────────────┘  └─────────────────┘               │
└─────────────────────────────────────────────────────────┘
```

### Usage Examples

#### Full System Check

```bash
# Run complete system diagnostics
./scripts/diagnostics.sh full

# Output:
# ╔══════════════════════════════════════════════════════╗
# ║          OPENCLAW SYSTEM DIAGNOSTICS                   ║
# ╠══════════════════════════════════════════════════════╣
# ║ Gateway:     [✓] healthy (45ms)                       ║
# ║ LiteLLM:     [✓] healthy (32ms)                       ║
# ║ Database:    [✓] connected (12ms)                     ║
# ║ Agents:      [✓] 7/8 running                          ║
# ║ System:      [✓] CPU 23%, Mem 45%, Disk 67%          ║
# ╠──────────────────────────────────────────────────────╣
# ║ HEALTH SCORE: 87/100 (Good)                          ║
# ╚══════════════════════════════════════════════════════╝
```

#### Component-Specific Checks

```bash
# Check Gateway health only
./scripts/diagnostics.sh component --name gateway

# Output:
# Gateway Health Check
# ────────────────────
# Process:     running (PID 1234)
# Port 18789:  listening
# WebSocket:   accepting connections
# Response:    45ms
# Errors (1h): 3
# Status:      HEALTHY
```

```bash
# Check database with integrity validation
./scripts/diagnostics.sh component --name database --integrity

# Output:
# Database Health Check
# ─────────────────────
# Connection:  established
# Latency:     12ms
# Tables:      24/24 accessible
# Indexes:     48/48 valid
# Constraints: all satisfied
# Integrity:   PASSED
# Status:      HEALTHY
```

#### Configuration Validation

```bash
# Validate all configuration files
./scripts/diagnostics.sh config --all

# Output:
# Configuration Validation
# ────────────────────────
# [✓] openclaw.json - valid JSON, all required fields present
# [✓] .env - all environment variables set
# [✓] litellm_config.yaml - valid YAML, models configured
# [✓] docker-compose.yml - valid YAML, services defined
#
# No configuration issues found.
```

```bash
# Validate with auto-fix
./scripts/diagnostics.sh config --all --auto-fix

# Output:
# Configuration Validation with Auto-Fix
# ──────────────────────────────────────
# [✓] openclaw.json - valid
# [!] .env - missing LOG_LEVEL, setting default: info
# [✓] litellm_config.yaml - valid
# [✓] docker-compose.yml - valid
#
# Fixed 1 issue automatically.
```

#### Dependency Verification

```bash
# Check all dependencies
./scripts/diagnostics.sh deps --all

# Output:
# Dependency Check
# ────────────────
# System Dependencies:
#   [✓] Docker 24.0.7
#   [✓] Node.js 18.19.0
#   [✓] npm 10.2.3
#
# Docker Dependencies:
#   [✓] Docker Compose 2.23.3
#   [✓] Container count: 12 running
#
# Node.js Dependencies:
#   [✓] ws 8.14.2
#   [✓] express 4.18.2
#   [✓] pg 8.11.3
#
# All dependencies satisfied.
```

#### Health Score Calculation

```bash
# Get health score with breakdown
./scripts/diagnostics.sh health-score --breakdown

# Output:
# Health Score: 87/100 (Good)
#
# Component Breakdown:
# ────────────────────
# Gateway   (25%):  25/25 ✓
# LiteLLM   (25%):  25/25 ✓
# Database  (20%):  18/20 ⚠ (minor latency)
# Agents    (20%):  17/20 ⚠ (1 agent offline)
# System    (10%):   9/10 ⚠ (disk usage 67%)
```

#### Log Aggregation

```bash
# Aggregate logs from all components
./scripts/diagnostics.sh logs --aggregate --since "1h ago"

# Output:
# Aggregated Logs (Last 1 hour)
# ─────────────────────────────
# [15:23:45] [gateway] INFO: WebSocket connection established
# [15:24:12] [litellm] WARN: Rate limit approaching for model X
# [15:25:00] [steward] ERROR: Failed to connect to alpha
# [15:25:01] [alpha] INFO: Agent restarted successfully
# ...
# Total entries: 1,247
# Errors: 3, Warnings: 12, Info: 1,232
```

### Health Score Weights

| Component | Weight | Criteria |
|-----------|--------|----------|
| Gateway | 25% | WebSocket connectivity, response time |
| LiteLLM | 25% | API availability, model access |
| Database | 20% | Connection, query response, integrity |
| Agents | 20% | Running count, health endpoints |
| System | 10% | CPU, memory, disk usage |

### Score Interpretation

| Score Range | Status | Action |
|-------------|--------|--------|
| 90-100 | Excellent | No action needed |
| 75-89 | Good | Monitor closely |
| 50-74 | Degraded | Investigate issues |
| 25-49 | Critical | Immediate attention |
| 0-24 | Failed | Emergency response |

---

## Log Analyzer

**Skill Path:** [`heretek-openclaw-core/.roo/skills/log-analyzer/SKILL.md`](../../../heretek-openclaw-core/.roo/skills/log-analyzer/SKILL.md)

### Overview

Log Analyzer provides intelligent log analysis with pattern detection, cross-log correlation, timeline reconstruction, and root cause suggestions for troubleshooting OpenClaw incidents.

### Architecture

```
┌─────────────────────────────────────────────────────────┐
│                     Log Analyzer                         │
├─────────────────────────────────────────────────────────┤
│  ┌─────────────────┐  ┌─────────────────┐               │
│  │ pattern-detector│  │ log-correlator  │               │
│  │     .js         │  │     .js         │               │
│  └─────────────────┘  └─────────────────┘               │
│  ┌─────────────────┐  ┌─────────────────┐               │
│  │timeline-builder │  │  index.js       │               │
│  │     .js         │  │                 │               │
│  └─────────────────┘  └─────────────────┘               │
└─────────────────────────────────────────────────────────┘
```

### Usage Examples

#### Full Log Analysis

```bash
# Analyze all logs from the last 2 hours
./scripts/analyze-logs.sh analyze --all --since "2h ago"

# Output:
# Log Analysis Report
# ───────────────────
# Time Range: 13:30:00 - 15:30:00 (2 hours)
# Components: gateway, litellm, steward, alpha, beta, scout, artisan, guardian
# Total Entries: 45,678
# Errors: 23, Warnings: 156, Info: 45,499
#
# Error Distribution:
#   Connection: 12 (52%)
#   Application: 8 (35%)
#   Configuration: 3 (13%)
```

#### Pattern Detection

```bash
# Detect common error patterns
./scripts/analyze-logs.sh patterns

# Output:
# Detected Patterns
# ─────────────────
# [HIGH] WebSocket connection failed (8 occurrences)
#   Pattern: "WebSocket connection failed.*"
#   First: 14:23:15, Last: 15:12:45
#   Components: gateway, steward
#
# [MEDIUM] Agent .* offline (3 occurrences)
#   Pattern: "Agent (steward|alpha|beta) offline"
#   First: 14:45:00, Last: 15:10:22
#   Components: gateway
#
# [LOW] RPC timeout (2 occurrences)
#   Pattern: "RPC timeout exceeded.*"
#   First: 14:56:33, Last: 15:01:12
#   Components: steward, alpha
```

```bash
# Search for specific pattern
./scripts/analyze-logs.sh patterns --search "connection refused"

# Output:
# Pattern Search Results
# ──────────────────────
# Pattern: "connection refused"
# Matches: 5
#
# [14:23:15] [gateway] ERROR: connection refused to litellm:4000
# [14:23:16] [gateway] ERROR: connection refused to litellm:4000
# [14:23:17] [gateway] ERROR: connection refused to litellm:4000
# [14:45:22] [steward] ERROR: connection refused to gateway:18789
# [15:10:45] [alpha] ERROR: connection refused to gateway:18789
```

#### Cross-Log Correlation

```bash
# Correlate events across all logs
./scripts/analyze-logs.sh correlate --window "10m"

# Output:
# Cross-Log Correlation (10 minute window)
# ────────────────────────────────────────
# Correlated Event Cluster #1:
#   Time: 14:23:15 - 14:23:20
#   Root: LiteLLM service restart
#   Effects:
#     - Gateway connection failures (3)
#     - Agent health check failures (5)
#     - RPC timeouts (2)
#   Confidence: 94%
#
# Correlated Event Cluster #2:
#   Time: 15:10:00 - 15:10:30
#   Root: Network partition detected
#   Effects:
#     - Agent offline events (2)
#     - WebSocket reconnections (4)
#   Confidence: 87%
```

#### Timeline Reconstruction

```bash
# Build event timeline for incident investigation
./scripts/analyze-logs.sh timeline --since "30m ago" --filter errors

# Output:
# Event Timeline (Last 30 minutes, errors only)
# ─────────────────────────────────────────────
# 15:00:00 [litellm] ERROR: API rate limit exceeded
# 15:00:01 [gateway] ERROR: LiteLLM returned 429
# 15:00:02 [steward] ERROR: Chat completion failed
# 15:00:15 [litellm] INFO: Rate limit reset
# 15:00:16 [gateway] INFO: LiteLLM connection restored
# 15:00:17 [steward] INFO: Retry succeeded
# 15:10:22 [gateway] ERROR: Agent alpha offline
# 15:10:25 [alpha] INFO: Agent restarting
# 15:10:45 [alpha] INFO: Agent started successfully
```

```bash
# Export timeline to JSON
./scripts/analyze-logs.sh timeline --output timeline.json

# Output:
# Timeline exported to timeline.json
# Events: 156
# Format: JSON
```

#### Root Cause Analysis

```bash
# Get root cause suggestions
./scripts/analyze-logs.sh root-cause

# Output:
# Root Cause Analysis
# ───────────────────
# Most Likely Root Causes:
#
# 1. Gateway WebSocket exhaustion (Confidence: 87%)
#    Evidence:
#      - 8 "WebSocket connection failed" errors
#      - Connection pool at 95% capacity
#      - Errors increased after 14:00
#    Recommendation: Increase WebSocket pool size
#
# 2. LiteLLM rate limiting (Confidence: 72%)
#    Evidence:
#      - 3 "rate limit exceeded" errors
#      - Token usage spike at 14:55
#      - Provider: anthropic
#    Recommendation: Implement request throttling
#
# 3. Memory pressure on agent containers (Confidence: 65%)
#    Evidence:
#      - 2 OOM kill events
#      - Memory usage > 90% before restarts
#    Recommendation: Increase container memory limits
```

#### Error Categorization

```bash
# Categorize all errors
./scripts/analyze-logs.sh categorize --summary

# Output:
# Error Categorization Summary
# ────────────────────────────
# Total Errors: 47
#
# By Category:
#   Connection:    23 (49%) ████████████████████
#   Application:   15 (32%) █████████████
#   Configuration:  5 (11%) ████
#   Resource:       3 (6%)  ██
#   External:       1 (2%)  █
#
# By Component:
#   Gateway:   18 (38%)
#   LiteLLM:   12 (26%)
#   Agents:    17 (36%)
```

### Error Categories

| Category | Description | Examples |
|----------|-------------|----------|
| Connection | Network/connectivity issues | WebSocket failures, timeouts |
| Authentication | Auth/permission errors | Invalid API key, expired token |
| Resource | System resource issues | Out of memory, disk full |
| Configuration | Config-related errors | Invalid JSON, missing field |
| Application | Logic/runtime errors | Null pointer, exception |
| External | Third-party service errors | Provider API failures |

### Detected Patterns

#### Gateway Patterns
- `WebSocket connection failed` - Connection issues
- `Agent .* offline` - Agent connectivity loss
- `RPC timeout` - Message timeout
- `Rate limit exceeded` - Throttling

#### LiteLLM Patterns
- `API key invalid` - Authentication failure
- `Model not found` - Model configuration issue
- `Rate limit` - Provider throttling
- `Timeout exceeded` - Slow response

#### Agent Patterns
- `Failed to connect to gateway` - Agent startup issue
- `Memory limit exceeded` - Resource constraint
- `Consensus failed` - Triad deliberation issue
- `Skill execution failed` - Plugin error

---

## State Inspector

**Skill Path:** [`heretek-openclaw-core/.roo/skills/state-inspector/SKILL.md`](../../../heretek-openclaw-core/.roo/skills/state-inspector/SKILL.md)

### Overview

State Inspector provides deep inspection of agent memory, session state, consensus ledger, and workspace integrity. It is a read-only tool for debugging state issues and auditing consensus decisions.

### Architecture

```
┌─────────────────────────────────────────────────────────┐
│                    State Inspector                       │
├─────────────────────────────────────────────────────────┤
│  ┌─────────────────┐  ┌─────────────────┐               │
│  │memory-inspector │  │ ledger-auditor  │               │
│  │     .js         │  │     .js         │               │
│  └─────────────────┘  └─────────────────┘               │
│  ┌─────────────────┐  ┌─────────────────┐               │
│  │workspace-checker│  │  index.js       │               │
│  │     .js         │  │                 │               │
│  └─────────────────┘  └─────────────────┘               │
└─────────────────────────────────────────────────────────┘
```

### Usage Examples

#### Agent Memory Inspection

```bash
# Inspect specific agent memory
./scripts/inspect-state.sh memory --agent steward

# Output:
# Agent Memory: steward
# ─────────────────────
# Short-term Memory:
#   Entries: 156
#   Size: 2.3 MB
#   Oldest: 2026-04-01T10:00:00Z
#   Newest: 2026-04-01T15:30:00Z
#
# Long-term Memory:
#   Entries: 1,247
#   Size: 18.5 MB
#   Categories: patterns, decisions, learnings
#
# Skill State:
#   Active Skills: 2
#   Pending Operations: 0
```

```bash
# Inspect all agent memories
./scripts/inspect-state.sh memory --all

# Output:
# Agent Memory Summary
# ────────────────────
# Agent       │ STM Entries │ LTM Entries │ Total Size
# ────────────┼─────────────┼─────────────┼────────────
# steward     │ 156         │ 1,247       │ 20.8 MB
# alpha       │ 142         │ 1,189       │ 19.2 MB
# beta        │ 138         │ 1,156       │ 18.9 MB
# scout       │ 89          │ 567         │ 8.4 MB
# artisan     │ 95          │ 623         │ 9.1 MB
# guardian    │ 102         │ 712         │ 10.5 MB
# ─────────────────────────────────────────────────────
# Total: 6 agents, 722 entries, 86.9 MB
```

#### Session State Visualization

```bash
# View current session state
./scripts/inspect-state.sh session

# Output:
# Session State
# ─────────────
# Active Sessions: 8
# Session Duration: 4h 30m
#
# Session Details:
#   steward:  active (messages: 1,247)
#   alpha:    active (messages: 1,189)
#   beta:     active (messages: 1,156)
#   scout:    active (messages: 567)
#   artisan:  active (messages: 623)
#   guardian: active (messages: 712)
#
# Shared Context:
#   Size: 4.5 MB
#   Last Sync: 15:29:45
#   Consistency: verified
```

```bash
# Export session visualization
./scripts/inspect-state.sh session --output session.json

# Output:
# Session exported to session.json
# Format: JSON
# Size: 4.7 MB
```

#### Consensus Ledger Audit

```bash
# Audit ledger entries
./scripts/inspect-state.sh ledger --since "1h ago"

# Output:
# Consensus Ledger Audit (Last 1 hour)
# ────────────────────────────────────
# Entries: 12
#
# Recent Entries:
#   [15:25:00] prop-20260401-001: APPROVED (3-0)
#   [15:18:33] prop-20260401-002: PENDING (2-1)
#   [15:12:15] prop-20260401-003: REJECTED (1-2)
#   [15:05:00] prop-20260401-004: EXECUTED
#
# Ledger Health:
#   Checksum: valid
#   Last Sync: 15:29:45
#   Divergence: none
```

```bash
# Verify ledger integrity
./scripts/inspect-state.sh ledger --verify

# Output:
# Ledger Integrity Check
# ──────────────────────
# Total Entries: 147
# Checksums Verified: 147/147 ✓
# Chain Integrity: valid ✓
# Signature Verification: valid ✓
# No corruption detected.
```

#### Workspace Integrity Check

```bash
# Check workspace integrity
./scripts/inspect-state.sh workspace

# Output:
# Workspace Integrity Check
# ─────────────────────────
# Files Checked: 1,247
# Directories: 89
#
# Status:
#   ✓ 1,240 files unchanged
#   ! 5 files modified (expected)
#   ✗ 2 files corrupted
#
# Corrupted Files:
#   - memory/steward/session.json (checksum mismatch)
#   - ledger/entries-20260401.db (truncated)
#
# Recommendation: Run corruption-recovery skill
```

```bash
# Detailed workspace check
./scripts/inspect-state.sh workspace --detailed

# Output:
# Detailed Workspace Report
# ─────────────────────────
# Modified Files:
#   config/openclaw.json
#     Expected: a3f5b2c1...
#     Actual:   b4e6c3d2...
#     Change:   rate_limit updated (100 → 200)
#
#   logs/gateway.log
#     Expected: rotating
#     Actual:   rotated
#     Change:   normal rotation
#
# Corrupted Files:
#   memory/steward/session.json
#     Expected: 1.2 MB
#     Actual:   0 bytes
#     Cause:    incomplete write
```

#### Corruption Scanning

```bash
# Scan for state corruption
./scripts/inspect-state.sh scan --full

# Output:
# State Corruption Scan
# ─────────────────────
# Components Scanned:
#   [✓] Agent Memory: clean
#   [✓] Collective Memory: clean
#   [!] Consensus Ledger: 2 entries corrupted
#   [✗] Workspace: 2 files corrupted
#
# Summary:
#   Total Issues: 4
#   Critical: 2
#   Warning: 2
#
# Recommendation: Run corruption-recovery skill
```

### State Components

#### Agent Memory
- **Short-term**: Current session context, recent messages
- **Long-term**: Persistent knowledge, learned patterns
- **Skill State**: Active skill execution context

#### Collective Memory
- **Shared Context**: Cross-agent knowledge
- **Consensus State**: Triad deliberation results
- **Global State**: System-wide shared state

#### Consensus Ledger
- **Proposals**: Submitted proposals for voting
- **Votes**: Agent votes on proposals
- **Decisions**: Finalized consensus decisions
- **History**: Complete deliberation history

#### Workspace State
- **File Integrity**: Checksums and modifications
- **Directory Structure**: Expected vs actual
- **Configuration State**: Config file consistency

---

## Corruption Recovery

**Skill Path:** [`heretek-openclaw-core/.roo/skills/corruption-recovery/SKILL.md`](../../../heretek-openclaw-core/.roo/skills/corruption-recovery/SKILL.md)

### Overview

Corruption Recovery provides automated detection and recovery from data corruption. It includes integrity scanning, automatic backup selection, staged recovery procedures, and post-recovery validation.

### Architecture

```
┌─────────────────────────────────────────────────────────┐
│                  Corruption Recovery                     │
├─────────────────────────────────────────────────────────┤
│  ┌─────────────────┐  ┌─────────────────┐               │
│  │integrity-scanner│  │ backup-selector │               │
│  │     .js         │  │     .js         │               │
│  └─────────────────┘  └─────────────────┘               │
│  ┌─────────────────┐  ┌─────────────────┐               │
│  │recovery-manager │  │  index.js       │               │
│  │     .js         │  │                 │               │
│  └─────────────────┘  └─────────────────┘               │
└─────────────────────────────────────────────────────────┘
```

### Usage Examples

#### Scanning for Corruption

```bash
# Full integrity scan
./scripts/recover.sh scan --full

# Output:
# Integrity Scan Report
# ─────────────────────
# Scan Started: 15:30:00
# Components: memory, ledger, workspace
#
# Results:
#   [✓] Agent Memory: clean (6 agents)
#   [!] Consensus Ledger: 2 entries corrupted
#   [✗] Workspace: 2 files corrupted
#   [✓] Configuration: valid
#
# Corruption Details:
#   - ledger/entries-20260401.db: entries 145-146 checksum mismatch
#   - memory/steward/session.json: file truncated (0 bytes)
#   - workspace/config/cache.json: invalid JSON
#
# Recommendation: Recovery required
```

#### Listing Available Backups

```bash
# List all available backups
./scripts/recover.sh list

# Output:
# Available Backups
# ─────────────────
# ID                  │ Timestamp           │ Components      │ Size
# ────────────────────┼─────────────────────┼─────────────────┼───────
# backup-20260401-1500│ 2026-04-01 15:00:00 │ all             │ 245MB
# backup-20260401-1400│ 2026-04-01 14:00:00 │ all             │ 243MB
# backup-20260401-1300│ 2026-04-01 13:00:00 │ all             │ 241MB
# backup-20260401-1200│ 2026-04-01 12:00:00 │ memory,ledger   │ 156MB
# backup-20260401-1100│ 2026-04-01 11:00:00 │ all             │ 238MB
#
# Total: 5 backups available
# Oldest: 2026-04-01 11:00:00
# Newest: 2026-04-01 15:00:00
```

```bash
# List recent backups only
./scripts/recover.sh list --recent --count 3

# Output:
# Recent Backups (Last 3)
# ───────────────────────
# backup-20260401-1500│ 2026-04-01 15:00:00 │ ✓ verified
# backup-20260401-1400│ 2026-04-01 14:00:00 │ ✓ verified
# backup-20260401-1300│ 2026-04-01 13:00:00 │ ✓ verified
```

#### Selecting Backup

```bash
# Auto-select best backup
./scripts/recover.sh select --auto

# Output:
# Backup Selection
# ────────────────
# Selected: backup-20260401-1500
#
# Selection Criteria:
#   Recency:      30% (score: 100)
#   Completeness: 25% (score: 100)
#   Integrity:    25% (score: 100)
#   Size:         10% (score: 98)
#   Age:          10% (score: 95)
#   ─────────────────────────────────
#   Total Score:  99.3/100
#
# Components: memory, ledger, workspace, config
# Size: 245 MB
# Verified: yes
```

```bash
# Select specific backup
./scripts/recover.sh select --backup backup-20260401-1400

# Output:
# Backup selected: backup-20260401-1400
# Timestamp: 2026-04-01 14:00:00
# Components: all
# Size: 243 MB
```

#### Previewing Recovery

```bash
# Preview recovery (dry-run)
./scripts/recover.sh preview --backup backup-20260401-1500

# Output:
# Recovery Preview
# ────────────────
# Backup: backup-20260401-1500
# Target Components: memory, ledger, workspace
#
# Changes:
#   - memory/steward/session.json (restore 1.2 MB)
#   - ledger/entries-20260401.db (restore 2 entries)
#   - workspace/config/cache.json (restore from backup)
#
# Services to Stop: steward, gateway
# Services to Restart: steward, gateway
# Estimated Downtime: 30 seconds
#
# Run with --execute to proceed.
```

```bash
# Preview with diff
./scripts/recover.sh preview --backup backup-20260401-1500 --diff

# Output:
# Recovery Diff
# ─────────────
# memory/steward/session.json
#   Current: 0 bytes (corrupted)
#   Backup:  1.2 MB
#   Action:  restore
#
# ledger/entries-20260401.db
#   Current: entries 145-146 corrupted
#   Backup:  entries 145-146 valid
#   Action:  patch entries
```

#### Executing Recovery

```bash
# Recover specific component
./scripts/recover.sh recover --component ledger --backup backup-20260401-1500 --validate

# Output:
# Recovery Execution
# ──────────────────
# Stage 1: Pre-Recovery
#   [✓] Backing up current state
#   [✓] Stopping affected services (steward)
#   [✓] Verifying backup integrity
#
# Stage 2: Recovery
#   [✓] Extracting backup data
#   [✓] Restoring ledger entries (145-146)
#   [✓] Updating metadata
#
# Stage 3: Post-Recovery
#   [✓] Validating restored data
#   [✓] Restarting services (steward)
#   [✓] Verifying system health
#
# Stage 4: Verification
#   [✓] Integrity checks passed
#   [✓] Functional tests passed
#   [✓] Consistency verified
#
# Recovery completed successfully.
```

```bash
# Auto-recover with validation
./scripts/recover.sh recover --auto --validate

# Output:
# Auto-Recovery Execution
# ───────────────────────
# Selected backup: backup-20260401-1500 (score: 99.3)
#
# Recovering components:
#   [✓] memory (3 files)
#   [✓] ledger (2 entries)
#   [✓] workspace (1 file)
#
# Validation: PASSED
# Rollback point: rollback-20260401-1530
```

#### Validating Recovery

```bash
# Post-recovery validation
./scripts/recover.sh validate

# Output:
# Recovery Validation
# ──────────────────
# Component Validation:
#   [✓] Agent Memory: 6/6 agents verified
#   [✓] Consensus Ledger: 147 entries verified
#   [✓] Workspace: 1,247 files verified
#
# Health Check:
#   Gateway: healthy (45ms)
#   LiteLLM: healthy (32ms)
#   Agents: 8/8 running
#
# Validation: PASSED
```

#### Rollback

```bash
# Rollback last recovery
./scripts/recover.sh rollback

# Output:
# Rollback Execution
# ──────────────────
# Rolling back to: rollback-20260401-1530
#
# Stage 1: Pre-Rollback
#   [✓] Backing up current state
#   [✓] Stopping affected services
#
# Stage 2: Rollback
#   [✓] Restoring pre-recovery state
#
# Stage 3: Post-Rollback
#   [✓] Restarting services
#   [✓] Verifying system health
#
# Rollback completed successfully.
```

### Recovery Stages

| Stage | Name | Actions |
|-------|------|---------|
| 1 | Pre-Recovery | Backup current state, stop services, verify backup |
| 2 | Recovery | Extract backup, restore files, update metadata |
| 3 | Post-Recovery | Validate data, restart services, verify health |
| 4 | Verification | Integrity checks, functional tests, consistency |

### Backup Selection Criteria

| Criterion | Weight | Description |
|-----------|--------|-------------|
| Recency | 30% | Prefer recent backups |
| Completeness | 25% | All components present |
| Integrity | 25% | Checksum verified |
| Size | 10% | Reasonable size (not truncated) |
| Age | 10% | Not too old |

---

## Common Debugging Scenarios

### Scenario 1: Agent Offline

**Symptoms:** Agent not responding, health check failures

**Workflow:**

```bash
# Step 1: Detect offline agent
./scripts/diagnostics.sh component --name agents --filter offline

# Output shows: alpha is offline

# Step 2: Check agent memory before restart
./scripts/inspect-state.sh memory --agent alpha --export

# Step 3: Check logs for cause
./scripts/analyze-logs.sh patterns --search "alpha.*offline\|alpha.*error"

# Step 4: Attempt restart
./scripts/lifecycle-manager.sh restart --agents alpha

# Step 5: Verify memory preserved
./scripts/inspect-state.sh memory --agent alpha --compare
```

### Scenario 2: LiteLLM Gateway Failure

**Symptoms:** API errors, model access failures

**Workflow:**

```bash
# Step 1: Check LiteLLM status
./scripts/diagnostics.sh component --name litellm --verbose

# Step 2: Analyze LiteLLM errors
./scripts/analyze-logs.sh analyze --component litellm --filter errors

# Step 3: Find timeout patterns
./scripts/analyze-logs.sh patterns --search "timeout\|rate limit"

# Step 4: Check LiteLLM state
./scripts/inspect-state.sh memory --component litellm

# Step 5: Restart if needed
docker compose -p heretek-openclaw-core restart litellm
```

### Scenario 3: Triad Deliberation Deadlock

**Symptoms:** Proposal stuck, votes not progressing

**Workflow:**

```bash
# Step 1: Check triad status
./scripts/triad-status.sh dashboard

# Step 2: Check for deadlock
./scripts/triad-status.sh check-deadlock

# Step 3: Inspect ledger for issues
./scripts/inspect-state.sh ledger --filter triad --since "1h ago"

# Step 4: Analyze triad logs
./scripts/analyze-logs.sh patterns --search "deliberation\|consensus\|deadlock"

# Step 5: Resolve deadlock
./scripts/triad-status.sh resolve-deadlock --proposal <id> --method steward-tiebreak
```

### Scenario 4: Database Corruption

**Symptoms:** Query failures, data inconsistencies

**Workflow:**

```bash
# Step 1: Check database integrity
./scripts/diagnostics.sh component --name database --integrity

# Step 2: Scan for corruption
./scripts/recover.sh scan --component database --full

# Step 3: List available backups
./scripts/recover.sh list --component database

# Step 4: Preview recovery
./scripts/recover.sh preview --component database --backup <backup-id>

# Step 5: Execute recovery
./scripts/recover.sh recover --component database --backup <backup-id> --validate
```

### Scenario 5: Memory Leak Detection

**Symptoms:** Increasing memory usage, OOM kills

**Workflow:**

```bash
# Step 1: Check memory usage
./scripts/diagnostics.sh health-score --breakdown

# Step 2: Inspect agent memories
./scripts/inspect-state.sh memory --all

# Step 3: Analyze memory-related logs
./scripts/analyze-logs.sh patterns --search "memory\|OOM\|heap"

# Step 4: Check for patterns over time
./scripts/analyze-logs.sh timeline --filter "memory" --since "24h ago"

# Step 5: Export state for analysis
./scripts/inspect-state.sh memory --all --output memory-report.json
```

### Scenario 6: Network Partition Recovery

**Symptoms:** Agents disconnected, WebSocket failures

**Workflow:**

```bash
# Step 1: Check network connectivity
./scripts/diagnostics.sh deps --network

# Step 2: Analyze connection errors
./scripts/analyze-logs.sh patterns --search "connection refused\|network\|partition"

# Step 3: Correlate events
./scripts/analyze-logs.sh correlate --event "agent-offline"

# Step 4: Check Gateway status
./scripts/gateway-pulse.sh status

# Step 5: Restart affected components
./scripts/lifecycle-manager.sh restart --agents gateway,litellm
```

---

## Related Documentation

- [Skills Operations Guide](./SKILLS_OPERATIONS_GUIDE.md) - Main operations guide
- [Orchestration Skills Guide](./ORCHESTRATION_SKILLS.md) - Orchestration skill usage
- [Skill Integration Patterns](./SKILL_INTEGRATION_PATTERNS.md) - Development patterns
- [Troubleshooting Workflows](./TROUBLESHOOTING_WORKFLOWS.md) - Scenario-based workflows

---

## Troubleshooting Tips

### Diagnostics Fail to Run

1. Verify Node.js version: `node --version` (requires >= 18.0.0)
2. Check script permissions: `chmod +x scripts/diagnostics.sh`
3. Verify dependencies: `./scripts/diagnostics.sh deps --node`

### Low Health Score

1. Run with breakdown: `./scripts/diagnostics.sh health-score --breakdown`
2. Check component status: `./scripts/diagnostics.sh component --name <component>`
3. Review aggregated logs: `./scripts/diagnostics.sh logs --aggregate --filter errors`

### Memory Inspection Fails

1. Verify agent is running: `docker ps | grep <agent>`
2. Check memory directory permissions: `ls -la /app/memory`
3. Use offline inspection: `./scripts/inspect-state.sh memory --offline`

### Recovery Fails

1. Check backup integrity: `./scripts/recover.sh validate-backup --backup <id>`
2. Try different backup: `./scripts/recover.sh list --valid-only`
3. Check disk space: `df -h`
