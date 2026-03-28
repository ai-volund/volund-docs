# ADR-005: Orchestrator + Specialist Agent Model

**Status:** Accepted
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
User ←→ Orchestrator ←→ Specialists
                     ←→ Sandboxes (tool execution)
```

All inter-agent communication uses CloudEvents over NATS:
- `io.volund.task.assigned` — orchestrator → specialist
- `io.volund.task.progress` — specialist → orchestrator
- `io.volund.task.completed` — specialist → orchestrator
- `io.volund.task.failed` — specialist → orchestrator

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
