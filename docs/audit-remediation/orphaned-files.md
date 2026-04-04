# Orphaned File Inventory

**Audit Reference:** AUDIT-FIX C1  
**Date:** 2026-04-04  
**Source:** Zero-trust audit triad (Agent-1, Agent-2, Agent-3)

## Summary

This document catalogs files that are orphaned, duplicated, or unused in the codebase. Each file has been analyzed for import references across the entire codebase.

## File Inventory

| File | Referenced By | Status | Action |
|------|---------------|--------|--------|
| `modules/curiosity-engine.js` | Tests (`tests/heavy-swarm.test.js`), skills, documentation | **DUPLICATE** - v2 exists | Add deprecation notice pointing to `curiosity-engine-v2.js` |
| `modules/task-state-machine.js` | Documentation only (audit reports, deployment docs) | **ORPHANED** - No imports in code | Add deprecation notice |
| `modules/heavy-swarm.js` | `tests/heavy-swarm.test.js` | **FUNCTIONAL** - Has test coverage | Keep, no action needed |
| `modules/lineage-tracking.js` | Documentation only (audit reports, skills audit) | **ORPHANED** - No imports in code | Add deprecation notice |
| `modules/observability/langfuse-client.js` | `modules/observability/index.js` (exported but gateway doesn't import) | **UNUSED IN GATEWAY** - Exported but not wired to runtime | Document in langfuse-wiring-plan.md |
| `gateway/README-ARCHIVE.md` | Audit documentation only | **ARCHIVED** - Historical documentation | Keep as archive, no action needed |

## Detailed Analysis

### 1. modules/curiosity-engine.js

**Location:** `/root/heretek/heretek-openclaw-core/modules/curiosity-engine.js`

**References Found:**
- Test file: `tests/heavy-swarm.test.js` (imports but may not execute)
- Skills: Multiple skill scripts reference it for execution
- Documentation: Referenced in gateway compatibility report

**Issue:** A newer version exists at `modules/curiosity-engine-v2.js`. Having both creates confusion about which is the authoritative implementation.

**Recommendation:** Add deprecation notice at top of file pointing to v2.

---

### 2. modules/task-state-machine.js

**Location:** `/root/heretek/heretek-openclaw-core/modules/task-state-machine.js`

**References Found:**
- Documentation: Audit reports, deployment status docs, skills audit
- **No actual code imports found** in `.js` or `.ts` files

**Issue:** This module implements EDICT pattern for task state management but is not imported anywhere in the runtime codebase.

**Recommendation:** Add deprecation notice. If functionality is needed, it should be re-integrated properly.

---

### 3. modules/heavy-swarm.js

**Location:** `/root/heretek/heretek-openclaw-core/modules/heavy-swarm.js`

**References Found:**
- Test: `tests/heavy-swarm.test.js` - imports and tests the module
- Documentation: Multiple deployment docs verify it as production-ready

**Status:** **FUNCTIONAL** - This module is NOT orphaned. It has test coverage and is referenced as production-ready in deployment documentation.

**Recommendation:** No action needed.

---

### 4. modules/lineage-tracking.js

**Location:** `/root/heretek/heretek-openclaw-core/modules/lineage-tracking.js`

**References Found:**
- Documentation: Audit reports, skills audit, gateway compatibility report
- **No actual code imports found** in `.js` or `.ts` files

**Issue:** Implements causal lineage tracking for swarm memories but is not integrated with the swarm-memory module or any other runtime component.

**Recommendation:** Add deprecation notice. If lineage tracking is needed, integrate with swarm-memory properly.

---

### 5. modules/observability/langfuse-client.js

**Location:** `/root/heretek/heretek-openclaw-core/modules/observability/langfuse-client.js`

**References Found:**
- Imported by: `modules/observability/index.js` (re-exported)
- Documentation: Setup docs, tracing docs, metrics docs
- **NOT imported by:** Gateway, agent-client, or any runtime code

**Issue:** Full implementation exists with triad consciousness metrics support, but the gateway does not import or use the observability module. The dashboard imports `DashboardSync` separately but not the langfuse client.

**Status:** Exported but unused in production runtime.

**Recommendation:** Create wiring plan document (`langfuse-wiring-plan.md`) to integrate with gateway, or deprecate if observability is handled elsewhere.

---

### 6. gateway/README-ARCHIVE.md

**Location:** `/root/heretek/heretek-openclaw-core/gateway/README-ARCHIVE.md`

**References Found:**
- Audit documentation only

**Status:** Historical/archival documentation. Not referenced in code.

**Recommendation:** Keep as archive. No action needed.

---

## Deprecation Notice Template

For orphaned files, add this comment at the top of the file:

```javascript
// DEPRECATED: This module is not imported anywhere in the codebase.
// Scheduled for removal. If you need this functionality, contact the team.
// Audit Reference: AUDIT-FIX C1
// Date: 2026-04-04
```

## Next Steps

1. Add deprecation notices to: `curiosity-engine.js`, `task-state-machine.js`, `lineage-tracking.js`
2. Create `langfuse-wiring-plan.md` for observability integration decision
3. Document decisions in `REMEDIATION_LOG.md`
