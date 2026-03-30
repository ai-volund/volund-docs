# VOLUND Platform — Gaps & Roadmap

Last updated: 2026-03-30

This document tracks known gaps, missing features, and the plan to address them across all repos. Each item is prioritized (P0–P3) and assigned to a phase.

---

## Phase 1: Security & Access Control (P0)

### 1.1 Admin UI Role Guard ✅
**Done:** `useAdminRole` hook checks JWT for `platform_admin`/`admin`. Shows `AccessDenied` component for non-admin users. `RequireAdmin` middleware on all `/v1/admin/*` endpoints.

### 1.2 Admin Role Promotion API ✅
**Done:** `PUT /v1/admin/tenants/{id}/members/{userId}/role` + `DELETE /v1/admin/tenants/{id}/members/{userId}`. Backend complete. Admin UI member management page TODO.

### 1.3 Admin API Authorization Audit ✅
**Done:** All `/v1/admin/*` routes now use `RequireAdmin` middleware. Only `platform_admin` and `admin` roles have access. `owner` is tenant-level only.

### 1.4 CORS Lockdown ✅
**Done:** `CORSMiddleware` reads `VOLUND_CORS_ORIGINS` env var. Defaults to `*` in dev. Validates `Origin` header when explicit origins are set.

### 1.5 CSRF Protection
**Gap:** better-auth sessions use cookies with `SameSite=Lax` but the gateway doesn't verify CSRF tokens for state-changing requests.
**Fix:** Add CSRF token header requirement for POST/PUT/DELETE when using cookie auth. better-auth's client already handles this for its own endpoints.
**Repos:** `volund`

---

## Phase 2: Dynamic LLM Provider Management (P0)

### 2.1 Admin API for LLM Providers ✅
**Done:** Full CRUD + test + models endpoints implemented. DB table `llm_providers`. Hot-reload on changes. Loads DB providers at startup alongside env vars.
**Fix:** New admin endpoints + database table for provider config:

```
POST   /v1/admin/llm/providers           — register a provider (name, type, api_key, base_url, models)
GET    /v1/admin/llm/providers           — list configured providers
GET    /v1/admin/llm/providers/{id}      — get provider detail + model list
PUT    /v1/admin/llm/providers/{id}      — update config (rotate API key, change base URL)
DELETE /v1/admin/llm/providers/{id}      — remove provider
POST   /v1/admin/llm/providers/{id}/test — test connection (list models, run health check)
GET    /v1/admin/llm/models              — list all models across all providers
```

**Database:**
```sql
CREATE TABLE llm_providers (
    id          UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name        TEXT NOT NULL UNIQUE,    -- "openai", "anthropic", "ollama", "azure-openai", "local-llama"
    type        TEXT NOT NULL,           -- "openai", "anthropic", "ollama", "openai-compatible"
    api_key     TEXT,                    -- encrypted at rest
    base_url    TEXT,                    -- custom endpoint (Azure, LM Studio, vLLM, etc.)
    config      JSONB DEFAULT '{}',     -- provider-specific config (org_id, api_version, etc.)
    enabled     BOOLEAN DEFAULT true,
    created_at  TIMESTAMPTZ DEFAULT NOW(),
    updated_at  TIMESTAMPTZ DEFAULT NOW()
);
```

**Router changes:**
- `Router.Register()` / `Router.Unregister()` called dynamically when admin adds/removes providers
- On startup, load providers from DB first, then overlay env vars (env vars take precedence for backward compat)
- Provider factory: `NewProviderFromConfig(type, apiKey, baseURL, config)` returns the right `Provider` impl
- Hot-reload: admin changes take effect without gateway restart

**Repos:** `volund` (new endpoints, DB migration, router changes), `volund-admin` (new Providers → LLM page)

### 2.2 Admin UI — LLM Provider Management Page ✅
**Done:** New "LLM Providers" page with Add/Edit/Delete, type selection (OpenAI, Anthropic, Ollama, OpenAI-compatible), Test Connection, Enable/Disable toggle, and Available Models catalog.

### 2.3 OpenAI-Compatible Provider Type ✅
**Done:** "openai-compatible" type supported in both API and UI. Uses the OpenAI provider implementation with custom `base_url`. Works with Azure, Together, Groq, LM Studio, vLLM, etc.

### 2.4 Per-Tenant Provider & Model Config
**Gap:** All tenants share the same providers. No way to restrict which models a tenant can use or set per-tenant API keys.
**Fix:** Add `tenant_llm_config` table mapping tenant → allowed providers/models. Admins can set per-tenant overrides. Default: all providers available to all tenants.
**Repos:** `volund`, `volund-admin`

---

## Phase 3: Operational Visibility (P1)

### 3.1 Agent Instance & Warm Pool Dashboard ✅
**Done:** GET /v1/admin/instances, DELETE /v1/admin/instances/{id}, GET /v1/admin/warmpool. Admin UI Instances page with pod table + warm pool stat cards.
**Original fix:** New endpoints:
```
GET    /v1/admin/instances          — list all instances (pod, state, tenant, profile, uptime)
GET    /v1/admin/instances/{id}     — instance detail
DELETE /v1/admin/instances/{id}     — force-release
GET    /v1/admin/warmpool           — pool stats (total, available, claimed, by profile)
```
Add "Instances" tab to admin Agents page with live pod status and warm pool utilization bars.
**Repos:** `volund`, `volund-admin`

### 3.2 Platform Health Endpoint ✅
**Done:** GET /v1/admin/health checks postgres, auth service, warm pool. Dashboard shows real health badges.
**Original gap:** `/healthz` only checks gateway. No visibility into NATS, Postgres, Redis, operator, auth service.
**Fix:** New `GET /v1/admin/health` that checks all dependencies:
```json
{
  "gateway": "ok",
  "postgres": "ok",
  "nats": "ok",
  "redis": "ok",
  "auth": "ok",
  "operator": "ok",
  "warm_pool": { "total": 4, "available": 2, "claimed": 2 }
}
```
Update admin dashboard to use this instead of `/healthz`.
**Repos:** `volund`, `volund-admin`

### 3.3 Audit Log ✅
**Done:** audit_log table (migration 016), AuditRepo, GET /v1/admin/audit. Admin UI Audit Log page with filterable table.
**Original gap:** No general audit trail. Only credential audit exists.
**Fix:** Add `audit_log` table + middleware that logs all state-changing API calls (POST/PUT/DELETE) with user, tenant, action, resource, timestamp. Admin UI page to browse/filter.
**Repos:** `volund`, `volund-admin`

### 3.4 Log Aggregation View
**Gap:** Admin can't see agent logs or gateway errors without `kubectl logs`.
**Fix:** Stream logs via NATS to a log aggregation endpoint. Admin UI shows recent logs filterable by service, level, tenant. Start simple: last 100 errors.
**Repos:** `volund`, `volund-admin`

### 3.5 Usage Trends & Charts
**Gap:** Usage page shows raw numbers, no trends. `recharts` is installed but unused.
**Fix:** Add time-series usage endpoint `GET /v1/usage/timeseries?from=&to=&interval=hour|day`. Render line charts for tokens/requests over time, per-model breakdown area charts, cost trend.
**Repos:** `volund`, `volund-admin`, `volund-desktop`

---

## Phase 4: User Experience (P1)

### 4.1 Tenant Switching
**Gap:** Users belonging to multiple tenants can't switch. JWT locks to the first tenant.
**Fix:** Add `GET /v1/auth/tenants` that lists all tenants the user belongs to. Add `POST /v1/auth/switch-tenant` that issues a new JWT scoped to the selected tenant. UI: tenant switcher dropdown in the header.
**Repos:** `volund-auth`, `volund`, `volund-desktop`, `volund-admin`

### 4.2 User Onboarding Flow
**Gap:** New users land on an empty chat screen with no guidance.
**Fix:** First-login detection → guided setup: choose a name, pick an agent profile, start a sample conversation. Show available system agents with descriptions.
**Repos:** `volund-desktop`

### 4.3 Skill Dependency Resolution
**Gap:** Agents reference skills (e.g. "email") but nothing checks if the skill is installed/enabled. User creates a conversation with Email Assistant, email skill isn't installed → silent failure.
**Fix:** When starting a conversation with an agent, check that all required skills are installed and enabled. Show a dialog: "Email Assistant requires the email skill. Install it now?"
**Repos:** `volund-desktop`, `volund`

### 4.4 Notifications
**Gap:** No push notifications when agents complete tasks or conversations get new messages.
**Fix:** Browser Notification API for web. Tauri native notifications for desktop. Subscribe to NATS events for the user's conversations.
**Repos:** `volund-desktop`

### 4.5 Email Transport ✅ (config added)
**Done:** SMTP env vars added to volund-auth (SMTP_HOST, SMTP_PORT, SMTP_USER, SMTP_PASS, SMTP_FROM). Wiring into better-auth email handlers is TODO.
**Repos:** `volund-auth`

---

## Phase 5: Multi-Tenancy & Billing (P2)

### 5.1 Tenant Quota Enforcement
**Gap:** Quota table exists but there's no enforcement. Users can exceed limits.
**Fix:** Check quotas in the LLM router before forwarding requests. Return 429 when exceeded. Admin UI for setting quotas per tenant.
**Repos:** `volund`, `volund-admin`

### 5.2 Cost Tracking
**Gap:** Usage tracks tokens but not cost. `estimated_cost` is always 0.
**Fix:** Model pricing table (input $/1K tokens, output $/1K tokens) populated from the model catalog. Calculate cost on each LLM response. Show in usage dashboards.
**Repos:** `volund`

### 5.3 Per-User Usage
**Gap:** Usage is tenant-level only. Can't see which user burns tokens.
**Fix:** Add `user_id` to usage events. New endpoint `GET /v1/usage/users` for per-user breakdown. Admin UI table.
**Repos:** `volund`, `volund-admin`

### 5.4 Billing Integration
**Gap:** `plan` field exists (free/pro/enterprise) but no payment processing.
**Fix:** Stripe integration — subscription management, usage-based billing, invoicing. Admin UI billing page.
**Repos:** `volund`, `volund-admin`

### 5.5 System Agent Cross-Tenant Dispatch
**Gap:** System agents are visible cross-tenant (fixed) but dispatching a task to one may fail because the profile's `tenant_id` doesn't match the requesting user's tenant.
**Fix:** When dispatching to a system agent, use the profile regardless of tenant_id match. The agent runtime should accept tasks for system profiles from any tenant.
**Repos:** `volund`, `volund-agent`

---

## Phase 6: Developer Experience (P2)

### 6.1 CI/CD for Skills & Agents Repos
**Gap:** READMEs mention "CI validates on merge" but no GitHub Actions exist.
**Fix:** Add workflows: validate skill.json/agent.yaml schema, build Docker images for MCP skills, auto-publish to Forge on merge to main.
**Repos:** `volund-skills`, `volund-agents`

### 6.2 Skill Versioning & Rollback
**Gap:** Publishing a broken skill update has no rollback.
**Fix:** Store version history in the Forge registry. `GET /v1/forge/skills/{name}/versions` lists all versions. `POST /v1/forge/skills/{name}/rollback?version=1.0.0` reverts.
**Repos:** `volund`

### 6.3 Forge Dev Integration
**Gap:** `forge dev` runs a local MCP server but isn't connected to the platform.
**Fix:** `forge dev --connect` registers a temporary skill pointing at the local dev server so agents can use it in real conversations.
**Repos:** `volund-forge`

### 6.4 API Key Management ✅ (API done)
**Done:** POST/GET/DELETE /v1/apikeys endpoints. Keys use sha256 hash, vk_ prefix. UI integration TODO.
**Repos:** `volund`, `volund-desktop`, `volund-admin`

---

## Phase 7: Hardening (P3)

### 7.1 Secret Rotation
**Gap:** No way to rotate `BETTER_AUTH_SECRET` or JWKS keys without downtime.
**Fix:** JWKS rotation is built into better-auth (set `rotationInterval`). For the gateway HS256 secret, support dual-secret validation during rotation window.
**Repos:** `volund-auth`, `volund`

### 7.2 Rate Limiting
**Gap:** better-auth rate limiting is skipped (can't determine IP behind proxy). Gateway has no per-tenant rate limiting.
**Fix:** Configure `trustedProxies` in better-auth. Add token bucket rate limiting in gateway middleware keyed by tenant_id.
**Repos:** `volund-auth`, `volund`

### 7.3 Backup & Restore
**Gap:** No automated PostgreSQL backups.
**Fix:** CronJob for `pg_dump` to S3/MinIO. Document restore procedure. Test disaster recovery.
**Repos:** `volund` (deploy manifests)

### 7.4 Tenant Data Isolation Tests
**Gap:** No automated tests proving one tenant can't access another's data.
**Fix:** Integration test suite: create two tenants, verify tenant A can't read tenant B's conversations, agents, memories, or usage.
**Repos:** `volund`

### 7.5 Session Revocation
**Gap:** Deactivating a user doesn't invalidate their JWT for up to 15 minutes.
**Fix:** Token revocation list in Redis. Gateway checks revocation list on each request. Admin "ban user" action adds all their active sessions to the list.
**Repos:** `volund`, `volund-auth`

---

## Priority Summary

| Phase | Focus | Priority | Done | Remaining |
|-------|-------|----------|------|-----------|
| **1** | Security & Access Control | P0 | 4/5 | 1.5 CSRF |
| **2** | Dynamic LLM Provider Management | P0 | 3/4 | 2.4 per-tenant config |
| **3** | Operational Visibility | P1 | 4/5 | 3.4 log aggregation |
| **4** | User Experience | P1 | 2/5 | 4.2 onboarding, 4.3 skill deps, 4.4 notifications |
| **5** | Multi-Tenancy & Billing | P2 | 2/5 | 5.1 enforcement, 5.4 Stripe, 5.5 cross-tenant dispatch |
| **6** | Developer Experience | P2 | 3/4 | 6.3 forge dev integration |
| **7** | Hardening | P3 | 1/5 | 7.1 rotation, 7.3 backups, 7.4 isolation tests, 7.5 revocation |

---

## Repo Impact Matrix

| Repo | P0 | P1 | P2 | P3 |
|------|----|----|----|----|
| `volund` (gateway) | 1.1,1.3,1.4,1.5,2.1,2.3 | 3.1,3.2,3.3,3.5 | 5.1,5.2,5.3,5.5,6.2,6.4 | 7.1,7.2,7.4,7.5 |
| `volund-admin` | 1.1,1.2,2.2 | 3.1,3.2,3.3,3.4,3.5 | 5.1,5.3,5.4,6.4 | — |
| `volund-auth` | — | — | 4.1,4.5 | 7.1,7.2,7.5 |
| `volund-desktop` | — | 3.5,4.2,4.3,4.4 | 6.4 | — |
| `volund-agent` | — | — | 5.5 | — |
| `volund-forge` | — | — | 6.3 | — |
| `volund-skills` | — | — | 6.1 | — |
| `volund-agents` | — | — | 6.1 | — |
| `volund-operator` | — | 3.1 | — | — |
| `volund-docs` | — | — | — | — |
