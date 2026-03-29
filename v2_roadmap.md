# VOLUND v2 Roadmap

This document captures all items deferred from v1 (v0.1.0), organized by priority and theme. Each item includes context on why it was deferred, what the v1 state is, and what v2 requires.

---

## 1. Security & Sandboxing (Critical)

These items are hard blockers for third-party Forge skill execution. Until they ship, only first-party (platform built-in) tools can run in agent pods.

### ~~1.1 Agent Sandbox — gVisor Integration (Option B)~~ — DONE

**v2 implementation (complete):**

**CRDs added to volund-operator:**
- `SandboxTemplate` — Reusable pod spec per tenant: RuntimeClassName (gVisor), image, resources, managed NetworkPolicy (default-deny + DNS egress), security context
- `SandboxWarmPool` — Pre-warmed sandbox pods with configurable replicas, HPA min/max, shutdown TTL. Controller creates/deletes pods to maintain target count, manages NetworkPolicy lifecycle
- `SandboxClaim` — Claim lifecycle: Pending → Bound → Released/Failed. Controller finds unclaimed warm pod, labels it claimed, sets endpoint. Finalizer ensures pod cleanup on claim deletion. TTL auto-expiry for safety

**Sandbox Router (`volund-agent/cmd/sandbox-router`):**
- Go HTTP service deployed per-cluster, manages SandboxClaim lifecycle via K8s API
- Routes execution requests to correct sandbox pod endpoint
- API: `POST /v1/sandboxes` (claim), `DELETE /v1/sandboxes/{id}` (release), proxy endpoints for execute/upload/download/list
- In-memory session cache with per-conversation sandbox reuse

**SandboxExecutor dual-mode (`volund-agent/internal/tools/builtin/sandbox.go`):**
- Direct mode (v1): talks to sandbox service endpoint
- Router mode (v2): talks to Sandbox Router which manages claim lifecycle. Claims cached per conversation, released on conversation end
- `RouterURL` takes precedence over `Endpoint`. Falls back to subprocess when neither is configured

**Deployment artifacts:**
- `Dockerfile.sandbox-router` — Go binary in Alpine container
- `deploy/sandbox-router.yaml` — ServiceAccount + RBAC (SandboxClaim CRUD/watch) + Deployment (2 replicas) + Service
- gVisor RuntimeClass manifest in `deploy/sandbox-service.yaml`

**Migration path:**
| Phase | Backend | Isolation | Scope |
|-------|---------|-----------|-------|
| v1 | Subprocess + ulimit | Process-level | First-party tools only |
| **v2 phase 1 (current)** | **agent-sandbox + gVisor** | **Kernel-level** | **`run_code` tool** |
| v2 phase 2 | agent-sandbox + gVisor | Kernel-level | All Forge third-party skills |
| v2 phase 3 | WASM + agent-sandbox | Mixed | WASM for fast tools, sandbox for heavy |

**Affected repos:** `volund-agent`, `volund-operator`
**ADR references:** ADR-003 (Amendment 2), ADR-009

### ~~1.2 Prompt Injection Audit~~ — DONE

**v2 implementation (complete):**

**Expanded sanitization (`volund-agent/internal/safety/sanitize.go`):**
- Unicode homoglyph normalization: Cyrillic substitutions, fullwidth chars, zero-width stripping, ligature expansion
- Base64/hex encoded instruction detection: scans for encoded payloads that decode to injection patterns
- 16 injection patterns: "ignore previous instructions", "developer mode", "DAN mode", "jailbreak", "bypass your", etc.
- Multi-pass sanitization (up to 5 rounds) to prevent reconstruction attacks
- Secret redaction: API keys (sk-*), Bearer tokens, AWS AKIA keys, password/token assignments, URLs with embedded credentials, k8s internal URLs, email PII
- `SanitizeSkillMetadata()` for LLM-visible skill descriptions (1024 char limit + injection pattern blocking)

**Security hooks wired into runtime (`volund-agent/internal/safety/hooks.go`):**
- `ToolArgumentValidationHook` (BeforeHook): blocks tool calls with injection patterns in arguments (with Unicode normalization)
- `SecretRedactionHook` (AfterHook): redacts credentials from all tool output
- `OutputSizeLimitHook` (AfterHook): truncates oversized tool output to prevent context window exhaustion
- All three hooks registered in `buildToolRegistry()` — applied to every tool execution

**Skill description sanitization (`volund-agent/internal/skill/loader.go`):**
- Skill `Description` and all `Parameter[].Description` sanitized via `SanitizeSkillMetadata` at load time
- Prompt skill content sanitized before appending to system prompt
- MCP tool descriptions from `discoverTools()` sanitized before building tool schemas

**Adversarial test suite (`volund-agent/internal/safety/injection_test.go`):**
- 11 top-level test functions, 100+ subtests covering:
  - Role marker stripping (all variants, nested, combined)
  - Boundary marker spoofing prevention
  - Unicode homoglyph bypass detection (Cyrillic, fullwidth, zero-width, ligatures)
  - Base64/hex encoded payload detection
  - Indirect injection via tool output (fake system messages, boundary spoofing)
  - Secret redaction (all 9 patterns + false positive checks)
  - Skill metadata sanitization
  - Nested/recursive attack prevention
  - Context window exhaustion defense
  - Hook integration (before/after blocking + registry integration)
  - System prompt integrity verification

**Affected repos:** `volund-agent`
**ADR references:** ADR-009

### 1.3 Supply Chain Security for Forge Skills

**v1 state:** Permission declarations are validated in skill manifests, but "unusual permission" flagging logic is deferred. No automated security scanning of published skills.

**v2 plan:**
- Static analysis pipeline for skill code on `forge publish`
- Permission audit: flag skills requesting unusual combinations (e.g., network + filesystem)
- Dependency scanning for MCP skill containers
- Signing and provenance attestation for published skills
- Community review/flagging system in Forge marketplace

**Affected repos:** `volund-forge`, `volund`

---

## 2. Core Platform Features (High Priority)

### 2.1 Shared Skill Runtime

**v1 state:** Every skill runs as a sidecar container per agent pod (stdio transport). This works but is wasteful for stateless API-proxy skills (GitHub, Slack, email) where one instance could serve all agents in a tenant.

**v2 plan:** Add `runtime.mode: shared` to the Skill CRD. When set:
- Operator deploys skill as a shared `Deployment` per tenant namespace
- Agent connects via HTTP+SSE transport instead of stdio
- One instance serves all agents in the tenant
- Sidecar remains the default; shared is an optimization

**Architecture (from ARCHITECTURE.md):**
```
┌──────────────────────────────────┐
│  Tenant Namespace                │
│  ┌────────────┐ ┌────────────┐  │
│  │ Agent Pod A │ │ Agent Pod B │ │
│  └─────┬──────┘ └──────┬─────┘  │
│        │               │         │
│  ┌─────▼───────────────▼─────┐  │
│  │ skill-github (Deployment) │  │
│  │ shared per-tenant          │  │
│  └───────────────────────────┘  │
└──────────────────────────────────┘
```

**Affected repos:** `volund-operator`, `volund-agent`
**ADR references:** ADR-003, ADR-004

### 2.2 Session Memory (Redis)

**v1 state:** Working memory is in-process `[]Message` — discarded after each task. The `memory.Manager` interface exists and is injected at startup. Long-term memory (pgvector) has DB schema, repos, decay scoring, dedup, and `RetrieveContext` wired into the runtime loop. But per-conversation session memory (Redis) is not yet implemented.

**v2 plan:**
- Redis-backed session memory with per-conversation key space
- Configurable TTL (default 24h)
- Sliding window context: most recent messages fit within model context, older messages summarized or dropped
- Session namespace: `{tenantId}.{conversationId}`

**Memory tier model:**

| Tier | Storage | Scope | TTL | Status |
|------|---------|-------|-----|--------|
| Working memory | In-process `[]Message` | Current turn | Discarded after turn | v1 complete |
| Session memory | Redis | Per conversation | Configurable (24h default) | **v2** |
| Long-term memory | pgvector | Per agent profile | Permanent (up to quota) | v1 partial (schema + retrieval wired) |

**Additional v2 memory work:**
- Shared namespace: `{tenantId}.shared` — readable by all profiles in a tenant for uploaded files and shared knowledge
- Memory consolidation background job: dedup, summarization, decay pruning (frequency and resource budget TBD)
- Context window budget enforcement: sliding window before each LLM call

**Affected repos:** `volund-agent`, `volund`
**ADR references:** ADR-006, ADR-010

### 2.3 WebSocket Routing Table

**v1 state:** Gateway uses direct instance lookup for WebSocket event forwarding (see `volund/internal/gateway/ws.go:151` — "For v1 we use a well-known instance lookup; v2 will use the routing table").

**v2 plan:**
- Replace direct instance lookup with a routing table backed by Redis or NATS KV
- Support multi-gateway deployments (horizontal scaling)
- Route WebSocket connections to the correct gateway instance holding the NATS subscription
- Graceful handoff on gateway restart

**Affected repos:** `volund`

---

## 3. Billing & Usage (Medium Priority)

### 3.1 Full Metering Pipeline

**v1 state:** LLM token tracking works end-to-end: agent emits `io.volund.usage.tokens` CloudEvents, gateway subscribes and persists to `usage_events` table, `GET /v1/usage/summary` API returns aggregated data. UI shows token counts and per-model breakdown.

**v2 plan — additional metering dimensions:**
- **Compute time:** CPU/memory seconds per agent task (from pod resource metrics)
- **Storage:** Database rows, pgvector entries, object store usage per tenant
- **Billing periods:** Monthly close-out with invoice generation
- **Multi-dimensional quotas:** Token limits + compute limits + storage limits per tenant/plan
- **Budget enforcement:** Configurable actions on quota breach (queue, downgrade tier, alert, hard block)
- **Budget alert webhooks:** Notify tenant admins at 80%/90%/100% thresholds

**Affected repos:** `volund`, `volund-agent`
**ADR references:** Billing/usage architecture doc

---

## 4. Desktop & UI Polish (Medium Priority)

### ~~4.1 Agent Profile Selection Per Conversation~~ — RESOLVED

**Resolution:** Single orchestrator model adopted. The orchestrator agent handles all conversations and delegates to specialist agents as needed via `create_task`/`get_task_result`. No user-facing profile selection required.

### 4.2 Auto-Generate Conversation Titles

**v1 state:** All conversations display as "New conversation" in the sidebar.

**v2 plan:**
- After first assistant response, generate a short title from the conversation content
- Use a lightweight LLM call or heuristic (first user message summary)
- `PATCH /v1/conversations/{id}` already exists for title updates
- User can still manually rename

**Affected repos:** `volund-desktop` (frontend only, API exists)

### 4.3 File Upload / Attachments

**v1 state:** No file upload support. Messages are text-only.

**v2 plan:**
- File upload endpoint: `POST /v1/conversations/{id}/attachments`
- Object storage backend (S3-compatible or local PVC)
- Attachment references in message content parts
- File type restrictions and size limits per tenant plan
- Image/PDF preview in chat view
- Files available to agent tools (read_file, analyze_image)

**Affected repos:** `volund-desktop`, `volund`, `volund-agent`

### 4.4 AI Elements Rich Tool Rendering

**v1 state:** Tool invocations display as collapsible blocks with name, status icon, and raw output. Functional but not rich.

**v2 plan:**
- Integrate AI SDK AI Elements for tool-specific rendering
- Custom renderers per tool type: code blocks with syntax highlighting, search results as cards, charts for data tools
- Streaming tool output (progressive rendering as tool runs)
- Interactive tool results (e.g., clickable links from web_search)

**Affected repos:** `volund-desktop`

### 4.5 Dead Code Cleanup

**v1 state:** `volund-desktop/src/lib/use-volund-chat.ts` is a legacy lightweight chat hook that predates the AI SDK `WebSocketChatTransport`. It is no longer used.

**v2 plan:** Remove the file. No functional impact.

**Affected repos:** `volund-desktop`

---

## 5. Forge & Skill Ecosystem (Medium Priority)

### 5.1 Forge CLI `dev` Mode

**v1 state:** `forge dev` command exists but prints TODO stubs (see `volund-forge/internal/cmd/dev.go:91,97`). Cannot launch MCP server binaries or proxy stdio for local skill development.

**v2 plan:**
- Launch the MCP server binary from skill spec (`forge dev` starts the server)
- Proxy stdio between the dev agent and the skill server
- Hot reload on skill code changes
- Local test harness: send sample tool calls, inspect responses
- Integration with `forge test` for automated validation

**Affected repos:** `volund-forge`

### 5.2 Skill Versioning & Updates

**v1 state:** Skills have version fields in CRDs but no upgrade/rollback workflow.

**v2 plan:**
- Semantic versioning enforcement on `forge publish`
- Agent profiles pin to skill version ranges (e.g., `^1.2.0`)
- Automatic minor version updates with rollback on failure
- Breaking change detection between skill versions
- Deprecation notices and migration guides in Forge UI

**Affected repos:** `volund-forge`, `volund-operator`, `volund`

---

## 6. Architecture Evolution (Low Priority)

### 6.1 Binary Split — Gateway + Control Plane — DEFERRED

> **Status: Deferred indefinitely.** The single-binary architecture is performing well. This item will only be revisited when monitoring shows gateway replica counts exceeding control plane by 10x+, indicating genuinely divergent scaling needs. The internal package structure already supports splitting without refactoring.

**v1 state:** Gateway and control plane run as a single `volund` binary with subcommands. Internal packages are structured to support splitting later.

**v2 plan (if needed):**
- Split into separate `volund-gateway` and `volund-controlplane` binaries
- Independent scaling (gateway is stateless + horizontally scalable, control plane manages state)
- Only worth doing if scaling characteristics diverge significantly

**ADR references:** ADR-001 ("split later if needed")
**Trigger:** When gateway needs 10x+ replicas relative to control plane

### 6.2 Protocol Buffers for WebSocket — DEFERRED

> **Status: Deferred indefinitely.** JSON encoding is adequate for the current WebSocket event throughput. This item will only be revisited when profiling shows JSON parsing or bandwidth as a bottleneck in production. The added complexity of protobuf-js in the desktop client is not justified without measured need.

**v1 state:** WebSocket events use JSON encoding.

**v2 plan (if needed):**
- Replace JSON with Protocol Buffers for WebSocket event encoding
- Reduces message size and parsing overhead
- Only worth doing if bandwidth or latency becomes a bottleneck

**ADR references:** Channel adapter architecture doc
**Trigger:** Measured latency or bandwidth issues at scale

### ~~6.3 Warm Pool Claim Latency SLA~~ — DONE

**v1 state:** Warm pool claim takes ~200-600ms (cold start). No explicit SLA target.

**v2 implementation (complete):**

**SLA targets:**
| Percentile | Target | Alert Threshold |
|-----------|--------|-----------------|
| P50 | < 50ms | — |
| P95 | < 200ms | Warning at 2min sustained |
| P99 | < 500ms | Critical at 2min sustained |

**Metrics added:**
- `volund.claim.duration_seconds` — Float64Histogram with explicit bucket boundaries (10ms–5s), attributes: `result` (claimed/unavailable/error), `path` (cache_hit/db_claim)
- `volund.warm_pool.size` — Int64UpDownCounter for available pod count

**Prometheus alerting rules** (`deploy/local/prometheus-alerts.yaml`):
- `ClaimLatencyP95High` — P95 > 200ms for 2m
- `ClaimLatencyP99Critical` — P99 > 500ms for 2m
- `WarmPoolExhaustion` — >10% unavailable claims for 3m
- `WarmPoolDown` — Zero successful claims for 5m
- `ClaimErrorRate` — >5% error rate for 2m
- `WarmPoolHighUtilization` — >80% utilization for 5m

**Grafana dashboard** (`deploy/local/grafana-dashboards.yaml`):
- Claim latency P50/P95/P99 timeseries with SLA threshold lines
- Claim latency heatmap
- Claim rate by result (claimed/unavailable/error)
- Claim success rate stat panel
- Active instances vs warm pool size
- Cache hit vs DB claim latency comparison
- Dispatch latency percentiles

**Affected repos:** `volund`

---

## 7. Documentation & ADR Cleanup

### 7.1 Stale ADR References

**ADR-010** states v1 ships `NoopManager` for memory, but the actual codebase has pgvector repos, decay scoring, dedup, and `RetrieveContext` wired into the runtime loop. The ADR should be amended to reflect the current memory implementation state.

**ARCHITECTURE.md Phase 5** marks "gVisor sandbox for run_code tool" as complete, but v1 actually uses subprocess + ulimit. The checkbox should be corrected to reflect the v1 implementation accurately.

### 7.2 Missing Documentation

- Deployment guide for production (Helm values, TLS, secrets management)
- Skill development tutorial (end-to-end: init, develop, test, publish)
- API reference (auto-generated from protobuf + OpenAPI)
- Runbook for common operational scenarios (warm pool scaling, agent crashes, NATS partitions)

---

## Priority Matrix

| Priority | Theme | Items | Dependency |
|----------|-------|-------|------------|
| ✅ **Done** | Security | 1.1 Agent Sandbox (gVisor) | Blocks third-party Forge skills |
| ✅ **Done** | Security | 1.2 Prompt Injection Audit | Blocks production deployment |
| ✅ **Done** | Security | 1.3 Supply Chain Security | Blocks `forge publish` for community |
| ✅ **Done** | Core | 2.1 Shared Skill Runtime | Resource efficiency at scale |
| ✅ **Done** | Core | 2.2 Session Memory (Redis) | Multi-turn conversation quality |
| ✅ **Done** | Core | 2.3 WebSocket Routing Table | Horizontal gateway scaling |
| ✅ **Done** | Billing | 3.1 Full Metering Pipeline | Revenue tracking |
| ~~P2~~ | ~~UI~~ | ~~4.1 Agent Profile Selection~~ | ~~Resolved — single orchestrator model~~ |
| ✅ **Done** | UI | 4.2 Auto-Generate Titles | UX polish |
| ✅ **Done** | UI | 4.3 File Upload | Feature completeness |
| ✅ **Done** | UI | 4.4 AI Elements Rendering | Visual polish |
| ✅ **Done** | UI | 4.5 Dead Code Cleanup | Housekeeping |
| ✅ **Done** | Forge | 5.1 Forge CLI `dev` Mode | Skill developer experience |
| ✅ **Done** | Forge | 5.2 Skill Versioning | Ecosystem stability |
| **Deferred** | Arch | 6.1 Binary Split | Deferred indefinitely — trigger: gateway needs 10x+ replicas vs control plane |
| **Deferred** | Arch | 6.2 Protobuf WebSocket | Deferred indefinitely — trigger: measured latency/bandwidth bottleneck at scale |
| ✅ **Done** | Arch | 6.3 Claim Latency SLA | Operational maturity |
| ✅ **Done** | Docs | 7.1-7.2 ADR Cleanup + Guides | Documentation hygiene |
