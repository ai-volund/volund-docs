# Observability Architecture

## Overview

VOLUND uses a three-signal observability stack: **wide events** (structured logs), **distributed traces**, and **metrics**. The system is built on OpenTelemetry with particular attention to tracing multi-agent flows and auditing access to sensitive data.

## Three Signals

```
┌─────────────────────────────────────────────────────────┐
│                    VOLUND Services                        │
│  (gateway, control plane, agent runtime, operator)       │
│                                                         │
│  ┌──────────────┐ ┌──────────────┐ ┌─────────────────┐ │
│  │  Wide Events  │ │   Traces     │ │   Metrics       │ │
│  │  (slog/JSON)  │ │  (OTel SDK)  │ │  (Prometheus)   │ │
│  └──────┬───────┘ └──────┬───────┘ └────────┬────────┘ │
└─────────┼────────────────┼──────────────────┼───────────┘
          │                │                  │
          ▼                ▼                  ▼
   ┌─────────────┐ ┌─────────────┐  ┌──────────────────┐
   │  Grafana     │ │  Grafana    │  │  Prometheus /    │
   │  Loki        │ │  Tempo      │  │  Grafana Mimir   │
   └──────┬──────┘ └──────┬──────┘  └────────┬─────────┘
          └────────────────┼──────────────────┘
                           ▼
                    ┌─────────────┐
                    │  Grafana    │
                    │  Dashboards │
                    └─────────────┘
```

## Signal 1: Wide Events (Canonical Log Lines)

Instead of scattered log statements, each service emits **one structured event per request** containing all relevant context.

### Principle

> Instead of logging what your code is doing, log what happened to this request.

A wide event for an LLM request might contain 30+ fields: tenant ID, agent ID, user ID, model, provider, input/output tokens, cost, latency, cache hit, fallback used, error, feature flags, deployment version — all in one JSON line.

### Implementation

Built on Go's `log/slog` (standard library since Go 1.21) with JSON output:

```go
// Build event context throughout the request lifecycle
ctx = WithField(ctx, "tenant_id", tenantID)
ctx = WithField(ctx, "agent_id", agentID)
ctx = WithField(ctx, "user_id", userID)
ctx = WithField(ctx, "request_id", requestID)

// ... later in LLM router ...
ctx = WithField(ctx, "provider", "anthropic")
ctx = WithField(ctx, "model", "claude-sonnet-4-6")
ctx = WithField(ctx, "input_tokens", 1523)
ctx = WithField(ctx, "output_tokens", 847)
ctx = WithField(ctx, "cost_usd", 0.0173)
ctx = WithField(ctx, "latency_ms", 2340)
ctx = WithField(ctx, "cache_hit", true)
ctx = WithField(ctx, "fallback_used", false)

// One emit at the end of the request
EmitWideEvent(ctx, "llm.request.completed")
```

Output (one JSON line):
```json
{
  "time": "2026-03-27T10:00:00Z",
  "level": "INFO",
  "msg": "llm.request.completed",
  "tenant_id": "acme-corp",
  "agent_id": "orch-123",
  "user_id": "user-456",
  "request_id": "req-789",
  "provider": "anthropic",
  "model": "claude-sonnet-4-6",
  "input_tokens": 1523,
  "output_tokens": 847,
  "cost_usd": 0.0173,
  "latency_ms": 2340,
  "cache_hit": true,
  "fallback_used": false,
  "trace_id": "abc123def456",
  "span_id": "789ghi",
  "deploy_version": "v0.3.1"
}
```

### Wide Event Types

Each service has a defined set of canonical events:

| Service | Event | Key Fields |
|---------|-------|------------|
| Gateway | `gateway.request` | method, path, status, tenant_id, user_id, latency_ms |
| LLM Router | `llm.request.completed` | provider, model, tokens, cost, latency, cache_hit, fallback |
| Agent Runtime | `agent.message.processed` | agent_type, profile, task_count, tool_invocations, memory_retrievals |
| Agent Runtime | `agent.task.delegated` | specialist_profile, task_description, required_skills |
| Operator | `operator.pod.claimed` | pool_type, claim_latency_ms, pool_size, pool_available |
| Operator | `operator.pod.released` | reason (idle/completed/error), active_duration_ms |
| Forge | `forge.skill.installed` | skill_name, version, tenant_id, runtime_type |

### Sensitive Field Handling in Logs

- **Never log**: API keys, passwords, auth tokens, full conversation content
- **Hash**: user email, IP addresses (SHA-256 with salt)
- **Truncate**: tool inputs/outputs (first 100 chars for debugging context)
- **Redact at collector**: Grafana Alloy pipeline strips fields matching `*_key`, `*_token`, `*_secret` patterns

## Signal 2: Distributed Traces (OpenTelemetry)

Traces follow requests across services and across async agent boundaries.

### Stack

- **SDK**: OpenTelemetry Go SDK (tracing + metrics stable, logs via `otelslog` bridge)
- **Exporter**: OTLP gRPC to Grafana Alloy
- **Backend**: Grafana Tempo (object store, low cost, no indexing required)
- **Visualization**: Grafana

### Trace Context Propagation

**Synchronous (gRPC/HTTP):** Standard OTel middleware — automatic.

**Async (NATS/CloudEvents):** Manual propagation via CloudEvents `traceparent` extension:

```go
// Publishing side: inject trace context into CloudEvent
event := cloudevents.NewEvent()
otel.GetTextMapPropagator().Inject(ctx, &cloudEventCarrier{event})
natsClient.Publish(subject, event)

// Subscribing side: extract trace context from CloudEvent
ctx = otel.GetTextMapPropagator().Extract(context.Background(), &cloudEventCarrier{event})
ctx, span := tracer.Start(ctx, "task.process", trace.WithLinks(
    trace.LinkFromContext(parentCtx), // Link to parent, not child — async fan-out
))
```

**Key decision:** Use **span Links** (not parent-child) for async agent-to-agent communication. This preserves the fan-out topology — an orchestrator spawning 3 specialists creates 3 linked traces, not a single deep tree.

### Span Naming Conventions

| Component | Span Name | Kind |
|-----------|-----------|------|
| Gateway | `HTTP POST /v1/agents/{id}/messages` | SERVER |
| Control Plane | `agent.route_message` | INTERNAL |
| Warm Pool | `pool.claim_pod` | INTERNAL |
| Agent Runtime | `agent.process_message` | INTERNAL |
| LLM Router | `llm.chat anthropic claude-sonnet-4-6` | CLIENT |
| Tool Exec | `tool.execute web-search` | INTERNAL |
| Sandbox | `sandbox.run code-executor` | CLIENT |
| NATS Publish | `task.assigned publish` | PRODUCER |
| NATS Subscribe | `task.completed receive` | CONSUMER |

### GenAI Semantic Conventions

Follow the OTel GenAI semantic conventions (Development status, but widely adopted):

```go
span.SetAttributes(
    attribute.String("gen_ai.system", "anthropic"),
    attribute.String("gen_ai.request.model", "claude-sonnet-4-6"),
    attribute.Int("gen_ai.request.max_tokens", 4096),
    attribute.String("gen_ai.response.model", "claude-sonnet-4-6"),
    attribute.Int("gen_ai.usage.input_tokens", 1523),
    attribute.Int("gen_ai.usage.output_tokens", 847),
    attribute.String("gen_ai.operation.name", "chat"),
)
```

**LLM content capture is OFF by default.** Enabled per-tenant or per-request via config flag. When enabled, prompt/completion content is recorded as span events (not attributes) so they can be filtered independently.

### Multi-Agent Trace Example

```
[Trace: user-request-abc]
├── gateway.request (HTTP POST /v1/agents/orch-1/messages)
│   ├── controlplane.route_message
│   │   └── pool.claim_pod (orchestrator pool, 120ms)
│   └── agent.process_message (orchestrator)
│       ├── llm.chat anthropic claude-sonnet-4-6 (1.2s)
│       ├── task.assigned publish (NATS → code-specialist)
│       └── task.assigned publish (NATS → research-agent)

[Trace: task-A-code] ──linked──▶ [user-request-abc]
├── task.assigned receive
├── pool.claim_pod (specialist pool, 80ms)
├── agent.process_message (code-specialist)
│   ├── llm.chat anthropic claude-sonnet-4-6 (2.1s)
│   ├── sandbox.run code-executor (3.5s)
│   ├── chat.post_message (via gateway)
│   └── task.completed publish (NATS)

[Trace: task-B-research] ──linked──▶ [user-request-abc]
├── task.assigned receive
├── agent.process_message (research-agent)
│   ├── tool.execute web-search (1.8s)
│   ├── llm.chat openai gpt-4o (1.5s)
│   └── task.completed publish (NATS)
```

## Signal 3: Metrics (Prometheus)

Standard Prometheus metrics exposed by each service.

### Key Metrics

**Gateway:**
- `volund_gateway_requests_total{method, path, status, tenant}`
- `volund_gateway_request_duration_seconds{method, path}`
- `volund_gateway_websocket_connections{tenant}`

**LLM Router:**
- `volund_llm_requests_total{provider, model, status}`
- `volund_llm_tokens_total{provider, model, direction}` (input/output)
- `volund_llm_request_duration_seconds{provider, model}`
- `volund_llm_cost_usd_total{tenant, provider, model}`
- `volund_llm_provider_health{provider}` (gauge: 1=healthy, 0=down)
- `volund_llm_circuit_breaker_state{provider}` (closed/open/half-open)

**Warm Pool / Operator:**
- `volund_pool_size{pool_type}` (orchestrator/specialist)
- `volund_pool_available{pool_type}` (unclaimed pods)
- `volund_pool_claim_duration_seconds{pool_type}`
- `volund_pool_claims_total{pool_type}`
- `volund_pool_releases_total{pool_type, reason}` (idle/completed/error)

**Agent Runtime:**
- `volund_agent_messages_processed_total{agent_type, profile}`
- `volund_agent_tasks_delegated_total{specialist_profile}`
- `volund_agent_tool_invocations_total{tool_name, runtime_type}`
- `volund_agent_memory_retrievals_total{tier}` (session/long_term)
- `volund_agent_active_duration_seconds{agent_type}`

**Billing:**
- `volund_tenant_budget_used_ratio{tenant}` (0.0 to 1.0)
- `volund_tenant_token_usage_total{tenant, direction}`

## Sensitive Data Audit Trail

For compliance and security, VOLUND maintains an audit trail of access to sensitive data.

### What Gets Audited

| Action | Details Captured |
|--------|-----------------|
| LLM API key access | Which agent, which tenant, timestamp |
| Conversation content access | Who viewed/exported, timestamp |
| Memory read/write | Agent ID, memory type, content hash (not content) |
| Tool execution | Tool name, input summary (truncated), sandbox ID |
| User data access | Which API endpoint, which user's data, accessor |
| Agent profile changes | Who changed, what changed, before/after |
| Skill installation | Which skill, version, permissions granted |

### Audit Event Format

Audit events are a special class of wide event written to a dedicated log stream:

```json
{
  "time": "2026-03-27T10:00:00Z",
  "audit": true,
  "action": "conversation.content.accessed",
  "actor_type": "agent",
  "actor_id": "code-specialist-456",
  "tenant_id": "acme-corp",
  "resource_type": "conversation",
  "resource_id": "conv-789",
  "context": {
    "task_id": "task-abc",
    "reason": "task_context_loading",
    "content_hash": "sha256:abc123..."
  }
}
```

### Audit Storage

- Written to a separate Loki stream (`{job="volund-audit"}`)
- Retained for configurable duration per tenant tier (free: 30 days, pro: 90 days, enterprise: 1 year+)
- Enterprise tier: exportable for external SIEM integration
- Immutable: append-only, no deletion API

## Observability Stack Deployment

```yaml
# deploy/helm/volund/values.yaml (observability section)
observability:
  # Grafana Alloy (collector)
  alloy:
    enabled: true
    # Receives OTLP from all services, forwards to backends

  # Grafana Tempo (traces)
  tempo:
    enabled: true
    storage:
      backend: s3       # MinIO in dev, S3 in prod

  # Grafana Loki (logs + wide events + audit)
  loki:
    enabled: true
    storage:
      backend: s3

  # Prometheus (metrics)
  prometheus:
    enabled: true
    retention: 15d

  # Grafana (visualization)
  grafana:
    enabled: true
    dashboards:
      - platform-overview
      - llm-router
      - warm-pool
      - tenant-usage
      - audit-trail
```

---

## Implementation Status

*Implemented 2026-03-28.*

### OTel SDK Bootstrap

Both `volund` (gateway) and `volund-agent` have an `internal/otel/otel.go` module:

```go
func Init(serviceName, otlpEndpoint string) (func(context.Context) error, error)
```

Returns a shutdown function. Sets up `TracerProvider` with OTLP/gRPC exporter, `MeterProvider` with Prometheus exporter, and a custom `slog.Handler` that extracts `trace_id` from span context.

### HTTP Tracing

The gateway wraps its mux with `otelhttp.NewMiddleware()` for automatic span creation on every HTTP request.

### NATS Trace Propagation

Trace context is injected into NATS message headers on dispatch and extracted on the agent side, using OTel's `TextMapPropagator` with a custom NATS header carrier.

### Prometheus Metrics

Key metrics registered via OTel MeterProvider:
- `volund_dispatch_total` — counter of dispatched tasks
- `volund_dispatch_duration_seconds` — histogram of dispatch latency
- `volund_claim_total` — counter of instance claims
- `volund_active_instances` — gauge of currently active agent instances
