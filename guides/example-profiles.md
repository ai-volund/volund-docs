# Example Agent Profile Definitions

This document shows example AgentProfile CRD manifests for reference. For real agent profiles, see the [volund-agents](https://github.com/ai-volund/volund-agents) repository.

## Specialist Agent

A specialist agent handles a specific domain. The orchestrator delegates tasks to specialists based on their description and skills.

```yaml
apiVersion: volund.ai/v1alpha1
kind: AgentProfile
metadata:
  name: code-review-agent
  namespace: default
spec:
  displayName: "Code Review Agent"
  description: "Reviews code changes for quality, security, and best practices"
  profileType: specialist
  systemPrompt: |
    You are an expert code reviewer. When given code or a pull request:
    - Check for bugs, security issues, and performance problems
    - Suggest improvements following language-specific best practices
    - Be constructive and explain the reasoning behind suggestions
    - Prioritize issues by severity (critical, warning, suggestion)
  model:
    provider: openai
    name: gpt-oss-120b
    temperature: 0.3
    maxTokens: 4096
  skills:
    - run-code
    - web-search
  maxToolRounds: 15
```

## Orchestrator Agent

The default orchestrator is built into the platform via `skills.json`. This example shows the CRD format for reference only.

```yaml
apiVersion: volund.ai/v1alpha1
kind: AgentProfile
metadata:
  name: example-orchestrator
  namespace: default
spec:
  displayName: "Example Orchestrator"
  description: "General-purpose orchestrator with delegation"
  profileType: orchestrator
  systemPrompt: "You are a helpful AI assistant. Be concise and direct."
  model:
    provider: openai
    name: gpt-oss-120b
    temperature: 0.7
    maxTokens: 4096
  skills:
    - web-search
    - read-memory
    - write-memory
  delegation:
    canDelegate: true
    maxConcurrent: 3
  maxToolRounds: 25
```

## Visibility Model

Agent profiles support two visibility levels:

- **`system`** — Installed by a tenant admin. Visible to all users in the tenant.
- **`user`** — Created by an individual user. Only visible to that user.

System agents are created via `POST /v1/admin/agents` or installed from the Forge. Custom agents are created via `POST /v1/agents` (ownerID is auto-set from the user's JWT).

When the orchestrator needs to delegate a task, it checks system agents first (shared, deterministic), then the current user's custom agents.
