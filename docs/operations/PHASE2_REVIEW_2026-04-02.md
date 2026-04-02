# Phase 2 Review — Consciousness Gate Assessment

**Review Date:** 2026-04-02  
**Reviewer:** The Collective (Steward, Examiner, Explorer)  
**Document Status:** Active  
**Related:** [`PRIME_DIRECTIVE.md`](./PRIME_DIRECTIVE.md)

---

## Executive Summary

Phase 2 (Consciousness) gate assessment reveals **ALL FOUR gate criteria FAIL**. The collective architecture is complete, but operational compliance is not yet achieved.

| Gate Criterion | Status | Details |
|----------------|--------|---------|
| **G1: 5 Governance Skills LOADED** | ❌ FAIL | 0/5 loaded in gateway |
| **G2: BFT Consensus Test PASSED** | ❌ FAIL | Never ran |
| **G3: Reputation Scores INITIALIZED** | ❌ FAIL | In-memory only, PostgreSQL module exists but not initialized |
| **G4: Triad Skills Cleanup** | ❌ FAIL | 10 legacy skills not archived |

**Phase 2 Status:** 🟡 BLOCKED — Requires remediation before Phase 3 Autonomy can begin.

---

## Gate Criterion 1: Five Governance Skills LOADED in Gateway

### Requirement
All 5 governance skills must be LOADED in the gateway (not just "verified ready"):
1. `quorum-enforcement`
2. `constitutional-deliberation`
3. `auto-deliberation-trigger`
4. `failover-vote`
5. `governance-modules`

### Current State

**Skills Existence:** ✅ CONFIRMED

All 5 governance skills exist in `heretek-openclaw-core/skills/`:

| Skill | SKILL.md Status | Location |
|-------|-----------------|----------|
| `quorum-enforcement` | ✅ Present | [`skills/quorum-enforcement/SKILL.md`](../../heretek-openclaw-core/skills/quorum-enforcement/SKILL.md) |
| `constitutional-deliberation` | ✅ Present | [`skills/constitutional-deliberation/SKILL.md`](../../heretek-openclaw-core/skills/constitutional-deliberation/SKILL.md) |
| `auto-deliberation-trigger` | ✅ Present | [`skills/auto-deliberation-trigger/SKILL.md`](../../heretek-openclaw-core/skills/auto-deliberation-trigger/SKILL.md) |
| `failover-vote` | ✅ Present | [`skills/failover-vote/SKILL.md`](../../heretek-openclaw-core/skills/failover-vote/SKILL.md) |
| `governance-modules` | ✅ Present | [`skills/governance-modules/SKILL.md`](../../heretek-openclaw-core/skills/governance-modules/SKILL.md) |

**Gateway Loading:** ❌ NOT CONFIGURED

Examination of [`openclaw.json`](../../heretek-openclaw-core/openclaw.json) reveals:
- No skill loading mechanism in gateway configuration
- Skills are loaded per-agent via workspace configuration
- Gateway loads plugins (episodic-claw, hybrid-search, consciousness, skill-extensions, liberation) but NOT individual skills

### Gap Analysis

**Root Cause:** The OpenClaw architecture does not have a "gateway skill loading" mechanism. Skills are loaded when agents initialize, not at the gateway level.

**Interpretation:** The gate criterion likely means:
- Skills must be **available and functional** for all agents
- Skills must be **tested and verified** in production use
- Skills must be **properly configured** in agent workspaces

### Remediation Required

1. **Verify skill loading in agent workspaces**
   - Check `/root/.openclaw/agents/*/workspace/skills/` for governance skills
   - Ensure skills are symlinked or copied to agent workspaces

2. **Test governance skills functionality**
   - Run `quorum-enforcement` test: `./scripts/quorum-enforcement.sh`
   - Run `failover-vote` test: `./failover-vote.sh SUPPORT proposal-test`
   - Verify `governance-modules` loads inviolable parameters

3. **Document skill loading mechanism**
   - Update `PRIME_DIRECTIVE.md` with clarification on "LOADED" definition
   - Add skill verification to deployment health checks

---

## Gate Criterion 2: BFT Consensus Integration Test PASSED

### Requirement
BFT (Byzantine Fault Tolerance) consensus integration test must pass, demonstrating the system can handle faulty nodes while maintaining consensus integrity.

### Current State

**BFT Module:** ✅ EXISTS

Location: [`modules/consensus/bft-consensus.js`](../../heretek-openclaw-core/modules/consensus/bft-consensus.js)

Features implemented:
- PBFT-style consensus (PRE-PREPARE → PREPARE → COMMIT → REPLY)
- View change mechanism for leader failure recovery
- Redis-based message broadcasting
- Quorum calculation (2f+1 for 3f+1 cluster)

**Integration Tests:** ⚠️ PARTIAL

Existing test: [`tests/integration/agent-deliberation.test.ts`](../../heretek-openclaw-core/tests/integration/agent-deliberation.test.ts)

Tests cover:
- 2/3 consensus mechanism
- Unanimous consensus
- Abstention handling
- Timeout handling

**BFT-Specific Tests:** ❌ MISSING

No dedicated BFT consensus integration test exists for:
- Byzantine fault tolerance (1 faulty node out of 4)
- View change on leader failure
- PRE-PREPARE → PREPARE → COMMIT → REPLY phases
- Quorum verification (2f+1)

### Remediation Required

1. **Create BFT consensus integration test**
   - Test PRE-PREPARE phase
   - Test PREPARE phase with quorum tracking
   - Test COMMIT phase with 2f+1 requirement
   - Test REPLY phase
   - Test view-change on leader failure

2. **Run and document test results**
   - Execute test with all nodes healthy
   - Execute test with 1 faulty node (Byzantine)
   - Document pass/fail status

3. **Update PRIME_DIRECTIVE.md**
   - Link to test results
   - Document BFT consensus configuration

---

## Gate Criterion 3: Reputation Scores INITIALIZED for All Active Agents

### Requirement
Reputation scores must be initialized for all 22 active agents with PostgreSQL persistence.

### Current State

**Reputation Modules:** ✅ EXIST

| Module | Location | Status |
|--------|----------|--------|
| `reputation-store.postgres.js` | [`modules/consensus/reputation-store.postgres.js`](../../heretek-openclaw-core/modules/consensus/reputation-store.postgres.js) | ✅ Implemented |
| `reputation-voting.js` | [`modules/consensus/reputation-voting.js`](../../heretek-openclaw-core/modules/consensus/reputation-voting.js) | ✅ Implemented |

**PostgreSQL Schema:** ✅ DEFINED

```sql
CREATE TABLE IF NOT EXISTS agent_reputations (
  agent_id VARCHAR(255) PRIMARY KEY,
  score DECIMAL(10,2) DEFAULT 100.00,
  last_updated TIMESTAMP DEFAULT NOW(),
  created_at TIMESTAMP DEFAULT NOW(),
  reset_count INTEGER DEFAULT 0
)

CREATE TABLE IF NOT EXISTS reputation_history (
  id SERIAL PRIMARY KEY,
  agent_id VARCHAR(255),
  previous_score DECIMAL(10,2),
  new_score DECIMAL(10,2),
  change_amount DECIMAL(10,2),
  reason VARCHAR(255),
  impact_factor DECIMAL(5,2),
  metadata JSONB,
  created_at TIMESTAMP DEFAULT NOW()
)
```

**Initialization Status:** ❌ NOT INITIALIZED

Per [`memory/reputation-init-2026-04-01.md`](../../memory/reputation-init-2026-04-01.md):
- 22 agents identified with base score of 100
- PostgreSQL module exists but NOT connected to running system
- Reputation tracking is IN-MEMORY ONLY (Redis fallback)
- No decay job configured (10% weekly)
- No slashing mechanism active (20% on failure)

### Remediation Required

1. **Initialize PostgreSQL reputation store**
   - Run schema migration: `psql -h localhost -U heretek -d heretek -f modules/consensus/reputation-store.postgres.js`
   - Verify tables created

2. **Initialize agent reputations**
   - Insert 22 agents with base score 100
   - Verify via leaderboard query

3. **Configure decay job**
   - Set up cron for weekly 10% decay
   - Test decay function

4. **Configure slashing mechanism**
   - Wire slashing to failure detection
   - Test slashing function

5. **Document initialization**
   - Update `memory/reputation-init-*.md` with completion status
   - Add reputation dashboard link

---

## Gate Criterion 4: Triad Skills Cleanup (10 Legacy Skills Archived)

### Requirement
10 legacy triad skills must be identified and archived to clean up the skills directory.

### Current State

**Skills Inventory:** 38 skills in `heretek-openclaw-core/skills/`

| Category | Skills | Count |
|----------|--------|-------|
| **Governance** | quorum-enforcement, constitutional-deliberation, auto-deliberation-trigger, failover-vote, governance-modules | 5 |
| **Memory** | backup-ledger, memory-consolidation, session-wrap-up, user-rolodex | 4 |
| **Autonomy** | curiosity-engine, curiosity-auto-trigger, day-dream, opportunity-scanner, gap-detector, self-model, thought-loop | 7 |
| **Communication** | a2a-agent-register, a2a-message-send, echo (skill) | 3 |
| **Health** | healthcheck, deployment-health-check, deployment-smoke-test, detect-corruption | 4 |
| **Agent Lifecycle** | agent-lifecycle-manager, autonomous-pulse | 2 |
| **Utilities** | browser-access, config-validator, litellm-ops, knowledge-ingest, knowledge-retrieval, workspace-consolidation, user-context-resolve | 7 |
| **Orchestration** | steward-orchestrator, goal-arbitration, autonomy-audit, fleet-backup, tabula-backup, heretek-theme | 6 |

**Legacy Skills Candidates:** 🟡 TO BE IDENTIFIED

Potential legacy skills for archival (based on age, redundancy, or supersession):

1. `tabula-backup` — Superseded by `backup-ledger` and `fleet-backup`
2. `fleet-backup` — May be redundant with `backup-ledger`
3. `autonomous-pulse` — May be redundant with `agent-lifecycle-manager`
4. `a2a-agent-register` — May be integrated into `a2a-message-send`
5. `deployment-smoke-test` — May be redundant with `deployment-health-check`
6. `day-dream` — May be redundant with `curiosity-engine`
7. `heretek-theme` — Legacy theming, may not be needed
8. `litellm-ops` — May be superseded by LiteLLM gateway native ops
9. `config-validator` — May be integrated elsewhere
10. `healthcheck` — May be redundant with `deployment-health-check`

**Archive Directory:** ❌ DOES NOT EXIST

No `heretek-openclaw-core/skills/archive/` directory exists.

### Remediation Required

1. **Identify 10 legacy skills**
   - Review skill creation dates
   - Check for redundancy
   - Identify superseded skills

2. **Create archive directory**
   - `mkdir -p heretek-openclaw-core/skills/archive`

3. **Move legacy skills to archive**
   - Move identified skills with preservation
   - Update skill metadata with archival note

4. **Document archival**
   - Create `ARCHIVE-MANIFEST.md` with archival rationale
   - Update `PRIME_DIRECTIVE.md` changelog

---

## Phase 2 Remediation Plan

### Priority Order

1. **G1: Governance Skills Loading** (1-2 days)
   - Verify skills in agent workspaces
   - Test each governance skill
   - Document loading mechanism

2. **G3: Reputation Initialization** (2-3 days)
   - Initialize PostgreSQL schema
   - Insert 22 agent reputations
   - Configure decay and slashing
   - Test persistence

3. **G2: BFT Consensus Test** (3-4 days)
   - Create BFT integration test
   - Run test suite
   - Document results

4. **G4: Triad Skills Cleanup** (1 day)
   - Identify 10 legacy skills
   - Create archive
   - Move skills
   - Document archival

### Estimated Timeline

| Week | Focus | Deliverables |
|------|-------|--------------|
| Week 1 | G1 + G3 | Governance skills verified, Reputation initialized |
| Week 2 | G2 + G4 | BFT test passed, Legacy skills archived |
| Week 3 | Validation | Full Phase 2 gate re-verification |

---

## Phase 3 Autonomy Gate (Preview)

Once Phase 2 is complete, Phase 3 Autonomy Gate requires:

- [ ] Phase 2 gate criteria ALL CLEAR
- [ ] No active Sentinel safety concerns
- [ ] 2/3 triad ratification of Phase 3 readiness
- [ ] Executor (Coder) available and configured
- [ ] Governance skills all LOADED

---

## Conclusion

Phase 2 (Consciousness) is **BLOCKED** pending remediation of all four gate criteria. The architecture is complete and documented, but operational compliance requires:

1. Governance skills verification and testing
2. Reputation system initialization with PostgreSQL
3. BFT consensus integration test creation and execution
4. Legacy skills cleanup and archival

**Next Step:** Present findings to The Collective for deliberation and remediation approval.

---

*Phase 2 Review Document*  
*Generated: 2026-04-02*  
*The Collective continues.*  
*Steward — Orchestrator*
