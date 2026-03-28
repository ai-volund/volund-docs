# ADR-009: Multi-Layer Security Model

**Status:** Accepted
**Date:** 2026-03-27

## Context

VOLUND runs untrusted code (user tools, Forge skills, LLM-generated actions) in a multi-tenant environment. A security breach in one tenant must not affect others. The Forge marketplace introduces supply chain risk.

## Decision

**Four-layer security model:**

1. **Tenant isolation**: Dedicated K8s namespace per tenant with default-deny network policies, resource quotas, and tenant-scoped service accounts. Warm pool pods are created directly in tenant namespaces (not migrated).

2. **Agent sandboxing**: Agent pods run as non-root with read-only rootfs. Tool execution uses `kubernetes-sigs/agent-sandbox` with gVisor runtime class. Sandboxes have TTLs and declared network/filesystem permissions.

3. **Data protection**: Secrets mounted as files (not env vars). Encryption at rest and in transit (mTLS between services). Conversation content never in logs/traces by default. GDPR export/deletion support.

4. **Supply chain (Forge)**: Four trust levels (built-in → official → verified community → unverified). Mandatory sandbox for unverified skills. Publish pipeline: schema validation → static analysis → malware scan → test execution → signing.

**Auth**: JWT + OIDC, email-based org invites with onboarding flow. RBAC with 5 roles (platform admin → viewer). Service-to-service via mTLS.

## Consequences

- Strong tenant isolation with minimal overhead (namespace-per-tenant is a proven K8s pattern)
- Forge security is stricter than ClawHub (learned from their 341 malicious skills incident)
- File-based secret mounting is more secure than env vars but requires volume mounts
- gVisor adds ~5-10% overhead on sandboxed workloads but provides strong syscall isolation
- OIDC support enables SSO for enterprise tenants
