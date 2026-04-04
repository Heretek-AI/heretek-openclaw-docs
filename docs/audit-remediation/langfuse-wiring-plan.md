# Langfuse Client Status Documentation

**Audit Reference:** AUDIT-FIX C8  
**Date:** 2026-04-04  
**Source:** Zero-trust audit triad (Agent-2, Agent-3)

## Executive Summary

The Langfuse client (`modules/observability/langfuse-client.js`) is **FULLY IMPLEMENTED** but **NOT WIRED** to the runtime. It's exported from the observability module but never imported by the gateway or any agent code.

**Status:** ⚠️ **IMPLEMENTED BUT UNUSED** - Observability stack exists but is not integrated with runtime.

---

## File Analysis

### langfuse-client.js

**Location:** `/root/heretek/heretek-openclaw-core/modules/observability/langfuse-client.js`

**Size:** ~500 lines (full implementation)

**Exports:**
- `HeretekLangfuseClient` - Main class wrapper around Langfuse SDK
- `createInstance()` - Singleton factory

**Features Implemented:**
- ✅ Triad deliberation tracing (Alpha/Beta/Charlie voting)
- ✅ Consciousness architecture metrics (GWT, IIT, AST signals)
- ✅ Consensus ledger event recording
- ✅ Agent decision cycle tracking
- ✅ Cost and latency monitoring per agent/model
- ✅ Offline mode support (local Langfuse instance)
- ✅ Cloud mode support (Langfuse cloud)

**Key Methods:**
```javascript
class HeretekLangfuseClient {
  constructor(config)
  async startTriadDeliberation({ sessionId, proposalId, agents })
  async recordTriadVote({ triadContext, agent, vote, reasoning })
  async finalizeTriadDeliberation({ triadContext, consensus, voteCount, stewardOverride, consciousnessMetrics })
  async recordConsciousnessMetrics({ trace, gwtScore, iitScore, astScore })
  async recordAgentDecision({ agentId, decision, context })
  async recordConsensusEvent({ sessionId, decision, votes })
  async shutdown()
  getStatus()
}
```

**Dependencies:**
- `langfuse` npm package (required)
- Environment variables: `LANGFUSE_PUBLIC_KEY`, `LANGFUSE_SECRET_KEY`, `LANGFUSE_HOST`

---

## Import Chain Analysis

### Current Import Chain

```
langfuse-client.js
    ↓ (imported by)
observability/index.js
    ↓ (exported from)
observability module
    ↓ (NOT IMPORTED BY)
❌ Gateway
❌ Agent-client
❌ Any runtime code
```

### Files That SHOULD Import It

Based on the implementation and documentation, these files should use the Langfuse client:

1. **`gateway/openclaw-gateway.js`** - Should trace all A2A messages and agent communications
2. **`agents/lib/agent-client.js`** - Should trace agent decision cycles
3. **`modules/consensus/bft-consensus.js`** - Should record consensus events
4. **`modules/swarm-memory/heretek-swarm-memory.js`** - Should trace memory operations

### Files That DO Import Observability Module

**Found:** `modules/observability/index.js` imports and re-exports langfuse-client

**But:** No runtime code imports from `modules/observability/`

**Audit Finding (Agent-2):**
> "No imports from `modules/observability/` found in gateway or agent-client"

---

## Why It's Not Wired

### Possible Reasons

1. **Development In Progress:** The observability stack was being built when audit occurred
2. **Missing Dependencies:** `langfuse` package may not be in dependencies
3. **Configuration Complexity:** Requires Langfuse instance (local or cloud)
4. **Priority:** Lower priority than core functionality fixes
5. **Architecture Decision:** May have been deferred to post-remediation phase

### Evidence

**From `modules/observability/index.js`:**
```javascript
// Full observability stack initialization code exists
function createObservabilityStack(config = {}) {
    const langfuse = new HeretekLangfuseClient({...});
    const metrics = new HeretekMetricsExporter({...});
    const dashboard = new DashboardSync({...});
    // ... wiring code
}
```

**The infrastructure is complete but not invoked anywhere.**

---

## Wiring Plan

### Option 1: Gateway Integration (Recommended)

**File:** `gateway/openclaw-gateway.js`

**Integration Point:** Initialize observability stack in gateway constructor

```javascript
const { createObservabilityStack } = require('../modules/observability');

class OpenClawGateway {
  constructor(config) {
    this.config = config;
    
    // Initialize observability
    this.observability = createObservabilityStack({
      langfuse: {
        publicKey: process.env.LANGFUSE_PUBLIC_KEY,
        secretKey: process.env.LANGFUSE_SECRET_KEY,
        host: process.env.LANGFUSE_HOST || 'http://localhost:3000',
      },
      metrics: {
        redisUrl: process.env.REDIS_URL,
        exportInterval: 60000,
      },
      dashboard: {
        dashboardUrl: process.env.DASHBOARD_URL,
        syncInterval: 5000,
      },
      enabled: process.env.OBSERVABILITY_ENABLED !== 'false',
    });
  }
  
  async initialize() {
    // Gateway initialization...
    
    // Observability is now ready
    console.log('[Gateway] Observability stack initialized');
  }
  
  async _handleMessage(ws, data) {
    // Trace incoming messages
    const trace = this.observability.langfuse.trace({
      name: 'gateway-message',
      sessionId: this.sessionId,
      metadata: { direction: 'inbound' }
    });
    
    // Process message...
    
    // Finalize trace
    trace.end();
  }
}
```

**Benefits:**
- Centralized tracing of all agent communications
- Automatic tracing of all A2A protocol messages
- Single initialization point

**Complexity:** MEDIUM

---

### Option 2: Agent-Client Integration

**File:** `agents/lib/agent-client.js`

**Integration Point:** Trace agent decision cycles

```javascript
const { getLangfuseClient } = require('../../modules/observability/langfuse-client');

class AgentClient {
  async executeSkill(skillId, context) {
    const langfuse = getLangfuseClient();
    
    const trace = langfuse.trace({
      name: 'agent-skill-execution',
      sessionId: this.sessionId,
      metadata: {
        agentId: this.agentId,
        skillId,
      }
    });
    
    try {
      const result = await this._executeSkillInternal(skillId, context);
      
      trace.end({
        output: { success: true },
      });
      
      return result;
    } catch (error) {
      trace.end({
        output: { error: error.message },
        level: 'ERROR',
      });
      throw error;
    }
  }
}
```

**Benefits:**
- Traces individual agent operations
- Captures skill execution details
- Error tracking

**Complexity:** MEDIUM

---

### Option 3: Full Stack Integration (Comprehensive)

Initialize observability in gateway AND instrument:
- Gateway message handling
- Agent-client skill execution
- Consensus operations
- Memory operations

**Benefits:**
- Complete observability coverage
- End-to-end tracing
- Full metrics collection

**Complexity:** HIGH

---

## Configuration Requirements

### Environment Variables

```bash
# Langfuse Configuration
LANGFUSE_PUBLIC_KEY=pk-xxxxxxxxxxxxx
LANGFUSE_SECRET_KEY=sk-xxxxxxxxxxxxx
LANGFUSE_HOST=http://localhost:3000  # or https://cloud.langfuse.com

# Enable/Disable Observability
OBSERVABILITY_ENABLED=true

# Langfuse Deployment Mode
LANGFUSE_MODE=production  # or 'offline' for local development
```

### Dependencies

Add to `heretek-openclaw-core/package.json`:
```json
{
  "dependencies": {
    "langfuse": "^3.x.x"
  }
}
```

### Langfuse Deployment

**Option A: Local Development**
```bash
docker run -d \
  -p 3000:3000 \
  -e DATABASE_URL=postgresql://postgres:postgres@db:5432/langfuse \
  -e SALT=randomsalt \
  -e ENCRYPTION_KEY=randomkey \
  langfuse/langfuse
```

**Option B: Cloud**
- Sign up at https://cloud.langfuse.com
- Get API keys from dashboard
- Set `LANGFUSE_HOST=https://cloud.langfuse.com`

---

## Implementation Recommendations

### Phase 1: Gateway Integration (C8 Task - Deferred)

1. Add `langfuse` to dependencies
2. Initialize observability stack in gateway constructor
3. Add basic message tracing
4. Test with local Langfuse instance

### Phase 2: Agent Integration (Future)

1. Instrument agent-client with tracing
2. Add skill execution traces
3. Record agent decision cycles

### Phase 3: Full Coverage (Future)

1. Instrument consensus module
2. Add memory operation traces
3. Configure metrics export
4. Set up dashboards

---

## Risk Assessment

### Low Risk
- Adding observability is non-breaking
- Can be disabled via environment variable
- No impact on existing functionality

### Medium Risk
- Performance overhead from tracing (typically <5%)
- Additional dependency to maintain
- Requires Langfuse instance (local or cloud)

### Mitigation
- Default to `OBSERVABILITY_ENABLED=false` initially
- Add performance monitoring to detect overhead
- Document Langfuse deployment options

---

## Completion Checklist

- [x] Read and understand langfuse-client.js implementation
- [x] Search for imports across codebase
- [x] Confirm it's exported but unused
- [x] Document current state
- [x] Create wiring plan with options
- [ ] Add langfuse to dependencies (deferred)
- [ ] Initialize in gateway (deferred)
- [ ] Add message tracing (deferred)
- [ ] Test with Langfuse instance (deferred)

---

## Related Documents

- `orphaned-files.md` - Documents langfuse-client.js as orphaned (exported but unused)
- `redis-consolidation-plan.md` - Redis client consolidation (metrics-exporter also uses Redis)
- `plugin-reality-check.md` - Plugin verification (some plugins may use observability)

---

**Audit Reference:** AUDIT-FIX C8  
**Status:** Documented, wiring plan created, implementation deferred
