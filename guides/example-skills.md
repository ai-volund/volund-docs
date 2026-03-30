# Example Skill Definitions

This document shows example Skill CRD manifests for reference. For real skills, see the [volund-skills](https://github.com/ai-volund/volund-skills) repository.

## Prompt Skill

A prompt skill injects instructions into the agent's system prompt. No container or runtime needed.

```yaml
apiVersion: volund.ai/v1alpha1
kind: Skill
metadata:
  name: my-guidelines
  namespace: default
spec:
  type: prompt
  version: "1.0.0"
  description: "Custom agent guidelines and behavior rules"
  author: "your-org"
  tags:
    - guidelines
    - core
  prompt: |
    # My Guidelines

    You are a helpful AI assistant.

    ## Behavior
    - Be concise and direct in your responses
    - When asked about your capabilities, mention that you can use tools and skills
```

## MCP Skill (Sidecar Mode)

An MCP skill runs as a sidecar container alongside the agent pod. Communication happens over stdio.

```yaml
apiVersion: volund.ai/v1alpha1
kind: Skill
metadata:
  name: my-mcp-tool
  namespace: default
spec:
  type: mcp
  version: "1.0.0"
  description: "Example MCP tool for demonstration"
  author: "your-org"
  tags:
    - mcp
    - example
  parameters:
    - name: message
      type: string
      description: "Input parameter for the tool"
      required: true
  runtime:
    image: "ghcr.io/your-org/my-mcp-tool:1.0.0"
    mode: sidecar
    transport: stdio
    resources:
      cpu: "50m"
      memory: "32Mi"
```

## MCP Skill (Shared Mode)

A shared MCP skill runs as a standalone Deployment with a Service. Multiple agents connect via HTTP.

```yaml
apiVersion: volund.ai/v1alpha1
kind: Skill
metadata:
  name: my-shared-tool
  namespace: default
spec:
  type: mcp
  version: "1.0.0"
  description: "Shared MCP tool serving multiple agents"
  author: "your-org"
  tags:
    - mcp
    - shared
  runtime:
    image: "ghcr.io/your-org/my-shared-tool:1.0.0"
    mode: shared
    transport: http-sse
    resources:
      cpu: "200m"
      memory: "256Mi"
```

## CLI Skill

A CLI skill wraps an existing command-line tool with an allowlist of permitted commands.

```yaml
apiVersion: volund.ai/v1alpha1
kind: Skill
metadata:
  name: gh-cli
  namespace: default
spec:
  type: cli
  version: "1.0.0"
  description: "GitHub CLI wrapper for PR and issue management"
  author: "your-org"
  tags:
    - cli
    - github
  cli:
    binary: "gh"
    allowedCommands:
      - "pr view"
      - "pr list"
      - "issue list"
      - "issue view"
```

## MCP Skill with Auth

An MCP skill that requires OAuth2 credentials to access external APIs.

```yaml
apiVersion: volund.ai/v1alpha1
kind: Skill
metadata:
  name: github-integration
  namespace: default
spec:
  type: mcp
  version: "2.0.0"
  description: "GitHub integration with OAuth2 authentication"
  author: "your-org"
  runtime:
    image: "ghcr.io/your-org/skill-github:2.0.0"
    mode: sidecar
    transport: stdio
    auth:
      type: oauth2
      provider: github
      scopes:
        - repo
        - read:org
```
