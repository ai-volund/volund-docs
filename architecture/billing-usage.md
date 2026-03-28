# Billing & Usage Architecture

## Overview

VOLUND tracks all resource consumption per tenant for billing, quota enforcement, and cost visibility. The billing system meters three dimensions: LLM token usage, compute time, and storage.

## Metering Dimensions

### 1. LLM Token Usage

Tracked by the LLM Router on every request (see [LLM Router](llm-router.md)).

| Metric | Source | Granularity |
|--------|--------|-------------|
| Input tokens | LLM Router | Per request |
| Output tokens | LLM Router | Per request |
| Cached tokens | LLM Router | Per request |
| Cost (USD) | Calculated from pricing config | Per request |

### 2. Compute Time

Agent and sandbox pod uptime.

| Metric | Source | Granularity |
|--------|--------|-------------|
| Agent pod uptime | Operator (pod lifecycle events) | Per minute |
| Sandbox pod uptime | agent-sandbox events | Per minute |
| CPU seconds consumed | K8s metrics API | Per minute |
| Memory-seconds | K8s metrics API | Per minute |

### 3. Storage

Persistent data usage.

| Metric | Source | Granularity |
|--------|--------|-------------|
| Database storage | PostgreSQL pg_stat | Daily |
| Object storage | MinIO/S3 usage API | Daily |
| Memory entries | Count of long-term memories | Daily |
| PVC usage | K8s storage metrics | Daily |

## Data Flow

```
┌──────────────┐  ┌──────────────┐  ┌──────────────┐
│  LLM Router  │  │   Operator   │  │ Storage APIs │
│  (tokens)    │  │  (compute)   │  │  (disk)      │
└──────┬───────┘  └──────┬───────┘  └──────┬───────┘
       │                 │                 │
       │   CloudEvents   │   CloudEvents   │   Periodic poll
       ▼                 ▼                 ▼
┌─────────────────────────────────────────────────────┐
│                  Usage Collector                      │
│                                                     │
│  Aggregates events into per-tenant usage records     │
│  Writes to usage_records table                       │
│  Checks against budget thresholds                    │
└──────────────────────┬──────────────────────────────┘
                       │
          ┌────────────┼────────────┐
          ▼            ▼            ▼
   ┌────────────┐ ┌─────────┐ ┌──────────┐
   │ PostgreSQL │ │ Alerts  │ │ Dashboard│
   │ (records)  │ │ (NATS)  │ │ (API)    │
   └────────────┘ └─────────┘ └──────────┘
```

## Database Schema

```sql
-- Raw usage events (append-only, partitioned by month)
CREATE TABLE usage_records (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL,
    agent_id        UUID,
    user_id         UUID,

    -- What was consumed
    record_type     TEXT NOT NULL,        -- 'llm', 'compute', 'storage'

    -- LLM-specific
    provider        TEXT,
    model           TEXT,
    input_tokens    INTEGER,
    output_tokens   INTEGER,
    cached_tokens   INTEGER,

    -- Compute-specific
    cpu_seconds     FLOAT,
    memory_mb_seconds FLOAT,

    -- Storage-specific
    storage_bytes   BIGINT,

    -- Cost
    cost_usd        NUMERIC(10, 6),

    -- Metadata
    recorded_at     TIMESTAMPTZ NOT NULL DEFAULT NOW()
) PARTITION BY RANGE (recorded_at);

-- Monthly partitions created automatically
CREATE TABLE usage_records_2026_03 PARTITION OF usage_records
    FOR VALUES FROM ('2026-03-01') TO ('2026-04-01');

-- Aggregated daily summaries (materialized for fast queries)
CREATE TABLE usage_daily (
    tenant_id       UUID NOT NULL,
    agent_id        UUID,
    date            DATE NOT NULL,
    record_type     TEXT NOT NULL,

    total_input_tokens    BIGINT DEFAULT 0,
    total_output_tokens   BIGINT DEFAULT 0,
    total_cost_usd        NUMERIC(12, 6) DEFAULT 0,
    total_cpu_seconds     FLOAT DEFAULT 0,
    total_storage_bytes   BIGINT DEFAULT 0,
    request_count         INTEGER DEFAULT 0,

    PRIMARY KEY (tenant_id, date, record_type, agent_id)
);

-- Tenant billing period summaries
CREATE TABLE billing_periods (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL,
    period_start    DATE NOT NULL,
    period_end      DATE NOT NULL,

    total_cost_usd  NUMERIC(12, 6) DEFAULT 0,
    total_tokens    BIGINT DEFAULT 0,
    total_requests  INTEGER DEFAULT 0,

    status          TEXT DEFAULT 'open',     -- open, closed, invoiced, paid

    UNIQUE(tenant_id, period_start)
);
```

## Tenant Tiers & Quotas

```yaml
tiers:
  free:
    monthlyTokenLimit: 100_000
    maxAgents: 2
    maxConcurrentSandboxes: 1
    storageGB: 1
    modelsAllowed: ["claude-haiku-*", "gpt-4o-mini"]
    rateLimit: 10/min

  pro:
    monthlyTokenLimit: 10_000_000
    maxAgents: 20
    maxConcurrentSandboxes: 5
    storageGB: 50
    modelsAllowed: ["*"]
    rateLimit: 60/min
    monthlyPriceUSD: 49

  enterprise:
    monthlyTokenLimit: unlimited
    maxAgents: unlimited
    maxConcurrentSandboxes: 50
    storageGB: 500
    modelsAllowed: ["*"]
    rateLimit: 300/min
    monthlyPriceUSD: custom
    features:
      - sso
      - audit_log
      - dedicated_namespace
      - priority_support
```

## Quota Enforcement

Enforcement happens at multiple layers:

| Layer | What's enforced | How |
|-------|----------------|-----|
| API Gateway | Request rate limits | Token bucket per tenant |
| LLM Router | Token budget | Pre-check before sending to provider |
| Operator | Max agents, max sandboxes | Admission webhook on CRD creation |
| Storage | PVC size limits | K8s ResourceQuota per tenant namespace |

**When quota is exceeded:**

```go
type QuotaAction string

const (
    QuotaReject    QuotaAction = "reject"     // Return 429
    QuotaDowngrade QuotaAction = "downgrade"  // Use cheaper model
    QuotaAlertOnly QuotaAction = "alert"      // Allow but notify
    QuotaQueue     QuotaAction = "queue"      // Queue for later execution
)
```

## Usage API

Tenants can query their usage via the API:

```
GET /v1/usage/summary?period=2026-03                    # Monthly summary
GET /v1/usage/daily?from=2026-03-01&to=2026-03-27       # Daily breakdown
GET /v1/usage/agents/{agentId}?period=2026-03            # Per-agent usage
GET /v1/usage/breakdown?groupBy=model&period=2026-03     # By model
GET /v1/billing/current                                  # Current billing period
```

## Alerts

CloudEvents emitted for billing events:

- `io.volund.billing.threshold.warning` — 50%, 75% of budget
- `io.volund.billing.threshold.critical` — 90% of budget
- `io.volund.billing.budget.exceeded` — 100% hit
- `io.volund.billing.quota.agent.limit` — max agents reached
- `io.volund.billing.period.closed` — monthly period finalized

Alerts delivered to:
- Dashboard notifications
- Desktop app system notifications
- Configurable webhook (for external alerting)

---

## Implementation Status

*Implemented 2026-03-28.*

### Agent-Side Usage Emission

The agent runtime emits `io.volund.usage.tokens` CloudEvents after each LLM call. Token counts are captured from the gRPC streaming `Complete` message which contains `UsageInfo` (input/output tokens).

```go
// volund-agent/internal/events/emitter.go
type UsageData struct {
    TenantID       string `json:"tenant_id"`
    ConversationID string `json:"conversation_id,omitempty"`
    TaskID         string `json:"task_id,omitempty"`
    InstanceID     string `json:"instance_id,omitempty"`
    Provider       string `json:"provider"`
    Model          string `json:"model"`
    InputTokens    int    `json:"input_tokens"`
    OutputTokens   int    `json:"output_tokens"`
}
```

Zero-token calls are silently skipped to avoid noise.

### Gateway-Side Persistence

The gateway's `InstanceSyncer` subscribes to usage CloudEvents via NATS and persists them to the `usage_events` table:

```sql
CREATE TABLE usage_events (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id),
    conversation_id UUID REFERENCES conversations(id),
    task_id         TEXT,
    instance_id     TEXT,
    provider        TEXT NOT NULL,
    model           TEXT NOT NULL,
    input_tokens    INTEGER NOT NULL DEFAULT 0,
    output_tokens   INTEGER NOT NULL DEFAULT 0,
    recorded_at     TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
```

### Usage API

`GET /v1/usage/summary` returns aggregated token usage for the authenticated user's tenant. The full metering pipeline (compute time, storage, billing periods) from the design doc is deferred to a later phase — the current implementation covers the LLM token tracking end-to-end.
