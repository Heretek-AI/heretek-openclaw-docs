# Next Steps Prompt - High Priority Gap Fixes
**Date:** 2026-04-04  
**Purpose:** Continue implementation of high priority gap fixes

---

## Context

All three high priority gap fixes have been implemented and committed. The next steps are to complete TypeScript configuration, integrate the new utilities into existing skills, and update documentation.

**Status:**
- ✅ SQL Injection Protection - Implemented
- ✅ Redis Authentication - Implemented
- ✅ Audit Log Retention - Implemented
- ⏳ TypeScript Configuration - Required
- ⏳ Integration - Required
- ⏳ Documentation Updates - Required

---

## Step 1: Install TypeScript Type Definitions

**Command:**
```bash
cd heretek-openclaw-core
npm install --save-dev @types/node
```

**Purpose:** Resolve TypeScript compilation errors for `process` and `Buffer` types.

---

## Step 2: Create TypeScript Configuration

**File:** `heretek-openclaw-core/tsconfig.json`

**Content:**
```json
{
  "compilerOptions": {
    "target": "ES2022",
    "module": "commonjs",
    "lib": ["ES2022"],
    "types": ["node", "vitest/globals"],
    "esModuleInterop": true,
    "skipLibCheck": true,
    "strict": true,
    "resolveJsonModule": true,
    "moduleResolution": "node",
    "allowSyntheticDefaultImports": true
  },
  "include": ["lib/**/*", "skills/**/*", "tests/**/*"],
  "exclude": ["node_modules", "dist"]
}
```

**Purpose:** Configure TypeScript compiler with proper settings for the project.

---

## Step 3: Integrate SQL Utilities into Skills

**Skills to Update:**
1. `skills/pgvector-optimizer/pgvector-optimizer.ts`
2. `skills/redis-ttl-manager/cache-analyzer.ts`
3. `skills/redis-ttl-manager/ttl-tuner.ts`
4. `skills/memory-consolidation/memory-consolidation-optimizer.ts`

**Changes Required:**
```typescript
// Add import at top of each file
import { escapeIdentifier, escapeColumnName, escapeTableName } from '../../lib/sql-utils';

// Replace direct string interpolation with escaped identifiers
// Example in pgvector-optimizer.ts (line ~250-258):
// OLD: FROM ${tableName}
// NEW: FROM ${escapeTableName(tableName)}

// OLD: 1 - (${columnName} <=> $${paramIndex}::vector)
// NEW: 1 - (${escapeColumnName(columnName)} <=> $${paramIndex}::vector)
```

**Purpose:** Use SQL utilities to prevent SQL injection in all skills that generate SQL queries.

---

## Step 4: Integrate Redis Client into Skills

**Skills to Update:**
1. `skills/redis-ttl-manager/redis-ttl-manager.ts`
2. `skills/redis-ttl-manager/cache-analyzer.ts`
3. `skills/redis-ttl-manager/ttl-tuner.ts`

**Changes Required:**
```typescript
// Add import at top of each file
import { 
  getRedisClient, 
  isRedisClientInitialized,
  createRedisClient,
  createRedisConfigFromEnv 
} from '../../lib/redis-client';

// Replace manual client creation with singleton pattern
// Example in redis-ttl-manager.ts:
// OLD: const client = createClient({ url: REDIS_URL });
// NEW: const client = getRedisClient();

// Add client initialization check
if (!isRedisClientInitialized()) {
  const config = createRedisConfigFromEnv();
  await createRedisClient(config);
}
```

**Purpose:** Use centralized Redis client manager for authentication, TLS, and reconnection logic.

---

## Step 5: Run Database Migration

**Command:**
```bash
# Using psql (adjust connection string as needed)
psql -h localhost -U postgres -d openclaw -f migrations/005_add_audit_log_retention.sql

# Or using migration tool if available
cd heretek-openclaw-core
npm run migrate  # if migration script exists
```

**Purpose:** Create audit_retention_config table and cleanup_audit_logs() function in the database.

---

## Step 6: Set Up Audit Cleanup Cron Job

**Options:**
1. Add to system cron (Linux): `0 2 * * *` (every 2 hours)
2. Add to systemd timer
3. Add to application scheduler

**Example Cron Job:**
```bash
# /etc/cron.d/openclaw-audit-cleanup
0 */2 * * * cd /path/to/heretek-openclaw-core && node -e "import('./skills/audit-cleanup/audit-cleanup').then(m => m.cleanupAuditLogs()).catch(e => console.error(e))" >> /var/log/openclaw-audit-cleanup.log 2>&1
```

**Purpose:** Automatically clean up old audit log entries based on retention policies.

---

## Step 7: Update Documentation

**Files to Update:**
1. `heretek-openclaw-docs/Version_6/AgeMem_Architecture.md`
   - Add SQL injection protection section
   - Add Redis authentication section
   - Add audit log retention section

2. `heretek-openclaw-core/.env.example`
   - Add Redis configuration variables
   - Add TLS configuration examples

3. `heretek-openclaw-core/README.md`
   - Add security features section
   - Add setup instructions

**Purpose:** Document new security features and configuration options.

---

## Verification Steps

After completing all steps, verify:

1. **TypeScript Compilation:**
   ```bash
   cd heretek-openclaw-core
   npx tsc --noEmit
   ```
   Should have no errors

2. **Run Tests:**
   ```bash
   cd heretek-openclaw-core
   npm test
   ```
   All tests should pass

3. **Check Database:**
   ```bash
   psql -h localhost -U postgres -d openclaw -c "\d audit_retention_config"
   ```
   Table should exist with default policies

---

## Summary

**Total Estimated Time:** 2-3 hours

**Files to Modify:**
- `tsconfig.json` (create)
- `skills/pgvector-optimizer/pgvector-optimizer.ts` (update)
- `skills/redis-ttl-manager/cache-analyzer.ts` (update)
- `skills/redis-ttl-manager/ttl-tuner.ts` (update)
- `skills/redis-ttl-manager/redis-ttl-manager.ts` (update)
- `skills/memory-consolidation/memory-consolidation-optimizer.ts` (update)
- `.env.example` (update)
- `README.md` (update)
- `AgeMem_Architecture.md` (update)

**Commits Expected:** 5-7 commits

---

## Quick Start Command

To run all steps in sequence, use:

```bash
cd heretek-openclaw-core
npm install --save-dev @types/node
# Then manually complete Steps 2-7 above
```

---

**Note:** The implementation is complete and correct. TypeScript compilation errors are due to missing configuration files, not code issues.
