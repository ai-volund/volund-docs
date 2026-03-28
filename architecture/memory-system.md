# Memory System Architecture

## Overview

VOLUND agents need memory at multiple time scales — from in-conversation context to long-term knowledge that persists across sessions and even across agent instances. The memory system is a core differentiator: agents that remember are agents users trust.

## Memory Tiers

```
┌─────────────────────────────────────────────────────────┐
│                    Agent Runtime                         │
│                                                         │
│  ┌─────────────┐  ┌──────────────┐  ┌───────────────┐  │
│  │  Working     │  │  Session     │  │  Long-Term    │  │
│  │  Memory      │  │  Memory      │  │  Memory       │  │
│  │              │  │              │  │               │  │
│  │  In-process  │  │  Redis       │  │  PostgreSQL   │  │
│  │  context     │  │  per-session │  │  + pgvector   │  │
│  │  window      │  │  TTL-based   │  │  persistent   │  │
│  └─────────────┘  └──────────────┘  └───────────────┘  │
│        │                 │                  │            │
│        ▼                 ▼                  ▼            │
│  ┌─────────────────────────────────────────────────┐    │
│  │           Memory Manager                         │    │
│  │  Handles retrieval, storage, summarization,      │    │
│  │  and relevance ranking across all tiers           │    │
│  └─────────────────────────────────────────────────┘    │
└─────────────────────────────────────────────────────────┘
```

### Tier 1: Working Memory (In-Process)

The active conversation context — what the LLM sees in its prompt.

| Property | Value |
|----------|-------|
| Storage | In-process (agent-runtime memory) |
| Lifetime | Single request/response cycle |
| Size | Bounded by model context window |
| Contents | System prompt, recent messages, retrieved memories, tool results |

**Key behavior:** The Memory Manager assembles working memory for each LLM call by combining the system prompt, recent conversation turns, and relevant memories retrieved from session and long-term storage.

**Context window management:**
- Track token count per message
- When approaching context limit, summarize older messages and swap in summaries
- Priority: system prompt > tool results > recent messages > retrieved memories > older messages

### Tier 2: Session Memory (Redis)

Per-conversation state that persists across multiple request/response cycles within a session.

| Property | Value |
|----------|-------|
| Storage | Redis (per-tenant keyspace) |
| Lifetime | Session duration + configurable TTL (default: 24h) |
| Size | Configurable per tenant tier |
| Contents | Full conversation history, extracted entities, task state |

**Key structures:**
```
volund:tenant:{orgId}:agent:{agentId}:session:{sessionId}:messages     # ordered list
volund:tenant:{orgId}:agent:{agentId}:session:{sessionId}:entities     # extracted entities
volund:tenant:{orgId}:agent:{agentId}:session:{sessionId}:tasks        # active task graph
volund:tenant:{orgId}:agent:{agentId}:session:{sessionId}:metadata     # session metadata
```

**Session lifecycle:**
1. New session created on first user message
2. Messages appended to session history
3. Entity extraction runs async after each exchange
4. Session archived to long-term memory on close or TTL expiry
5. Sessions can be resumed if within TTL

### Tier 3: Long-Term Memory (PostgreSQL + pgvector)

Persistent knowledge that spans sessions — facts, preferences, learned behaviors.

| Property | Value |
|----------|-------|
| Storage | PostgreSQL with pgvector extension |
| Lifetime | Permanent (until explicitly deleted) |
| Size | Quota per tenant tier |
| Contents | Facts, preferences, conversation summaries, learned patterns |

**Schema:**
```sql
CREATE TABLE memories (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id),
    agent_id        UUID NOT NULL REFERENCES agents(id),
    user_id         UUID REFERENCES users(id),        -- NULL for agent-global memories

    -- Content
    content         TEXT NOT NULL,                      -- The memory itself
    memory_type     TEXT NOT NULL,                      -- fact, preference, summary, learned
    source          TEXT,                               -- conversation, tool, system

    -- Vector embedding for semantic search
    embedding       vector(1536),                       -- text-embedding-3-small dimensions

    -- Metadata
    importance      FLOAT DEFAULT 0.5,                  -- 0.0 to 1.0, affects retrieval ranking
    access_count    INTEGER DEFAULT 0,                  -- How often retrieved
    last_accessed   TIMESTAMPTZ,
    created_at      TIMESTAMPTZ DEFAULT NOW(),
    updated_at      TIMESTAMPTZ DEFAULT NOW(),
    expires_at      TIMESTAMPTZ,                        -- Optional TTL for temporal memories

    -- Deduplication
    content_hash    TEXT GENERATED ALWAYS AS (md5(content)) STORED
);

CREATE INDEX idx_memories_embedding ON memories
    USING ivfflat (embedding vector_cosine_ops) WITH (lists = 100);
CREATE INDEX idx_memories_tenant_agent ON memories (tenant_id, agent_id);
CREATE INDEX idx_memories_type ON memories (memory_type);
```

**Memory types:**

| Type | Description | Example |
|------|-------------|---------|
| `fact` | Concrete information about the user or domain | "User prefers Python over JavaScript" |
| `preference` | User preferences and working style | "User wants concise responses without summaries" |
| `summary` | Compressed conversation summaries | "On 2026-03-27, discussed project architecture..." |
| `learned` | Patterns the agent has learned | "When user says 'ship it', they mean deploy to staging first" |
| `entity` | Named entities extracted from conversations | "Project VOLUND is a K8s agent platform" |

## Memory Operations

### Storage Flow

```
User message arrives
  │
  ▼
Memory Manager: retrieve relevant memories
  │ (semantic search on long-term + session lookup)
  ▼
Assemble working memory (system prompt + memories + conversation)
  │
  ▼
LLM processes and responds
  │
  ▼
Post-processing (async):
  ├── Append to session history
  ├── Extract entities
  ├── Evaluate for long-term storage
  │   ├── Is this a new fact? → store as fact
  │   ├── Is this a correction? → update existing memory
  │   ├── Is this a preference? → store as preference
  │   └── Nothing notable → skip
  └── Update memory importance scores
```

### Retrieval Strategy

For each LLM call, the Memory Manager retrieves relevant context:

1. **Recency** — last N messages from session (always included)
2. **Semantic search** — embed the current query, find top-K similar memories from long-term store
3. **Entity match** — if entities are mentioned, fetch related memories
4. **Importance weighting** — rank by `relevance_score * importance * recency_decay`

```go
type MemoryQuery struct {
    Text        string    // Current user message (for embedding)
    Entities    []string  // Extracted entities from current message
    AgentID     string
    UserID      string
    Limit       int       // Max memories to return
    MinScore    float64   // Minimum relevance threshold
}

type RetrievedMemory struct {
    Memory      Memory
    Score       float64   // Combined relevance score
    Source      string    // "semantic", "entity", "recency"
}
```

### Memory Consolidation

Periodic background process that maintains memory health:

- **Deduplication** — merge memories with similar content (cosine similarity > 0.95)
- **Summarization** — compress old session histories into summary memories
- **Decay** — reduce importance of memories that are never accessed
- **Pruning** — remove expired memories and those below importance threshold
- **Conflict resolution** — when contradictory memories exist, keep the most recent

## Memory Sharing

### Per-User vs Per-Agent

- **User memories** — specific to a user-agent pair (preferences, facts about the user)
- **Agent memories** — shared across all users of that agent (domain knowledge, learned patterns)
- **Tenant memories** — shared across all agents in a tenant (org-wide knowledge)

### Specialist Agent Memory

When the orchestrator delegates to a specialist:
1. Orchestrator packages relevant memories as context in the task assignment
2. Specialist has its own working + session memory during task execution
3. Notable findings from the specialist are reported back via CloudEvents
4. Orchestrator decides whether to promote specialist findings to long-term memory

## Privacy & Security

- Memories are tenant-isolated at the database level (row-level security)
- User memories are accessible only to agents within that user's scope
- Memory content is encrypted at rest (PostgreSQL TDE or application-level)
- Users can view, edit, and delete their memories via the API
- GDPR: full memory export and deletion per user

---

## Memory Implementation Decisions

*Decided during implementation phase (2026-03).*

### Three Tiers

| Tier | Storage | Scope | TTL | Status |
|---|---|---|---|---|
| Working memory | In-process `[]Message` | Current turn only | Discarded after turn ends | ✅ v1 |
| Session memory | Redis | Per conversation | Configurable (default 24h) | Phase 4 |
| Long-term memory | pgvector | Per agent profile | Permanent (up to quota) | Phase 4 |

Working memory is just the `messages` slice the agent holds during a single turn. It's the LLM's context window — no external storage needed. The agent holds it; when the turn ends and the pod returns to warm, it's gone.

### Memory Namespace

Memory is **scoped per agent profile** within a tenant, not per pod instance. This is essential because warm pool pods are interchangeable — any pod claiming a profile must have the same memory view.

```
Namespace format:  {tenantId}.{profileId}
Shared namespace:  {tenantId}.shared       ← readable by all profiles in tenant
```

The shared namespace is used for:
- Tenant-level uploaded documents
- Knowledge base entries (product info, company context, etc.)
- Artifacts produced by specialists that others need to read

### Context Window Budget

Before each LLM call the runtime applies a sliding window over working memory:

```
1. Count tokens in full message history
2. If over model's context limit:
   a. Summarize oldest messages (using a cheap/fast model call)
   b. Replace summary block in messages
   c. Retrieve relevant long-term memories via similarity search
   d. Inject as "recalled context" system message
3. Pass trimmed context to LLM
```

This is implemented in the `memory.Manager` interface so the no-op v1 implementation can be replaced without changing the agent loop.

### Interface (v1 — no-op, v2 — Redis + pgvector)

```go
type Manager interface {
    // Load recent messages for a conversation (session memory).
    LoadSession(ctx context.Context, convID string, limit int) ([]Message, error)

    // Persist a message to session memory.
    SaveMessage(ctx context.Context, convID string, msg Message) error

    // Search long-term memory by semantic similarity.
    Search(ctx context.Context, query string, limit int) ([]MemoryEntry, error)

    // Persist a fact or artifact to long-term memory.
    Store(ctx context.Context, entry MemoryEntry) error
}
```

The v1 `NoopManager` returns empty slices. The agent loop calls the interface — swapping in the Redis+pgvector implementation requires no loop changes.
- Embedding vectors are stored alongside content — deleting content deletes the vector
