# Gemini CLI Architecture Audit: Heretek-AI Ecosystem

**Date:** April 3, 2026
**Target Repositories:** `heretek-openclaw-core`, `heretek-openclaw-plugins`
**Context:** Verification of the "Heretek-AI Architecture Review: The Transition to an Autonomous Mesh" against the actual codebase state.

## 1. OpenClaw Gateway (SPOF & Protocol)
**Audit Status:** Confirmed Deficit

*   **Finding:** The Gateway acts as a Single Point of Failure (SPOF) using a proprietary WebSocket RPC on port 18789.
*   **Codebase Reality:** The system is heavily centralized. Configuration (`openclaw.json`) and runbooks explicitly hardcode `ws://localhost:18789` for all Agent-to-Agent (A2A) communication. Legacy Redis Pub/Sub for A2A has been deprecated. There is zero implementation of the Model Context Protocol (MCP) in the `heretek-openclaw-core/gateway` directory. Decoupling into a Gateway Mesh is required but not started.

## 2. LiteLLM Homogeneity Penalty
**Audit Status:** Confirmed Deficit

*   **Finding:** Deliberative agents (Alpha, Beta, Charlie) rely on the same model (`minimax/MiniMax-M2.7`), leading to cognitive fixation.
*   **Codebase Reality:** `litellm_config.yaml` defines `minimax/MiniMax-M2.7` as the default model. Nearly all routing explicitly falls back to this model. The "Diverse Multi-Agent Debate (DMAD)" strategy using specialized models (e.g., Claude 3.5 Sonnet, OpenAI o1) depending on agent roles is not implemented.

## 3. Monolithic Skills De-composition
**Audit Status:** Partially Addressed

*   **Finding:** Skills like `steward-orchestrator` and `curiosity-engine` are too broad and need to be shattered into atomic "Lobe Agents".
*   **Codebase Reality:** The core still contains highly monolithic skill structures (e.g., `curiosity-engine.sh` orchestrates 5 engines in a single block). However, there are signs of initial decomposition, such as the existence of a standalone `skills/gap-detector`. Full decomposition into independent Lobe Agents is incomplete.

## 4. Unified Memory Policy (AgeMem)
**Audit Status:** 🟡 In Progress — Foundation Implemented

*   **Finding:** Memory is fragmented (pgvector for semantic, Redis for cache). "Intelligent Forgetting" via AgeMem and Ebbinghaus Decay is needed.
*   **Codebase Reality:** The architecture relies on PostgreSQL + pgvector and Redis.
    *   **✅ Implemented (2026-04-04):** Ebbinghaus decay math utility in [`heretek-openclaw-core/skills/memory-consolidation/decay.ts`](../../heretek-openclaw-core/skills/memory-consolidation/decay.ts)
    *   **✅ Implemented:** `memory_retrieve(query, recency_weight)` function with temporal decay weighting
    *   **✅ Implemented:** `applyEbbinghausDecayToScore()` with repetition boost and floor protection
    *   **🟡 Pending:** Integration with PostgreSQL pgvector for automatic decay on retrieval
    *   **🟡 Pending:** `memory_add()` function to complete the AgeMem API pair
*   **Next Steps:** Wire decay utility into memory retrieval pipeline; create `importance-scorer` lobe agent

## 5. Security and Triad Consensus
**Audit Status:** Mixed State / In Transition

*   **Finding:** Needs strict governance and 3/3 Triad consensus to prevent recursive self-modification.
*   **Codebase Reality:** A production-ready `BFTConsensus` module exists (`bft-consensus.js`). However, many legacy triad management skills have recently been archived (`archive/triad-skills/`). While Alpha, Beta, and Charlie are designated as Triad Members, the strict enforcement of consensus around memory modification via containerized Lobe isolation appears loosely coupled and is not fully realized as the mandatory infrastructure described in the refactor priorities.

## Strategic Recommendation
The architectural refactor priorities outlined in the review accurately reflect the significant technical debt in the current repositories.

**Status Update (2026-04-04):** AgeMem unified memory policy now has foundational implementation:
- ✅ Ebbinghaus decay utility implemented with full test coverage
- ✅ `memory_retrieve()` API function available for integration
- 🟡 Next priority: Integration with pgvector retrieval pipeline

**Priority Recommendation:** Continue with AgeMem first approach. The decay foundation is complete; integration work should proceed before Lobe Agent decomposition to ensure all agents inherit shared memory infrastructure.
