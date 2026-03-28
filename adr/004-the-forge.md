# ADR-004: The Forge — Skill and Agent Marketplace

**Status:** Accepted
**Date:** 2026-03-27

## Context

VOLUND needs an extensible system for adding tools/skills to agents and sharing pre-built specialist agent profiles. Similar platforms (OpenClaw's ClawHub) have proven the marketplace model works but exposed security gaps (341+ malicious skills discovered in early 2026).

## Decision

Build "The Forge" — a marketplace for skills, agent profiles, and templates. Hosted as part of the control plane API, with a separate CLI (`forge`) for development.

### What Can Be Published

1. **Skills/Tools** — reusable agent capabilities with typed inputs/outputs
2. **Agent Profiles** — pre-built specialist agent definitions
3. **Templates** — sandbox templates, workflow templates

### Skill Format

```
my-skill/
├── SKILL.yaml     # Structured manifest (not markdown frontmatter)
├── SKILL.md       # Natural language instructions for the agent
├── handler.go     # Optional compiled tool (WASM) or handler.py (script)
├── schema.json    # Input/output JSON schema
└── test/          # Required for publishing
```

### Security Improvements Over ClawHub

- Mandatory sandbox execution for untrusted skills
- Structured SKILL.yaml with schema validation on publish
- Explicit permission declarations (network, secrets, filesystem)
- Test cases required for publishing
- Signed packages + org verification badges
- VirusTotal-style scanning at publish time

### Runtime Discovery

Agents can query The Forge at runtime, suggest skill installations to users, and hot-reload skills mid-session.

## Consequences

- Users can extend agents without code changes
- Community-driven skill ecosystem
- Forge API lives in `volund` repo (part of control plane)
- `volund-forge` repo contains the CLI + SDK for skill development
- Need to invest in security scanning infrastructure
- Skill format must be stable early — it's a public API
