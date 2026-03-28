# Channel Adapters Architecture

## Overview

Channel adapters connect agents to the outside world. VOLUND has two client channels (Web UI and Desktop App) that share a React frontend codebase. Both connect through the WebSocket gateway for real-time streaming.

The UI has three primary views:
- **Chat** — multi-agent conversation thread (orchestrator + specialists + user)
- **Tasks** — async job dashboard for tracking long-running work
- **Dashboard** — agent management, usage, billing, Forge marketplace

## Channel Types

```
┌──────────────────────────────────────────────────────────┐
│                     User Interfaces                       │
│                                                          │
│  ┌────────────────┐         ┌────────────────┐           │
│  │  Web UI (SPA)  │         │ Desktop (Tauri) │           │
│  └───────┬────────┘         └───────┬─────────┘           │
│          └──────────┬───────────────┘                     │
│                     ▼                                     │
│             ┌───────────────┐                             │
│             │  WebSocket    │                             │
│             │  Gateway      │                             │
│             └───────┬───────┘                             │
└─────────────────────┼─────────────────────────────────────┘
                      │
┌─────────────────────┼─────────────────────────────────────┐
│                     ▼            Control Plane              │
│             ┌───────────────┐                              │
│             │  Channel      │                              │
│             │  Router       │                              │
│             └───────┬───────┘                              │
└─────────────────────┼─────────────────────────────────────┘
                      │
┌─────────────────────┼─────────────────────────────────────┐
│                     ▼          Agent Runtime                │
│             ┌───────────────┐                              │
│             │  WebSocket    │                              │
│             │  Handler      │                              │
│             └───────────────┘                              │
└───────────────────────────────────────────────────────────┘
```

## Channels

### 1. Web Chat UI

The default interaction channel — a web-based chat interface.

- **Tech:** React SPA with WebSocket connection to the gateway
- **Auth:** JWT from the gateway, WebSocket authenticated on connect
- **Three primary views:**

#### Chat View
The primary interaction surface. Shows a **multi-agent conversation thread** where the orchestrator, specialists, and user all appear in one unified timeline.

- Real-time streaming responses (token-by-token)
- Rich message rendering (markdown, code blocks, tables, images)
- Each message tagged with agent name and avatar — visual distinction between orchestrator and specialists
- `[Agent Name] joined/left the conversation` system messages
- When a specialist asks a question, reply input shows "Replying to [Code Agent]"
- File upload/download

#### Tasks View
Dashboard for monitoring async long-running jobs.

- Active tasks with progress bars and status
- Which specialist agent is assigned to each task
- Cancel, view details, jump to related chat message
- Task history (completed/failed)
- Scheduled/recurring tasks

#### Dashboard View
Management and monitoring.

- Agent management: create, configure, start/stop
- Agent Builder wizard for creating specialist profiles
- Usage and billing dashboards
- Forge marketplace browser (search, install skills/profiles)
- Org/team/user management

### 2. Desktop App (Tauri)

Cross-platform desktop application wrapping the web UI with native capabilities.

- **Tech:** Tauri 2.x (Rust backend + web frontend)
- **Why Tauri over Electron:**
  - ~10x smaller binary size (Tauri: ~5MB vs Electron: ~50MB+)
  - Lower memory footprint (uses OS webview, not bundled Chromium)
  - Rust backend for native integrations (filesystem, notifications, system tray)
  - Better security model (fine-grained permissions per API)
- **Native features:**
  - System tray with agent status
  - Native notifications for task completion
  - Global keyboard shortcut to invoke agent (like Spotlight)
  - Local file access for agent tools (with user permission)
  - Offline message queuing
  - Auto-updates via Tauri's built-in updater
- **Architecture:**
  - Frontend: same React SPA as web UI (shared codebase)
  - Backend: Tauri Rust commands for native APIs
  - Connection: WebSocket to VOLUND gateway (same as web)
  - Offline: SQLite for local message cache, sync on reconnect

### 3. REST API

Programmatic access for building custom clients and integrations.

- **Endpoints:**
  - `POST /v1/agents/{id}/messages` — send a message, get response
  - `POST /v1/agents/{id}/messages/stream` — SSE streaming response
  - `GET /v1/agents/{id}/conversations` — list conversations
  - `POST /v1/agents/{id}/tasks` — submit async task
  - `GET /v1/agents/{id}/tasks/{taskId}` — poll task status
- **Auth:** API key or JWT bearer token
- **Rate limiting:** per-key, configurable per tenant tier

Both the Web UI and Desktop app use the WebSocket gateway for real-time interaction. The REST API exists for programmatic/automation use cases.

## WebSocket Gateway

The central real-time communication hub between clients and agents.

**Protocol:**
- WebSocket with JSON messages (consider Protocol Buffers for performance later)
- Heartbeat/ping-pong for connection health
- Automatic reconnection with exponential backoff (client-side)
- Message acknowledgment for delivery guarantees

**Message types:**
```
Client → Server:
  message.send        — User sends a message
  message.stream.ack  — Client acknowledges stream chunk
  session.create      — Start new conversation
  session.resume      — Resume existing conversation
  typing.start        — User is typing

Server → Client:
  message.received    — Agent received the message
  message.chunk       — Streaming response token
  message.complete    — Full response complete
  agent.status        — Agent state change (thinking, tool_exec, delegating)
  task.progress       — Specialist agent progress update
  task.complete       — Delegated task finished
  error               — Error occurred
```

## Channel Router

Part of the control plane — routes messages between channels and agent pods.

**Responsibilities:**
- Map channel connections to agent instances
- Handle agent pod discovery (which pod serves which agent)
- Buffer messages if agent pod is starting up (warm pool miss)
- Fan-out: single agent response → multiple connected channels
- Rate limiting per channel per agent

## Desktop App Repo Strategy

The desktop app will live in a separate repo: `volund-desktop`

```
volund-desktop/
├── src-tauri/              # Rust backend
│   ├── src/
│   │   ├── main.rs
│   │   ├── commands/       # Tauri commands (native APIs)
│   │   ├── tray.rs         # System tray
│   │   └── updater.rs      # Auto-update logic
│   ├── Cargo.toml
│   └── tauri.conf.json
├── src/                    # Shared web frontend (React)
│   ├── components/
│   ├── hooks/
│   └── ...
├── package.json
└── Makefile
```

This means we'd need a **7th repo**: `ai-volund/volund-desktop`.
