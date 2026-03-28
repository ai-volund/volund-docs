# ADR-003: kubernetes-sigs/agent-sandbox for Tool Execution

**Status:** Amended
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

---

## Amendment (2026-03): v1 Uses Subprocess, Not agent-sandbox

The original decision assumed `agent-sandbox` from day one. After reviewing the dependency complexity and the fact that `kubernetes-sigs/agent-sandbox` is still v0.2.1 with no stable Go SDK, we are phasing the sandboxing approach:

**v1 (current):** The `run_code` built-in tool executes code as a subprocess with:
- Configurable timeout (default 10s, max 60s)
- Working directory scoped to a temp dir per tool call
- Stdout/stderr captured and returned to the agent
- No network access from the subprocess (env var `NO_NETWORK=true` for cooperative tools)

This is intentionally pragmatic. It ships fast and covers the primary use case (running scripts in the agent's own pod).

**v2 (planned):** Replace subprocess execution with `agent-sandbox` + gVisor runtime class for full kernel-level isolation. The `run_code` tool interface does not change — only the execution backend changes.

The three-tier model (WASM / Sandbox / Sidecar) from the original ADR remains the target architecture for v2.

**Implication for ADR-009:** The security model's "Agent sandboxing" layer currently relies on pod-level isolation (non-root, read-only rootfs) rather than a dedicated sandbox CRD. This is documented as a known gap until v2.
