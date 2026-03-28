# Agent Runtime Lifecycle & Scaling

## Overview

VOLUND agents follow a **serverless-style execution model**. The `volund-agent` binary is a generic runtime that assumes identity at startup by loading a profile and user context. Pods come and go, but agent identity and state persist externally. This enables efficient scaling — pods are claimed from warm pools on demand and released when idle.

## Core Principle

```
volund-agent binary = generic compute
AgentProfile        = the "code" (personality, skills, model config)
External state      = the "memory" (Redis sessions, Postgres long-term)
```

An agent pod has no inherent identity. It becomes an orchestrator or specialist based on what gets injected at claim time.

## Agent Pod Lifecycle

```
                    ┌─────────────────┐
                    │   Warm Pool     │
                    │                 │
                    │  ┌───┐ ┌───┐   │
                    │  │   │ │   │   │
                    │  │ ? │ │ ? │   │  Generic volund-agent pods
                    │  │   │ │   │   │  Pre-started, no identity
                    │  └───┘ └───┘   │
                    └───────┬─────────┘
                            │
              User sends    │   Orchestrator delegates
              message       │   task
              ▼             │             ▼
     ┌────────────────┐    │    ┌────────────────┐
     │  Claim Pod     │    │    │  Claim Pod     │
     │                │    │    │                │
     │  Inject:       │    │    │  Inject:       │
     │  - Orch profile│    │    │  - Specialist  │
     │  - User context│    │    │    profile     │
     │  - Memory state│    │    │  - Task context│
     └───────┬────────┘    │    └───────┬────────┘
             │             │            │
             ▼             │            ▼
     ┌────────────────┐    │    ┌────────────────┐
     │  ACTIVE        │    │    │  ACTIVE        │
     │  Processing    │    │    │  Working on    │
     │  messages      │    │    │  task          │
     └───────┬────────┘    │    └───────┬────────┘
             │             │            │
             ▼             │            │
     ┌────────────────┐    │            │
     │  WARM IDLE     │    │            │  Task completes
     │  Waiting for   │    │            │  (CloudEvent)
     │  next message  │    │            │
     │  (timeout: 5m) │    │            │
     └───────┬────────┘    │            │
             │             │            │
             ▼             │            ▼
     ┌────────────────┐    │    ┌────────────────┐
     │  COLD          │    │    │  Release Pod   │
     │  State saved   │    │    │  back to pool  │
     │  Pod released  │    │    └────────────────┘
     │  back to pool  │    │
     └────────────────┘    │
                           │
      ┌────────────────────┘
      │  Control plane sees task.completed
      │  → Claim pod from pool
      │  → Reinject orchestrator profile
      │  → Notify user
      ▼
```

## Three Pod States

| State | Pod Status | What's Happening | Transition |
|-------|-----------|------------------|------------|
| **Active** | Claimed, processing | Handling messages, executing tools, delegating tasks | → Warm Idle (no activity for N seconds) |
| **Warm Idle** | Claimed, waiting | Pod is reserved for this agent, waiting for next message | → Active (message received) or → Cold (timeout) |
| **Cold** | Released to pool | All state externalized, pod has no identity | → Active (new message triggers claim) |

### Idle Timeouts by Tier

| Tier | Warm Idle Timeout | Behavior |
|------|------------------|----------|
| Free | 2 minutes | Quick release back to pool |
| Pro | 10 minutes | Reasonable wait for follow-up messages |
| Enterprise | Configurable / always-on | Option to keep orchestrator permanently claimed |

## State Externalization

When an agent transitions from Warm Idle → Cold, all state must be saved externally:

| State | Storage | Restore Time |
|-------|---------|-------------|
| Conversation history | Redis (session store) | ~1ms |
| Active task graph | Redis (task state key) | ~1ms |
| Long-term memories | PostgreSQL + pgvector | ~5ms (top-K query) |
| User context | PostgreSQL | ~2ms |
| Agent profile | PostgreSQL / ConfigMap | ~1ms |
| Installed skills | Control plane registry | ~5ms |

**Total cold-start restore: ~15-50ms** (after pod claim from warm pool)

The user experience for Cold → Active:
1. User sends message (~0ms)
2. Gateway routes to control plane (~5ms)
3. Control plane claims pod from warm pool (~100-500ms)
4. Profile + context injection (~50ms)
5. State restore from Redis/Postgres (~50ms)
6. Agent processes message (~varies, LLM call dominates)

**Total overhead: ~200-600ms** before the LLM call starts. Since LLM responses take 1-5 seconds anyway, this is barely noticeable.

## Warm Pools

Two separate warm pools managed by the operator:

### Orchestrator Pool

Pre-warmed pods optimized for orchestrator workloads:

```yaml
apiVersion: volund.io/v1alpha1
kind: AgentWarmPool
metadata:
  name: orchestrator-pool
spec:
  minReady: 5                    # Always keep 5 pods warm
  maxSize: 50                    # Scale up to 50
  scaleUpThreshold: 2            # Scale up when fewer than 2 unclaimed
  podTemplate:
    image: volund/agent-runtime:latest
    resources:
      requests: { cpu: "200m", memory: "512Mi" }
      limits:   { cpu: "1",    memory: "1Gi"   }
    preloadSkills:               # Common skills pre-loaded
      - web-search
      - calculator
      - file-manager
```

### Specialist Pool

Pre-warmed pods for specialist workloads, potentially with different resource profiles:

```yaml
apiVersion: volund.io/v1alpha1
kind: AgentWarmPool
metadata:
  name: specialist-pool
spec:
  minReady: 3
  maxSize: 100
  scaleUpThreshold: 1
  podTemplate:
    image: volund/agent-runtime:latest
    resources:
      requests: { cpu: "500m", memory: "1Gi" }
      limits:   { cpu: "2",    memory: "2Gi"  }
```

### Pool Scaling

The operator watches pool utilization and scales:
- **Scale up** when unclaimed pods < `scaleUpThreshold`
- **Scale down** when unclaimed pods > `minReady` for sustained period
- **Burst** for sudden demand (tenant onboarding, peak hours)

## Pod Claim Protocol

When a message arrives for an agent that has no active pod:

```
1. Gateway receives message
2. Gateway → Control Plane: "route message for agent X"
3. Control Plane checks:
   a. Is agent X currently Active or Warm Idle? → route to existing pod
   b. Is agent X Cold? → initiate claim:
      i.   Select pod from appropriate warm pool (orchestrator/specialist)
      ii.  Mark pod as claimed (atomic operation)
      iii. Inject: AgentProfile, user context, tenant config, secrets
      iv.  Pod loads state from Redis/Postgres
      v.   Pod signals ready
      vi.  Route message to pod
4. If warm pool is empty:
   a. Queue the message (bounded queue, ~30s timeout)
   b. Operator emergency-scales the pool
   c. When pod available, resume from step 3b
```

## Session Affinity

While an agent is Active or Warm Idle, all messages route to the same pod. This is critical for:
- In-flight tool executions
- Streaming responses
- Conversation coherence (working memory is in-process)

The control plane maintains a routing table:

```
agent_id → pod_id (with TTL matching warm idle timeout)
```

When the pod releases, the routing entry is deleted. Next message triggers a new claim.

## Task Completion Without Orchestrator

When a specialist completes a task and the orchestrator is Cold:

```
1. Specialist emits io.volund.task.completed CloudEvent
2. Control plane task watcher receives event
3. Control plane checks: is orchestrator for this conversation active?
   a. Yes → forward task result to orchestrator pod
   b. No (Cold) →
      i.   Store task result in Redis (task:{taskId}:result)
      ii.  Claim orchestrator pod from warm pool
      iii. Inject profile + context + pending task results
      iv.  Orchestrator processes results
      v.   Orchestrator sends notification to user via gateway
      vi.  If user responds → stay Active
      vii. If no response → Warm Idle → Cold
```

## Specialist Agent Creation Flow

### Admin-Defined (System-Wide)

Platform admins create AgentProfiles that are available to all tenants:

```
Admin → API/Dashboard
  → Create AgentProfile CR (namespace: volund-system)
  → Available in The Forge as "official" profiles
  → All orchestrators can delegate to these
```

### User-Created via Chat

```
User: "I want an agent that monitors our GitHub repos
       and summarizes new PRs every morning"

Orchestrator:
  1. Recognizes intent to create a specialist
  2. Asks clarifying questions:
     - "Which repos should it monitor?"
     - "What level of detail in summaries?"
     - "Should it post summaries somewhere or just tell you?"
  3. Generates AgentProfile:
     - system_prompt: crafted from conversation
     - skills: [github-api, text-summarizer]
     - model: claude-haiku (cost-effective for summaries)
     - schedule: cron "0 9 * * 1-5" (weekday mornings)
  4. Shows preview to user:
     "Here's the agent I'd create:
      Name: GitHub PR Summarizer
      Skills: github-api, text-summarizer
      Runs: Weekday mornings at 9am
      Model: Claude Haiku (to keep costs low)
      Does this look right?"
  5. User approves
  6. Orchestrator → Control Plane API:
     - Create AgentProfile CR (namespace: volund-tenant-{orgId})
     - If scheduled: create CronJob or scheduled task trigger
  7. Profile now available for orchestrator to delegate to
```

### User-Created via Web UI

```
Web UI → Agent Builder wizard
  → Step 1: Name and description
  → Step 2: System prompt (with AI-assisted writing)
  → Step 3: Select skills from installed + Forge
  → Step 4: Model and resource config
  → Step 5: Review and create
  → API call → AgentProfile CR created
```
