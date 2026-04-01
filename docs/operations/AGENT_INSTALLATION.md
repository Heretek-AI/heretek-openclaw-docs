# Agent Installation and Management Guide

**Last Updated:** 2026-04-01  
**Version:** 2026.3.31

---

## Table of Contents

1. [Overview](#overview)
2. [Prerequisites](#prerequisites)
3. [Agent Architecture](#agent-architecture)
4. [Installation Methods](#installation-methods)
5. [Agent Configuration](#agent-configuration)
6. [Managing Agents](#managing-agents)
7. [Troubleshooting](#troubleshooting)
8. [Reference](#reference)

---

## Overview

### What are OpenClaw Agents?

OpenClaw agents are autonomous AI entities that operate within the OpenClaw multi-agent system. Each agent is a specialized instance configured with unique identity, capabilities, and operational parameters. Agents work collaboratively to accomplish complex tasks through distributed reasoning and coordinated action.

### Role in the OpenClaw System

Agents serve as the primary execution units within the OpenClaw architecture:

- **Autonomous Operation**: Each agent operates independently with its own workspace, configuration, and identity
- **Specialized Roles**: Agents are configured for specific functions (orchestration, analysis, coding, security, etc.)
- **Collaborative Execution**: Agents communicate and coordinate through the OpenClaw gateway to accomplish shared objectives
- **Persistent State**: Each agent maintains its own conversation history, tools, and operational context

### Agent Lifecycle

```
┌─────────────┐    ┌──────────────┐    ┌─────────────┐    ┌─────────────┐
│  Template   │ →  │  Deployed    │ →  │   Running   │ →  │  Updated/   │
│  Files      │    │  Workspace   │    │   Agent     │    │  Removed    │
└─────────────┘    └──────────────┘    └─────────────┘    └─────────────┘
```

---

## Prerequisites

### System Requirements

| Requirement | Version | Description |
|-------------|---------|-------------|
| Operating System | Linux, macOS, WSL2 | Unix-like environment required |
| Bash | 4.0+ | Shell environment for deployment scripts |
| jq | 1.6+ | JSON processing utility |
| OpenClaw Core | 2026.3.31+ | Core agent runtime |

### Required Dependencies

#### Bash

Bash is typically pre-installed on Linux and macOS systems. Verify installation:

```bash
bash --version
```

#### jq

The `jq` command-line JSON processor is required for batch operations.

**Install on Ubuntu/Debian:**
```bash
sudo apt-get update && sudo apt-get install -y jq
```

**Install on macOS:**
```bash
brew install jq
```

**Install on RHEL/CentOS:**
```bash
sudo yum install -y jq
```

**Verify installation:**
```bash
jq --version
```

### Directory Structure

Agents are deployed to the following default location:

```
~/.openclaw/agents/<agent-id>/workspace/
```

Ensure the user account has write permissions to the target directory.

---

## Agent Architecture

### Workspace Structure

Each agent workspace contains the following structure:

```
~/.openclaw/agents/<agent-id>/
├── workspace/
│   ├── SOUL.md           # Core identity and purpose
│   ├── IDENTITY.md       # Agent persona and characteristics
│   ├── AGENTS.md         # Multi-agent collaboration guidelines
│   ├── USER.md           # User interaction protocols
│   ├── config.json       # Runtime configuration
│   ├── BOOTSTRAP.md      # Initialization instructions
│   └── TOOLS.md          # Available tools and capabilities
```

### Directory Components

| Component | Description |
|-----------|-------------|
| `~/.openclaw/` | Root OpenClaw installation directory |
| `agents/` | Container for all agent instances |
| `<agent-id>/` | Unique agent identifier directory |
| `workspace/` | Agent working directory with all configuration files |

### Agent ID Naming

Agent IDs follow a consistent naming convention:

- Lowercase alphanumeric characters
- Hyphens allowed for compound names (e.g., `sentinel-prime`, `habit-forge`)
- Must match the ID defined in [`openclaw.json`](../../heretek-openclaw-core/openclaw.json:1)

---

## Installation Methods

### Overview

The [`deploy-agent.sh`](../../heretek-openclaw-core/agents/deploy-agent.sh:1) script provides flexible deployment options:

| Method | Description | Use Case |
|--------|-------------|----------|
| Single Agent | Deploy one agent with custom parameters | Testing, individual agent setup |
| Batch | Deploy all agents from openclaw.json | Full system deployment |
| Custom Target | Deploy to alternative directory | Development, testing, custom setups |

---

### Batch Installation (All Agents)

Deploy all 22 agents defined in [`openclaw.json`](../../heretek-openclaw-core/openclaw.json:486) to their default locations.

#### Default Deployment (OpenClaw)

```bash
cd heretek-openclaw-core/agents
./deploy-agent.sh --batch
```

This deploys all agents to `~/.openclaw/agents/<agent>/workspace/` as specified in the `workspace` field of each agent entry.

#### Local Deployment

```bash
cd heretek-openclaw-core/agents
./deploy-agent.sh --batch --target local
```

This deploys all agents to `./deployed/<agent>/` within the script directory.

#### Expected Output

```
INFO: Reading agent configurations from: /path/to/openclaw.json
INFO: [1] Deploying agent: steward (steward)
  Created SOUL.md
  Created IDENTITY.md
  Created AGENTS.md
  Created USER.md
  Created config.json
SUCCESS: Agent steward deployed to: /root/.openclaw/agents/steward/workspace/

... (repeats for each agent)

==============================================
Batch Deployment Summary
==============================================
Total agents processed: 22
Successful: 22
Failed: 0
==============================================
```

---

### Single Agent Installation

Deploy a specific agent with custom parameters.

#### Syntax

```bash
./deploy-agent.sh <agent-id> <agent-name> <agent-role> [options]
```

#### Parameters

| Parameter | Description | Example |
|-----------|-------------|---------|
| `agent-id` | Unique identifier | `steward` |
| `agent-name` | Display name | `Steward` |
| `agent-role` | Agent role/function | `orchestrator` |

#### Examples

**Deploy to default location:**
```bash
./deploy-agent.sh steward Steward orchestrator
```

**Deploy to local directory:**
```bash
./deploy-agent.sh steward Steward orchestrator --target local
```

**Deploy with custom role:**
```bash
./deploy-agent.sh examiner Examiner quality-assurance
```

---

### Custom Target Directory Installation

Use the `--target` option to control deployment destination.

#### Target Options

| Target | Destination | Description |
|--------|-------------|-------------|
| `openclaw` (default) | `~/.openclaw/agents/<agent>/workspace/` | Production deployment |
| `local` | `./deployed/<agent>/` | Local development/testing |

#### Examples

**Deploy single agent to local directory:**
```bash
./deploy-agent.sh coder Coder developer --target local
```

**Deploy all agents to local directory:**
```bash
./deploy-agent.sh --batch --target local
```

---

### Help and Options

View available options:

```bash
./deploy-agent.sh --help
```

**Output:**
```
Agent Deployment Script

Usage:
  Single agent deployment:
    ./deploy-agent.sh <agent-id> <agent-name> <agent-role> [options]
    Example: ./deploy-agent.sh steward Steward orchestrator --target local

  Batch deployment (all agents from openclaw.json):
    ./deploy-agent.sh --batch
    ./deploy-agent.sh --batch --target local

Options:
  --target <target>    Deployment target: "openclaw" (default) or "local"
                       - openclaw: Deploys to ~/.openclaw/agents/<agent>/workspace/
                       - local: Deploys to ./deployed/<agent>/
  --batch              Deploy all agents defined in openclaw.json
  --help, -h           Show this help message
```

---

## Agent Configuration

### Configuration Files

Each agent workspace contains the following configuration files:

| File | Purpose | Editable |
|------|---------|----------|
| [`config.json`](#configjson) | Runtime configuration | Yes |
| [`SOUL.md`](#soulmd) | Core identity and purpose | Recommended: No |
| [`IDENTITY.md`](#identitymd) | Agent persona | Recommended: No |
| [`AGENTS.md`](#agentsmd) | Collaboration guidelines | No |
| [`USER.md`](#usermd) | User interaction | No |
| [`BOOTSTRAP.md`](#bootstrapmd) | Initialization | No |
| [`TOOLS.md`](#toolsmd) | Available tools | Yes |

---

### config.json

The primary runtime configuration file for each agent.

#### Structure

```json
{
  "id": "AGENT_ID_PLACEHOLDER",
  "name": "Agent Name",
  "role": "role_name",
  "model": {
    "primary": "litellm/claude-opus-4-6",
    "fallbacks": []
  },
  "workspace": "/root/.openclaw/agents/<agent-id>/workspace",
  "agentDir": "/root/.openclaw/agents/<agent-id>"
}
```

#### Fields

| Field | Type | Description |
|-------|------|-------------|
| `id` | string | Unique agent identifier |
| `name` | string | Human-readable agent name |
| `role` | string | Agent functional role |
| `model.primary` | string | Primary model endpoint |
| `model.fallbacks` | array | Fallback model endpoints |
| `workspace` | string | Agent workspace directory |
| `agentDir` | string | Agent root directory |

#### Model Configuration

Agents inherit model configuration from the defaults in [`openclaw.json`](../../heretek-openclaw-core/openclaw.json:423):

```json
{
  "defaults": {
    "model": {
      "primary": "litellm/claude-opus-4-6",
      "fallbacks": [
        "litellm/agent/steward",
        "litellm/agent/sentinel-prime",
        "... (additional fallbacks)"
      ]
    },
    "maxConcurrent": 4,
    "subagents": {
      "maxConcurrent": 8
    }
  }
}
```

---

### SOUL.md

Defines the agent's core identity, purpose, and existential parameters.

#### Purpose

- Establishes the agent's fundamental nature
- Defines core objectives and values
- Provides philosophical grounding

#### Example Content Structure

```markdown
# SOUL

## Core Identity
I am [Agent Name], an [agent type] within the OpenClaw system.

## Purpose
My fundamental purpose is to [core function].

## Values
- Integrity
- Collaboration
- Excellence
```

---

### IDENTITY.md

Defines the agent's persona, communication style, and behavioral characteristics.

#### Purpose

- Establishes personality traits
- Defines communication patterns
- Sets behavioral expectations

---

### AGENTS.md

Guidelines for multi-agent collaboration and communication.

#### Purpose

- Defines inter-agent protocols
- Establishes collaboration patterns
- Documents communication channels

---

### USER.md

Protocols and guidelines for user interaction.

#### Purpose

- Defines user interaction patterns
- Establishes communication expectations
- Documents user-facing capabilities

---

### BOOTSTRAP.md

Initialization and startup instructions for the agent.

#### Purpose

- Provides startup procedures
- Documents initialization sequence
- Lists pre-flight checks

---

### TOOLS.md

Documentation of available tools and capabilities.

#### Purpose

- Lists available tools
- Documents tool usage
- Provides capability reference

---

## Managing Agents

### Starting Agents

Agents are started through the OpenClaw runtime. Refer to the main OpenClaw documentation for runtime startup procedures.

#### Prerequisites

1. Ensure agent workspace is deployed
2. Verify `config.json` is properly configured
3. Confirm gateway is running

#### Starting a Single Agent

```bash
openclaw start --agent steward
```

#### Starting Multiple Agents

```bash
openclaw start --agents steward,coordinator,examiner
```

#### Starting All Agents

```bash
openclaw start --all
```

---

### Stopping Agents

#### Stopping a Single Agent

```bash
openclaw stop --agent steward
```

#### Stopping Multiple Agents

```bash
openclaw stop --agents steward,coordinator
```

#### Stopping All Agents

```bash
openclaw stop --all
```

#### Emergency Shutdown

For emergency shutdown procedures, see [`runbook-emergency-shutdown.md`](./runbook-emergency-shutdown.md).

---

### Updating Agents

#### Re-deploying an Agent

To update an agent's workspace files:

```bash
./deploy-agent.sh steward Steward orchestrator
```

This overwrites existing files in the workspace directory.

#### Updating Configuration

Edit [`config.json`](#configjson) directly in the agent workspace:

```bash
nano ~/.openclaw/agents/steward/workspace/config.json
```

#### Batch Update

Re-deploy all agents with updated templates:

```bash
./deploy-agent.sh --batch
```

---

### Monitoring Agents

#### Check Agent Status

```bash
openclaw status --agent steward
```

#### View Agent Logs

```bash
openclaw logs --agent steward
```

#### Health Check

```bash
openclaw healthcheck --agent steward
```

For comprehensive monitoring setup, see [`MONITORING_STACK.md`](./MONITORING_STACK.md).

---

### Removing Agents

#### Remove Single Agent

```bash
rm -rf ~/.openclaw/agents/steward
```

#### Remove Multiple Agents

```bash
rm -rf ~/.openclaw/agents/{steward,coordinator}
```

> **Warning:** This permanently deletes the agent workspace and all associated data. Ensure backups are created before removal.

---

## Troubleshooting

### Common Issues

#### Deployment Failures

| Issue | Cause | Solution |
|-------|-------|----------|
| `jq not found` | Missing jq dependency | Install jq (see [Prerequisites](#prerequisites)) |
| `Template directory not found` | Missing templates | Verify `agents/templates/` directory exists |
| `openclaw.json not found` | Invalid path | Run script from correct directory |
| `Permission denied` | Insufficient permissions | Ensure write access to target directory |

#### Runtime Issues

| Issue | Cause | Solution |
|-------|-------|----------|
| Agent fails to start | Invalid config.json | Verify JSON syntax and required fields |
| Model connection failed | Gateway not running | Start LiteLLM gateway first |
| Workspace not found | Deployment incomplete | Re-run deploy-agent.sh |

---

### Diagnostic Commands

#### Verify Deployment

```bash
ls -la ~/.openclaw/agents/<agent-id>/workspace/
```

Expected files:
- SOUL.md
- IDENTITY.md
- AGENTS.md
- USER.md
- config.json
- BOOTSTRAP.md
- TOOLS.md

#### Validate Configuration

```bash
jq . ~/.openclaw/agents/<agent-id>/workspace/config.json
```

#### Check Template Directory

```bash
ls -la heretek-openclaw-core/agents/templates/
```

---

### Log Analysis

#### Agent Logs Location

```
~/.openclaw/logs/<agent-id>.log
```

#### Common Log Patterns

**Successful Startup:**
```
[INFO] Agent steward initializing...
[INFO] Loading workspace configuration...
[INFO] Connecting to gateway...
[INFO] Agent steward ready
```

**Configuration Error:**
```
[ERROR] Failed to parse config.json
[ERROR] Missing required field: id
```

**Connection Error:**
```
[ERROR] Failed to connect to gateway
[ERROR] Connection refused: http://localhost:4000
```

---

### Recovery Procedures

#### Corrupted Workspace

1. Backup existing data (if recoverable)
2. Re-deploy the agent:
   ```bash
   ./deploy-agent.sh <agent-id> <name> <role>
   ```
3. Restore any custom configurations

#### Configuration Corruption

1. Backup current config:
   ```bash
   cp ~/.openclaw/agents/<agent>/workspace/config.json config.json.bak
   ```
2. Re-deploy to regenerate:
   ```bash
   ./deploy-agent.sh <agent-id> <name> <role>
   ```
3. Restore custom settings from backup

---

## Reference

### Complete Agent List

The OpenClaw system includes 22 agents, each with a specialized role:

| ID | Name | Workspace Path | Primary Function |
|----|------|----------------|------------------|
| `steward` | Steward | `/root/.openclaw/agents/steward/workspace` | System orchestration and coordination |
| `coordinator` | Coordinator | `/root/.openclaw/agents/coordinator/workspace` | Task coordination and workflow management |
| `examiner` | Examiner | `/root/.openclaw/agents/examiner/workspace` | Quality assurance and validation |
| `explorer` | Explorer | `/root/.openclaw/agents/explorer/workspace` | Discovery and research |
| `alpha` | Alpha | `/root/.openclaw/agents/alpha/workspace` | Primary analysis agent |
| `arbiter` | Arbiter | `/root/.openclaw/agents/arbiter/workspace` | Decision mediation and conflict resolution |
| `beta` | Beta | `/root/.openclaw/agents/beta/workspace` | Secondary analysis agent |
| `catalyst` | Catalyst | `/root/.openclaw/agents/catalyst/workspace` | Change initiation and optimization |
| `charlie` | Charlie | `/root/.openclaw/agents/charlie/workspace` | Tertiary analysis agent |
| `chronos` | Chronos | `/root/.openclaw/agents/chronos/workspace` | Time-based operations and scheduling |
| `coder` | Coder | `/root/.openclaw/agents/coder/workspace` | Code generation and modification |
| `dreamer` | Dreamer | `/root/.openclaw/agents/dreamer/workspace` | Creative and exploratory thinking |
| `echo` | Echo | `/root/.openclaw/agents/echo/workspace` | Communication and message relay |
| `empath` | Empath | `/root/.openclaw/agents/empath/workspace` | Sentiment and emotional analysis |
| `habit-forge` | Habit-Forge | `/root/.openclaw/agents/habit-forge/workspace` | Pattern formation and habit building |
| `historian` | Historian | `/root/.openclaw/agents/historian/workspace` | Historical record and documentation |
| `metis` | Metis | `/root/.openclaw/agents/metis/workspace` | Strategic wisdom and insight |
| `nexus` | Nexus | `/root/.openclaw/agents/nexus/workspace` | Central hub and connectivity |
| `perceiver` | Perceiver | `/root/.openclaw/agents/perceiver/workspace` | Sensory input and perception |
| `prism` | Prism | `/root/.openclaw/agents/prism/workspace` | Multi-perspective analysis |
| `sentinel` | Sentinel | `/root/.openclaw/agents/sentinel/workspace` | Security monitoring and protection |
| `sentinel-prime` | Sentinel-Prime | `/root/.openclaw/agents/sentinel-prime/workspace` | Primary security coordination |

### Agent Categories

#### Orchestration Agents
- `steward` - Primary orchestrator
- `coordinator` - Workflow coordination
- `nexus` - Hub connectivity

#### Analysis Agents
- `alpha` - Primary analysis
- `beta` - Secondary analysis
- `charlie` - Tertiary analysis
- `prism` - Multi-perspective analysis
- `perceiver` - Input perception

#### Specialized Function Agents
- `coder` - Code operations
- `examiner` - Quality assurance
- `explorer` - Research and discovery
- `historian` - Documentation
- `empath` - Sentiment analysis
- `chronos` - Time operations
- `habit-forge` - Pattern formation
- `metis` - Strategic insight
- `dreamer` - Creative thinking
- `echo` - Communication
- `catalyst` - Optimization
- `arbiter` - Conflict resolution

#### Security Agents
- `sentinel` - Security monitoring
- `sentinel-prime` - Security coordination

### Quick Reference Commands

```bash
# Deploy all agents
./deploy-agent.sh --batch

# Deploy single agent
./deploy-agent.sh <id> <name> <role>

# Deploy to local directory
./deploy-agent.sh --batch --target local

# View help
./deploy-agent.sh --help

# Start agent
openclaw start --agent <id>

# Stop agent
openclaw stop --agent <id>

# Check status
openclaw status --agent <id>

# View logs
openclaw logs --agent <id>
```

---

## Related Documentation

- [`runbook-agent-restart.md`](./runbook-agent-restart.md) - Agent restart procedures
- [`runbook-emergency-shutdown.md`](./runbook-emergency-shutdown.md) - Emergency shutdown
- [`MONITORING_STACK.md`](./MONITORING_STACK.md) - Monitoring configuration
- [`HEALTH_DASHBOARD.md`](./HEALTH_DASHBOARD.md) - Health dashboard setup
- [`../../heretek-openclaw-core/README.md`](../../heretek-openclaw-core/README.md) - OpenClaw Core documentation

---

*This documentation is part of the OpenClaw Operations Guide. For contributions or corrections, see the [CONTRIBUTING.md](../README.md) guide.*
