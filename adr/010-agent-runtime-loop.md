# ADR-010: Agent Runtime Loop Design

**Status:** Accepted
**Date:** 2026-03-28

## Context

The `volund-agent` binary needs a runtime loop that handles:
- Streaming LLM responses to the user in real-time
- Tool execution mid-stream (tool call in LLM output → execute → continue)
- User steering (corrections sent while the agent is executing tools)
- Follow-up messages (user sends a second message before agent idles)
- Clean shutdown and state externalization

We reviewed [pi-agent](https://github.com/badlogic/pi-mono/tree/main/packages/agent) as a reference implementation. Its two-tier loop pattern was the primary design inspiration.

## Decision

### Two-Tier Loop

```
┌─── OUTER LOOP ──────────────────────────────────────────────────────┐
│  Runs while there are pending user messages (follow-ups)            │
│                                                                     │
│  ┌── INNER LOOP ───────────────────────────────────────────────┐   │
│  │  1. Drain steering messages (mid-run user corrections)      │   │
│  │  2. Build context: system prompt + messages + tool schemas  │   │
│  │  3. LLM call (streaming) → publish delta events to NATS    │   │
│  │  4. If tool call → execute → append result → goto 1        │   │
│  │  5. If final answer → persist message → publish agent_end  │   │
│  └────────────────────────────────────────────────────────────┘   │
│                                                                     │
│  Check follow-up queue. Messages waiting? → re-enter inner loop    │
└─────────────────────────────────────────────────────────────────────┘
```

**Steering messages** are injected between tool calls, before the next LLM call. They do not interrupt an in-flight LLM request — they wait for the current streaming response to finish, then get prepended to context for the next round. This allows real-time user correction without introducing race conditions.

**Follow-up messages** are only picked up after the inner loop is fully idle (final answer delivered). This prevents the agent from starting a new LLM call mid-stream. No re-claim needed — the pod stays active and just re-enters the inner loop.

**Max rounds** are enforced per profile (`max_tool_rounds`, default 10). A wall-clock timeout also applies. Both prevent unbounded loops.

### NATS Event Protocol

Every event is published to `volund.conv.{convId}.stream`. The gateway subscribes on WebSocket connect and fans events to the client.

| Event | Payload | When |
|-------|---------|------|
| `agent_start` | `{convId, agentId, profileType}` | Agent claimed, turn beginning |
| `turn_start` | `{turnId}` | New inner loop iteration |
| `delta` | `{turnId, content}` | Streaming text token from LLM |
| `tool_start` | `{turnId, toolName, args}` | Tool execution beginning |
| `tool_update` | `{turnId, toolName, partialResult}` | Incremental tool output (optional) |
| `tool_end` | `{turnId, toolName, result, isError}` | Tool execution complete |
| `turn_end` | `{turnId, stopReason}` | Inner loop iteration done |
| `agent_end` | `{convId, agentId, messageId}` | Turn complete, agent returning to warm |
| `error` | `{message, fatal}` | Error during execution |

### Tool Dispatch Hooks

```go
// Called before every tool execution. Return block:true to prevent execution.
type BeforeToolHook func(ctx context.Context, call ToolCall) (BlockDecision, error)

// Called after every tool execution. Can transform or redact the result.
type AfterToolHook func(ctx context.Context, call ToolCall, result ToolResult) (ToolResult, error)
```

Common uses:
- `beforeToolCall`: validate args, enforce allow-list, inject tenant context
- `afterToolCall`: redact secrets from results, add attribution metadata

### Built-in Tools

Tools are registered at startup based on the profile's `skills` array. The following are always available:

| Tool | Scope | Description |
|------|-------|-------------|
| `web_search` | All agents | Search the web via configured provider |
| `run_code` | All agents | Execute code as subprocess (v1), agent-sandbox (v2) |
| `read_memory` | All agents | Retrieve from long-term memory (semantic search) |
| `write_memory` | All agents | Store a fact/preference to long-term memory |
| `create_task` | Orchestrators only | Delegate work to a specialist agent |
| `get_task_result` | Orchestrators only | Retrieve completed task result from inbox |

`create_task` is gated to orchestrators to prevent specialists from spawning unbounded sub-agents.

### Memory Interface

The `memory.Manager` interface is injected at claim time:

```go
type Manager interface {
    LoadSession(ctx context.Context, convID string, limit int) ([]Message, error)
    SaveMessage(ctx context.Context, convID string, msg Message) error
    Search(ctx context.Context, query string, limit int) ([]MemoryEntry, error)
    Store(ctx context.Context, entry MemoryEntry) error
}
```

**Amendment (2026-03-28):** The original plan was to ship v1 with `NoopManager`. In practice, v1 shipped with a functional long-term memory implementation: pgvector-backed semantic search, decay scoring, deduplication, and `RetrieveContext` wired into the runtime loop. The `memory.Manager` interface is injected at startup and the production implementation uses PostgreSQL + pgvector for storage with embedding-based retrieval. Session memory (Redis-backed, per-conversation) was added in v2. The loop does not change between versions.

## Consequences

- Clean separation: inner loop handles LLM/tool execution; outer loop handles message queuing
- Steering enables real-time user control without introducing interruption complexity
- Follow-up batching keeps the loop simple — no concurrent LLM calls per agent
- NATS event protocol decouples agent execution from WebSocket handling entirely
- The `memory.Manager` interface makes v1 → v2 memory upgrade a drop-in swap
- Max rounds + timeout guard against infinite loops and runaway tool chains
- Tool hooks enable tenant-level policy enforcement without modifying tool implementations
