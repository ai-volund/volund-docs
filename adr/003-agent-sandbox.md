# ADR-003: kubernetes-sigs/agent-sandbox for Tool Execution

**Status:** Amended (v2 plan finalized 2026-03-28)
**Date:** 2026-03-27

## Context

AI agents need to execute untrusted code and tools (user-provided scripts, LLM-generated code, third-party skills). This requires strong isolation to prevent tenant escape, data exfiltration, and resource abuse.

## Decision

Use [kubernetes-sigs/agent-sandbox](https://github.com/kubernetes-sigs/agent-sandbox) (v0.2.x, Apache 2.0) as the sandbox primitive for **tool execution only** (Option B). Agent pods remain managed by our own operator.

agent-sandbox provides:
- `Sandbox` CRD — isolated, stateful, single-pod workloads with stable identity
- `SandboxTemplate` — reusable templates with managed NetworkPolicies (default-deny)
- `SandboxWarmPool` — pre-warmed pods for fast allocation (critical for latency)
- `SandboxClaim` — user-facing abstraction for claiming sandboxes from templates
- Runtime API: `/execute`, `/upload`, `/download`, `/list`, `/exists` endpoints inside each sandbox
- Sandbox Router — FastAPI proxy that routes to sandbox pods via `X-Sandbox-ID` header
- Supports gVisor and Kata Containers for runtime isolation
- Under kubernetes-sigs governance (SIG Apps)
- Go typed client at `clients/k8s/clientset/versioned/`
- Python SDK: `pip install k8s-agent-sandbox`

## Three Execution Tiers

| Tier | Use Case | Mechanism |
|------|----------|-----------|
| In-process (WASM) | Fast, stateless tools (text transform, calculations) | WASM module in agent-runtime process |
| Sandbox | Untrusted/heavy tools (code exec, file processing) | agent-sandbox CRD, isolated pod |
| Sidecar | Long-running services (DB connections, API proxies) | Sidecar container in agent pod |

## Consequences

- Strong isolation for untrusted code without building our own sandbox
- Warm pools minimize latency for code execution
- CRD-based — fits our hybrid operator model
- v1alpha1 API may have breaking changes — we should pin versions
- Go typed client exists; Python SDK is mature

---

## Amendment 1 (2026-03-27): v1 Uses Subprocess

The original decision assumed `agent-sandbox` from day one. After reviewing the dependency complexity and the fact that `kubernetes-sigs/agent-sandbox` is still v0.2.1 with no stable Go SDK, we are phasing the sandboxing approach:

**v1 (current):** The `run_code` built-in tool executes code as a subprocess with:
- Configurable timeout (default 10s, max 60s)
- Working directory scoped to a temp dir per tool call
- Stdout/stderr captured and returned to the agent
- No network access from the subprocess (env var `NO_NETWORK=true` for cooperative tools)

This is intentionally pragmatic. It ships fast and covers the primary use case (running scripts in the agent's own pod).

**Implication for ADR-009:** The security model's "Agent sandboxing" layer currently relies on pod-level isolation (non-root, read-only rootfs) rather than a dedicated sandbox CRD. This is documented as a known gap until v2.

---

## Amendment 2 (2026-03-28): v2 Integration Plan — Option B

### Decision: Tool Execution Sandboxes Only

After reviewing `agent-sandbox` v0.2.1 architecture, CRDs, and SDK, we adopt **Option B**: use agent-sandbox exclusively for tool execution, not for agent pod management.

**Rationale:**

Our agent pods require NATS subscription setup, skill sidecar injection, credential mounting, and custom health probes — all managed by our operator. Replacing our `AgentWarmPool`/`AgentInstance` CRDs with agent-sandbox would require layering all this custom logic on top of their `v1alpha1` API, creating tight coupling with an unstable upstream.

The actual security gap is narrow: only the `run_code` tool executes untrusted code. Agent pods themselves run trusted platform code. Sandboxing tool execution is the minimal, highest-value change.

### Architecture

```
┌─────────────────────────────────────────────────────────┐
│ Tenant Namespace                                        │
│                                                         │
│  ┌──────────────────────┐    ┌───────────────────────┐  │
│  │ Agent Pod (ours)     │    │ Sandbox Router (their)│  │
│  │ ┌──────────────────┐ │    │ FastAPI proxy          │  │
│  │ │ run_code tool     │─┼───▶ X-Sandbox-ID header   │  │
│  │ │ (Go HTTP client) │ │    │ routes to sandbox pod │  │
│  │ └──────────────────┘ │    └───────────┬───────────┘  │
│  │ managed by our       │                │              │
│  │ volund-operator      │                ▼              │
│  └──────────────────────┘    ┌───────────────────────┐  │
│                              │ Sandbox Pod (theirs)  │  │
│                              │ gVisor RuntimeClass   │  │
│                              │ ┌───────────────────┐ │  │
│                              │ │ Runtime API       │ │  │
│                              │ │ POST /execute     │ │  │
│                              │ │ POST /upload      │ │  │
│                              │ │ GET  /download    │ │  │
│                              │ └───────────────────┘ │  │
│                              │ default-deny network  │  │
│                              │ no service account    │  │
│                              └───────────────────────┘  │
│                                                         │
│  ┌───────────────────────┐                              │
│  │ SandboxWarmPool       │                              │
│  │ replicas: 3           │                              │
│  │ HPA-compatible        │                              │
│  │ pre-warmed gVisor pods│                              │
│  └───────────────────────┘                              │
└─────────────────────────────────────────────────────────┘
```

### Separation of Concerns

| Component | Managed By | CRDs |
|-----------|-----------|------|
| Agent pods (orchestrator, specialist) | volund-operator | `AgentWarmPool`, `AgentInstance`, `AgentProfile` |
| Tool execution sandboxes | agent-sandbox controller | `Sandbox`, `SandboxTemplate`, `SandboxClaim`, `SandboxWarmPool` |
| Skill sidecars | volund-operator | `Skill` CRD (sidecar injection) |
| Network policies for agents | volund Helm charts | Helm-templated `NetworkPolicy` |
| Network policies for sandboxes | agent-sandbox controller | `SandboxTemplate.spec.networkPolicy` (auto-managed) |

### SandboxTemplate per Tenant

Each tenant namespace gets a `SandboxTemplate` configured for tool execution:

```yaml
apiVersion: extensions.agents.x-k8s.io/v1alpha1
kind: SandboxTemplate
metadata:
  name: tool-execution
  namespace: tenant-abc
spec:
  podTemplate:
    spec:
      runtimeClassName: gvisor
      securityContext:
        runAsUser: 65534     # nobody
        runAsGroup: 65534
        runAsNonRoot: true
        fsGroup: 65534
      containers:
      - name: runtime
        image: ghcr.io/ai-volund/sandbox-runtime:v2
        ports:
        - containerPort: 8888
          protocol: TCP
        resources:
          requests:
            cpu: 100m
            memory: 128Mi
          limits:
            cpu: "1"
            memory: 512Mi

  # Controller auto-manages NetworkPolicy:
  # - Ingress: only from Sandbox Router
  # - Egress: public internet only (blocks RFC1918, metadata 169.254.169.254, API server)
  networkPolicyManagement: Managed
  networkPolicy:
    egress:
    # DNS only — no internet access for tool execution
    - ports:
      - protocol: UDP
        port: 53
      - protocol: TCP
        port: 53
```

### SandboxWarmPool per Tenant

Pre-warm sandbox pods for low-latency tool execution:

```yaml
apiVersion: extensions.agents.x-k8s.io/v1alpha1
kind: SandboxWarmPool
metadata:
  name: tool-execution-pool
  namespace: tenant-abc
spec:
  replicas: 2
  sandboxTemplateRef:
    name: tool-execution
```

The `SandboxWarmPool` has HPA scale subresource support, so pool sizing can be automated based on tool execution demand.

### run_code Tool Integration

The `run_code` tool in `volund-agent` gets a new execution backend:

```go
// internal/tools/builtin/run_code.go

type SandboxExecutor struct {
    routerURL  string         // e.g. "http://sandbox-router-svc:8080"
    namespace  string
    template   string         // SandboxTemplate name
    k8sClient  sandboxclient.Interface  // agent-sandbox Go typed client
    timeout    time.Duration
}

func (s *SandboxExecutor) Execute(ctx context.Context, req RunCodeRequest) (*RunCodeResult, error) {
    // 1. Create SandboxClaim (or reuse existing for this conversation)
    claim := s.getOrCreateClaim(ctx, req.ConversationID)

    // 2. Wait for sandbox ready
    sandboxName := s.waitForReady(ctx, claim)

    // 3. Upload code file if needed
    s.upload(ctx, sandboxName, "/tmp/code.py", req.Code)

    // 4. Execute via Router
    resp := s.execute(ctx, sandboxName, req.Command)

    // 5. Return results
    return &RunCodeResult{
        Stdout:   resp.Stdout,
        Stderr:   resp.Stderr,
        ExitCode: resp.ExitCode,
    }, nil
}
```

**Sandbox lifecycle per conversation:**
1. First `run_code` call in a conversation → create `SandboxClaim` → claim gets a pod from `SandboxWarmPool` (fast) or creates a new `Sandbox` (slower)
2. Subsequent `run_code` calls reuse the same sandbox (stateful — files persist between calls)
3. Conversation ends → `SandboxClaim` deleted → sandbox pod cleaned up
4. Timeout safety: `SandboxClaim.spec.lifecycle.shutdownTime` set to conversation TTL

**Fallback:** If `VOLUND_SANDBOX_ENDPOINT` is not configured (e.g., local dev without gVisor), fall back to subprocess execution (v1 behavior). The tool interface doesn't change — only the backend.

### Sandbox Runtime Image

We build a minimal runtime image (`ghcr.io/ai-volund/sandbox-runtime`) that exposes the required HTTP API:

**Endpoints:**
- `POST /execute` — `{"command": "python3 /tmp/code.py"}` → `{"stdout": "...", "stderr": "...", "exit_code": 0}`
- `POST /upload` — multipart file upload to sandbox filesystem
- `GET /download/{path}` — download file from sandbox
- `GET /list/{path}` — list directory → `[{"name": "...", "size": 0, "type": "file", "mod_time": 0.0}]`
- `GET /exists/{path}` — `{"exists": true}`

**Pre-installed runtimes:** Python 3.12, Node.js 22, bash. Additional language support via tenant-specific `SandboxTemplate` images.

**Security hardening:**
- Non-root (nobody:65534)
- Read-only rootfs (writable `/tmp` only via emptyDir)
- No service account token (`automountServiceAccountToken: false` — SandboxTemplate default)
- gVisor kernel isolation (syscalls intercepted by gVisor sentry)
- Network: DNS only (no internet, no internal cluster access)
- Resource limits enforced by kubelet + gVisor

### Cluster Setup Requirements

**One-time per cluster:**
1. Install gVisor RuntimeClass: `kubectl apply -f gvisor-runtimeclass.yaml`
2. Install agent-sandbox controller: `kubectl apply -f manifest.yaml`
3. Install agent-sandbox extensions: `kubectl apply -f extensions.yaml`
4. Deploy Sandbox Router: `kubectl apply -f sandbox-router.yaml`

**Per tenant namespace (managed by volund-operator):**
1. Create `SandboxTemplate` with gVisor + network policy
2. Create `SandboxWarmPool` (replica count based on tenant plan)
3. RBAC: agent service account needs `create`/`delete`/`get`/`watch` on `SandboxClaim`

### Migration Path

| Phase | Backend | Isolation | Scope |
|-------|---------|-----------|-------|
| v1 (current) | Subprocess + ulimit | Process-level | First-party tools only |
| v2 phase 1 | agent-sandbox + gVisor | Kernel-level | `run_code` tool |
| v2 phase 2 | agent-sandbox + gVisor | Kernel-level | All Forge third-party skills |
| v2 phase 3 (future) | WASM + agent-sandbox | Mixed | WASM for fast tools, sandbox for heavy |

### Risks and Mitigations

| Risk | Mitigation |
|------|-----------|
| agent-sandbox `v1alpha1` API breaks | Pin version in manifests; wrap client behind interface for easy swap |
| Go client coverage gaps | Generated typed client exists; contribute upstream if gaps found |
| Sandbox startup latency | `SandboxWarmPool` with HPA keeps pre-warmed pods ready |
| gVisor compatibility issues | Some syscalls unsupported; test runtime image against gVisor compatibility list |
| Router single point of failure | Deploy Router as Deployment with replicas + PDB |
| Sandbox pod resource exhaustion | Per-sandbox resource limits in `SandboxTemplate` + tenant `ResourceQuota` |

### What We Do NOT Adopt

- **Agent pod management** — stays with our operator (`AgentWarmPool`, `AgentInstance`)
- **Skill sidecars** — stays with our operator (sidecar injection on agent pods)
- **Session/identity management** — stays with our `ClaimManager`
- **NATS subscription routing** — stays with our gateway dispatch layer
- **Sandbox hibernate/resume** — not needed for ephemeral tool execution
