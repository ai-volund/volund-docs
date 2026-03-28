# ADR-003: kubernetes-sigs/agent-sandbox for Tool Execution

**Status:** Accepted
**Date:** 2026-03-27

## Context

AI agents need to execute untrusted code and tools (user-provided scripts, LLM-generated code, third-party skills). This requires strong isolation to prevent tenant escape, data exfiltration, and resource abuse.

## Decision

Use [kubernetes-sigs/agent-sandbox](https://github.com/kubernetes-sigs/agent-sandbox) (v0.2.1, Apache 2.0) as the sandbox primitive for tool execution.

agent-sandbox provides:
- `Sandbox` CRD — isolated, stateful, single-pod workloads with stable identity
- `SandboxTemplate` — reusable templates with managed NetworkPolicies (default-deny)
- `SandboxWarmPool` — pre-warmed pods for fast allocation (critical for latency)
- Runtime API: `/execute`, `/upload`, `/download` endpoints inside each sandbox
- Supports gVisor and Kata Containers for runtime isolation
- Under kubernetes-sigs governance (SIG Apps)

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
- Python SDK available now, Go SDK on their roadmap — may need to contribute or wrap
