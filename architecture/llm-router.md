# LLM Router Architecture

## Overview

The LLM Router is a control plane service that sits between agent runtimes and LLM providers. It handles provider selection, failover, key management, streaming, token counting, and cost tracking. Agents never talk directly to LLM APIs — everything goes through the router.

## Architecture

```
Agent Runtime (volund-agent)
  │
  │  gRPC: ChatRequest
  ▼
┌──────────────────────────────────────────────────────────┐
│                     LLM Router                            │
│                                                          │
│  ┌────────────┐  ┌─────────────┐  ┌──────────────────┐  │
│  │  Request    │  │  Provider   │  │  Token Counter   │  │
│  │  Classifier │  │  Selector   │  │  & Cost Tracker  │  │
│  └─────┬──────┘  └──────┬──────┘  └────────┬─────────┘  │
│        │                │                   │            │
│  ┌─────▼────────────────▼───────────────────▼─────────┐  │
│  │                Provider Pool                        │  │
│  │  ┌───────────┐ ┌────────┐ ┌───────┐ ┌───────────┐  │  │
│  │  │ Anthropic │ │ OpenAI │ │ Ollama│ │ BYO Key   │  │  │
│  │  │           │ │        │ │ (local)│ │ (tenant)  │  │  │
│  │  └───────────┘ └────────┘ └───────┘ └───────────┘  │  │
│  └─────────────────────────────────────────────────────┘  │
└──────────────────────────────────────────────────────────┘
```

## Provider Interface

```go
type LLMProvider interface {
    // Chat completion
    Chat(ctx context.Context, req *ChatRequest) (*ChatResponse, error)

    // Streaming chat completion
    StreamChat(ctx context.Context, req *ChatRequest) (ChatStream, error)

    // Embeddings
    Embeddings(ctx context.Context, req *EmbeddingRequest) (*EmbeddingResponse, error)

    // Metadata
    ListModels(ctx context.Context) ([]Model, error)
    ProviderName() string
    HealthCheck(ctx context.Context) error
}

type ChatStream interface {
    Recv() (*ChatChunk, error)   // Receive next token chunk
    Close() error
}
```

## Provider Implementations

### Anthropic
- Models: Claude Opus, Sonnet, Haiku families
- Features: tool use, vision, extended thinking, streaming
- API: Messages API via `github.com/anthropics/anthropic-sdk-go`

### OpenAI
- Models: GPT-4o, o1, o3 families
- Features: function calling, vision, streaming
- API: Chat Completions API via `github.com/openai/openai-go`

### Ollama (Local Models)
- Models: Llama, Mistral, Qwen, etc.
- Use case: cost-sensitive tasks, privacy-sensitive data, offline operation
- Connection: HTTP to Ollama server running in-cluster or on-prem

### BYO Key (Bring Your Own)
- Tenant provides their own API keys for any supported provider
- Keys encrypted at rest, injected via K8s secrets
- Metering still tracks usage for billing (compute cost, not API cost)

## Request Flow

### 1. Request Classification

When a request arrives, the router classifies it to determine optimal routing:

```go
type RequestClass struct {
    Complexity    string   // "simple", "moderate", "complex"
    RequiresTool  bool     // Does the prompt reference tool use
    RequiresVision bool   // Are images attached
    EstTokens     int     // Estimated input + output tokens
    LatencySLA    string  // "realtime" (streaming), "batch" (async)
}
```

### 2. Provider Selection

Selection follows a priority chain:

```
1. Agent config specifies exact model?
   → Use that provider + model

2. Tenant has BYO key for preferred provider?
   → Use tenant's key

3. Route by request class:
   → simple: cheapest available model (Haiku, GPT-4o-mini, local)
   → moderate: balanced model (Sonnet, GPT-4o)
   → complex: most capable model (Opus, o3)

4. Check provider health + rate limits
   → If primary is down/throttled, fall to next in chain

5. Check tenant budget
   → If budget exhausted, reject or downgrade
```

### 3. Failover Chain

Each agent config defines a fallback sequence:

```yaml
llm:
  primary:
    provider: anthropic
    model: claude-sonnet-4-6
  fallback:
    - provider: openai
      model: gpt-4o
    - provider: ollama
      model: llama3.3:70b
  failover:
    maxRetries: 2
    retryDelay: 1s
    circuitBreaker:
      failureThreshold: 5      # failures before opening circuit
      recoveryTimeout: 30s      # time before half-open
```

**Circuit breaker per provider:**
- Closed (healthy): requests flow normally
- Open (unhealthy): requests immediately fail over to next provider
- Half-open (recovering): send one test request, if success → close

### 4. Streaming

The router proxies streaming responses from providers to agent runtimes:

```
Provider (SSE/streaming) → LLM Router (gRPC stream) → Agent Runtime → WebSocket → Client
```

- Token counting happens on each chunk (running total)
- If provider stream errors mid-response, router can retry from the last complete message
- Backpressure: if client disconnects, router cancels the provider request

## Token Counting & Cost

### Per-Request Tracking

Every request/response pair is metered:

```go
type UsageRecord struct {
    RequestID     string
    TenantID      string
    AgentID       string
    UserID        string
    Provider      string
    Model         string
    InputTokens   int
    OutputTokens  int
    CachedTokens  int        // Prompt cache hits (Anthropic)
    TotalTokens   int
    CostUSD       float64    // Calculated from provider pricing
    LatencyMs     int64
    Timestamp     time.Time
}
```

### Cost Calculation

Provider pricing is configured and updatable without code changes:

```yaml
pricing:
  anthropic:
    claude-opus-4-6:
      input: 15.00      # per 1M tokens
      output: 75.00
      cachedInput: 1.50
    claude-sonnet-4-6:
      input: 3.00
      output: 15.00
      cachedInput: 0.30
  openai:
    gpt-4o:
      input: 2.50
      output: 10.00
```

### Budget Controls

Per-tenant and per-agent budget enforcement:

```go
type BudgetConfig struct {
    // Tenant level
    MonthlyBudgetUSD  float64    // Hard cap for the tenant
    AlertThresholds   []float64  // [0.50, 0.75, 0.90] — alert at 50%, 75%, 90%

    // Agent level
    DailyTokenLimit   int        // Max tokens per day per agent
    RequestRateLimit  int        // Max requests per minute per agent

    // On budget exceeded
    Action            string     // "reject", "downgrade", "alert_only"
    DowngradeTo       string     // Model to downgrade to if action=downgrade
}
```

## Prompt Caching

The router supports prompt caching strategies:

- **Anthropic prompt caching**: automatically uses cache breakpoints for system prompts and long context
- **Semantic caching**: hash the request (system prompt + last N messages), if cache hit within TTL return cached response (configurable, off by default — only for idempotent queries)
- **System prompt caching**: system prompts are hashed and reused across requests for the same agent

## Observability

The LLM Router emits CloudEvents for every request:

- `io.volund.llm.request.started` — provider, model, estimated tokens
- `io.volund.llm.request.completed` — actual tokens, cost, latency
- `io.volund.llm.request.failed` — error, failover action taken
- `io.volund.llm.provider.circuit.opened` — provider circuit breaker tripped
- `io.volund.llm.budget.threshold` — tenant hit budget alert threshold

Prometheus metrics:
- `volund_llm_requests_total` (provider, model, status)
- `volund_llm_tokens_total` (provider, model, direction: input/output)
- `volund_llm_request_duration_seconds` (provider, model)
- `volund_llm_cost_usd_total` (tenant, provider, model)
- `volund_llm_provider_health` (provider: healthy/degraded/down)
