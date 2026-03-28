# ADR-002: CloudEvents over NATS for Event Bus

**Status:** Amended
**Date:** 2026-03-27

## Context

Agent pods need to communicate lifecycle events, task assignments, progress updates, and results back to the control plane and to other agents. We need a standard event envelope and transport.

## Decision

Use [CloudEvents](https://cloudevents.io/) (CNCF Graduated) as the event envelope format, transported over NATS.

Event types follow the pattern `io.volund.<domain>.<action>`:
- `io.volund.agent.started`
- `io.volund.agent.heartbeat`
- `io.volund.task.assigned`
- `io.volund.task.completed`
- `io.volund.tool.invoked`
- `io.volund.skill.installed`

The `source` field encodes tenant/agent path: `//volund/tenants/{orgId}/agents/{agentId}`

## Consequences

- Standard envelope — no custom event format to maintain
- Go SDK (`github.com/cloudevents/sdk-go`) has native NATS bindings
- Tenants can subscribe to agent events via webhooks using standard format
- Easy to pipe into observability systems
- Protobuf and JSON format bindings available

---

## Amendment (2026-03): Two Coexisting Event Systems

During implementation we clarified that two distinct event systems coexist on the NATS backplane. CloudEvents handles lifecycle and control; a separate lightweight NATS stream handles real-time LLM token delivery.

| System | Subject pattern | Format | Persisted | Consumer |
|--------|-----------------|--------|-----------|----------|
| CloudEvents | `io.volund.*` subjects | CloudEvents JSON envelope | Yes (JetStream) | Control plane, webhooks |
| NATS streaming | `volund.conv.{convId}.stream` | Lightweight JSON (no CE envelope) | No | Gateway → WebSocket → UI |
| NATS task dispatch | `volund.agent.{instanceId}.task` | JSON | No | Agent runtime |
| NATS steering | `volund.agent.{instanceId}.steer` | JSON | No | Agent runtime |

**Why two systems:**

CloudEvents wrapping adds ~200 bytes of envelope overhead per event. For LLM token streaming (delta events firing every ~50ms), that overhead is unnecessary — the consumer is always the gateway WebSocket bridge, not a generic subscriber. Using lightweight JSON there keeps the hot path lean.

CloudEvents remain the right choice for lifecycle events (`task.assigned`, `task.completed`, `agent.heartbeat`) where discoverability, webhooks, and observability integrations matter.

**Gateway bridge pattern:**

```
Agent → NATS: volund.conv.{convId}.stream (delta/tool events)
             ↓
Gateway subscribes on WebSocket connect
             ↓
WebSocket → UI (real-time token stream)
```

The gateway subscribes to the conversation stream the moment a WebSocket connects. Agent pods have no socket handles — they only publish to NATS.
