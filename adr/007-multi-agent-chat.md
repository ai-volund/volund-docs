# ADR-007: Multi-Agent Chat with Orchestrator Inbox

**Status:** Accepted
**Date:** 2026-03-27

## Context

When the orchestrator delegates tasks to specialists, the user needs visibility into what's happening. The orchestrator follows a serverless model and may not be running while specialists work. We need a communication pattern that keeps the user informed without requiring the orchestrator to relay every message.

## Decision

### Multi-Agent Chat Thread
Specialist agents post messages **directly into the conversation thread** via the gateway API. The user sees a unified timeline with messages from all participating agents, each tagged with agent name.

The chat is the single source of truth for all user-facing communication. No separate notification channels needed.

### User Input Routing
When a specialist needs user input mid-task, it posts a question to the chat and enters a WAITING state. The gateway routes the user's reply directly to the waiting specialist — the orchestrator is not involved.

### Orchestrator Inbox
The orchestrator has a Redis-backed inbox queue. When specialists complete tasks:
1. Task result goes into the orchestrator's inbox
2. Control plane triggers orchestrator spin-up
3. Orchestrator drains inbox, synthesizes results, posts summary to chat

### Tasks View
A separate UI view for monitoring long-running jobs — progress bars, status, cancel. This is a read-only dashboard driven by CloudEvents, not a communication channel.

## Consequences

- Chat stays clean: the user sees who's talking and can reply directly to specialists
- Orchestrator doesn't need to relay messages — reduces coupling and enables serverless model
- The conversation model supports multiple participants (proto updated with agent_id, participant tracking)
- Gateway needs a message routing table per conversation (which agent is active/waiting)
- Tasks view gives users a dashboard for async work without cluttering the chat

---

## Amendment (2026-03): Real-Time Architecture

The NATS backplane handles both lifecycle events (CloudEvents) and real-time streaming (lightweight NATS). See ADR-002 amendment for details on the two coexisting event systems.

**Task dispatch flow:**
1. User sends message → Gateway → Control Plane gRPC `DispatchMessage`
2. Control Plane persists message, claims warm instance (`FOR UPDATE SKIP LOCKED`), publishes to `volund.agent.{instanceId}.task`
3. Gateway subscribes to `volund.conv.{convId}.stream` on WebSocket connect
4. Agent publishes all events (delta, tool_start, tool_end, agent_end) to that stream
5. Gateway forwards to WebSocket — it knows nothing about agent internals

**Steering vs follow-up:**

| Pattern | Trigger | Delivery | When processed |
|---------|---------|----------|----------------|
| Steering | User sends correction mid-run | `volund.agent.{instanceId}.steer` | Before next LLM call (between tool executions) |
| Follow-up | User sends next message before agent idles | Redis queue per conversation | After current inner loop completes |

**Specialist WAITING state:**
When a specialist needs user input it publishes a `question` event to `volund.conv.{convId}.stream`. The gateway routes the user's reply to `volund.agent.{specialistId}.steer`. The specialist receives it as a steering message and continues without re-claiming a pod.
