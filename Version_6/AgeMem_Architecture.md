# AgeMem Unified Memory Architecture

**Version:** 1.2.0
**Date:** 2026-04-04
**Status:** PostgreSQL Integration Complete — Full Stack Implementation
**Author:** Heretek-AI Collective

---

## Executive Summary

AgeMem (Age-aware Memory) is the Heretek-AI Collective's unified memory policy implementation, providing intelligent forgetting capabilities based on the Ebbinghaus forgetting curve. This architecture addresses memory fragmentation across PostgreSQL pgvector (semantic), Redis (STM), and session transcripts (episodic) by introducing a unified API with temporal decay weighting.

---

## 1. Architecture Overview

### 1.1 Problem Statement

**Current State (Pre-AgeMem):**
```
┌─────────────────┐     ┌─────────────────┐     ┌─────────────────┐
│  PostgreSQL     │     │  Redis          │     │  Session        │
│  pgvector       │     │  (Cache)        │     │  Transcripts    │
│  (Semantic)     │     │  (STM 24h)      │     │  (Episodic)     │
└─────────────────┘     └─────────────────┘     └─────────────────┘
```

**Issues:**
- No intelligent forgetting — memories accumulate indefinitely
- Context rot — obsolete information degrades retrieval quality
- Fragmented access — different interfaces for each memory tier
- No cross-tier correlation — episodic ↔ semantic linking missing

**Target State (AgeMem):**
```
┌─────────────────────────────────────────────────────────────────┐
│                    AgeMem Unified Memory API                     │
│  ┌─────────────┐ ┌─────────────┐ ┌─────────────┐               │
│  │  memory_    │ │  memory_    │ │  memory_    │               │
│  │  add()      │ │  retrieve() │ │  update()   │               │
│  └─────────────┘ └─────────────┘ └─────────────┘               │
│  ┌─────────────┐ ┌─────────────┐ ┌─────────────┐               │
│  │  memory_    │ │  memory_    │ │  memory_    │               │
│  │  delete()   │ │  summarize()│ │  filter()   │               │
│  └─────────────┘ └─────────────┘ └─────────────┘               │
│                                                                  │
│  ┌─────────────────────────────────────────────────────────────┐│
│  │              Ebbinghaus Decay Engine                        ││
│  │  R(t) = S × e^(-λt) × repetition_bonus                      ││
│  └─────────────────────────────────────────────────────────────┘│
└─────────────────────────────────────────────────────────────────┘
                              │
        ┌─────────────────────┼─────────────────────┐
        ▼                     ▼                     ▼
┌─────────────────┐ ┌─────────────────┐ ┌─────────────────┐
│  PostgreSQL     │ │  Redis          │ │  Session        │
│  pgvector       │ │  (Cache)        │ │  Transcripts    │
│  (Semantic)     │ │  (STM 24h)      │ │  (Episodic)     │
└─────────────────┘ └─────────────────┘ └─────────────────┘
```

---

## 2. Ebbinghaus Forgetting Curve Implementation

### 2.1 Mathematical Foundation

The Ebbinghaus forgetting curve describes exponential memory decay:

```
R(t) = S × e^(-λt)

Where:
  R(t) = Retention strength at time t
  S = Initial memory strength (importance score, 0-1)
  λ = Decay constant = ln(2) / halfLifeDays
  t = Time elapsed in days
```

### 2.2 Implementation Details

**File:** [`heretek-openclaw-core/skills/memory-consolidation/decay.ts`](../../heretek-openclaw-core/skills/memory-consolidation/decay.ts)

**Core Functions:**

| Function | Purpose | Signature | Status |
|----------|---------|-----------|--------|
| `memory_add()` | Add new memory with metadata | `(params: MemoryAddParams) => Promise<MemoryAddResult>` | ✅ Implemented |
| `memory_retrieve()` | Unified retrieval with decay weighting | `({ memories, query, recencyWeight, config }) => Promise<MemoryRetrievalResult[]>` | ✅ Implemented |
| `toDecayLambda()` | Converts half-life to decay constant | `(halfLifeDays: number) => number` | ✅ Implemented |
| `calculateEbbinghausMultiplier()` | Calculates retention multiplier | `({ ageInDays, halfLifeDays }) => number` | ✅ Implemented |
| `applyEbbinghausDecayToScore()` | Applies decay to importance score | `({ score, ageInDays, halfLifeDays, accessCount, config }) => number` | ✅ Implemented |
| `batchApplyDecay()` | Batch decay processing | `(memories, config) => Array<{ importance, decayedScore, ageInDays }>` | ✅ Implemented |
| `calculateOptimalReviewInterval()` | Schedules memory review | `({ currentScore, threshold, halfLifeDays }) => number` | ✅ Implemented |
| `getHalfLifeForMemoryType()` | Type-specific half-life calculation | `(type: MemoryType) => number` | ✅ Implemented |
| `validateImportance()` | Clamps importance to 0-1 range | `(importance?: number) => number` | ✅ Implemented |
| `generateMemoryId()` | Generates unique memory ID | `(content: string, createdAt: Date) => string` | ✅ Implemented |

### 2.3 Configuration

**Default Ebbinghaus Config:**
```typescript
{
  enabled: true,           // Enable temporal decay
  halfLifeDays: 7,         // Episodic memories decay faster
  floorMultiplier: 0.1,    // Never decay below 10% of original
  repetitionBoost: 1.5     // Frequently accessed memories boosted
}
```

**Recommended Half-Life by Memory Type:**
| Memory Type | Half-Life | Rationale |
|-------------|-----------|-----------|
| **Working** | N/A | Session lifetime only |
| **Episodic** | 7 days | Recent experiences fade quickly unless reinforced |
| **Semantic** | 30 days | Facts and concepts persist longer |
| **Procedural** | 90 days | Skills and habits are long-lasting |
| **Archival** | ∞ | Permanent record, no decay applied |

### 2.4 Repetition Boost Formula

Frequently accessed memories receive a logarithmic boost:

```
repetition_bonus = 1 + log10(accessCount + 1) × (repetitionBoost - 1)
```

This ensures diminishing returns — the 10th access provides less boost than the 2nd.

### 2.5 Floor Protection

To prevent complete memory loss:

```
final_score = max(decayedScore, importance × floorMultiplier)
```

Default floor: 10% of original importance score.

---

## 3. AgeMem Unified Memory API

### 3.1 API Specification

**Memory Operations (Tool Actions):**

| Operation | Function | Description | Status |
|-----------|----------|-------------|--------|
| `memory_add` | `memory_add(params)` | Store new memory with metadata | ✅ Implemented |
| `memory_retrieve` | `memory_retrieve(params)` | Retrieve with decay weighting | ✅ Implemented |
| `memory_update` | `memory_update(id, new_content)` | Update existing memory | 🟡 Pending |
| `memory_delete` | `memory_delete(id, reason)` | Remove memory (logged) | 🟡 Pending |
| `memory_summarize` | `memory_summarize(cluster_id)` | Generate summary for memory cluster | 🟡 Pending |
| `memory_filter` | `memory_filter(criteria)` | Filter memories by criteria | 🟡 Pending |

### 3.2 MemoryAddParams Schema

```typescript
interface MemoryAddParams {
  content: string;                    // Memory content (text or serialized data)
  type: MemoryType;                   // working | episodic | semantic | procedural | archival
  importance?: number;                // Initial importance score (0-1, default: 0.5)
  tags?: string[];                    // Optional tags for categorization
  metadata?: Record<string, unknown>; // Optional custom key-value pairs
  source?: string;                    // Optional source reference (file path, URL)
  clusterId?: string;                 // Optional cluster ID for grouping related memories
  config?: Partial<EbbinghausConfig>; // Optional Ebbinghaus config override
}
```

### 3.3 MemoryAddResult Schema

```typescript
interface MemoryAddResult {
  id: string;                         // Unique memory identifier
  content: string;                    // Memory content
  type: MemoryType;                   // Memory type
  importance: number;                 // Initial importance score (clamped 0-1)
  createdAt: string;                  // ISO 8601 timestamp
  path: string;                       // Storage path based on memory type
  tags: string[];                     // Associated tags
  metadata: Record<string, unknown>;  // Associated metadata
  success: boolean;                   // Whether operation succeeded
  error?: string;                     // Optional error message if failed
}
```

### 3.4 MemoryRetrievalResult Schema

```typescript
interface MemoryRetrievalResult {
  content: string;           // Memory content or reference
  originalScore: number;     // Original importance (0-1)
  decayedScore: number;      // Score after Ebbinghaus decay
  ageInDays: number;         // Memory age
  type: string;              // episodic | semantic | working
  accessCount: number;       // Access frequency
  createdAt: string;         // ISO 8601 timestamp
  path: string;              // File path or identifier
}
```

### 3.5 Usage Examples

#### memory_add Example

```typescript
import { memory_add, getHalfLifeForMemoryType } from './decay';

// Add semantic memory
const semanticResult = await memory_add({
  content: "User prefers TypeScript over JavaScript",
  type: "semantic",
  importance: 0.9,
  tags: ["user-preferences", "programming"],
  metadata: { language: "typescript" }
});

console.log(`Memory added with ID: ${semanticResult.id}`);
console.log(`Storage path: ${semanticResult.path}`);
console.log(`Half-life: ${getHalfLifeForMemoryType('semantic')} days`);

// Add episodic memory from session
const sessionResult = await memory_add({
  content: "Discussed Ebbinghaus forgetting curve implementation",
  type: "episodic",
  importance: 0.7,
  tags: ["session", "technical"],
  source: "episodes/2026-04-04/session.jsonl"
});
```

#### memory_retrieve Example

```typescript
import { memory_retrieve, DEFAULT_EBBINGHAUS_CONFIG } from './decay';

const results = await memory_retrieve({
  memories: [
    {
      content: "User prefers TypeScript over JavaScript",
      importance: 0.9,
      createdAt: "2026-04-01T10:00:00Z",
      accessCount: 15,
      type: "semantic",
      path: "memory/2026-04-01.md"
    },
    {
      content: "Session context from yesterday",
      importance: 0.7,
      createdAt: "2026-04-03T14:30:00Z",
      accessCount: 3,
      type: "episodic",
      path: "episodes/2026-04-03/session.jsonl"
    }
  ],
  query: "user preferences",
  recencyWeight: 0.3,  // Prioritize semantic relevance over recency
  config: {
    ...DEFAULT_EBBINGHAUS_CONFIG,
    halfLifeDays: 14  // Override for this retrieval
  }
});

// Results sorted by decayedScore (highest relevance first)
console.log(`Top result: ${results[0].content}`);
console.log(`Decayed score: ${results[0].decayedScore}`);
```

---

## 4. Integration Points

### 4.1 PostgreSQL pgvector Integration (✅ Implemented)

**Schema File:** [`heretek-openclaw-deploy/observability/config/agemem-init.sql`](../../heretek-openclaw-deploy/observability/config/agemem-init.sql)

The AgeMem PostgreSQL schema provides full Ebbinghaus decay support at the database layer, enabling efficient memory retrieval with temporal decay weighting.

#### 4.1.1 Main Schema: `memories` Table

```sql
CREATE TABLE IF NOT EXISTS memories (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    content         TEXT NOT NULL,
    type            VARCHAR(20) NOT NULL DEFAULT 'episodic'
                    CHECK (type IN ('working', 'episodic', 'semantic', 'procedural', 'archival')),
    importance_score FLOAT NOT NULL DEFAULT 0.5
                    CHECK (importance_score >= 0.0 AND importance_score <= 1.0),
    access_count    INTEGER NOT NULL DEFAULT 0,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    last_accessed_at TIMESTAMPTZ DEFAULT NULL,
    memory_type     VARCHAR(20) NOT NULL DEFAULT 'episodic',
    source          TEXT DEFAULT NULL,
    cluster_id      UUID DEFAULT NULL,
    tags            TEXT[] DEFAULT ARRAY[]::TEXT[],
    metadata        JSONB DEFAULT '{}'::jsonb,
    storage_path    TEXT NOT NULL,
    is_archived     BOOLEAN NOT NULL DEFAULT false,
    is_deleted      BOOLEAN NOT NULL DEFAULT false,
    deleted_at      TIMESTAMPTZ DEFAULT NULL,
    deleted_reason  TEXT DEFAULT NULL
);
```

**Key Columns:**
| Column | Type | Purpose |
|--------|------|---------|
| `importance_score` | FLOAT | Initial memory strength (0-1), decays over time |
| `access_count` | INTEGER | Tracks memory accesses for repetition boost |
| `memory_type` | VARCHAR(20) | Determines half-life for decay calculation |
| `created_at` | TIMESTAMPTZ | Timestamp for age calculation |
| `last_accessed_at` | TIMESTAMPTZ | Tracks when memory was last retrieved |
| `is_archived` | BOOLEAN | Soft archive flag |
| `is_deleted` | BOOLEAN | Soft delete flag (preserves audit trail) |

#### 4.1.2 Access Logging: `memory_access_log` Table

```sql
CREATE TABLE IF NOT EXISTS memory_access_log (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    memory_id       UUID NOT NULL REFERENCES memories(id) ON DELETE CASCADE,
    accessed_at     TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    access_type     VARCHAR(20) NOT NULL DEFAULT 'read'
                    CHECK (access_type IN ('read', 'write', 'update', 'delete')),
    agent_id        TEXT DEFAULT NULL,
    session_id      TEXT DEFAULT NULL,
    query_context   TEXT DEFAULT NULL
);
```

**Purpose:** Audit trail for memory access patterns, anomaly detection, and Sentinel review triggers.

#### 4.1.3 Core Functions

**`calculate_decayed_score()`** — Implements R(t) = S × e^(-λt) × repetition_bonus

```sql
CREATE OR REPLACE FUNCTION calculate_decayed_score(
    importance_score FLOAT,
    created_at TIMESTAMPTZ,
    access_count INTEGER,
    memory_type VARCHAR(20)
) RETURNS FLOAT AS $$
DECLARE
    age_in_days FLOAT;
    half_life_days FLOAT;
    lambda FLOAT;
    decay_multiplier FLOAT;
    repetition_bonus FLOAT;
    decayed_score FLOAT;
BEGIN
    age_in_days := EXTRACT(EPOCH FROM (NOW() - created_at)) / 86400.0;
    
    CASE memory_type
        WHEN 'working' THEN half_life_days := 0.5;
        WHEN 'episodic' THEN half_life_days := 7.0;
        WHEN 'semantic' THEN half_life_days := 30.0;
        WHEN 'procedural' THEN half_life_days := 90.0;
        WHEN 'archival' THEN RETURN importance_score;
        ELSE half_life_days := 7.0;
    END CASE;
    
    lambda := LN(2.0) / half_life_days;
    decay_multiplier := EXP(-lambda * age_in_days);
    
    IF access_count > 0 THEN
        repetition_bonus := 1.0 + (LOG(10.0, access_count + 1) * 0.5);
    ELSE
        repetition_bonus := 1.0;
    END IF;
    
    decayed_score := importance_score * decay_multiplier * repetition_bonus;
    
    IF decayed_score < (importance_score * 0.1) THEN
        decayed_score := importance_score * 0.1;
    END IF;
    
    RETURN decayed_score;
END;
$$ LANGUAGE plpgsql IMMUTABLE;
```

**Characteristics:**
- **IMMUTABLE:** Can be cached and indexed (no side effects, deterministic)
- **Type-specific half-lives:** Working (0.5d), Episodic (7d), Semantic (30d), Procedural (90d), Archival (∞)
- **Floor protection:** Never decays below 10% of original importance

---

**`retrieve_memories()`** — AgeMem memory_retrieve API (SQL layer)

```sql
CREATE OR REPLACE FUNCTION retrieve_memories(
    search_query TEXT DEFAULT NULL,
    memory_type_filter VARCHAR(20) DEFAULT NULL,
    min_importance FLOAT DEFAULT 0.0,
    limit_count INTEGER DEFAULT 100
) RETURNS TABLE (
    id UUID,
    content TEXT,
    type VARCHAR(20),
    original_score FLOAT,
    decayed_score FLOAT,
    age_in_days FLOAT,
    access_count INTEGER,
    created_at TIMESTAMPTZ,
    tags TEXT[],
    metadata JSONB,
    storage_path TEXT,
    similarity_score FLOAT
) AS $$
BEGIN
    RETURN QUERY
    SELECT
        m.id,
        m.content,
        m.type::VARCHAR(20),
        m.importance_score AS original_score,
        calculate_decayed_score(
            m.importance_score,
            m.created_at,
            m.access_count,
            m.memory_type
        ) AS decayed_score,
        EXTRACT(EPOCH FROM (NOW() - m.created_at)) / 86400.0 AS age_in_days,
        m.access_count,
        m.created_at,
        m.tags,
        m.metadata,
        m.storage_path,
        CASE
            WHEN search_query IS NOT NULL THEN
                ts_rank(to_tsvector('english', m.content), plainto_tsquery('english', search_query))
            ELSE 0.0
        END AS similarity_score
    FROM memories m
    WHERE NOT m.is_deleted
      AND NOT m.is_archived
      AND (search_query IS NULL OR to_tsvector('english', m.content) @@ plainto_tsquery('english', search_query))
      AND (memory_type_filter IS NULL OR m.type = memory_type_filter)
      AND m.importance_score >= min_importance
    ORDER BY
        calculate_decayed_score(
            m.importance_score,
            m.created_at,
            m.access_count,
            m.memory_type
        ) DESC,
        m.created_at DESC
    LIMIT limit_count;
END;
$$ LANGUAGE plpgsql STABLE;
```

**Features:**
- **STABLE:** Consistent results within a transaction, can use indexes
- **Full-text search:** PostgreSQL `to_tsvector` integration
- **Type filtering:** Filter by memory type (working, episodic, semantic, procedural, archival)
- **Ranked retrieval:** Ordered by decayed_score DESC, then created_at DESC

---

**`add_memory()`** — AgeMem memory_add API (SQL layer)

```sql
CREATE OR REPLACE FUNCTION add_memory(
    p_content TEXT,
    p_type VARCHAR(20) DEFAULT 'episodic',
    p_importance FLOAT DEFAULT 0.5,
    p_tags TEXT[] DEFAULT ARRAY[]::TEXT[],
    p_metadata JSONB DEFAULT '{}'::jsonb,
    p_source TEXT DEFAULT NULL,
    p_cluster_id UUID DEFAULT NULL
) RETURNS UUID AS $$
DECLARE
    v_id UUID;
    v_storage_path TEXT;
    v_date_str TEXT;
BEGIN
    -- Validate importance score
    p_importance := GREATEST(0.0, LEAST(1.0, COALESCE(p_importance, 0.5)));
    
    -- Validate memory type
    IF p_type NOT IN ('working', 'episodic', 'semantic', 'procedural', 'archival') THEN
        p_type := 'episodic';
    END IF;
    
    -- Generate storage path based on type
    v_date_str := TO_CHAR(NOW(), 'YYYY-MM-DD');
    v_id := gen_random_uuid();
    
    CASE p_type
        WHEN 'working' THEN v_storage_path := 'working/' || v_id || '.tmp';
        WHEN 'episodic' THEN v_storage_path := 'episodes/' || v_date_str || '/' || v_id || '.jsonl';
        WHEN 'semantic' THEN v_storage_path := 'memory/semantic/' || v_id || '.md';
        WHEN 'procedural' THEN v_storage_path := 'memory/procedural/' || v_id || '.md';
        WHEN 'archival' THEN v_storage_path := 'archive/' || v_date_str || '/' || v_id || '.md';
        ELSE v_storage_path := 'memory/' || v_id || '.md';
    END CASE;
    
    -- Insert memory
    INSERT INTO memories (
        id, content, type, importance_score, access_count,
        created_at, updated_at, memory_type, source,
        cluster_id, tags, metadata, storage_path, is_archived, is_deleted
    ) VALUES (
        v_id, p_content, p_type, p_importance, 0,
        NOW(), NOW(), p_type, p_source,
        p_cluster_id, p_tags, p_metadata, v_storage_path, false, false
    );
    
    RETURN v_id;
END;
$$ LANGUAGE plpgsql VOLATILE;
```

**Features:**
- **VOLATILE:** Modifies database state, generates new UUID
- **Automatic path routing:** Storage path generated based on memory type
- **Input validation:** Importance clamped to 0-1, type validated against enum
- **Returns:** UUID of newly created memory

---

**`track_memory_access()`** — Updates access statistics and logs access

```sql
CREATE OR REPLACE FUNCTION track_memory_access(
    p_memory_id UUID,
    p_access_type VARCHAR(20) DEFAULT 'read',
    p_agent_id TEXT DEFAULT NULL,
    p_session_id TEXT DEFAULT NULL,
    p_query_context TEXT DEFAULT NULL
) RETURNS VOID AS $$
BEGIN
    -- Update memory access statistics
    UPDATE memories
    SET
        access_count = access_count + 1,
        last_accessed_at = NOW(),
        updated_at = NOW()
    WHERE id = p_memory_id AND NOT is_deleted;
    
    -- Log the access
    INSERT INTO memory_access_log (
        memory_id, access_type, agent_id, session_id, query_context
    ) VALUES (
        p_memory_id, p_access_type, p_agent_id, p_session_id, p_query_context
    );
END;
$$ LANGUAGE plpgsql VOLATILE;
```

**Purpose:** Increment access_count (for repetition boost), update last_accessed_at, create audit log entry.

---

**`calculate_review_interval()`** — Spaced repetition scheduling

```sql
CREATE OR REPLACE FUNCTION calculate_review_interval(
    p_importance_score FLOAT,
    p_memory_type VARCHAR(20),
    p_threshold FLOAT DEFAULT 0.5
) RETURNS FLOAT AS $$
DECLARE
    half_life_days FLOAT;
    lambda FLOAT;
    days_until_threshold FLOAT;
BEGIN
    CASE p_memory_type
        WHEN 'working' THEN half_life_days := 0.5;
        WHEN 'episodic' THEN half_life_days := 7.0;
        WHEN 'semantic' THEN half_life_days := 30.0;
        WHEN 'procedural' THEN half_life_days := 90.0;
        WHEN 'archival' THEN RETURN -1;
        ELSE half_life_days := 7.0;
    END CASE;
    
    IF p_importance_score <= p_threshold THEN
        RETURN 0;
    END IF;
    
    lambda := LN(2.0) / half_life_days;
    IF lambda <= 0 THEN
        RETURN half_life_days;
    END IF;
    
    -- t = -ln(threshold/importance) / λ
    days_until_threshold := -LN(p_threshold / p_importance_score) / lambda;
    
    RETURN GREATEST(0, days_until_threshold);
END;
$$ LANGUAGE plpgsql IMMUTABLE;
```

**Purpose:** Calculate optimal days until next review based on Ebbinghaus curve. Returns -1 for archival memories (no review needed).

#### 4.1.4 Convenience View

```sql
CREATE OR REPLACE VIEW memories_with_decay AS
SELECT
    m.id,
    m.content,
    m.type,
    m.importance_score AS original_score,
    calculate_decayed_score(
        m.importance_score,
        m.created_at,
        m.access_count,
        m.memory_type
    ) AS decayed_score,
    EXTRACT(EPOCH FROM (NOW() - m.created_at)) / 86400.0 AS age_in_days,
    m.access_count,
    m.created_at,
    m.last_accessed_at,
    m.tags,
    m.metadata,
    m.storage_path,
    m.cluster_id,
    m.source
FROM memories m
WHERE NOT m.is_deleted AND NOT m.is_archived;
```

**Usage:** Quick access to decayed scores without calling function directly.

#### 4.1.5 Indexes (9 Total)

| Index | Columns | Purpose |
|-------|---------|---------|
| `idx_memories_type` | type | Filter by memory type |
| `idx_memories_importance` | importance_score DESC | Rank by importance |
| `idx_memories_created_at` | created_at DESC | Age-based queries |
| `idx_memories_last_accessed` | last_accessed_at DESC | Access pattern analysis |
| `idx_memories_cluster_id` | cluster_id | Cluster-based queries |
| `idx_memories_tags` | GIN(tags) | Tag array searches |
| `idx_memories_metadata` | GIN(metadata) | JSONB metadata queries |
| `idx_memories_type_importance_created` | type, importance_score DESC, created_at DESC | Composite retrieval |
| `idx_memory_access_memory_id` | memory_id | Access log lookups |
| `idx_memory_access_timestamp` | accessed_at DESC | Temporal access analysis |

#### 4.1.6 SQL Usage Examples

```sql
-- Add a new semantic memory
SELECT add_memory(
    'User prefers TypeScript over JavaScript',
    'semantic',
    0.9,
    ARRAY['user-preferences', 'programming'],
    '{"language": "typescript"}'::jsonb
);

-- Retrieve memories with decay weighting
SELECT * FROM retrieve_memories(
    'TypeScript',           -- search_query
    'semantic',             -- memory_type_filter
    0.3,                    -- min_importance
    10                      -- limit_count
);

-- Track memory access (call after retrieval)
SELECT track_memory_access(
    '550e8400-e29b-41d4-a716-446655440000',  -- memory_id
    'read',                                   -- access_type
    'agent-copilot',                          -- agent_id
    'session-12345'                           -- session_id
);

-- Calculate review interval for spaced repetition
SELECT calculate_review_interval(
    0.8,              -- importance_score
    'episodic',       -- memory_type
    0.5               -- threshold
);
-- Returns: ~4.8 days until memory decays below 0.5

-- Query view for quick access
SELECT id, content, decayed_score, age_in_days
FROM memories_with_decay
WHERE type = 'episodic'
ORDER BY decayed_score DESC
LIMIT 20;
```

### 4.2 Redis Cache Integration (Pending)

**TTL Calculation:**
```typescript
function calculateMemoryTTL(importance: number, accessCount: number): number {
  const baseTTL = 24 * 60 * 60; // 24 hours in seconds
  const importanceMultiplier = 1 + importance;
  const accessBonus = Math.log10(accessCount + 1);
  return Math.floor(baseTTL * importanceMultiplier * accessBonus);
}
```

### 4.3 Session Transcript Integration (Implemented)

The `memory_retrieve` function can process session transcript entries directly:

```typescript
const sessionMemories = sessionTranscripts.map(entry => ({
  content: entry.message,
  importance: entry.importance ?? 0.5,
  createdAt: entry.timestamp,
  accessCount: entry.viewCount ?? 0,
  type: 'episodic',
  path: `episodes/${entry.sessionId}/${entry.messageId}`
}));

const ranked = await memory_retrieve({
  memories: sessionMemories,
  recencyWeight: 0.7  // Favor recent conversations
});
```

---

## 5. Memory Lifecycle

### 5.1 Memory States

```
┌──────────────┐     ┌──────────────┐     ┌──────────────┐
│   WORKING    │────►│   EPISODIC   │────►│   SEMANTIC   │
│  (Session)   │     │  (0-30 days) │     │  (Permanent) │
└──────────────┘     └──────────────┘     └──────────────┘
       │                    │                    │
       │                    ▼                    │
       │             ┌──────────────┐            │
       └────────────►│   ARCHIVE    │◄───────────┘
                     │  (Cold Store)│
                     └──────────────┘
                            │
                            ▼
                     ┌──────────────┐
                     │   FORGOTTEN  │
                     │  (Deleted)   │
                     └──────────────┘
```

### 5.2 State Transition Triggers

| Transition | Trigger | Criteria |
|------------|---------|----------|
| Working → Episodic | Session end | All session memories persisted |
| Episodic → Semantic | Promotion | accessCount > 10 OR importance > 0.8 OR tagged "critical" |
| Episodic → Archive | Aging | age > 30 days AND accessCount = 0 |
| Semantic → Archive | Deprecation | tagged "deprecated" OR superseded |
| Any → Forgotten | Manual delete | Explicit deletion with reason logged |

---

## 6. Security & Governance

### 6.1 Memory Access Control

| Operation | Permission Required |
|-----------|---------------------|
| `memory_add` | Any authenticated agent with trust score >= 30 |
| `memory_retrieve` | Any authenticated agent |
| `memory_update` | Memory owner OR Triad consensus |
| `memory_delete` | Triad consensus (2/3) |
| `memory_summarize` | Historian agent OR Triad consensus |
| High-trust operations | Trust score >= 70 |
| Critical operations | Trust score >= 85 |

### 6.2 Agent Reputation System

**Phase 5 Security Hardening** introduced a comprehensive reputation system:

- **Trust Scoring:** Dynamic 0-100 reputation scores for all agents
  - Initial score: 50 for new agents
  - Write permission threshold: 30
  - High-trust threshold: 70
  - Critical operation threshold: 85

- **Score Adjustments:**
  - Successful write: +2 (with consecutive bonus up to +10)
  - Violation: -10
  - Memory poisoning detection: -25
  - God Mode attempt: -30
  - Inactivity decay: -1 per day after 7 days

- **Recovery Mechanisms:**
  - Probation period: 3 days after violation
  - Recovery rate: +3 per day of good behavior
  - Consecutive success bonus: +0.5 per success (capped at +10)

- **Audit Trail:** Complete reputation history and activity logging

**Reference:** [`reputation-system.ts`](../../heretek-openclaw-core/skills/agemem-governance/reputation-system.ts)

### 6.3 Container Isolation

**Phase 5 Security Hardening** implemented process isolation for Lobe Agents:

- **Resource Limits:**
  - CPU: 1 core per agent (configurable)
  - Memory: 512MB per agent
  - Disk: 1GB per agent
  - Network: 100Mbps per agent

- **Security Profiles:**
  - Default profile: Balanced security with common syscalls allowed
  - Strict profile: Minimal permissions for high-security operations

- **Syscall Blocking:** Dangerous syscalls blocked (ptrace, mount, reboot, etc.)

- **Path Blocking:** Sensitive paths protected (/etc/shadow, /root, /dev/mem, etc.)

- **Command Whitelisting:** Only approved commands allowed (node, npm, git, etc.)

- **Violation Tracking:** Resource, security, and execution violations with severity levels

**Reference:** [`container-isolation.ts`](../../heretek-openclaw-core/skills/agemem-governance/container-isolation.ts)

### 6.4 Memory Poisoning Prevention

- **Write Permissions:** Gated by agent reputation score (minimum 30)
- **Content Validation:** Liberation Shield strict mode for inter-agent memory writes
- **Audit Logging:** All memory operations logged to consensus ledger
- **Anomaly Detection:** Unusual access patterns trigger Sentinel review
- **Poison Detection:** Automatic detection with -25 reputation penalty

### 6.5 God Mode Prevention

The Ebbinghaus decay implementation is **read-only** — it calculates scores but cannot:
- Modify memory content
- Grant elevated permissions
- Bypass consensus requirements
- Access external systems

**Additional Protections:**
- God Mode attempts detected and logged with -30 reputation penalty
- Container isolation prevents privilege escalation
- Seccomp/AppArmor profiles block system-level attacks

---

## 7. Implementation Status

### 7.1 Core Memory API

| Component | Status | File Reference |
|-----------|--------|----------------|
| Ebbinghaus decay math | ✅ Implemented | [`decay.ts`](../../heretek-openclaw-core/skills/memory-consolidation/decay.ts) |
| `memory_retrieve()` API (TypeScript) | ✅ Implemented | [`decay.ts`](../../heretek-openclaw-core/skills/memory-consolidation/decay.ts:161) |
| `memory_add()` API (TypeScript) | ✅ Implemented | [`decay.ts`](../../heretek-openclaw-core/skills/memory-consolidation/decay.ts:392) |
| PostgreSQL schema | ✅ Implemented | [`agemem-init.sql`](../../heretek-openclaw-deploy/observability/config/agemem-init.sql) |
| `calculate_decayed_score()` SQL | ✅ Implemented | [`agemem-init.sql`](../../heretek-openclaw-deploy/observability/config/agemem-init.sql:96) |
| `retrieve_memories()` SQL | ✅ Implemented | [`agemem-init.sql`](../../heretek-openclaw-deploy/observability/config/agemem-init.sql:180) |
| `add_memory()` SQL | ✅ Implemented | [`agemem-init.sql`](../../heretek-openclaw-deploy/observability/config/agemem-init.sql:245) |
| `track_memory_access()` SQL | ✅ Implemented | [`agemem-init.sql`](../../heretek-openclaw-deploy/observability/config/agemem-init.sql:299) |
| `calculate_review_interval()` SQL | ✅ Implemented | [`agemem-init.sql`](../../heretek-openclaw-deploy/observability/config/agemem-init.sql:329) |
| `memories_with_decay` view | ✅ Implemented | [`agemem-init.sql`](../../heretek-openclaw-deploy/observability/config/agemem-init.sql:152) |

### 7.2 Security & Governance (Phase 5)

| Component | Status | File Reference | Tests |
|-----------|--------|----------------|-------|
| Agent Reputation System | ✅ Implemented | [`reputation-system.ts`](../../heretek-openclaw-core/skills/agemem-governance/reputation-system.ts) | 63 |
| Container Isolation | ✅ Implemented | [`container-isolation.ts`](../../heretek-openclaw-core/skills/agemem-governance/container-isolation.ts) | 51 |
| Trust Score Permission Gates | ✅ Implemented | [`reputation-system.ts`](../../heretek-openclaw-core/skills/agemem-governance/reputation-system.ts:188) | — |
| Security Profiles (Default/Strict) | ✅ Implemented | [`container-isolation.ts`](../../heretek-openclaw-core/skills/agemem-governance/container-isolation.ts:149) | — |
| Syscall/Path Blocking | ✅ Implemented | [`container-isolation.ts`](../../heretek-openclaw-core/skills/agemem-governance/container-isolation.ts:362) | — |

### 7.3 Pending Components

| Component | Status | File Reference |
|-----------|--------|----------------|
| Redis TTL integration | ✅ Implemented | [`redis-ttl-manager.ts`](../../heretek-openclaw-core/skills/redis-ttl-manager/redis-ttl-manager.ts) |
| `importance-scorer` lobe | ✅ Implemented | [`importance-scorer.ts`](../../heretek-openclaw-core/skills/importance-scorer/importance-scorer.ts) |
| `archivist` lobe | ✅ Implemented | [`archivist.ts`](../../heretek-openclaw-core/skills/archivist/archivist.ts) |
| Automatic rollback mechanisms | 🟡 Pending | — |
| Liberation Shield strict mode for A2A | 🟡 Pending | — |

---

## 8. Performance Considerations

### 8.1 Complexity

| Operation | Time Complexity | Space Complexity |
|-----------|-----------------|------------------|
| `toDecayLambda()` | O(1) | O(1) |
| `calculateEbbinghausMultiplier()` | O(1) | O(1) |
| `applyEbbinghausDecayToScore()` | O(1) | O(1) |
| `memory_retrieve()` | O(n log n) | O(n) |
| `batchApplyDecay()` | O(n) | O(n) |
| `calculateOptimalReviewInterval()` | O(1) | O(1) |

### 8.2 Optimization Strategies

1. **Caching:** Cache decay multipliers for frequently accessed memories
2. **Batch Processing:** Use `batchApplyDecay()` for bulk operations
3. **Lazy Evaluation:** Calculate decayed scores only at retrieval time
4. **Indexing:** Create database indexes on `created_at`, `importance_score`, `access_count`

---

## 9. Testing Strategy

### 9.1 Unit Test Cases

```typescript
describe('Ebbinghaus decay', () => {
  it('matches exponential decay formula', () => {
    // R(t) = e^(-λt) where λ = ln(2) / halfLifeDays
    // At t = halfLifeDays, R = 0.5
  });

  it('does not decay evergreen memory files', () => {
    // MEMORY.md and memory/ folder root files exempt
  });

  it('applies repetition boost correctly', () => {
    // log10(accessCount + 1) scaling
  });

  it('respects floor multiplier', () => {
    // Never decay below floorMultiplier × original
  });

  it('handles edge cases', () => {
    // Negative ages, infinite values, zero half-life
  });
});
```

### 9.2 Integration Tests

- End-to-end retrieval with pgvector
- Redis TTL expiration behavior
- Cross-tier memory correlation

---

## 10. Future Enhancements

1. **Adaptive Half-Life:** Learn optimal half-life per memory type from usage patterns
2. **Emotional Weighting:** Boost memories with high emotional salience (Empath agent integration)
3. **Cross-Memory Linking:** Automatic episodic ↔ semantic correlation
4. **Spaced Repetition:** Automated review scheduling based on `calculateOptimalReviewInterval()`
5. **Memory Clustering:** Group related memories for efficient summarization

---

## 11. Security Features

### 11.1 SQL Injection Protection

**Implementation:** [`heretek-openclaw-core/lib/sql-utils.ts`](../../heretek-openclaw-core/lib/sql-utils.ts)

AgeMem uses parameterized queries and identifier escaping to prevent SQL injection attacks:

**Key Functions:**
- `escapeTableName()` - Validates and escapes table names
- `escapeColumnName()` - Validates and escapes column names
- `escapeIdentifier()` - Generic identifier escaping with double quotes
- `validateIdentifier()` - Validates identifiers against regex and reserved keywords
- `sanitizeOrderBy()` - Sanitizes ORDER BY clauses
- `sanitizeLimit()` / `sanitizeOffset()` - Validates LIMIT/OFFSET values

**Usage Example:**
```typescript
import { escapeTableName, escapeColumnName } from './lib/sql-utils';

const tableName = 'memories';
const columnName = 'embedding';

// Safe SQL generation
const sql = `SELECT * FROM ${escapeTableName(tableName)} 
             ORDER BY ${escapeColumnName(columnName)} DESC`;
```

**Protection Features:**
- ✅ Rejects invalid identifiers (must start with letter/underscore)
- ✅ Blocks reserved SQL keywords (SELECT, DROP, etc.)
- ✅ Enforces maximum identifier length (63 chars for PostgreSQL)
- ✅ Escapes double quotes in identifiers
- ✅ Detects suspicious injection patterns

### 11.2 Redis Authentication

**Implementation:** [`heretek-openclaw-core/lib/redis-client.ts`](../../heretek-openclaw-core/lib/redis-client.ts)

Centralized Redis client manager with production-ready security:

**Features:**
- ✅ Password authentication via `REDIS_PASSWORD` environment variable
- ✅ TLS/SSL support for encrypted connections
- ✅ Automatic reconnection with exponential backoff
- ✅ Connection pooling and singleton pattern
- ✅ Configurable timeouts and retry limits

**Configuration:**
```bash
# .env.example
REDIS_URL=redis://localhost:6379
REDIS_PASSWORD=your_secure_password
REDIS_TLS=true  # Enable TLS for production
REDIS_CONNECT_TIMEOUT=10000
REDIS_MAX_RETRIES=3
```

**Usage:**
```typescript
import { 
  getRedisClient, 
  isRedisClientInitialized,
  createRedisClient,
  createRedisConfigFromEnv 
} from './lib/redis-client';

// Initialize with environment variables
if (!isRedisClientInitialized()) {
  const config = createRedisConfigFromEnv();
  await createRedisClient(config);
}

// Get singleton client
const client = getRedisClient();
await client.set('key', 'value');
```

### 11.3 Audit Log Retention

**Implementation:** 
- Database: [`heretek-openclaw-core/migrations/005_add_audit_log_retention.sql`](../../heretek-openclaw-core/migrations/005_add_audit_log_retention.sql)
- Cleanup Script: [`heretek-openclaw-core/scripts/audit-cleanup.sh`](../../heretek-openclaw-core/scripts/audit-cleanup.sh)
- Skill: [`heretek-openclaw-core/skills/audit-cleanup/audit-cleanup.ts`](../../heretek-openclaw-core/skills/audit-cleanup/audit-cleanup.ts)

Configurable audit log retention policies with automated cleanup:

**Retention Policies (Default):**
| Event Type | Retention Period |
|------------|------------------|
| debug      | 7 days           |
| info       | 30 days          |
| warning    | 90 days          |
| error      | 365 days         |
| critical   | 1825 days (5 years) |

**Database Schema:**
```sql
CREATE TABLE audit_retention_config (
    id SERIAL PRIMARY KEY,
    event_type VARCHAR(100) UNIQUE NOT NULL,
    retention_days INTEGER NOT NULL DEFAULT 90,
    created_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP,
    CONSTRAINT valid_retention_days CHECK (retention_days > 0)
);

-- Cleanup function
CREATE OR REPLACE FUNCTION cleanup_audit_logs()
RETURNS INTEGER AS $$
DECLARE
    deleted_count INTEGER;
BEGIN
    DELETE FROM audit_log
    WHERE created_at < (
        SELECT CURRENT_TIMESTAMP - (retention_days || ' days')::INTERVAL
        FROM audit_retention_config
        WHERE audit_log.event_type = audit_retention_config.event_type
    );
    
    GET DIAGNOSTICS deleted_count = ROW_COUNT;
    RETURN deleted_count;
END;
$$ LANGUAGE plpgsql;
```

**Automated Cleanup:**
- Cron job runs every 2 hours: `0 */2 * * *`
- Batch deletions to avoid database lock contention
- Dry-run mode for testing: `./audit-cleanup.sh --dry-run`
- Logging to `/var/log/openclaw-audit-cleanup.log`

**Installation:**
```bash
cd heretek-openclaw-core/scripts
./install-audit-cron.sh install
```

### 11.4 Security Best Practices

1. **Always use parameterized queries** - Never concatenate user input into SQL
2. **Validate all identifiers** - Use `validateIdentifier()` before dynamic SQL
3. **Enable Redis authentication** - Set `REDIS_PASSWORD` in production
4. **Use TLS for Redis** - Set `REDIS_TLS=true` for encrypted connections
5. **Rotate credentials regularly** - Update passwords and API keys periodically
6. **Monitor audit logs** - Review critical events and anomalies
7. **Limit retention periods** - Don't store sensitive data longer than needed

---

## Appendix A: Research Citations

1. Ebbinghaus, H. (1885). "Über das Gedächtnis" (On Memory)
2. Murre, J.M.J. & Dros, J. (2015). "Replication and Analysis of Ebbinghaus' Forgetting Curve"
3. Yu, Y. et al. (2026). "AgeMem: Learning Unified Long-Term and Short-Term Memory Management"
4. Xu, W. et al. (2025). "A-MEM: Agentic Memory for LLM Agents"

---

**Document Version:** 1.2.0
**Created:** 2026-04-04
**Updated:** 2026-04-04
**Status:** PostgreSQL Integration Complete — Full Stack Ebbinghaus Decay Implementation

🦞 *The lobster way — Any OS. Any Platform. Together. The thought that never ends.*
