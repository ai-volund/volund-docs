# VOLUND v3 Roadmap — First-Party Content & Platform Cleanup

This document captures the v3 work: creating first-party skills and agent repos, cleaning up test/sample content from the platform, and establishing the architecture for admin-installed vs user-created agents.

---

## Architecture Overview

| Concept | Where it lives | How it gets into the platform |
|---|---|---|
| **Default orchestrator** | `volund/deploy/local/skills.json` | Built into the app, always present |
| **First-party skills** | `volund-skills` repo | Forge publish pipeline on merge |
| **First-party agents** | `volund-agents` repo | Forge publish pipeline on merge |
| **Builtin tools** (run-code, web-search, memory) | Compiled into agent binary | Always available, listed in Forge UI for discoverability |
| **System agents** | Installed by admin via API/Forge | Visible to all tenant users |
| **Custom agents** | Created by user via UI | Visible only to that user |

---

## 1. Create `volund-skills` Repository

**Goal:** First-party maintained skills that publish into the Forge marketplace.

### Structure

```
volund-skills/
├── README.md
├── .github/
│   └── workflows/
│       └── publish.yaml          # CI: validate + forge publish on merge
├── skills/
│   ├── <skill-name>/
│   │   ├── skill.yaml            # Skill CRD definition
│   │   ├── README.md             # Usage docs, examples
│   │   └── Dockerfile            # Optional, for MCP/containerized skills
│   ├── <skill-name>/
│   │   ├── skill.yaml
│   │   └── README.md
│   └── ...
```

### Directory-per-skill pattern

Each skill is a self-contained directory with:
- `skill.yaml` — The Skill CRD manifest (type, version, description, runtime config, parameters)
- `README.md` — Human-readable docs: what it does, usage examples, configuration
- `Dockerfile` — Only for MCP-type skills that need a container image

### Forge publish pipeline

On merge to `main`, CI runs:
1. Validate all `skill.yaml` files against the CRD schema
2. Build container images for MCP skills (if Dockerfile present)
3. `volund-forge publish` for each skill to push into the platform registry

### Skills to create

**TBD** — Will be determined in discussion before implementation. Skills should be useful, real capabilities (not test/mock).

---

## 2. Create `volund-agents` Repository

**Goal:** First-party specialist agent profiles that the orchestrator can delegate to.

### Structure

```
volund-agents/
├── README.md
├── .github/
│   └── workflows/
│       └── publish.yaml          # CI: validate + publish on merge
├── agents/
│   ├── <agent-name>/
│   │   ├── agent.yaml            # AgentProfile CRD definition
│   │   └── README.md             # What this agent does, skills it uses
│   ├── <agent-name>/
│   │   ├── agent.yaml
│   │   └── README.md
│   └── ...
```

### Agent definition

Each `agent.yaml` is an AgentProfile CRD with:
- `displayName`, `description` — What users see in the Forge
- `profileType: specialist` — All first-party agents are specialists (orchestrator is built-in)
- `systemPrompt` — The agent's persona and instructions
- `skills` — List of skills this agent uses (references skills from `volund-skills` or builtins)
- `model` — LLM configuration

### Agents to create

**TBD** — Will be determined in discussion before implementation.

---

## 3. Agent Visibility Model

### Two-tier system

**System agents** — Installed by a tenant admin or seeded from `volund-agents` via Forge.
- `visibility: system` on the AgentProfile
- Visible to all users in the tenant
- The default orchestrator can delegate to these
- Examples: code-review agent, DevOps agent, security agent

**Custom agents** — Created by individual users through the UI.
- `visibility: user` on the AgentProfile
- `ownerID` set to the creating user's ID (from JWT)
- Only visible to and usable by that specific user
- Users compose them from any available skills (system + their own)

### API design

| Endpoint | Who | What |
|---|---|---|
| `POST /v1/admin/agents` | Admin | Create system agent (visible to all) |
| `POST /v1/agents` | User | Create custom agent (auto-sets ownerID) |
| `GET /v1/agents` | User | Returns system agents + current user's custom agents |
| `PUT /v1/agents/{id}` | User/Admin | Update (user can only edit own custom agents) |
| `DELETE /v1/agents/{id}` | User/Admin | Delete (user can only delete own custom agents) |

### Data model changes

AgentProfile needs two new fields:
- `visibility` — `system` | `user` (default: `system` for admin-created, `user` for user-created)
- `ownerID` — Empty for system agents, user ID for custom agents

### Orchestrator delegation

When resolving which specialist to delegate to:
1. Check system agents first (shared, deterministic)
2. Then check current user's custom agents (personal extensions)

---

## 4. Platform Cleanup

### Remove test/sample content

| File | Action |
|---|---|
| `volund/deploy/local/sample-skills.yaml` | Delete — contains echo-test and volund-guidelines test skills |
| `volund/deploy/local/sample-profile.yaml` | Delete — sample orchestrator profile (real one is in skills.json) |

### Move to documentation

| Source | Destination | Purpose |
|---|---|---|
| `volund-guidelines` prompt content | `volund-docs/guides/example-skills.md` | Example of how to write a prompt skill |
| `sample-profile.yaml` | `volund-docs/guides/example-profiles.md` | Example of how to write an agent profile |
| `sample-skills.yaml` | `volund-docs/guides/example-skills.md` | Example of skill CRD definitions |

### Keep as-is (app config, not samples)

| File | Reason |
|---|---|
| `volund/deploy/local/skills.json` | App config — defines default orchestrator. Remove echo-test and volund-guidelines references. |
| `volund/deploy/local/builtin-skills.yaml` | Forge registry stubs for built-in tools. Stay in platform. |

### Update skills.json

Remove test skill references from the default orchestrator config:
- Remove `echo-test` skill entry
- Remove `volund-guidelines` skill entry (will be replaced by real skills from `volund-skills` repo)
- Keep the default orchestrator system prompt and model config

---

## 5. Forge Publish Pipeline (CI)

### For both `volund-skills` and `volund-agents`:

```yaml
# .github/workflows/publish.yaml
on:
  push:
    branches: [main]

jobs:
  publish:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Validate manifests
        run: # validate all skill.yaml / agent.yaml against CRD schema
      - name: Build images (skills only)
        run: # docker build for any skill with a Dockerfile
      - name: Publish to Forge
        run: # volund-forge publish for each skill/agent
```

---

## Execution Order

1. **Create `volund-skills` repo** — scaffold, README, CI workflow, empty skills/ directory
2. **Create `volund-agents` repo** — scaffold, README, CI workflow, empty agents/ directory
3. **Platform cleanup** — remove sample files, move content to docs, update skills.json
4. **Add visibility model** — AgentProfile schema changes, API endpoints, delegation logic
5. **Create first-party skills** — *TBD in discussion*
6. **Create first-party agents** — *TBD in discussion*

Items 1-3 can proceed immediately. Item 4 is a platform code change. Items 5-6 require design discussion on what skills/agents to build.

---

## Verification

- [ ] `volund-skills` repo exists with proper structure and CI
- [ ] `volund-agents` repo exists with proper structure and CI
- [ ] `sample-skills.yaml` and `sample-profile.yaml` removed from `volund/deploy/local/`
- [ ] Example content moved to `volund-docs/guides/`
- [ ] `skills.json` cleaned of test skill references
- [ ] AgentProfile supports `visibility` and `ownerID` fields
- [ ] `GET /v1/agents` returns merged system + user agents
- [ ] First-party skills created and publishable
- [ ] First-party agents created and publishable
