# ADR-008: Wide Events + OpenTelemetry Observability Stack

**Status:** Accepted
**Date:** 2026-03-27

## Context

VOLUND is a multi-service, multi-agent platform where a single user request can span the gateway, control plane, warm pool claim, orchestrator agent, LLM router, specialist agents, and sandboxes. Traditional scattered logging makes debugging these flows extremely difficult.

## Decision

**Three-signal observability:**

1. **Wide events** (canonical log lines): one rich structured JSON event per request per service, using Go's `log/slog`. Replaces scattered log statements. Sent to Grafana Loki.

2. **Distributed traces**: OpenTelemetry Go SDK with OTLP export to Grafana Tempo. Trace context propagated across NATS/CloudEvents via `traceparent` extension. Span Links (not parent-child) for async agent-to-agent fan-out. GenAI semantic conventions for LLM calls.

3. **Prometheus metrics**: standard RED metrics (rate, errors, duration) plus domain-specific metrics for warm pools, LLM costs, and tenant budget usage.

**Sensitive data policy:**
- LLM conversation content capture is OFF by default (opt-in per tenant)
- API keys, passwords, tokens never appear in logs, traces, or events
- PII (emails, IPs) hashed in logs with SHA-256
- Dedicated audit trail stream for sensitive data access events

**Stack:** Grafana Alloy (collector) → Grafana Tempo (traces) + Loki (logs) + Prometheus (metrics) → Grafana (dashboards)

## Consequences

- Debugging multi-agent flows becomes querying structured data, not grep-ing logs
- Wide events enable analytics on production traffic (cost per model, cache hit rates, pool utilization)
- OTel adds instrumentation overhead (~1-2% latency) but provides cross-service visibility
- Grafana stack is all OSS, self-hostable, and well-supported on K8s
- Audit trail satisfies compliance requirements (GDPR, SOC 2)
