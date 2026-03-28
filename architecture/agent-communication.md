# Agent-to-Agent Communication

## Overview

Agents communicate asynchronously via CloudEvents over NATS. The orchestrator does not need to stay alive while specialists work. Specialists post updates directly into the chat conversation, and the orchestrator has an inbox for receiving results when it spins back up.

## Communication Pattern: Event-Driven with Chat Integration

```
┌─────────┐     ┌──────────────┐     ┌─────────────┐
│  User   │────▶│  Gateway /   │────▶│ Orchestrator│
│  (Chat) │◀────│  WebSocket   │◀────│   Agent     │
└─────────┘     └──────┬───────┘     └──────┬──────┘
                       │                    │
                       │                    │ task.assigned
                       │                    ▼
                       │             ┌──────────────┐
                       │             │    NATS      │
                       │             │  (CloudEvents│
                       │             │   bus)       │
                       │             └──────┬───────┘
                       │                    │
                       │    ┌───────────────┼───────────────┐
                       │    │               │               │
                       │    ▼               ▼               ▼
                       │ ┌─────────┐ ┌──────────┐ ┌──────────────┐
                       │ │Specialist│ │Specialist│ │ Control Plane│
                       │ │ Agent A │ │ Agent B  │ │ (task watcher│
                       │ └────┬────┘ └──────────┘ │  + inbox mgr)│
                       │      │                   └──────────────┘
                       │      │ chat.message
                       │      │ (writes directly
                       │      │  to conversation)
                       │◀─────┘
                       │
               User sees specialist
               messages in real-time
```

## The Three Communication Channels

### 1. Chat Thread (User-Facing)

Specialists write messages directly into the conversation thread via the gateway. The user sees a multi-agent conversation in real-time.

**Who writes to chat:**
- User → via WebSocket
- Orchestrator → via WebSocket (when active) or queued message (on spin-up)
- Specialists → via gateway API (post message to conversation)

**Message routing for user replies:**
When the user replies during a specialist's active task:
1. Gateway checks: which agent is currently `ACTIVE` or `WAITING` in this conversation?
2. If a specialist has `PARTICIPANT_STATUS_WAITING` (asked a question) → route reply to specialist
3. If no specialist is waiting → queue for orchestrator inbox
4. If orchestrator is active → route to orchestrator

### 2. NATS Events (System-Level)

CloudEvents for lifecycle coordination. Not shown to the user directly.

| Event | From | To | Purpose |
|-------|------|----|---------|
| `task.assigned` | Orchestrator | Control Plane | "Spin up a specialist for this task" |
| `task.progress` | Specialist | Control Plane | Update task progress (reflected in Tasks UI) |
| `task.completed` | Specialist | Control Plane → Orchestrator Inbox | "Work is done, here are the results" |
| `task.failed` | Specialist | Control Plane → Orchestrator Inbox | "Something went wrong" |
| `task.input_needed` | Specialist | Control Plane → Gateway | "I need user input" (triggers WAITING state) |
| `agent.joined` | Control Plane | Gateway | Notify UI that specialist joined conversation |
| `agent.left` | Control Plane | Gateway | Notify UI that specialist left conversation |

### 3. Orchestrator Inbox (Async Queue)

A per-orchestrator message queue for things that happen while it's cold.

**Storage:** Redis list per orchestrator agent ID

```
volund:inbox:{orchestrator_agent_id}  →  [message1, message2, ...]
```

**What goes in the inbox:**
- Task completion results (from specialists)
- Task failure notifications
- Scheduled trigger events ("time to run your daily job")
- System notifications (budget alerts, skill updates)

**Inbox processing on spin-up:**

```
Orchestrator starts (pod claimed from warm pool)
  │
  ├── Load profile + user context
  ├── Restore session state from Redis
  │
  ├── Drain inbox (FIFO):
  │   ├── task.completed: "Code Agent refactored 12 files"
  │   ├── task.completed: "Research Agent found 3 CVEs"
  │   └── scheduled: "Daily PR summary trigger"
  │
  ├── Prioritize:
  │   1. Pending user message (if any) → respond first
  │   2. Task results → synthesize and post to chat
  │   3. Scheduled triggers → execute
  │
  └── Process each item
```

## Full Flow: Complex Task

```
User: "Analyze our codebase for security issues and write a report"

1. ORCHESTRATOR (Active)
   ├── Decomposes into subtasks:
   │   ├── Task A: "Scan codebase for vulnerabilities" → Code Agent
   │   └── Task B: "Research CVE databases for matches" → Research Agent
   ├── Posts to chat: "I'll have the Code Agent scan the code and
   │   the Research Agent check CVE databases. You can track progress
   │   in Tasks."
   ├── Emits: task.assigned (Task A) → NATS
   ├── Emits: task.assigned (Task B) → NATS
   └── Idle timeout → COLD (spins down)

2. CONTROL PLANE
   ├── Receives task.assigned events
   ├── Claims 2 pods from specialist pool
   ├── Injects Code Agent profile + task context → Pod A
   ├── Injects Research Agent profile + task context → Pod B
   ├── Emits: agent.joined (Code Agent) → Gateway → UI
   └── Emits: agent.joined (Research Agent) → Gateway → UI

3. CODE AGENT (Active in Pod A)
   ├── Posts to chat: "Starting codebase scan..."
   ├── Executes tools (in sandbox): static analysis, dependency audit
   ├── Posts to chat: "Found 3 potential SQL injection points
   │   and 2 outdated dependencies with known CVEs"
   ├── Posts to chat: "Question: should I also check for
   │   hardcoded secrets? This will take an additional ~5 minutes."
   ├── Emits: task.input_needed → NATS → Gateway
   └── Status: WAITING (for user input)

4. USER sees in chat:
   [Code Agent] "Question: should I also check for hardcoded
   secrets? This will take an additional ~5 minutes."

   User: "Yes, check for secrets too"

5. GATEWAY routes reply to Code Agent (it's WAITING)

6. CODE AGENT (resumes)
   ├── Posts to chat: "Scanning for hardcoded secrets..."
   ├── Posts to chat: "Found 1 API key in config.go. Scan complete."
   ├── Emits: task.completed (with full results) → NATS
   ├── Emits: agent.left → Gateway → UI
   └── Pod released back to specialist pool

7. RESEARCH AGENT (Active in Pod B, working in parallel)
   ├── Posts to chat: "Checking CVE databases..."
   ├── Posts to chat: "Found 2 CVEs matching your dependencies:
   │   CVE-2026-1234 (high), CVE-2026-5678 (medium)"
   ├── Emits: task.completed (with full results) → NATS
   ├── Emits: agent.left → Gateway → UI
   └── Pod released back to specialist pool

8. CONTROL PLANE
   ├── Both tasks complete
   ├── Puts both results in orchestrator inbox
   └── Triggers orchestrator spin-up

9. ORCHESTRATOR (Claims pod, restores state, drains inbox)
   ├── Reads: Task A result (code scan) + Task B result (CVE research)
   ├── Synthesizes results into a security report
   ├── Posts to chat: "Security analysis complete. Here's the
   │   full report: [3 SQL injection points, 1 hardcoded API key,
   │   2 CVEs in dependencies]. Want me to create tickets for these?"
   └── Waits for user response (WARM IDLE)
```

## Message Routing Table

The control plane maintains routing state per conversation:

```
conversation:{id}:routing
  ├── orchestrator_agent_id: "agent-123"
  ├── orchestrator_pod: "pod-abc" (or null if COLD)
  ├── active_specialists:
  │   ├── "agent-456": { pod: "pod-def", task: "task-A", status: ACTIVE }
  │   └── "agent-789": { pod: "pod-ghi", task: "task-B", status: ACTIVE }
  └── waiting_specialist: "agent-456" (currently asking user a question)
```

**User message routing logic:**

```
User sends message to conversation
  │
  ├── Is there a waiting_specialist? → route to that specialist
  │
  ├── Is orchestrator pod active? → route to orchestrator
  │
  ├── Is orchestrator COLD?
  │   ├── Queue message in orchestrator inbox
  │   └── Trigger orchestrator spin-up
  │
  └── Edge case: no orchestrator, no specialists
      → Error: "Your agent is starting up, one moment..."
```

## UI Integration

### Chat View

The primary view. Shows all messages from all agents in chronological order.

- Each message tagged with agent name and avatar/icon
- Visual distinction between orchestrator messages and specialist messages
- "[Agent Name] joined/left the conversation" system messages
- When a specialist asks a question, the reply input could show "Replying to [Code Agent]"

### Tasks View

Dashboard showing all active and recent tasks.

| Task | Agent | Status | Progress | Started |
|------|-------|--------|----------|---------|
| Codebase scan | Code Agent | Waiting for input | 75% | 2m ago |
| CVE research | Research Agent | Running | 40% | 2m ago |
| Daily PR summary | PR Agent | Scheduled | — | 9:00 AM |

Actions: Cancel task, View details, Jump to chat message

### Orchestrator Inbox (Internal — not a user-facing UI)

This is an internal system concept. Users don't see it. It's how the control plane queues work for the orchestrator to process on spin-up.
