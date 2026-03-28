# ADR-005: Orchestrator + Specialist Agent Model

**Status:** Amended
**Date:** 2026-03-27

## Context

VOLUND is a general-purpose agent platform. Users need to interact with agents that can handle complex, multi-step tasks spanning different domains (code, data, communication, research).

## Decision

A two-tier agent model: **Orchestrator** + **Specialists**.

### Orchestrator Agent
- One per user, long-running
- Primary chat interface — all user interaction goes through the orchestrator
- Understands user intent, decomposes complex requests into tasks
- Can execute simple tools directly
- Delegates specialized work to specialist agents
- Manages task lifecycle, aggregates results

### Specialist Agents
- Defined by `AgentProfile` CRDs
- Spawned on-demand by the orchestrator (creates `AgentInstance` CRs)
- Have focused system prompts, specific skill sets, and tuned model configs
- Can work in parallel on independent sub-tasks
- Report results back to orchestrator via CloudEvents
- Can form teams for complex multi-agent workflows

### Communication

```
User ←→ Orchestrator ←→ Specialists (via task tools + NATS)
                     ←→ Tool execution (subprocess v1 / agent-sandbox v2)
```

Orchestrators delegate to specialists using the **`create_task` tool** — it's a tool call in the LLM's output, not a direct API call. This keeps task creation composable and auditable in the conversation history.

Specialist agents post updates **directly into the chat thread** (not relayed through the orchestrator), giving users real-time visibility. Task results flow back to the orchestrator via its Redis inbox.

CloudEvents over NATS for lifecycle/control:
- `io.volund.task.assigned` — control plane → specialist pod
- `io.volund.task.progress` — specialist → control plane (Tasks UI)
- `io.volund.task.completed` — specialist → control plane → orchestrator inbox
- `io.volund.task.failed` — specialist → control plane → orchestrator inbox
- `io.volund.task.input_needed` — specialist → control plane → gateway (triggers WAITING state)

Lightweight NATS stream for real-time token delivery:
- `volund.conv.{convId}.stream` — agent → gateway → WebSocket → UI

### Agent Profile Creation

Two paths:
1. **Declarative** — YAML `AgentProfile` resources, publishable to The Forge
2. **Conversational** — orchestrator helps user build a profile through chat

## Consequences

- Clean separation of concerns — orchestrator handles routing, specialists handle execution
- Parallel task execution — multiple specialists can work simultaneously
- Composable — specialists can be mixed and matched per workflow
- Overhead — spawning specialist pods adds latency (mitigated by warm pools)
- Complexity — need robust task tracking and failure handling between agents

---

## Amendment (2026-03): Built-in Tools for Task Orchestration

The orchestrator delegates work through two built-in tools registered in the runtime:

| Tool | Agent | Description |
|------|-------|-------------|
| `create_task` | Orchestrator only | Emits `task.assigned` CloudEvent, returns task ID |
| `get_task_result` | Orchestrator only | Retrieves result for a completed task from inbox |

Restricting `create_task` to orchestrators prevents specialists from spawning unbounded sub-agents without oversight. Specialists request user input via `task.input_needed` events (routing WAITING state in gateway), not by spawning further agents.

The `create_task` tool takes `specialist_profile_id`, `task_description`, and optional `context` — the control plane handles pod claim and injection. The orchestrator does not manage pod lifecycle directly.
