# ADR-007: Multi-Agent Chat with Orchestrator Inbox

**Status:** Accepted
**Date:** 2026-03-27

## Context

When the orchestrator delegates tasks to specialists, the user needs visibility into what's happening. The orchestrator follows a serverless model and may not be running while specialists work. We need a communication pattern that keeps the user informed without requiring the orchestrator to relay every message.

## Decision

### Multi-Agent Chat Thread
Specialist agents post messages **directly into the conversation thread** via the gateway API. The user sees a unified timeline with messages from all participating agents, each tagged with agent name.

The chat is the single source of truth for all user-facing communication. No separate notification channels needed.

### User Input Routing
When a specialist needs user input mid-task, it posts a question to the chat and enters a WAITING state. The gateway routes the user's reply directly to the waiting specialist — the orchestrator is not involved.

### Orchestrator Inbox
The orchestrator has a Redis-backed inbox queue. When specialists complete tasks:
1. Task result goes into the orchestrator's inbox
2. Control plane triggers orchestrator spin-up
3. Orchestrator drains inbox, synthesizes results, posts summary to chat

### Tasks View
A separate UI view for monitoring long-running jobs — progress bars, status, cancel. This is a read-only dashboard driven by CloudEvents, not a communication channel.

## Consequences

- Chat stays clean: the user sees who's talking and can reply directly to specialists
- Orchestrator doesn't need to relay messages — reduces coupling and enables serverless model
- The conversation model supports multiple participants (proto updated with agent_id, participant tracking)
- Gateway needs a message routing table per conversation (which agent is active/waiting)
- Tasks view gives users a dashboard for async work without cluttering the chat
