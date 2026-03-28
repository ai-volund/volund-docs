# Security Architecture

## Overview

VOLUND's security model addresses four threat domains: **tenant isolation** (preventing cross-tenant access), **agent sandboxing** (containing untrusted code/tools), **data protection** (secrets, conversations, PII), and **supply chain** (malicious skills from The Forge).

## Threat Model

| Threat | Attack Surface | Impact | Mitigation |
|--------|---------------|--------|------------|
| Tenant escape | Network, storage, K8s API | Access another tenant's data | Namespace isolation, network policies, RBAC |
| Sandbox escape | Tool execution pods | Host compromise, data exfil | gVisor/Kata runtime, agent-sandbox network policies |
| LLM prompt injection | Agent conversations | Unauthorized actions, data leak | Tool permission model, user approval for actions |
| Malicious skill | The Forge marketplace | Code execution, secret theft | Mandatory sandbox, permission declarations, scanning |
| API key theft | Config, env vars, memory | Unauthorized LLM usage, billing | K8s secrets, no env var injection, encrypted at rest |
| Conversation data leak | Database, Redis, logs | PII exposure | Encryption at rest/transit, audit logging, RBAC |
| Privilege escalation | Agent runtime, K8s | Platform compromise | Non-root, read-only rootfs, minimal RBAC |

## Tenant Isolation

Each tenant gets a dedicated Kubernetes namespace with strict boundaries.

### Namespace Layout

```
volund-system/              # Platform components (control plane, operator)
volund-pools/               # Warm pool pods (generic, no tenant affinity)
volund-tenant-{orgId}/      # Per-tenant namespace
  ├── NetworkPolicy
  ├── ResourceQuota
  ├── LimitRange
  ├── ServiceAccount
  ├── Secrets
  └── Agent pods (claimed from pool, running in tenant namespace)
```

### Network Policies

```yaml
# Default: deny all ingress and egress
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny
  namespace: volund-tenant-acme
spec:
  podSelector: {}
  policyTypes:
    - Ingress
    - Egress

---
# Allow: agent pods → control plane (for LLM router, chat API)
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-control-plane
  namespace: volund-tenant-acme
spec:
  podSelector:
    matchLabels:
      app: volund-agent
  policyTypes:
    - Egress
  egress:
    - to:
        - namespaceSelector:
            matchLabels:
              name: volund-system
      ports:
        - port: 8443   # gRPC (LLM router, chat API)

---
# Allow: agent pods → NATS (for CloudEvents)
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-nats
  namespace: volund-tenant-acme
spec:
  podSelector:
    matchLabels:
      app: volund-agent
  policyTypes:
    - Egress
  egress:
    - to:
        - namespaceSelector:
            matchLabels:
              name: volund-system
      ports:
        - port: 4222   # NATS

---
# Deny: agent pods cannot reach other tenant namespaces
# Deny: agent pods cannot reach K8s API server
# Deny: agent pods cannot reach cloud metadata (169.254.169.254)
```

### Pod Claim Namespace Migration

When a pod is claimed from the warm pool for a tenant:

1. Pod starts in `volund-pools` namespace (generic, no tenant data)
2. Control plane assigns tenant → pod gets tenant-scoped service account + secrets
3. Pod network policies enforce tenant boundaries

**Note:** Actual K8s pod namespace migration isn't possible — the operator creates the pod directly in the tenant namespace from the warm pool template. The "pool" is a set of pre-warmed pod templates, not running pods that move between namespaces.

Revised flow:
1. Warm pool maintains **ready-to-create** pod specs (images pulled, templates cached)
2. On claim: operator creates pod in `volund-tenant-{orgId}` namespace from template
3. Pod starts with tenant service account, secrets, and network policies already in place
4. Fast startup because image is already cached on the node

### Resource Quotas

```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: tenant-quota
  namespace: volund-tenant-acme
spec:
  hard:
    pods: "20"                    # Max concurrent pods (agents + sandboxes)
    requests.cpu: "10"
    requests.memory: "20Gi"
    limits.cpu: "20"
    limits.memory: "40Gi"
    persistentvolumeclaims: "10"
    requests.storage: "50Gi"
```

Quotas configured per tenant tier (free/pro/enterprise).

## Agent Sandboxing

### Runtime Security

Every agent pod runs with strict security constraints:

```yaml
securityContext:
  runAsNonRoot: true
  runAsUser: 65534                 # nobody
  readOnlyRootFilesystem: true
  allowPrivilegeEscalation: false
  capabilities:
    drop: ["ALL"]
  seccompProfile:
    type: RuntimeDefault
```

### Tool Execution Sandboxing

**v1 (implemented 2026-03-28) — subprocess isolation:**
- **Process groups**: `Setpgid = true` isolates child processes
- **Resource limits**: ulimit wrappers on Linux — CPU time (60s), virtual memory (256MB), file size (64MB), file descriptors (64), processes (32)
- **Restricted environment**: only `PATH`, `HOME`, `LANG`, `TMPDIR` passed to subprocesses
- **Platform-aware**: build tags (`rlimit_linux.go` / `rlimit_other.go`) handle Linux vs other platforms
- **Sandbox service**: optional HTTP endpoint (`VOLUND_SANDBOX_ENDPOINT`) for delegating execution to an isolated service

**v2 (planned) — `kubernetes-sigs/agent-sandbox`:**
- **gVisor runtime class** for compute isolation (syscall filtering)
- **Network policies**: default-deny, only allow explicitly declared hosts
- **Filesystem**: isolated `/app` directory, no access to host filesystem
- **Time-limited**: sandboxes have a TTL, auto-destroyed after expiry
- **Resource-limited**: CPU/memory limits per sandbox

### LLM Prompt Injection Defense

Agents can be manipulated via prompt injection in user messages or tool outputs. Mitigations:

1. **Tool permission model**: Skills declare what they can access (network hosts, secrets, filesystem). Users approve permissions on install.
2. **Action confirmation**: For high-risk actions (file deletion, API calls, purchases), the agent must request user approval via the chat thread.
3. **Output sanitization**: Tool outputs are treated as untrusted data — agents are instructed not to execute commands found in tool outputs without user confirmation.
4. **System prompt hardening**: Orchestrator and specialist system prompts include injection defense instructions.
5. **Audit trail**: All tool invocations logged with input/output for post-incident review.

## Data Protection

### Encryption

| Data | At Rest | In Transit |
|------|---------|-----------|
| PostgreSQL | TDE or volume encryption | TLS (mTLS between services) |
| Redis | Encrypted volumes | TLS |
| NATS | Encrypted volumes | TLS |
| MinIO/S3 | Server-side encryption (SSE) | TLS |
| K8s Secrets | etcd encryption at rest | TLS |
| Agent ↔ Control Plane | N/A | mTLS (gRPC) |
| Client ↔ Gateway | N/A | TLS (HTTPS/WSS) |

### Secret Management

LLM API keys and other secrets follow a strict handling chain:

1. User provides API key via dashboard → encrypted in transit (TLS)
2. Control plane encrypts key → stored in K8s Secret (encrypted at rest in etcd)
3. When agent pod claims for this tenant → Secret mounted as volume (not env var)
4. Agent reads key from mounted file → passes to LLM Router via gRPC (mTLS)
5. Key **never** appears in: logs, traces, wide events, error messages, CloudEvents

**Why not env vars?** Environment variables are visible in `/proc/*/environ`, logged by some libraries, and exposed in core dumps. File mounts are more secure and can be made read-only.

### Conversation Data

- Conversation history stored in PostgreSQL with tenant-level row isolation
- Session data in Redis with tenant-prefixed keys
- Long-term memories in PostgreSQL with row-level security
- **GDPR support**: full export and deletion per user via API
- **Retention policies**: configurable per tenant (auto-delete conversations older than N days)

### PII Handling

| Data Type | Storage | Logging | Tracing |
|-----------|---------|---------|---------|
| User email | PostgreSQL (plaintext, needed for auth) | SHA-256 hash | Never |
| Conversation content | PostgreSQL/Redis (plaintext, needed for agent context) | Never | Opt-in only, span events |
| LLM API keys | K8s Secrets (encrypted) | Never | Never |
| Tool inputs/outputs | PostgreSQL (plaintext) | Truncated (100 chars) | Opt-in only |
| IP addresses | Gateway logs | SHA-256 hash | Never |

## Supply Chain Security (The Forge)

### Skill Trust Model

```
┌──────────────────────────────────────────────┐
│                Trust Levels                   │
│                                              │
│  ┌────────────────────────────────────────┐  │
│  │  Built-in Skills                       │  │
│  │  (shipped with volund-agent)           │  │
│  │  Trust: FULL — run in-process          │  │
│  └────────────────────────────────────────┘  │
│                                              │
│  ┌────────────────────────────────────────┐  │
│  │  Official Skills                       │  │
│  │  (published by ai-volund org)          │  │
│  │  Trust: HIGH — run in WASM or sandbox  │  │
│  └────────────────────────────────────────┘  │
│                                              │
│  ┌────────────────────────────────────────┐  │
│  │  Verified Community Skills             │  │
│  │  (verified author, scanned, tested)    │  │
│  │  Trust: MEDIUM — run in sandbox only   │  │
│  └────────────────────────────────────────┘  │
│                                              │
│  ┌────────────────────────────────────────┐  │
│  │  Unverified Community Skills           │  │
│  │  (anyone can publish)                  │  │
│  │  Trust: LOW — sandbox + extra limits   │  │
│  └────────────────────────────────────────┘  │
└──────────────────────────────────────────────┘
```

### Skill Security Pipeline

On publish to The Forge:

1. **Schema validation**: SKILL.yaml must conform to spec, all required fields present
2. **Static analysis**: scan for known malicious patterns (env var exfiltration, network calls to undeclared hosts)
3. **Malware scanning**: automated scan of package contents
4. **Permission review**: declared permissions flagged if unusual (e.g., skill claims to need filesystem write but description says "calculator")
5. **Test execution**: required test cases run in isolated sandbox
6. **Signing**: package signed with author's key, signature verified on install

On install:

1. **Permission prompt**: user shown exactly what the skill can access — must approve
2. **Signature verification**: package hash matches registry
3. **Sandbox enforcement**: unverified skills ALWAYS run in sandbox, no exceptions

### Dependency Pinning

- Agent runtime image tags pinned to specific digests in production
- Skill packages include content hashes for integrity verification
- Helm chart values reference exact image digests, not tags

## Authentication & Authorization

### Auth Flow

```
User → Login (email/password or OIDC) → JWT access token + refresh token
  │
  ├── Access token: short-lived (15 min), stateless validation
  ├── Refresh token: long-lived (7 days), stored server-side, rotated on use
  └── OIDC: supports Google, GitHub, Azure AD, Okta
```

**OIDC Implementation (2026-03-28):**

OIDC SSO is implemented in `volund/internal/auth/oidc.go` using `github.com/coreos/go-oidc/v3`. Configuration holds an array of `OIDCProvider{Name, ClientID, ClientSecret, Issuer}`.

Routes:
- `GET /v1/auth/oidc/providers` — returns configured OIDC providers (public, no auth required)
- `GET /v1/auth/oidc/{provider}` — generates state parameter, redirects to provider authorize URL
- `GET /v1/auth/oidc/{provider}/callback` — exchanges authorization code, verifies ID token, finds or creates user via `FindOrCreateByOIDC(provider, sub, email)`, issues JWT

The `users` table has `oidc_provider` and `oidc_sub` columns (migration `007_oidc_provider`). Users created via OIDC are auto-provisioned with a personal tenant. Existing email/password auth is unchanged — OIDC is additive.

### Org Invite Flow

```
Admin → Invite user (email) → System sends invite email
  │
  ├── Email contains: signed invite link with org ID + role
  ├── User clicks link → onboarding flow:
  │   ├── Create account (if new) or link existing account
  │   ├── Accept org membership
  │   └── Configure personal orchestrator agent
  └── User is now a member of the org with assigned role
```

### RBAC Model

```
Platform Admin
  └── Org Admin
       └── Team Admin
            └── User
                 └── Viewer

Permissions:
  Platform Admin: manage all tenants, system config, official skills
  Org Admin: manage org members, billing, tenant config, all agents
  Team Admin: manage team members, team agents
  User: create/manage own agents, use shared agents, install skills
  Viewer: read-only access to conversations and dashboards
```

### Service-to-Service Auth

Internal services use mTLS with certificates issued by an internal CA (cert-manager):

- Agent Runtime ↔ LLM Router: mTLS
- Agent Runtime ↔ NATS: TLS with JWT auth
- Control Plane ↔ Operator: K8s RBAC (service account)
- Gateway ↔ Control Plane: mTLS

## Incident Response

### Security Events

CloudEvents emitted for security-relevant actions:

- `io.volund.security.auth.failed` — failed login attempt
- `io.volund.security.auth.brute_force` — rate limit hit on auth endpoint
- `io.volund.security.secret.accessed` — API key read from secret store
- `io.volund.security.sandbox.escape_attempt` — blocked syscall or network access in sandbox
- `io.volund.security.skill.malicious` — scan flagged a skill package
- `io.volund.security.rbac.denied` — authorization check failed

### Response Automation

- **Brute force**: auto-block IP after N failed auth attempts (configurable)
- **Sandbox escape**: kill sandbox pod, alert platform admin, quarantine skill
- **Malicious skill detected**: auto-hide from Forge, notify installers, alert platform admin
- **Budget anomaly**: sudden spike in token usage → alert tenant admin, optional auto-throttle
