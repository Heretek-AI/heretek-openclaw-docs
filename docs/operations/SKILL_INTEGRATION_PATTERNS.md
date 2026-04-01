# Skill Integration Patterns

**Version:** 1.0  
**Last Updated:** 2026-04-01  
**Scope:** Skill Development and Integration

---

## Table of Contents

1. [Skill Architecture Overview](#skill-architecture-overview)
2. [Gateway WebSocket RPC Protocol](#gateway-websocket-rpc-protocol)
3. [AgentClient Integration Patterns](#agentclient-integration-patterns)
4. [Skill Development Workflow](#skill-development-workflow)
5. [Testing Strategies](#testing-strategies)
6. [Extending Existing Skills](#extending-existing-skills)

---

## Skill Architecture Overview

### Directory Structure

Each skill follows a standardized directory structure:

```
skill-name/
├── SKILL.md                 # Skill documentation and usage guide
├── package.json             # Node.js dependencies
├── src/
│   ├── index.js             # Main skill entry point
│   ├── module-a.js          # Feature module
│   ├── module-b.js          # Feature module
│   └── module-c.js          # Feature module
├── scripts/
│   └── skill-name.sh        # CLI wrapper script
└── tests/
    └── skill-name.test.js   # Skill tests
```

### Component Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                        Skill                                 │
├─────────────────────────────────────────────────────────────┤
│  ┌─────────────────────────────────────────────────────┐    │
│  │                    index.js                          │    │
│  │  - Command parsing                                   │    │
│  │  - Module orchestration                              │    │
│  │  - Output formatting                                 │    │
│  └─────────────────────────────────────────────────────┘    │
│                                                              │
│  ┌─────────────┐ ┌─────────────┐ ┌─────────────┐            │
│  │  Module A   │ │  Module B   │ │  Module C   │            │
│  │             │ │             │ │             │            │
│  │  Feature    │ │  Feature    │ │  Feature    │            │
│  │  Logic      │ │  Logic      │ │  Logic      │            │
│  └─────────────┘ └─────────────┘ └─────────────┘            │
└─────────────────────────────────────────────────────────────┘
```

### Example: System Diagnostics Structure

```
system-diagnostics/
├── SKILL.md
├── package.json
├── src/
│   ├── index.js              # Main entry, command routing
│   ├── config-validator.js   # Configuration validation
│   ├── dependency-checker.js # Dependency verification
│   └── health-scorer.js      # Health score calculation
├── scripts/
│   └── diagnostics.sh        # CLI wrapper
└── tests/
    └── diagnostics.test.js
```

### Skill Lifecycle

```
┌──────────┐    ┌──────────┐    ┌──────────┐    ┌──────────┐
│  Init    │ →  │ Execute  │ →  │  Report  │ →  │ Cleanup  │
└──────────┘    └──────────┘    └──────────┘    └──────────┘
     │               │               │               │
     ▼               ▼               ▼               ▼
 Load config   Run commands    Format output   Release resources
 Validate      Collect results Publish status  Close connections
 Initialize    Handle errors   Log results     Save state
```

---

## Gateway WebSocket RPC Protocol

### Connection Setup

All skills connect to the OpenClaw Gateway via WebSocket RPC on port 18789.

```javascript
const WebSocket = require('ws');

class SkillClient {
  constructor(options = {}) {
    this.gatewayUrl = options.gatewayUrl || 'ws://localhost:18789/ws';
    this.agentId = options.agentId;
    this.ws = null;
    this.requestId = 0;
    this.pendingRequests = new Map();
  }

  async connect() {
    return new Promise((resolve, reject) => {
      this.ws = new WebSocket(this.gatewayUrl);
      
      this.ws.on('open', () => {
        console.log('Connected to Gateway');
        resolve();
      });
      
      this.ws.on('message', (data) => {
        this.handleMessage(JSON.parse(data));
      });
      
      this.ws.on('error', reject);
    });
  }

  async send(method, params) {
    const id = ++this.requestId;
    
    return new Promise((resolve, reject) => {
      this.pendingRequests.set(id, { resolve, reject });
      
      this.ws.send(JSON.stringify({
        jsonrpc: '2.0',
        id,
        method,
        params
      }));
      
      // Timeout after 30 seconds
      setTimeout(() => {
        this.pendingRequests.delete(id);
        reject(new Error('Request timeout'));
      }, 30000);
    });
  }

  handleMessage(response) {
    const { id, result, error } = response;
    const pending = this.pendingRequests.get(id);
    
    if (pending) {
      this.pendingRequests.delete(id);
      if (error) {
        pending.reject(new Error(error.message));
      } else {
        pending.resolve(result);
      }
    }
  }
}
```

### RPC Methods

| Method | Parameters | Returns | Description |
|--------|------------|---------|-------------|
| `skill.execute` | `skill`, `command`, `args` | `result` | Execute skill command |
| `skill.status` | `skill`, `executionId` | `status` | Get execution status |
| `skill.cancel` | `executionId` | `boolean` | Cancel running skill |
| `health.check` | `component` | `health` | Check component health |
| `agent.list` | `filter` | `agents[]` | List agents |
| `agent.status` | `agentId` | `status` | Get agent status |

### Request Format

```json
{
  "jsonrpc": "2.0",
  "id": 1,
  "method": "skill.execute",
  "params": {
    "skill": "system-diagnostics",
    "command": "health-score",
    "args": {
      "breakdown": true,
      "format": "json"
    }
  }
}
```

### Response Format

```json
{
  "jsonrpc": "2.0",
  "id": 1,
  "result": {
    "status": "success",
    "data": {
      "score": 87,
      "status": "Good",
      "breakdown": {
        "gateway": 25,
        "litellm": 25,
        "database": 18,
        "agents": 17,
        "system": 9
      }
    },
    "executionTime": 1234
  }
}
```

### Error Format

```json
{
  "jsonrpc": "2.0",
  "id": 1,
  "error": {
    "code": -32000,
    "message": "Skill execution failed",
    "data": {
      "skill": "system-diagnostics",
      "command": "health-score",
      "reason": "Gateway not responding"
    }
  }
}
```

### Error Codes

| Code | Description |
|------|-------------|
| -32000 | Skill execution failed |
| -32001 | Skill not found |
| -32002 | Invalid command |
| -32003 | Agent offline |
| -32004 | Gateway unavailable |
| -32005 | Timeout exceeded |

---

## AgentClient Integration Patterns

### Basic AgentClient Usage

```javascript
const { AgentClient } = require('./src/agent-client');

// Create client for specific agent
const client = new AgentClient({
  gatewayUrl: 'ws://localhost:18789',
  agentId: 'steward'
});

// Connect to Gateway
await client.connect();

// Send message to another agent
await client.send('alpha', {
  type: 'skill.invocation',
  skill: 'triad-orchestrator',
  action: 'vote',
  data: {
    proposal: 'prop-001',
    vote: 'approve'
  }
});

// Receive messages
client.on('message', (message) => {
  console.log('Received:', message);
});
```

### Message Types

```javascript
// Skill invocation message
{
  type: 'skill.invocation',
  skill: 'skill-name',
  action: 'command',
  data: { /* command args */ }
}

// Status query message
{
  type: 'status.query',
  target: 'agent-id',
  requestId: 'unique-id'
}

// Status response message
{
  type: 'status.response',
  requestId: 'unique-id',
  status: {
    state: 'running',
    health: 'healthy',
    uptime: 3600
  }
}

// Event notification
{
  type: 'event.notification',
  event: 'agent.offline',
  data: {
    agentId: 'alpha',
    timestamp: '2026-04-01T15:30:00Z'
  }
}
```

### Subscription Pattern

```javascript
const { AgentClient } = require('./src/agent-client');

class SkillWithSubscriptions {
  constructor(agentId) {
    this.client = new AgentClient({ agentId });
    this.subscriptions = new Map();
  }

  async subscribe(eventType, handler) {
    await this.client.subscribe(eventType);
    
    this.client.on('message', (message) => {
      if (message.type === 'event.notification' && 
          message.event === eventType) {
        handler(message.data);
      }
    });
    
    this.subscriptions.set(eventType, handler);
  }

  async unsubscribe(eventType) {
    await this.client.unsubscribe(eventType);
    this.subscriptions.delete(eventType);
  }
}

// Usage
const skill = new SkillWithSubscriptions('steward');

await skill.subscribe('agent.offline', (data) => {
  console.log(`Agent ${data.agentId} went offline`);
  // Trigger recovery workflow
});
```

### Request-Response Pattern

```javascript
const { AgentClient } = require('./src/agent-client');

class SkillWithRPC {
  constructor(agentId) {
    this.client = new AgentClient({ agentId });
    this.pendingRequests = new Map();
  }

  async request(targetAgent, action, data) {
    const requestId = this.generateRequestId();
    
    return new Promise((resolve, reject) => {
      this.pendingRequests.set(requestId, { resolve, reject });
      
      this.client.send(targetAgent, {
        type: 'rpc.request',
        requestId,
        action,
        data
      });
      
      // Timeout after 30 seconds
      setTimeout(() => {
        this.pendingRequests.delete(requestId);
        reject(new Error('Request timeout'));
      }, 30000);
    });
  }

  async handleMessage(message) {
    if (message.type === 'rpc.response') {
      const pending = this.pendingRequests.get(message.requestId);
      if (pending) {
        this.pendingRequests.delete(message.requestId);
        if (message.error) {
          pending.reject(new Error(message.error));
        } else {
          pending.resolve(message.data);
        }
      }
    }
  }

  generateRequestId() {
    return `req-${Date.now()}-${Math.random().toString(36).substr(2, 9)}`;
  }
}
```

---

## Skill Development Workflow

### Step 1: Create Skill Directory

```bash
# Create skill directory structure
mkdir -p my-skill/{src,scripts,tests}
cd my-skill

# Initialize package.json
npm init -y

# Install dependencies
npm install ws commander chalk
```

### Step 2: Create SKILL.md

```markdown
---
name: my-skill
description: Description of what the skill does
---

# My Skill

## When to use this skill

Use when you need to...

## When NOT to use this skill

Do NOT use when...

## Inputs required

1. **Parameter 1**: Description
2. **Parameter 2**: Description

## Workflow

### Command examples

```bash
./scripts/my-skill.sh command --option value
```
```

### Step 3: Implement Main Entry Point

```javascript
// src/index.js
const { Command } = require('commander');
const chalk = require('chalk');
const { featureModule } = require('./feature-module');

const program = new Command();

program
  .name('my-skill')
  .description('My skill description')
  .version('1.0.0');

program
  .command('execute')
  .description('Execute the skill')
  .requiredOption('-t, --target <target>', 'Target to operate on')
  .option('-v, --verbose', 'Verbose output', false)
  .action(async (options) => {
    try {
      console.log(chalk.blue('Executing skill...'));
      
      const result = await featureModule.execute(options.target);
      
      console.log(chalk.green('Success!'));
      console.log(JSON.stringify(result, null, 2));
    } catch (error) {
      console.error(chalk.red('Error:'), error.message);
      process.exit(1);
    }
  });

program.parse();
```

### Step 4: Implement Feature Module

```javascript
// src/feature-module.js
const { AgentClient } = require('./agent-client');

class FeatureModule {
  constructor(options = {}) {
    this.client = new AgentClient(options);
  }

  async execute(target) {
    await this.client.connect();
    
    // Perform operations
    const status = await this.client.getStatus(target);
    
    // Process results
    const result = await this.processStatus(status);
    
    await this.client.disconnect();
    
    return result;
  }

  async processStatus(status) {
    // Implementation logic
    return {
      target: status.target,
      healthy: status.healthy,
      details: status.details
    };
  }
}

module.exports = new FeatureModule();
```

### Step 5: Create CLI Wrapper

```bash
#!/bin/bash
# scripts/my-skill.sh

SCRIPT_DIR="$(cd "$(dirname "${BASH_SOURCE[0]}")" && pwd)"
SKILL_DIR="$(dirname "$SCRIPT_DIR")"

cd "$SKILL_DIR"

# Run the Node.js skill
node src/index.js "$@"
```

Make executable:

```bash
chmod +x scripts/my-skill.sh
```

### Step 6: Add Tests

```javascript
// tests/my-skill.test.js
const { describe, it, expect } = require('vitest');
const featureModule = require('../src/feature-module');

describe('FeatureModule', () => {
  it('should execute successfully', async () => {
    const result = await featureModule.execute('test-target');
    expect(result).toBeDefined();
    expect(result.target).toBe('test-target');
  });

  it('should handle errors gracefully', async () => {
    await expect(featureModule.execute('invalid'))
      .rejects.toThrow('Invalid target');
  });
});
```

---

## Testing Strategies

### Unit Testing

```javascript
// tests/unit/feature-module.test.js
const { describe, it, expect, vi, beforeEach } = require('vitest');
const FeatureModule = require('../../src/feature-module');
const { AgentClient } = require('../../src/agent-client');

// Mock AgentClient
vi.mock('../../src/agent-client');

describe('FeatureModule Unit Tests', () => {
  let module;
  let mockClient;

  beforeEach(() => {
    mockClient = {
      connect: vi.fn().mockResolvedValue(),
      disconnect: vi.fn().mockResolvedValue(),
      getStatus: vi.fn().mockResolvedValue({ healthy: true })
    };
    AgentClient.mockImplementation(() => mockClient);
    module = new FeatureModule();
  });

  it('should connect before executing', async () => {
    await module.execute('target');
    expect(mockClient.connect).toHaveBeenCalled();
  });

  it('should disconnect after executing', async () => {
    await module.execute('target');
    expect(mockClient.disconnect).toHaveBeenCalled();
  });

  it('should process status correctly', async () => {
    mockClient.getStatus.mockResolvedValue({
      target: 'test',
      healthy: true,
      details: { cpu: 50 }
    });

    const result = await module.execute('test');
    expect(result.healthy).toBe(true);
  });
});
```

### Integration Testing

```javascript
// tests/integration/skill-integration.test.js
const { describe, it, expect, beforeAll, afterAll } = require('vitest');
const { execSync } = require('child_process');
const { spawn } = require('child_process');

describe('Skill Integration Tests', () => {
  let gatewayProcess;

  beforeAll(async () => {
    // Start test Gateway
    gatewayProcess = spawn('docker', [
      'compose', '-p', 'test-openclaw', 'up', 'gateway'
    ]);
    
    // Wait for Gateway to be ready
    await waitForGateway();
  });

  afterAll(() => {
    // Clean up
    execSync('docker compose -p test-openclaw down');
  });

  it('should execute health check', () => {
    const output = execSync('./scripts/my-skill.sh health-check', {
      encoding: 'utf8'
    });
    expect(output).toContain('healthy');
  });

  it('should handle invalid input', () => {
    expect(() => {
      execSync('./scripts/my-skill.sh execute --invalid', {
        encoding: 'utf8'
      });
    }).toThrow();
  });
});

async function waitForGateway() {
  // Wait for Gateway to be available
  return new Promise((resolve) => {
    const check = setInterval(() => {
      try {
        execSync('curl -s http://localhost:18789/health');
        clearInterval(check);
        resolve();
      } catch {
        // Keep waiting
      }
    }, 1000);
  });
}
```

### End-to-End Testing

```javascript
// tests/e2e/workflow.test.js
const { describe, it, expect } = require('vitest');
const { AgentClient } = require('../../src/agent-client');

describe('End-to-End Workflow Tests', () => {
  it('should complete full skill workflow', async () => {
    const client = new AgentClient({ agentId: 'steward' });
    await client.connect();

    // Step 1: Check initial state
    const initialState = await client.getStatus('alpha');
    expect(initialState.state).toBe('running');

    // Step 2: Execute skill command
    await client.send('alpha', {
      type: 'skill.invocation',
      skill: 'my-skill',
      action: 'execute',
      data: { target: 'test' }
    });

    // Step 3: Wait for completion
    await sleep(1000);

    // Step 4: Verify result
    const finalState = await client.getStatus('alpha');
    expect(finalState.lastAction).toBe('execute');

    await client.disconnect();
  });
});

function sleep(ms) {
  return new Promise(resolve => setTimeout(resolve, ms));
}
```

### Test Coverage Configuration

```javascript
// vitest.config.js
import { defineConfig } from 'vitest/config';

export default defineConfig({
  test: {
    coverage: {
      provider: 'v8',
      reporter: ['text', 'json', 'html'],
      threshold: {
        lines: 80,
        functions: 80,
        branches: 70,
        statements: 80
      }
    }
  }
});
```

---

## Extending Existing Skills

### Extending System Diagnostics

```javascript
// src/custom-health-check.js
const { HealthScorer } = require('./health-scorer');

class CustomHealthScorer extends HealthScorer {
  constructor(options = {}) {
    super(options);
    this.customWeights = options.customWeights || {};
  }

  calculateBreakdown(components) {
    const breakdown = super.calculateBreakdown(components);
    
    // Add custom component
    if (this.customWeights.custom) {
      breakdown.custom = this.scoreCustomComponent(components.custom);
    }
    
    return breakdown;
  }

  scoreCustomComponent(component) {
    // Custom scoring logic
    return component.healthy ? 10 : 0;
  }
}

module.exports = { CustomHealthScorer };
```

### Extending Log Analyzer

```javascript
// src/custom-pattern-detector.js
const { PatternDetector } = require('./pattern-detector');

class CustomPatternDetector extends PatternDetector {
  constructor(options = {}) {
    super(options);
    this.customPatterns = options.customPatterns || [];
  }

  detectPatterns(logs) {
    const patterns = super.detectPatterns(logs);
    
    // Add custom pattern detection
    const customPatterns = this.detectCustomPatterns(logs);
    
    return [...patterns, ...customPatterns];
  }

  detectCustomPatterns(logs) {
    return this.customPatterns.map(pattern => ({
      name: pattern.name,
      matches: logs.filter(log => pattern.regex.test(log.message)),
      severity: pattern.severity
    }));
  }
}

// Usage
const detector = new CustomPatternDetector({
  customPatterns: [
    {
      name: 'Custom Error',
      regex: /CUSTOM_ERROR_.*/,
      severity: 'high'
    }
  ]
});
```

### Extending Corruption Recovery

```javascript
// src/custom-backup-selector.js
const { BackupSelector } = require('./backup-selector');

class CustomBackupSelector extends BackupSelector {
  constructor(options = {}) {
    super(options);
    this.customCriteria = options.customCriteria || [];
  }

  calculateScore(backup) {
    const score = super.calculateScore(backup);
    
    // Add custom scoring criteria
    let customScore = 0;
    
    for (const criterion of this.customCriteria) {
      customScore += criterion.weight * criterion.scoreFn(backup);
    }
    
    return score + customScore;
  }
}

// Usage
const selector = new CustomBackupSelector({
  customCriteria: [
    {
      name: 'location',
      weight: 0.1,
      scoreFn: (backup) => backup.location === 'local' ? 10 : 5
    }
  ]
});
```

### Plugin Architecture

```javascript
// src/plugin-system.js
class SkillPluginSystem {
  constructor() {
    this.plugins = new Map();
  }

  register(name, plugin) {
    this.plugins.set(name, plugin);
  }

  async executeHook(hookName, ...args) {
    const results = [];
    
    for (const [name, plugin] of this.plugins) {
      if (typeof plugin[hookName] === 'function') {
        try {
          const result = await plugin[hookName](...args);
          results.push({ plugin: name, result });
        } catch (error) {
          console.error(`Plugin ${name} hook ${hookName} failed:`, error);
        }
      }
    }
    
    return results;
  }
}

// Plugin example
const myPlugin = {
  name: 'my-plugin',
  
  async preExecute(context) {
    console.log('Pre-execute hook');
    return { modified: true };
  },
  
  async postExecute(result) {
    console.log('Post-execute hook');
    return result;
  }
};

// Register plugin
const pluginSystem = new SkillPluginSystem();
pluginSystem.register('my-plugin', myPlugin);
```

---

## Related Documentation

- [Skills Operations Guide](./SKILLS_OPERATIONS_GUIDE.md) - Main operations guide
- [Orchestration Skills Guide](./ORCHESTRATION_SKILLS.md) - Orchestration skill usage
- [Debugging Skills Guide](./DEBUGGING_SKILLS.md) - Debugging skill usage
- [Troubleshooting Workflows](./TROUBLESHOOTING_WORKFLOWS.md) - Scenario-based workflows

---

## Best Practices

### Code Organization

1. **Single Responsibility**: Each module should have one clear purpose
2. **Dependency Injection**: Pass dependencies rather than importing directly
3. **Error Handling**: Always handle errors gracefully with meaningful messages
4. **Logging**: Use consistent logging levels (debug, info, warn, error)

### Performance

1. **Connection Pooling**: Reuse Gateway connections when possible
2. **Caching**: Cache frequently accessed data
3. **Batching**: Batch multiple operations when possible
4. **Timeouts**: Always set reasonable timeouts for operations

### Security

1. **Input Validation**: Validate all inputs before processing
2. **Authentication**: Verify agent identity for sensitive operations
3. **Authorization**: Check permissions before executing commands
4. **Audit Logging**: Log all skill invocations for audit trails

### Documentation

1. **Inline Comments**: Document complex logic
2. **JSDoc**: Use JSDoc for function documentation
3. **Examples**: Include usage examples in documentation
4. **CHANGELOG**: Maintain changelog for skill versions
