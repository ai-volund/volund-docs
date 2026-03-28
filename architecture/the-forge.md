# The Forge — Skill & Agent Marketplace

## Overview

The Forge is VOLUND's marketplace for skills, agent profiles, and templates. It enables a community-driven ecosystem where users can publish and discover reusable agent capabilities.

![The Forge — Skill Lifecycle](diagrams/the-forge-lifecycle.svg)

> [Edit in Excalidraw](https://excalidraw.com/#json=La5q1AVLG0YjlBDKzw8n4,iD59MvX89eRYEB-NVaMCdA) | [Source JSON](diagrams/the-forge-lifecycle.excalidraw.json)

## What Can Be Published

| Type | Description | Example |
|------|-------------|---------|
| **Skills/Tools** | Reusable agent capabilities with typed I/O | `github-pr-review`, `web-search`, `csv-analyzer` |
| **Agent Profiles** | Pre-built specialist agent definitions | `code-reviewer`, `data-analyst`, `sysadmin` |
| **Templates** | Sandbox templates, workflow templates | `python-data-sandbox`, `node-dev-sandbox` |

## Skill Format

```
my-skill/
├── SKILL.yaml          # Structured manifest
├── SKILL.md            # Natural language instructions for the agent
├── handler.go          # Optional: compiled tool (Go → WASM)
├── handler.py          # Optional: script-based tool
├── schema.json         # Input/output JSON schema
└── test/               # Required for publishing
    └── test_skill.yaml
```

### SKILL.yaml

```yaml
apiVersion: forge.volund.io/v1
kind: Skill
metadata:
  name: github-pr-review
  version: 1.2.0
  author: ai-volund
  license: MIT
  category: development
  tags: [github, code-review, git]

spec:
  description: "Reviews GitHub pull requests and posts inline comments"
  instructions: SKILL.md

  runtime:
    type: sandbox
    image: volund/skill-github:1.2.0

  input:
    schema: schema.json
    example:
      repo: "ai-volund/volund"
      prNumber: 42

  output:
    schema: schema.json

  permissions:
    network:
      - "api.github.com"
    secrets:
      - GITHUB_TOKEN
    filesystem: read-only

  requires:
    skills: []
    binaries: [git]
```

## Forge CLI

```bash
forge init my-skill --type sandbox     # scaffold a new skill
forge dev my-skill/                    # local dev with hot-reload
forge test my-skill/                   # run test cases
forge publish my-skill/ --version 1.0.0  # publish to The Forge
forge install github-pr-review         # install a skill
forge install --agent data-analyst     # install an agent profile
forge search "code review"             # search the marketplace
```

## Security Model

| Threat | Mitigation |
|--------|-----------|
| Malicious skills | Mandatory sandbox execution for untrusted skills |
| Env var exfiltration | Explicit permission declarations, user approval required |
| Supply chain attacks | Signed packages, org verification badges |
| Unparseable manifests | Structured SKILL.yaml with schema validation on publish |
| Untested skills | Test cases required for Forge publishing |
| Malware | Automated scanning at publish time |

## Runtime Discovery

Agents can discover and suggest skills at runtime:

1. Agent encounters a task it lacks tools for
2. Queries Forge API with semantic search
3. Presents matching skills to user with trust signals (stars, verified author, permissions needed)
4. User approves installation
5. Skill is installed, sandbox provisioned, skill available immediately

## Architecture

The Forge API is part of the `volund` control plane. The `volund-forge` repo contains:
- `forge` CLI binary
- Skill development SDK
- Skill testing framework
- Template scaffolding
