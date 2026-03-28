# ADR-001: Three-Binary Split

**Status:** Accepted
**Date:** 2026-03-27

## Context

VOLUND has multiple components (API gateway, control plane, K8s operator, agent runtime) that could be packaged as a single binary, individual binaries per component, or something in between.

## Decision

Three binaries, split by trust boundary and deployment pattern:

| Binary | Components | Rationale |
|--------|-----------|-----------|
| `volund` | API gateway + control plane (subcommands) | Same trust boundary, scale together early, split later if needed |
| `volund-operator` | K8s operator | Separate RBAC, leader election, different deployment pattern |
| `volund-agent` | Agent runtime | Runs in tenant pods, different trust boundary, minimal image |

## Consequences

- Fast iteration: one binary for core platform during development (`volund serve --all`)
- Clear separation where it matters: operator and agent runtime are isolated
- Can split `volund` into gateway + controlplane later without restructuring — internal packages support it
- Three repos, three CI pipelines, three container images
