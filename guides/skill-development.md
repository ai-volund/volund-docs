# Skill Development Guide

This guide walks through building, testing, and publishing skills for VOLUND's agent platform via The Forge marketplace.

---

## Table of Contents

1. [Overview of Skill Types](#overview-of-skill-types)
2. [Creating a Prompt Skill](#creating-a-prompt-skill)
3. [Creating an MCP Skill](#creating-an-mcp-skill)
4. [Creating a CLI Skill](#creating-a-cli-skill)
5. [Local Development Workflow](#local-development-workflow)
6. [Testing](#testing)
7. [Publishing and the Security Audit Pipeline](#publishing-and-the-security-audit-pipeline)
8. [Versioning](#versioning)
9. [Troubleshooting Common Issues](#troubleshooting-common-issues)

---

## Overview of Skill Types

Skills extend agent capabilities on the VOLUND platform. Every skill is defined by a `skill.json` manifest and falls into one of three types:

| Type | What It Is | When to Use |
|--------|-----------------------------------------------|-----------------------------------------------|
| `prompt` | A prompt template exposed as a tool | Simple text transformations, structured output, domain-specific instructions |
| `mcp` | A Model Context Protocol server in a Docker container communicating over JSON-RPC stdio | Complex logic, external API calls, stateful operations |
| `cli` | A CLI binary wrapped via `mcp-cli-adapter` | Existing command-line tools you want to expose to agents |

All three types share the same `skill.json` structure:

```json
{
  "name": "my-skill",
  "version": "1.0.0",
  "description": "What this skill does",
  "author": "username",
  "type": "prompt",
  "tags": ["category"],
  "spec": {}
}
```

The `spec` field varies by skill type and is covered in each section below.

---

## Creating a Prompt Skill

Prompt skills are the simplest type. They wrap a prompt template as a callable tool that agents can invoke.

### Step 1: Scaffold the project

```bash
forge init --type prompt my-summarizer
cd my-summarizer
```

This creates the following structure:

```
my-summarizer/
  skill.json
  prompt.md
  tests/
    test_basic.yaml
```

### Step 2: Edit `skill.json`

```json
{
  "name": "my-summarizer",
  "version": "1.0.0",
  "description": "Summarizes long documents into concise bullet points",
  "author": "your-username",
  "type": "prompt",
  "tags": ["text", "summarization"],
  "spec": {
    "prompt_file": "prompt.md",
    "parameters": [
      {
        "name": "document",
        "type": "string",
        "description": "The text to summarize",
        "required": true
      },
      {
        "name": "max_bullets",
        "type": "integer",
        "description": "Maximum number of bullet points",
        "required": false,
        "default": 5
      }
    ]
  }
}
```

### Step 3: Write the prompt template

Edit `prompt.md`:

```markdown
Summarize the following document into at most {{max_bullets}} concise bullet points.
Each bullet should capture a distinct key idea. Use plain language.

---

{{document}}
```

Template variables use `{{variable_name}}` syntax and map directly to the parameters defined in `skill.json`.

### Step 4: Test locally

```bash
forge dev
```

This starts an interactive session where you can invoke the skill:

```
[forge dev] Skill "my-summarizer" running locally
[forge dev] Type a JSON input or press Ctrl+C to exit

> {"document": "VOLUND is a multi-tenant AI agent platform...", "max_bullets": 3}

[output]
- VOLUND provides multi-tenant AI agent hosting
- Agents are deployed on Kubernetes with serverless warm pools
- The Forge marketplace enables skill sharing across tenants
```

### Step 5: Publish

```bash
forge publish
```

See the [Publishing](#publishing-and-the-security-audit-pipeline) section for details on what happens during publish.

---

## Creating an MCP Skill

MCP skills are Docker containers that implement the Model Context Protocol over JSON-RPC via stdio. Use this type when you need real logic, API calls, or state.

### Step 1: Scaffold the project

```bash
forge init --type mcp my-weather-tool
cd my-weather-tool
```

This creates:

```
my-weather-tool/
  skill.json
  Dockerfile
  src/
    main.go
  tests/
    test_basic.yaml
```

### Step 2: Implement the MCP server

An MCP server communicates over stdin/stdout using JSON-RPC 2.0. The server must handle three core methods:

- `initialize` -- returns server capabilities and metadata
- `tools/list` -- returns the list of tools this skill exposes
- `tools/call` -- executes a tool invocation

Here is a minimal Go implementation in `src/main.go`:

```go
package main

import (
	"bufio"
	"encoding/json"
	"fmt"
	"net/http"
	"os"
)

type JSONRPCRequest struct {
	JSONRPC string          `json:"jsonrpc"`
	ID      interface{}     `json:"id"`
	Method  string          `json:"method"`
	Params  json.RawMessage `json:"params,omitempty"`
}

type JSONRPCResponse struct {
	JSONRPC string      `json:"jsonrpc"`
	ID      interface{} `json:"id"`
	Result  interface{} `json:"result,omitempty"`
	Error   interface{} `json:"error,omitempty"`
}

func main() {
	scanner := bufio.NewScanner(os.Stdin)
	for scanner.Scan() {
		line := scanner.Bytes()
		var req JSONRPCRequest
		if err := json.Unmarshal(line, &req); err != nil {
			continue
		}

		var resp JSONRPCResponse
		resp.JSONRPC = "2.0"
		resp.ID = req.ID

		switch req.Method {
		case "initialize":
			resp.Result = map[string]interface{}{
				"protocolVersion": "2024-11-05",
				"serverInfo": map[string]string{
					"name":    "my-weather-tool",
					"version": "1.0.0",
				},
				"capabilities": map[string]interface{}{
					"tools": map[string]bool{},
				},
			}

		case "tools/list":
			resp.Result = map[string]interface{}{
				"tools": []map[string]interface{}{
					{
						"name":        "get_weather",
						"description": "Get the current weather for a city",
						"inputSchema": map[string]interface{}{
							"type": "object",
							"properties": map[string]interface{}{
								"city": map[string]string{
									"type":        "string",
									"description": "City name",
								},
							},
							"required": []string{"city"},
						},
					},
				},
			}

		case "tools/call":
			var params struct {
				Name      string                 `json:"name"`
				Arguments map[string]interface{} `json:"arguments"`
			}
			json.Unmarshal(req.Params, &params)

			if params.Name == "get_weather" {
				city := params.Arguments["city"].(string)
				weather := fetchWeather(city)
				resp.Result = map[string]interface{}{
					"content": []map[string]interface{}{
						{
							"type": "text",
							"text": weather,
						},
					},
				}
			}

		default:
			resp.Error = map[string]interface{}{
				"code":    -32601,
				"message": "Method not found",
			}
		}

		out, _ := json.Marshal(resp)
		fmt.Println(string(out))
	}
}

func fetchWeather(city string) string {
	resp, err := http.Get(
		fmt.Sprintf("https://wttr.in/%s?format=%%C+%%t", city),
	)
	if err != nil {
		return "Error fetching weather: " + err.Error()
	}
	defer resp.Body.Close()
	buf := make([]byte, 256)
	n, _ := resp.Body.Read(buf)
	return string(buf[:n])
}
```

### Step 3: Write the Dockerfile

```dockerfile
FROM golang:1.22-alpine AS builder
WORKDIR /app
COPY src/ .
RUN go build -o /skill main.go

FROM alpine:3.19
RUN apk add --no-cache ca-certificates
COPY --from=builder /skill /skill
ENTRYPOINT ["/skill"]
```

The container must read JSON-RPC from stdin and write responses to stdout. Do not write logs or debug output to stdout -- use stderr instead.

### Step 4: Configure `skill.json`

```json
{
  "name": "my-weather-tool",
  "version": "1.0.0",
  "description": "Fetches current weather data for any city",
  "author": "your-username",
  "type": "mcp",
  "tags": ["weather", "api"],
  "spec": {
    "image": "my-weather-tool:latest",
    "transport": "stdio",
    "permissions": ["network"],
    "runtime": {
      "mode": "sidecar",
      "resources": {
        "memory": "128Mi",
        "cpu": "100m"
      }
    }
  }
}
```

Key `spec` fields for MCP skills:

| Field | Description |
|-------|-------------|
| `image` | Docker image reference. On publish, The Forge builds and hosts the image. |
| `transport` | Must be `stdio`. |
| `permissions` | Array of permissions the skill requires: `network`, `filesystem`, `env`. |
| `runtime.mode` | `sidecar` (default, one container per agent) or `shared` (single Deployment serving multiple agents). |
| `runtime.resources` | Kubernetes resource requests. |

---

## Creating a CLI Skill

CLI skills wrap an existing command-line binary using `mcp-cli-adapter`. The adapter translates MCP tool calls into CLI invocations.

### Step 1: Scaffold the project

```bash
forge init --type cli my-jq-tool
cd my-jq-tool
```

### Step 2: Configure `skill.json`

```json
{
  "name": "my-jq-tool",
  "version": "1.0.0",
  "description": "Run jq queries against JSON data",
  "author": "your-username",
  "type": "cli",
  "tags": ["json", "data"],
  "spec": {
    "image": "my-jq-tool:latest",
    "binary": "/usr/bin/jq",
    "allowed_commands": [
      {
        "name": "query",
        "description": "Run a jq expression against JSON input",
        "command_template": "echo '{{input}}' | jq '{{expression}}'",
        "parameters": [
          {
            "name": "input",
            "type": "string",
            "description": "JSON input data",
            "required": true
          },
          {
            "name": "expression",
            "type": "string",
            "description": "jq filter expression",
            "required": true
          }
        ]
      }
    ],
    "permissions": ["filesystem"],
    "runtime": {
      "mode": "sidecar"
    }
  }
}
```

The `allowed_commands` array is critical. It defines exactly which commands the adapter will execute. Commands not listed here are rejected. Each command uses `{{param}}` template syntax to inject parameters.

### Step 3: Write the Dockerfile

```dockerfile
FROM alpine:3.19
RUN apk add --no-cache jq
ENTRYPOINT ["mcp-cli-adapter"]
```

The `mcp-cli-adapter` binary is injected into the container at runtime by the VOLUND platform. You do not need to include it in your image.

### Step 4: Test and publish

```bash
forge dev
forge test
forge publish
```

---

## Local Development Workflow

The `forge dev` command is the primary tool for local iteration. It behaves differently depending on skill type:

| Skill Type | What `forge dev` Does |
|------------|----------------------|
| `prompt` | Starts an interactive REPL. You type JSON inputs and see the rendered prompt and simulated output. |
| `mcp` | Builds the Docker image locally, starts the container, and proxies stdio. You interact via JSON-RPC. |
| `cli` | Builds the Docker image, starts the container with `mcp-cli-adapter`, and proxies stdio. |

### Common flags

```bash
# Start with verbose logging (logs appear on stderr)
forge dev --verbose

# Override a specific environment variable
forge dev --env API_KEY=test-key-123

# Use a specific Docker build context
forge dev --build-context ./custom-dir

# Skip Docker image rebuild (use cached image)
forge dev --no-build
```

### Interactive JSON-RPC session (MCP and CLI skills)

When `forge dev` is running for an MCP or CLI skill, you can send raw JSON-RPC requests:

```json
{"jsonrpc":"2.0","id":1,"method":"tools/list"}
```

```json
{"jsonrpc":"2.0","id":2,"method":"tools/call","params":{"name":"get_weather","arguments":{"city":"Seattle"}}}
```

### Hot reload

For prompt skills, `forge dev` watches `prompt.md` and reloads on save. For MCP and CLI skills, modify your source and press `r` in the terminal to trigger a rebuild.

---

## Testing

### Writing tests

Tests live in the `tests/` directory and are written as YAML files:

```yaml
# tests/test_weather.yaml
name: "Weather tool returns data"
steps:
  - call:
      tool: get_weather
      arguments:
        city: "London"
    expect:
      status: success
      content_contains: "London"

  - call:
      tool: get_weather
      arguments:
        city: ""
    expect:
      status: error
```

Each test file contains a `name` and a list of `steps`. Each step issues a `call` and checks the response against `expect` conditions.

### Available assertions

| Assertion | Description |
|-----------|-------------|
| `status: success` | The tool call returned without error |
| `status: error` | The tool call returned an error |
| `content_contains: "text"` | Response content includes the given substring |
| `content_matches: "regex"` | Response content matches the given regex |
| `error_contains: "text"` | Error message includes the given substring |
| `json_path: {path: "$.key", equals: "value"}` | Evaluate a JSONPath expression against the response |

### Running tests

```bash
# Run all tests
forge test

# Run a specific test file
forge test tests/test_weather.yaml

# Run with verbose output
forge test --verbose
```

`forge test` starts the skill in a local container (or REPL for prompt skills), executes each test step sequentially, and reports results:

```
Running: Weather tool returns data
  [PASS] get_weather with city=London
  [PASS] get_weather with empty city returns error
Results: 2 passed, 0 failed
```

---

## Publishing and the Security Audit Pipeline

When you run `forge publish`, your skill goes through a multi-step security audit before it reaches The Forge registry.

### The publish flow

```
forge publish
    |
    v
[1] Validate skill.json schema
    |
    v
[2] Semver version check
    - Version must be valid semver (e.g., 1.0.0, 2.1.3-beta.1)
    - Version must be greater than the latest published version
    |
    v
[3] Prompt injection scan
    - Description and prompt templates scanned for injection patterns
    - Detected patterns: role overrides, instruction leaks, jailbreak phrases
    - Flagged skills require manual review
    |
    v
[4] Permission combination audit
    - Unusual permission combos flagged for review
    - Example: network + filesystem is flagged (potential data exfiltration)
    - Single permissions like network-only pass without flags
    |
    v
[5] Name safety check
    - Name checked for path traversal (../, ..\)
    - Name checked for special characters and reserved words
    - Must match pattern: [a-z0-9][a-z0-9-]{0,62}[a-z0-9]
    |
    v
[6] Docker image build and push (MCP and CLI types)
    - Image built from your Dockerfile
    - Pushed to The Forge container registry
    |
    v
[7] Published to The Forge registry
```

### Handling audit failures

If the audit flags your skill, `forge publish` prints the specific issue:

```
[FAIL] Permission audit: combination [network, filesystem] flagged for review.
       Provide a justification with --justify "reason" or remove one permission.
```

To provide justification:

```bash
forge publish --justify "Needs filesystem to cache API responses locally"
```

Justified skills are published but marked for human review. They remain available while under review.

### First-time publishing

The first time you publish a skill, you must authenticate with The Forge:

```bash
forge login
```

This opens a browser-based OAuth flow and stores your credentials locally.

---

## Versioning

The Forge enforces strict semver for all published skills.

### Rules

- Every published version must be valid semver: `MAJOR.MINOR.PATCH` with optional pre-release suffix (e.g., `1.0.0-beta.1`).
- Each new publish must have a version strictly greater than the latest published version.
- You cannot overwrite or delete a published version.

### When to bump what

| Change Type | Version Bump | Example |
|-------------|-------------|---------|
| Breaking change to tool parameters or behavior | MAJOR | `1.2.3` to `2.0.0` |
| New tool added, backward-compatible | MINOR | `1.2.3` to `1.3.0` |
| Bug fix, no API change | PATCH | `1.2.3` to `1.2.4` |

### Compatibility ranges in agent configs

When agents declare skill dependencies, they use caret ranges to allow compatible updates:

```json
{
  "skills": {
    "my-weather-tool": "^1.2.0"
  }
}
```

The caret (`^`) allows any version that is backward-compatible with `1.2.0`:

| Range | Matches | Does Not Match |
|-------|---------|----------------|
| `^1.2.0` | `1.2.0`, `1.2.5`, `1.3.0`, `1.9.9` | `2.0.0`, `1.1.0` |
| `^0.2.0` | `0.2.0`, `0.2.9` | `0.3.0`, `1.0.0` |
| `^0.0.3` | `0.0.3` | `0.0.4`, `0.1.0` |

Note that `0.x` versions are treated as unstable -- the caret range is more restrictive for pre-1.0 versions, as shown above.

### Pre-release versions

Pre-release versions are supported and sort before the release:

```
1.0.0-alpha.1 < 1.0.0-beta.1 < 1.0.0-rc.1 < 1.0.0
```

Agents using `^1.0.0` will not match pre-release versions. To use a pre-release, the agent must explicitly specify it.

---

## Troubleshooting Common Issues

### `forge dev` fails to start

**Docker not running.** MCP and CLI skills require Docker. Make sure the Docker daemon is running:

```bash
docker info
```

**Port conflict.** If `forge dev` reports a port conflict, kill the existing process or use a different port:

```bash
forge dev --port 9090
```

### `forge publish` fails with "version must be greater"

You are trying to publish a version that already exists or is lower than the current latest. Bump the version in `skill.json`:

```bash
# Check the latest published version
forge info my-skill

# Update skill.json version, then retry
forge publish
```

### Prompt injection scan false positive

If the security scanner flags your description or prompt as containing injection patterns, you have two options:

1. Rephrase the flagged content. Avoid phrases like "ignore previous instructions" or "you are now" in your prompt templates.
2. Use the `--justify` flag with an explanation.

### Permission combination flagged

Common flagged combinations and how to address them:

| Flagged Combo | Why | Resolution |
|---------------|-----|------------|
| `network` + `filesystem` | Potential data exfiltration | Justify the need or remove one |
| `network` + `env` | Could leak secrets via network | Justify or use a secrets manager |

### Container crashes on startup

Check the container logs:

```bash
forge dev --verbose
```

Common causes:

- **Binary not found.** Ensure the `ENTRYPOINT` in your Dockerfile points to the correct path.
- **Writing to stdout.** Only JSON-RPC responses should go to stdout. All logs, debug output, and errors must go to stderr.
- **Missing CA certificates.** If your skill makes HTTPS calls, include `ca-certificates` in your image:

  ```dockerfile
  RUN apk add --no-cache ca-certificates
  ```

### Tests pass locally but fail in CI

- **Network access.** CI environments may not have external network access. Mock external APIs in your test fixtures.
- **Docker image mismatch.** Run `forge dev --no-cache` to force a clean image build before testing.

### Skill works locally but not after publish

- **Environment variables.** Local `--env` flags are not carried to production. Declare required environment variables in `skill.json`:

  ```json
  {
    "spec": {
      "env": [
        {
          "name": "API_KEY",
          "description": "API key for the weather service",
          "required": true
        }
      ]
    }
  }
  ```

  Agents configure these values when they install the skill.

- **Resource limits.** The default sidecar resource limits may be too low. Increase them in `spec.runtime.resources`.

### Name rejected on publish

Skill names must match `[a-z0-9][a-z0-9-]{0,62}[a-z0-9]`. Common rejections:

- Underscores: use hyphens instead (`my_skill` should be `my-skill`)
- Leading hyphens or dots: start with a letter or number
- Uppercase letters: use all lowercase
