# ADR-002: CloudEvents over NATS for Event Bus

**Status:** Accepted
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
