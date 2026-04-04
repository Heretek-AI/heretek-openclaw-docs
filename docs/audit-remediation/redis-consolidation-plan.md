# Redis Client Consolidation Plan

**Audit Reference:** AUDIT-FIX C2  
**Date:** 2026-04-04  
**Source:** Zero-trust audit triad (Agent-1, Agent-2, Agent-3)

## Executive Summary

The codebase has **inconsistent Redis client patterns** with multiple files creating their own Redis clients instead of using the centralized singleton. This creates:
- Connection pooling inefficiency
- Inconsistent error handling
- Duplicate reconnection logic
- Potential memory leaks from orphaned connections

## Current State Analysis

### Redis Client Library Inconsistency

| Library | Files Using | Status |
|---------|-------------|--------|
| **ioredis** | 12+ files | ✅ Primary choice |
| **node-redis** (`redis` npm package) | 1 file | ❌ Should migrate to ioredis |

### Files Creating Redis Clients

| File | Library | Pattern | Line | Issues |
|------|---------|---------|------|--------|
| `gateway/openclaw-gateway.js` | ioredis | `new Redis()` inline | 296, 301 | No connection pooling, inline error handling |
| `skills/a2a-message-send/a2a-redis.js` | ioredis | `new Redis()` inline | 96, 102 | Creates client per skill execution |
| `modules/observability/dashboard-sync.js` | ioredis | `new Redis()` inline | 161, 162 | Duplicate client instance |
| `modules/observability/metrics-exporter.js` | ioredis | `new Redis()` inline | 232, 233 | Duplicate client instance |
| `modules/consensus/bft-consensus.js` | ioredis | `new Redis()` inline | 17 | No centralized config |
| `modules/consensus/reputation-voting.js` | ioredis | `new Redis()` inline | 18 | No centralized config |
| `modules/communication/redis-websocket-bridge.js` | ioredis | `new Redis()` inline | 205, 210, 248, 252 | Multiple clients, no pooling |
| `modules/adapters/bft-consensus-adapter.js` | ioredis | `new Redis()` inline | 62 | No centralized config |
| `modules/adapters/acp-adapter.js` | ioredis | `new Redis()` inline | 405 | No centralized config |
| `modules/provider-abstraction/src/provider-router.js` | ioredis | `new Redis()` inline | 13 | No centralized config |
| `modules/swarm-memory/heretek-swarm-memory.js` | ioredis | `new Redis()` inline | 14 | No centralized config |
| `modules/curiosity-engine-v2.js` | ioredis | `new Redis()` inline | 18 | No centralized config |
| `modules/a2a-protocol/event-mesh.js` | **node-redis** | `redis.createClient()` | 40 | ❌ **WRONG LIBRARY** |

### Existing Singleton Solution

**File:** `lib/redis-client.ts`

**Features:**
- ✅ Singleton pattern (single connection shared across app)
- ✅ TypeScript support
- ✅ Authentication support (username/password)
- ✅ TLS configuration
- ✅ Reconnection logic with event handlers
- ✅ Connection timeout handling
- ✅ Ready check
- ✅ Offline queue management
- ✅ State management
- ✅ Graceful shutdown

**Exports:**
- `createRedisClient(config)` - Initialize singleton
- `getRedisClient()` - Get existing client
- `isRedisClientInitialized()` - Check if ready
- `getRedisClientState()` - Get state info
- `closeRedisClient()` - Graceful shutdown
- `forceCloseRedisClient()` - Force disconnect
- `createRedisConfigFromEnv()` - Config from env vars

## Consolidation Strategy

### Phase 1: Standardize on ioredis

**Decision:** Use `ioredis` as the standard library (already used by 92% of files)

**Rationale:**
1. Already the dominant library in codebase
2. Better TypeScript support
3. More mature clustering support
4. Built-in pipeline support
5. Better event handling

**Action Required:**
- Convert `modules/a2a-protocol/event-mesh.js` from `node-redis` to `ioredis`

### Phase 2: Migrate to Singleton Pattern

**Target Files for Migration:**

| Priority | File | Risk Level | Migration Complexity |
|----------|------|------------|---------------------|
| 1 | `modules/observability/dashboard-sync.js` | Low | Easy - already imports from observability |
| 2 | `modules/observability/metrics-exporter.js` | Low | Easy - already imports from observability |
| 3 | `modules/consensus/bft-consensus.js` | Medium | Moderate - core consensus logic |
| 4 | `modules/consensus/reputation-voting.js` | Medium | Moderate - depends on bft-consensus |
| 5 | `modules/adapters/bft-consensus-adapter.js` | Medium | Moderate - adapter layer |
| 6 | `modules/adapters/acp-adapter.js` | Medium | Moderate - adapter layer |
| 7 | `modules/provider-abstraction/src/provider-router.js` | Low | Easy - isolated module |
| 8 | `modules/swarm-memory/heretek-swarm-memory.js` | High | Complex - core memory system |
| 9 | `modules/curiosity-engine-v2.js` | Low | Easy - exploratory feature |
| 10 | `modules/communication/redis-websocket-bridge.js` | High | Complex - bridge logic |
| 11 | `skills/a2a-message-send/a2a-redis.js` | Medium | Moderate - skill execution |
| 12 | `gateway/openclaw-gateway.js` | High | Complex - core gateway |

### Phase 3: Gateway Integration (Special Case)

The gateway should initialize the singleton at startup:

```javascript
// gateway/openclaw-gateway.js
const { createRedisClient } = require('../lib/redis-client');

class OpenClawGateway {
  async initialize() {
    // Initialize Redis singleton
    const redisConfig = {
      url: process.env.REDIS_URL || 'redis://localhost:6379',
      password: process.env.REDIS_PASSWORD,
      maxRetriesPerRequest: 3,
      connectTimeout: 10000,
    };
    
    await createRedisClient(redisConfig);
    
    // Get the singleton for local use
    this.redisClient = getRedisClient();
  }
}
```

## Migration Pattern

### Before (Inline Client)

```javascript
const { Redis } = require('ioredis');

class MyModule {
  constructor(options = {}) {
    this.redis = new Redis(options.redisUrl || 'redis://localhost:6379');
  }
  
  async doSomething() {
    await this.redis.set('key', 'value');
  }
}
```

### After (Singleton Pattern)

```javascript
const { getRedisClient, isRedisClientInitialized } = require('../lib/redis-client');

class MyModule {
  constructor(options = {}) {
    if (!isRedisClientInitialized()) {
      throw new Error('Redis client not initialized. Call createRedisClient() first.');
    }
    this.redis = getRedisClient();
  }
  
  async doSomething() {
    await this.redis.set('key', 'value');
  }
}
```

## Risk Assessment

### High Risk Files

| File | Risk | Mitigation |
|------|------|------------|
| `gateway/openclaw-gateway.js` | **HIGH** - Core infrastructure | Requires full integration test suite |
| `modules/swarm-memory/heretek-swarm-memory.js` | **HIGH** - Core memory system | Requires memory operation tests |
| `modules/communication/redis-websocket-bridge.js` | **HIGH** - Bridge logic | Requires pub/sub testing |

### Medium Risk Files

| File | Risk | Mitigation |
|------|------|------------|
| `modules/consensus/bft-consensus.js` | **MEDIUM** - Consensus critical | Test consensus rounds |
| `modules/consensus/reputation-voting.js` | **MEDIUM** - Voting logic | Test vote recording |
| `modules/adapters/*` | **MEDIUM** - Adapter layer | Test adapter operations |

### Low Risk Files

| File | Risk | Mitigation |
|------|------|------------|
| `modules/observability/*` | **LOW** - Observability | Already in same module family |
| `modules/provider-abstraction/src/provider-router.js` | **LOW** - Isolated | Simple migration |
| `modules/curiosity-engine-v2.js` | **LOW** - Exploratory | Non-critical feature |

## Testing Requirements

Before any migration:

1. **Unit Tests:** Each module must have passing unit tests
2. **Integration Tests:** Redis operations must be tested end-to-end
3. **Connection Tests:** Verify singleton is shared correctly
4. **Error Tests:** Test behavior when Redis is unavailable
5. **Shutdown Tests:** Verify graceful cleanup

## Recommended Migration Order

1. ✅ **Start with low-risk files** (observability modules)
2. ✅ **Move to medium-risk** (adapters, provider-router)
3. ⚠️ **Tackle high-risk last** (gateway, swarm-memory, bridge)
4. ✅ **Convert event-mesh.js** from node-redis to ioredis (isolated change)

## Configuration Standardization

All modules should use environment variables:

```bash
# .env.example
REDIS_URL=redis://localhost:6379
REDIS_PASSWORD=
REDIS_USERNAME=
REDIS_TLS=false
REDIS_CONNECT_TIMEOUT=10000
REDIS_MAX_RETRIES=3
```

## Implementation Notes

**DO NOT** actually perform this migration as part of C2. This document is for **planning only**.

The actual migration requires:
1. Careful testing of each module
2. Verification that singleton is initialized before use
3. Error handling for uninitialized client scenarios
4. Update to package.json if dependencies change
5. Full regression testing

## Related Documents

- `orphaned-files.md` - Documents unused/legacy code
- `langfuse-wiring-plan.md` - Observability integration (related to metrics-exporter)
- `config-consolidation.md` - Configuration file cleanup

## Completion Checklist

- [x] Document all Redis client creation points
- [x] Identify library inconsistency (node-redis vs ioredis)
- [x] Document existing singleton solution
- [x] Create migration strategy
- [x] Assess risk for each migration target
- [x] Define testing requirements
- [ ] **DO NOT IMPLEMENT** - This is planning only

---

**Audit Reference:** AUDIT-FIX C2  
**Status:** Plan documented, implementation deferred to separate task
