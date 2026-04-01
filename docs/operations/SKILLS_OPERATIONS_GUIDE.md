# OpenClaw Skills Operations Guide

**Version:** 1.0  
**Last Updated:** 2026-04-01  
**Scope:** Orchestration and Debugging Skills

---

## Table of Contents

1. [Overview](#overview)
2. [Quick Reference](#quick-reference)
3. [Installation and Setup](#installation-and-setup)
4. [Integration with OpenClaw Gateway](#integration-with-openclaw-gateway)
5. [Best Practices](#best-practices)

---

## Overview

OpenClaw skills are modular, reusable components that provide specialized functionality for system orchestration and debugging. This guide covers the 7 core skills organized into two categories:

### Orchestration Skills (3)

Orchestration skills manage system operations, monitoring, and coordination:

| Skill | Purpose | Primary Use Case |
|-------|---------|------------------|
| [`agent-lifecycle-manager`](../../../heretek-openclaw-core/.roo/skills/agent-lifecycle-manager/SKILL.md:1) | Agent process management | Start, stop, restart, and monitor agent fleet |
| [`triad-orchestrator`](../../../heretek-openclaw-core/.roo/skills/triad-orchestrator/SKILL.md:1) | Governance deliberation | Manage proposals, votes, and consensus |
| [`gateway-pulse`](../../../heretek-openclaw-core/.roo/skills/gateway-pulse/SKILL.md:1) | Infrastructure monitoring | Monitor Gateway and LiteLLM health |

### Debugging Skills (4)

Debugging skills provide diagnostic and recovery capabilities:

| Skill | Purpose | Primary Use Case |
|-------|---------|------------------|
| [`system-diagnostics`](../../../heretek-openclaw-core/.roo/skills/system-diagnostics/SKILL.md:1) | Comprehensive health checks | Full system validation and health scoring |
| [`log-analyzer`](../../../heretek-openclaw-core/.roo/skills/log-analyzer/SKILL.md:1) | Log analysis and correlation | Pattern detection and root cause analysis |
| [`state-inspector`](../../../heretek-openclaw-core/.roo/skills/state-inspector/SKILL.md:1) | Deep state inspection | Memory, ledger, and workspace auditing |
| [`corruption-recovery`](../../../heretek-openclaw-core/.roo/skills/corruption-recovery/SKILL.md:1) | Automated recovery | Backup restoration and disaster recovery |

---

## Quick Reference

### Skill Execution Methods

All skills support three execution methods:

| Method | Command | Use Case |
|--------|---------|----------|
| **CLI Script** | `./scripts/<skill-name>.sh <command>` | Interactive operations |
| **Gateway RPC** | WebSocket RPC via port 18789 | Programmatic access |
| **Direct Node.js** | `node src/index.js <args>` | Custom integrations |

### Command Quick Reference

#### Agent Lifecycle Manager

```bash
./scripts/lifecycle-manager.sh status              # View all agent statuses
./scripts/lifecycle-manager.sh start-all           # Start all agents
./scripts/lifecycle-manager.sh restart --agents <ids>  # Restart specific
./scripts/lifecycle-manager.sh rolling-restart     # Zero-downtime restart
./scripts/lifecycle-manager.sh dashboard           # Status dashboard
```

#### Triad Orchestrator

```bash
./scripts/triad-status.sh propose --title <title>  # Create proposal
./scripts/triad-status.sh vote --proposal <id>     # Submit vote
./scripts/triad-status.sh check-deadlock           # Detect deadlocks
./scripts/triad-status.sh dashboard                # Triad dashboard
```

#### Gateway Pulse

```bash
./scripts/gateway-pulse.sh status                  # Service status
./scripts/gateway-pulse.sh monitor                 # Start monitoring
./scripts/gateway-pulse.sh alerts --set            # Configure alerts
./scripts/gateway-pulse.sh metrics --export        # Export Prometheus
```

#### System Diagnostics

```bash
./scripts/diagnostics.sh full                      # Full system check
./scripts/diagnostics.sh component --name <name>   # Component check
./scripts/diagnostics.sh health-score              # Health score only
./scripts/diagnostics.sh config --all              # Config validation
```

#### Log Analyzer

```bash
./scripts/analyze-logs.sh analyze --all            # Analyze all logs
./scripts/analyze-logs.sh patterns                 # Detect patterns
./scripts/analyze-logs.sh correlate                # Cross-log correlation
./scripts/analyze-logs.sh timeline                 # Build timeline
```

#### State Inspector

```bash
./scripts/inspect-state.sh memory --agent <id>     # Inspect memory
./scripts/inspect-state.sh session                 # Session state
./scripts/inspect-state.sh ledger                  # Audit ledger
./scripts/inspect-state.sh scan                    # Corruption scan
```

#### Corruption Recovery

```bash
./scripts/recover.sh scan --full                   # Scan for corruption
./scripts/recover.sh list                          # List backups
./scripts/recover.sh recover --auto                # Auto-recover
./scripts/recover.sh validate                      # Validate recovery
```

---

## Installation and Setup

### Prerequisites

- Node.js >= 18.0.0
- Docker and Docker Compose (for containerized deployments)
- OpenClaw Gateway running on port 18789
- LiteLLM running on port 4000

### Installation Steps

1. **Clone or navigate to the OpenClaw core directory:**

   ```bash
   cd heretek-openclaw-core
   ```

2. **Install skill dependencies:**

   ```bash
   # Install all skill dependencies
   for skill in .roo/skills/*/; do
     cd "$skill" && npm install && cd -
   done
   ```

3. **Verify script permissions:**

   ```bash
   chmod +x .roo/skills/*/scripts/*.sh
   ```

4. **Configure environment variables:**

   Create `.env` file in each skill directory:

   ```bash
   GATEWAY_URL=ws://localhost:18789
   LITELLM_URL=http://localhost:4000
   LOG_LEVEL=info
   ```

5. **Test skill execution:**

   ```bash
   # Test system diagnostics
   .roo/skills/system-diagnostics/scripts/diagnostics.sh health-score
   ```

### Verification

Run a quick health check to verify all skills are accessible:

```bash
# Expected output: Health Score with component breakdown
.roo/skills/system-diagnostics/scripts/diagnostics.sh health-score --breakdown
```

---

## Integration with OpenClaw Gateway

### WebSocket RPC Protocol

All skills integrate with the OpenClaw Gateway via WebSocket RPC on port 18789.

#### Connection Setup

```javascript
const WebSocket = require('ws');
const ws = new WebSocket('ws://localhost:18789/ws');

ws.on('open', () => {
  // Send RPC request
  ws.send(JSON.stringify({
    jsonrpc: '2.0',
    id: 1,
    method: 'skill.execute',
    params: {
      skill: 'system-diagnostics',
      command: 'health-score'
    }
  }));
});
```

#### Available RPC Methods

| Method | Parameters | Description |
|--------|------------|-------------|
| `skill.execute` | `skill`, `command`, `args` | Execute skill command |
| `skill.status` | `skill` | Get skill execution status |
| `skill.cancel` | `executionId` | Cancel running skill |
| `health.check` | `component` | Check component health |

#### Response Format

```json
{
  "jsonrpc": "2.0",
  "id": 1,
  "result": {
    "status": "success",
    "data": {
      "healthScore": 87,
      "components": {...}
    }
  }
}
```

### AgentClient Integration

Skills use the AgentClient for inter-agent communication:

```javascript
const { AgentClient } = require('./src/agent-client');

const client = new AgentClient({
  gatewayUrl: 'ws://localhost:18789',
  agentId: 'steward'
});

// Send message to another agent
await client.send('alpha', {
  type: 'skill.invocation',
  skill: 'triad-orchestrator',
  action: 'vote',
  data: { proposal: 'prop-001', vote: 'approve' }
});
```

---

## Best Practices

### Skill Execution

1. **Always check status before operations:**
   ```bash
   # Check agent status before restart
   ./scripts/lifecycle-manager.sh status --agent <agent-id>
   ```

2. **Use dry-run for destructive operations:**
   ```bash
   # Preview recovery before executing
   ./scripts/recover.sh preview --backup <backup-id> --diff
   ```

3. **Enable validation after recovery:**
   ```bash
   # Recover with automatic validation
   ./scripts/recover.sh recover --auto --validate
   ```

### Monitoring and Alerting

1. **Set appropriate alert thresholds:**
   ```bash
   # Configure warning at 5s, critical at 10s
   ./scripts/gateway-pulse.sh alerts --set --warning 5000 --critical 10000
   ```

2. **Export metrics for external monitoring:**
   ```bash
   # Export to Prometheus format
   ./scripts/gateway-pulse.sh metrics --export --format prometheus
   ```

3. **Enable auto-remediation for common failures:**
   ```bash
   # Enable automatic recovery
   ./scripts/gateway-pulse.sh remediate --enable
   ```

### Troubleshooting

1. **Start with system diagnostics:**
   ```bash
   # Get overall health score first
   ./scripts/diagnostics.sh health-score --breakdown
   ```

2. **Use log analyzer for root cause:**
   ```bash
   # Find root cause suggestions
   ./scripts/analyze-logs.sh root-cause
   ```

3. **Inspect state for corruption:**
   ```bash
   # Scan for state corruption
   ./scripts/inspect-state.sh scan --full
   ```

### Security Considerations

1. **Restrict skill access to authorized users**
2. **Validate all inputs before skill execution**
3. **Log all skill invocations for audit**
4. **Use secure WebSocket connections (wss://) in production**

---

## Related Documentation

- [Orchestration Skills Guide](./ORCHESTRATION_SKILLS.md) - Detailed orchestration skill usage
- [Debugging Skills Guide](./DEBUGGING_SKILLS.md) - Detailed debugging skill usage
- [Skill Integration Patterns](./SKILL_INTEGRATION_PATTERNS.md) - Development and integration guide
- [Troubleshooting Workflows](./TROUBLESHOOTING_WORKFLOWS.md) - Scenario-based workflows

---

## Support

For issues or questions:
1. Check the specific skill documentation in `heretek-openclaw-core/.roo/skills/<skill>/SKILL.md`
2. Review troubleshooting workflows in [`TROUBLESHOOTING_WORKFLOWS.md`](./TROUBLESHOOTING_WORKFLOWS.md)
3. Run system diagnostics to identify issues
