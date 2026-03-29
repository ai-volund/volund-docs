# VOLUND - Multi-User Agent Platform Architecture Plan

## Overview

VOLUND is a multi-tenant, general-purpose AI agent platform running on Kubernetes. Users can deploy, manage, and interact with AI agents that connect to multiple LLM providers and use extensible tool/skill systems. Built in Go with a hybrid K8s operator model.

---

## High-Level Architecture

![Platform Overview](diagrams/platform-overview.svg)

> [Edit in Excalidraw](https://excalidraw.com/#json=S_kARHcS93bRfzEb0FrCU,QzJq5FtdtI5ngkL-5TH9EA) | [Source JSON](diagrams/platform-overview.excalidraw.json)

```
┌─────────────────────────────────────────────────────────────────┐
│                        INGRESS (Traefik/Nginx)                  │
│                   TLS termination, rate limiting                │
└──────────┬──────────────────┬───────────────────┬───────────────┘
           │                  │                   │
    ┌──────▼──────┐   ┌──────▼──────┐    ┌───────▼───────┐
    │  Web UI /   │   │  REST/gRPC  │    │  WebSocket    │
    │  Dashboard  │   │  API Gateway│    │  Gateway      │
    │  (SPA)      │   │             │    │  (real-time)  │
    └─────────────┘   └──────┬──────┘    └───────┬───────┘
                             │                   │
                      ┌──────▼───────────────────▼───────┐
                      │         CONTROL PLANE             │
                      │  ┌───────────┐ ┌───────────────┐  │
                      │  │ Auth/RBAC │ │ Tenant Mgr    │  │
                      │  └───────────┘ └───────────────┘  │
                      │  ┌───────────┐ ┌───────────────┐  │
                      │  │ Agent Mgr │ │ Billing/Usage │  │
                      │  └───────────┘ └───────────────┘  │
                      └──────────┬───────────────────────┘
                                 │
                      ┌──────────▼───────────────────────┐
                      │      K8S OPERATOR (HYBRID)        │
                      │  Watches CRDs, reconciles state   │
                      │  AgentInstance | AgentWorkspace    │
                      └──────────┬───────────────────────┘
                                 │
           ┌─────────────────────┼─────────────────────┐
           │                     │                     │
    ┌──────▼──────┐    ┌────────▼────────┐    ┌───────▼───────┐
    │ Agent Pod   │    │  Agent Pod      │    │  Agent Pod    │
    │ (tenant-a)  │    │  (tenant-b)     │    │  (tenant-c)   │
    │ ┌─────────┐ │    │  ┌───────────┐  │    │               │
    │ │ Runtime │ │    │  │  Runtime   │  │    │  ...          │
    │ │ Sandbox │ │    │  │  Sandbox   │  │    │               │
    │ └─────────┘ │    │  └───────────┘  │    └───────────────┘
    └─────────────┘    └─────────────────┘
           │                     │
    ┌──────▼─────────────────────▼─────────────────────┐
    │              DATA PLANE                           │
    │  ┌──────────┐  ┌──────────┐  ┌────────────────┐  │
    │  │ PostgreSQL│  │  Redis   │  │  Object Store  │  │
    │  │ (per-     │  │ (sessions│  │  (S3/Minio -   │  │
    │  │  tenant)  │  │  pubsub) │  │   artifacts)   │  │
    │  └──────────┘  └──────────┘  └────────────────┘  │
    │  ┌──────────────────────────────────────────────┐ │
    │  │  NATS / Kafka  (event bus)                   │ │
    │  └──────────────────────────────────────────────┘ │
    └──────────────────────────────────────────────────┘
```

---

## Component Breakdown

### 1. API Gateway & Auth (`cmd/gateway`)
- **Go service** using gRPC + REST (grpc-gateway)
- JWT-based auth with refresh tokens + OIDC SSO (Google, GitHub, Okta via `go-oidc/v3`)
- Multi-tenant RBAC: Org → Team → User → Agent
- Rate limiting per tenant/user
- API versioning (v1)
- OpenTelemetry instrumentation (`otelhttp` middleware, trace-ID in structured logs, Prometheus metrics)

### 2. Control Plane (`cmd/controlplane`)
- **Tenant Manager**: org/team/user CRUD, onboarding
- **Agent Manager**: CRUD for agent definitions, handles lifecycle requests
- **LLM Router**: proxy layer that routes to configured providers (Anthropic, OpenAI, local, etc.) with key management per tenant
- **Billing/Usage**: tracks token usage, API calls, compute time per tenant
- **Skill Registry**: catalog of available tools/skills agents can use

### 3. Kubernetes Operator (`cmd/operator`)
- **Custom Resources**:
  - `AgentInstance` — represents a running agent (desired state: running/stopped/suspended)
  - `AgentWorkspace` — PVC + config for an agent's persistent state
  - `TenantNamespace` — namespace + network policies + resource quotas per tenant
- **Reconciliation loops**:
  - Agent lifecycle: create/update/delete agent pods
  - Workspace provisioning: PVCs, secrets, configmaps
  - Resource enforcement: quotas, limits
- **Standard K8s resources** for platform infra (control plane, databases, etc.) managed via Helm

### 4. Agent Runtime (`volund-agent`)
- **Generic, identity-less binary** that assumes identity at startup by loading an AgentProfile
- Follows a **serverless execution model** — pods are claimed from warm pools and released when idle
- Becomes an orchestrator or specialist based on injected profile
- **Components**:
  - **LLM Client**: connects to the LLM Router via gRPC, supports streaming
  - **Tool Executor**: sandboxed execution via subprocess + ulimit resource limits (v1); kubernetes-sigs/agent-sandbox with gVisor planned for v2
  - **Memory Manager**: working (in-process) + session (Redis) + long-term (PostgreSQL + pgvector)
  - **Inbox Processor**: drains queued messages/task results on spin-up
  - **Chat Writer**: posts messages directly to conversation threads via gateway
- Communicates status back to control plane via CloudEvents over NATS
- **Pod lifecycle**: Active → Warm Idle → Cold (released to pool)
- See [Agent Runtime Lifecycle](agent-runtime-lifecycle.md) for full details

### 5. Data Layer
- **PostgreSQL** (via CloudNativePG operator): tenant databases, agent configs, conversation history
- **Redis**: session state, caching, pub/sub for real-time updates
- **NATS**: event bus for agent ↔ control plane communication, inter-agent messaging (CloudEvents format)
- **MinIO/S3**: artifact storage (uploaded files, generated outputs)
- **Vector DB** (pgvector extension): agent memory embeddings

### 6. Web Dashboard & Desktop App
- **Web UI**: React SPA with five views: Chat, Tasks, Agents, Forge, Settings
- **Desktop App**: Tauri 2.x (Rust backend + shared React frontend), system tray, native notifications
- **Auth**: Login/register with email/password + OIDC SSO (Google, GitHub, Okta). JWT persisted in localStorage with auto-refresh.
- **Error handling**: Global ErrorBoundary, WebSocket reconnection with 5s timeout
- Both connect via WebSocket gateway for real-time streaming
- See [Channel Adapters](channel-adapters.md) for full details

### 7. Testing Infrastructure
- **Unit tests**: Full coverage for gateway (credentials, claim manager, circuit breaker) and agent (runtime, skills, tools, memory, orchestrator)
- **Integration tests**: `testcontainers-go` spins up `pgvector/pgvector:pg16` + `nats:2-alpine`, runs all 8 migrations, exercises the full API flow (register → login → me → tenants → agents → conversations → usage → forge)
- **Frontend**: TypeScript type checking + production build verification

---

## Tenant Isolation Model

```
Namespace: volund-tenant-{orgId}
├── NetworkPolicy: deny-all + allow control-plane + allow egress to LLM router
├── ResourceQuota: cpu/memory/storage limits per tenant tier
├── LimitRange: per-pod defaults
├── ServiceAccount: tenant-scoped, no cluster access
├── Secrets: LLM API keys (encrypted at rest)
├── AgentWarmPool: pre-started generic pods (orchestrator + specialist pools)
└── Agent Pods: claimed from warm pool on demand
    ├── Profile injected at claim time (orchestrator or specialist)
    ├── State restored from Redis/Postgres
    ├── Sidecar: log collector + metrics exporter
    └── SecurityContext: non-root, read-only rootfs, no privilege escalation
```

---

## LLM Provider Architecture

```go
// Provider interface — all LLM providers implement this
type LLMProvider interface {
    Chat(ctx context.Context, req *ChatRequest) (*ChatResponse, error)
    StreamChat(ctx context.Context, req *ChatRequest) (ChatStream, error)
    ListModels(ctx context.Context) ([]Model, error)
    Embeddings(ctx context.Context, req *EmbeddingRequest) (*EmbeddingResponse, error)
}

// Router selects provider based on tenant config + model requested
type LLMRouter struct {
    providers map[string]LLMProvider  // "anthropic", "openai", "ollama", etc.
    tenantCfg TenantConfigStore
}
```

- Tenant-level provider config: which providers enabled, API keys (encrypted), default model
- Request-level model selection: agent config specifies preferred model
- Fallback chains: if primary provider fails, try secondary
- Token counting + cost tracking per request

---

## Repository & Binary Strategy

Seven repos, split by trust boundary and deployment pattern:

| Repo | Purpose |
|------|---------|
| [ai-volund/volund](https://github.com/ai-volund/volund) | Platform core — API gateway + control plane + Forge API. Subcommands: `volund gateway`, `volund controlplane`, `volund serve --all` |
| [ai-volund/volund-operator](https://github.com/ai-volund/volund-operator) | K8s operator — manages CRDs (AgentInstance, AgentWorkspace, AgentWarmPool), runs with leader election |
| [ai-volund/volund-agent](https://github.com/ai-volund/volund-agent) | Generic agent runtime — becomes orchestrator or specialist via profile injection. Runs in warm pool pods |
| [ai-volund/volund-proto](https://github.com/ai-volund/volund-proto) | Shared protobuf definitions, CloudEvent types, schemas. Managed with buf |
| [ai-volund/volund-forge](https://github.com/ai-volund/volund-forge) | Forge CLI + SDK — skill and agent profile development toolkit |
| [ai-volund/volund-desktop](https://github.com/ai-volund/volund-desktop) | Desktop app — Tauri 2.x with shared React frontend |
| [ai-volund/volund-docs](https://github.com/ai-volund/volund-docs) | Documentation, architecture decisions, and guides |

### volund (platform core)
```
volund/
├── cmd/
│   └── volund/            # Single binary, subcommands for gateway/controlplane
├── internal/
│   ├── gateway/           # API gateway, routing, middleware
│   ├── auth/              # JWT, RBAC, OIDC
│   ├── tenant/            # Tenant management
│   ├── agent/             # Agent lifecycle, definitions
│   ├── llm/               # LLM provider interface + router
│   ├── skill/             # Tool/skill registry
│   ├── events/            # CloudEvents over NATS publishing/subscribing
│   ├── billing/           # Usage tracking, metering
│   └── storage/           # Database repos, object store clients
├── api/
│   ├── proto/             # Protobuf definitions (gRPC)
│   └── openapi/           # OpenAPI specs (REST)
├── deploy/
│   └── helm/volund/       # Platform Helm chart
├── pkg/
│   ├── sdk/               # Go SDK for building skills/tools
│   └── client/            # Generated API client
├── web/                   # Dashboard SPA
├── Makefile
├── go.mod
└── Dockerfile
```

### volund-operator
```
volund-operator/
├── cmd/
│   └── operator/          # Operator binary
├── internal/
│   ├── controller/        # Reconciliation loops
│   └── webhook/           # Admission webhooks
├── api/
│   └── v1alpha1/          # CRD Go types (AgentInstance, AgentWorkspace, TenantNamespace)
├── config/
│   ├── crd/               # Generated CRD manifests
│   ├── rbac/              # Operator RBAC
│   └── manager/           # Operator deployment
├── deploy/
│   └── helm/volund-operator/
├── Makefile
├── go.mod
└── Dockerfile
```

### volund-agent
```
volund-agent/
├── cmd/
│   └── agent/             # Agent runtime binary
├── internal/
│   ├── runtime/           # Core agent loop, state machine
│   ├── llm/               # LLM client (connects to platform LLM Router)
│   ├── tool/              # Tool executor, sandbox
│   ├── memory/            # Short-term + long-term memory manager
│   ├── channel/           # Messaging channel adapters
│   └── events/            # CloudEvents emitter (status, heartbeats)
├── Makefile
├── go.mod
└── Dockerfile
```

---

## Custom Resource Definitions

### AgentInstance CRD
```yaml
apiVersion: volund.io/v1alpha1
kind: AgentInstance
metadata:
  name: my-agent
  namespace: volund-tenant-acme
spec:
  displayName: "Acme Support Bot"
  runtime:
    image: volund/agent-runtime:latest
    resources:
      requests: { cpu: "100m", memory: "256Mi" }
      limits:   { cpu: "500m", memory: "512Mi" }
  llm:
    provider: anthropic
    model: claude-sonnet-4-6
    fallback:
      provider: openai
      model: gpt-4o
  skills:
    - name: web-search
    - name: code-executor
    - name: slack-channel
      config:
        channelId: "C1234567"
  memory:
    shortTerm: redis
    longTerm:
      type: postgresql
      embeddingModel: text-embedding-3-small
  state: running
status:
  phase: Running
  lastHeartbeat: "2026-03-27T10:00:00Z"
  tokenUsage:
    today: 45230
    month: 1203400
```

### AgentWorkspace CRD
```yaml
apiVersion: volund.io/v1alpha1
kind: AgentWorkspace
metadata:
  name: my-agent-workspace
  namespace: volund-tenant-acme
spec:
  agentRef: my-agent
  storage:
    size: 1Gi
    storageClass: fast-ssd
  config:
    systemPrompt: |
      You are a helpful support agent for Acme Corp...
    personality: professional
    maxConversationLength: 100
```

---

## Implementation Phases

### Phase 1: Foundation
- [ ] Project scaffolding: Go modules, Makefiles, CI/CD (GitHub Actions → GHCR)
- [x] Protobuf definitions for core APIs (auth, tenant, agent, chat, llm, task, forge, usage)
- [ ] PostgreSQL schema + migrations
- [ ] Auth service: JWT + OIDC, email-based org invites with onboarding flow
- [ ] Tenant CRUD + namespace provisioning
- [ ] Basic API gateway with auth middleware
- [ ] Local dev setup: Kind + Tilt

### Phase 2: Agent Runtime
- [ ] Generic agent runtime binary with profile injection
- [ ] LLM provider interface + Anthropic implementation
- [ ] Agent main loop + state machine
- [ ] Tool/skill execution framework (WASM + agent-sandbox integration)
- [ ] Memory manager: working (in-process) + session (Redis) + long-term (Postgres + pgvector)
- [ ] WebSocket gateway for real-time streaming
- [ ] Multi-agent chat: specialists posting to conversation threads

### Phase 3: K8s Operator + Serverless Scaling
- [ ] CRD definitions (AgentInstance, AgentWorkspace, AgentWarmPool)
- [ ] Operator scaffolding (kubebuilder)
- [ ] Warm pool management: orchestrator pool + specialist pool
- [ ] Pod claim/release protocol
- [ ] Orchestrator inbox (Redis queue, drain on spin-up)
- [ ] Session affinity routing table in control plane
- [ ] Network policies + resource quotas per tenant

### Phase 4: Multi-Provider, Skills & Forge
- [ ] OpenAI provider implementation
- [ ] Ollama/local model provider
- [ ] LLM Router with fallback + circuit breakers
- [ ] The Forge: skill registry, search, install, publish
- [ ] Forge CLI (`forge init`, `forge dev`, `forge test`, `forge publish`)
- [ ] Built-in skills (web search, code exec, file manager)

### Phase 5: UI & Desktop
- [ ] Web UI: Chat view (multi-agent conversation thread)
- [ ] Web UI: Tasks view (async job dashboard)
- [ ] Web UI: Agent management, Forge marketplace browser
- [ ] Desktop app (Tauri 2.x): shared React frontend, system tray, notifications
- [ ] Agent Builder: create specialist profiles via chat or UI wizard
- [ ] Billing/usage dashboards

### Phase 6: Observability & Hardening
- [ ] Wide events (canonical log lines): one structured event per request per service
- [ ] OpenTelemetry tracing across multi-agent flows
- [ ] Sensitive data access audit trail
- [ ] Budget enforcement + alerts
- [ ] Helm chart for full platform deployment
- [ ] Security hardening: tenant escape testing, sandbox escape scenarios

---

## Key Technical Decisions

| Decision | Choice | Rationale |
|----------|--------|-----------|
| Language | Go | K8s-native, excellent for operators, high performance |
| API | gRPC + REST (grpc-gateway) | Type-safe internal comms, REST for external |
| Database | PostgreSQL + pgvector | Mature, handles relational + vector search |
| Event bus | NATS + CloudEvents | Lightweight, Go-native, CNCF standard event envelope |
| Cache/sessions | Redis | Standard, fast, pub/sub for real-time |
| Operator framework | kubebuilder | Official K8s operator SDK for Go |
| Object storage | MinIO (S3-compatible) | Self-hosted, K8s-native, or swap for cloud S3 |
| Auth | JWT + OIDC | Stateless, supports SSO integration |
| Tool sandboxing | Subprocess + ulimit (v1), kubernetes-sigs/agent-sandbox (v2) | v1: process-level isolation for first-party tools. v2: CRD-based gVisor pods for third-party skill execution |
| Agent execution | Serverless warm pools | Generic pods claimed on demand, released when idle |
| Desktop app | Tauri 2.x | Lightweight, Rust backend, shared React frontend |
| Observability | Wide events + OpenTelemetry | One structured event per request, distributed tracing |
| Local dev | Kind + Tilt | K8s-native local development |

---

## Security Considerations

- **Tenant isolation**: Network policies prevent cross-tenant traffic
- **Secret management**: LLM API keys encrypted at rest, injected as K8s secrets
- **Agent sandboxing**: Non-root containers, read-only rootfs, seccomp profiles
- **Tool execution**: Code execution in subprocess with ulimit resource limits (v1); ephemeral gVisor sandbox pods planned for v2 third-party skills
- **RBAC**: Fine-grained permissions (org admin, team admin, user, viewer)
- **Audit logging**: All API calls and agent actions logged with tenant context
