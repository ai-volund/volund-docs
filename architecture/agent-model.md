# Agent Model Architecture

## Overview

VOLUND uses a two-tier agent model with a **serverless execution pattern**:
- An **Orchestrator** agent per user is the single entry point for all interaction
- **Specialist** agents are spawned on demand for complex tasks
- All agents run on **generic pods claimed from warm pools** — identity is injected at claim time
- Specialists post updates **directly into the chat thread** — the user sees a multi-agent conversation
- The orchestrator has an **inbox** for receiving results while it's spun down

## Execution Flow

![Agent Model](diagrams/agent-model.svg)

> [Edit in Excalidraw](https://excalidraw.com/#json=SDhCf9nuA0MJ0eGhPD42p,jSQeERQhjcbNTTV2OfU1fA) | [Source JSON](diagrams/agent-model.excalidraw.json)

```
User (Chat UI / Desktop App)
  │
  ▼
┌─────────────────────────────────┐
│  ORCHESTRATOR AGENT             │
│  (per-user, serverless)         │
│                                 │
│  - Single entry point           │
│  - Handles general chat + tools │
│  - Decomposes complex tasks     │
│  - Delegates to specialists     │
│  - Has an inbox for async results│
│  ┌───────────────────────────┐  │
│  │ Agent Profile + Memory    │  │
│  └───────────────────────────┘  │
└──────┬──────────┬───────────────┘
       │          │
  ┌────▼───┐  ┌──▼──────────────────┐
  │ Direct │  │ Delegate to          │
  │ Tool   │  │ Specialist Agents    │
  │ Exec   │  └──┬──────┬───────┬───┘
  │        │     │      │       │
  └────────┘  ┌──▼──┐┌──▼──┐┌──▼──┐
              │Code ││Data ││Rsrch│  ...
              │Agent││Agent││Agent│
              └──┬──┘└──┬──┘└─────┘
                 │      │
              ┌──▼──────▼──┐
              │ agent-sandbox│
              │ (tool exec) │
              └─────────────┘
```

## Orchestrator Agent

The orchestrator is the user's personal agent and the **single entry point** for all interaction. It:

- Handles general conversation and simple tool use directly
- Decomposes complex requests into tasks and delegates to specialists
- Synthesizes results from specialists and presents them to the user
- Maintains long-term memory about the user
- Processes its **inbox** on spin-up (task results, scheduled triggers, system notifications)
- **Does NOT need to stay alive** while specialists work

### Orchestrator Lifecycle

The orchestrator follows the serverless warm pool pattern:

1. **User sends message** → control plane claims pod from orchestrator warm pool
2. **Profile + context injected** → pod becomes this user's orchestrator
3. **Inbox drained** → any pending task results or notifications processed first
4. **Message processed** → orchestrator responds (may delegate tasks)
5. **Idle timeout** → state saved, pod released back to warm pool

See [Agent Runtime Lifecycle](agent-runtime-lifecycle.md) for full details on the warm pool model.

## Specialist Agents

Specialists are focused agents with specific skills, defined by `AgentProfile` resources:

```yaml
apiVersion: volund.io/v1alpha1
kind: AgentProfile
metadata:
  name: code-specialist
spec:
  description: "Writes, reviews, and debugs code"
  personality:
    systemPrompt: |
      You are a senior software engineer...
    temperature: 0.3
  model:
    provider: anthropic
    name: claude-sonnet-4-6
  skills:
    - code-executor
    - git-operations
    - file-manager
  sandbox:
    template: code-sandbox
    warmPool: true
  delegation:
    canSpawnSubAgents: false
    maxConcurrentTasks: 3
```

### Creating Specialists

**System-wide (admin-defined):**
Platform admins create AgentProfiles available to all tenants. These appear as "official" profiles in The Forge.

**User-created via chat:**
```
User: "I want an agent that monitors our GitHub repos and summarizes PRs"

Orchestrator:
  → Asks clarifying questions (which repos, what detail level)
  → Generates AgentProfile (system prompt, skills, model config)
  → Shows preview: "Here's the agent I'd create..."
  → User approves → AgentProfile CR created in tenant namespace
```

**User-created via Web UI:**
Agent Builder wizard: name → system prompt → skills → model → review → create.

## Multi-Agent Chat

Specialists post messages **directly into the conversation thread**. The user sees all agents in one unified chat view:

```
User: "Refactor this codebase to use the new API"

Orchestrator: "I'll hand this to the Code Agent."

  [Code Agent joined the conversation]
  Code Agent: "Starting analysis... found 12 files using the old API."
  Code Agent: "Refactoring src/auth/handler.go (3/12)"
  Code Agent: "Question: handler.go is 800 lines. Should I split it?"

User: "Split it"

  Code Agent: "Got it. Splitting into handler.go and middleware.go."
  Code Agent: "Done. 13 files updated, all tests passing."
  [Code Agent left the conversation]

Orchestrator: "The refactor is complete. Here's what changed..."
```

**Message routing:** When the user replies during an active specialist task:
1. Is a specialist currently waiting for user input? → route reply to that specialist
2. Is the orchestrator active? → route to orchestrator
3. Orchestrator is cold? → queue in inbox, trigger spin-up

## Orchestrator Inbox

A per-orchestrator async queue (Redis list) for things that happen while it's spun down:

- Task completion/failure results from specialists
- Scheduled trigger events ("run the daily PR summary")
- System notifications (budget alerts, skill updates)

On spin-up, the orchestrator drains its inbox in priority order:
1. Pending user messages (interactive, highest priority)
2. Task results (synthesize and post to chat)
3. Scheduled triggers (execute)

## Inter-Agent Communication

All coordination uses **CloudEvents over NATS** — fully async, event-driven:

| Event | From → To | Purpose |
|-------|-----------|---------|
| `task.assigned` | Orchestrator → Control Plane | "Spin up a specialist for this task" |
| `task.progress` | Specialist → Control Plane | Update task progress (shown in Tasks UI) |
| `task.completed` | Specialist → Orchestrator Inbox | "Work is done, here are the results" |
| `task.failed` | Specialist → Orchestrator Inbox | "Something went wrong" |
| `task.input_needed` | Specialist → Gateway | "I need user input" (routes to chat) |
| `agent.joined` | Control Plane → Gateway | Notify UI specialist joined conversation |
| `agent.left` | Control Plane → Gateway | Notify UI specialist left conversation |
| `chat.message` | Specialist → Gateway | Post message directly to conversation thread |

See [Agent Communication](agent-communication.md) for the full protocol.

## Tool Execution Tiers

| Tier | Use Case | Mechanism | Latency |
|------|----------|-----------|---------|
| In-process (WASM) | Stateless tools | WASM in agent-runtime | <10ms |
| Sandbox | Untrusted/heavy tools | agent-sandbox CRD | ~2-5s (warm pool: <500ms) |
| Sidecar | Long-running services | Sidecar container | Persistent |

## Agent Teams

For complex workflows, the orchestrator assembles teams of specialists working in parallel:

```
User: "Analyze our codebase for security issues and write a report"

Orchestrator delegates:
  ├── Code Agent: scans code for vulnerabilities (sandbox)
  ├── Research Agent: checks CVE databases (tools)
  └── Both post progress to chat thread in real-time
      │
      ├── Code Agent: "Found 3 SQL injection points"
      ├── Research Agent: "Found 2 CVEs in dependencies"
      │
      └── Both complete → results in orchestrator inbox
          → Orchestrator spins up, synthesizes report
          → Posts to chat: "Security analysis complete..."
```

The user sees the full workflow in the chat thread and can track jobs in the Tasks view.
