# ADR-006: Serverless Agent Execution Model

**Status:** Accepted
**Date:** 2026-03-27

## Context

Agents need to scale efficiently across hundreds of tenants. Having always-on pods per agent is wasteful — most agents are idle most of the time. We need a model that supports instant responsiveness for active users while releasing resources when agents are idle.

## Decision

Adopt a **serverless execution model** for agents:

1. The `volund-agent` binary is **generic and identity-less** — it becomes an orchestrator or specialist by loading an AgentProfile at claim time
2. **Warm pools** of pre-started pods (separate pools for orchestrators and specialists)
3. **Three pod states**: Active → Warm Idle (configurable timeout) → Cold (released to pool)
4. **All state externalized**: conversation history in Redis, long-term memory in PostgreSQL, task state in Redis
5. **Orchestrator inbox**: Redis queue for task results, scheduled triggers, and notifications received while cold
6. **Cold start overhead**: ~200-600ms (dominated by pod claim, not state restore)

## Consequences

- Efficient resource usage: pods serve many agents over time
- Near-instant response for active users (Warm Idle → Active is ~0ms)
- Acceptable cold start latency (~500ms, hidden by LLM response time of 1-5s)
- State externalization adds complexity but enables resilience (pod crash = no data loss)
- Idle timeouts configurable per tenant tier (free: 2min, pro: 10min, enterprise: always-on option)
- The operator must manage warm pool sizing and scaling
