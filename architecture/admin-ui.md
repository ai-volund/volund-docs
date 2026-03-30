# VOLUND Admin UI — Architecture & Implementation Plan

## Problem

The platform has two user-facing clients (desktop app via Tauri, web via the same React SPA), but **no admin console**. System administrators currently have no UI to manage tenants, monitor the warm pool, install skills, configure OAuth providers, view cross-tenant usage, or manage system agents. All admin operations require `curl` against `/v1/admin/*` endpoints.

---

## Goals

1. **Dedicated admin web UI** — separate SPA served by the gateway on `/admin`
2. **Platform-wide visibility** — not scoped to a single tenant like the user app
3. **Leverage existing APIs** — use the `/v1/admin/*`, `/v1/tenants`, `/v1/usage/*`, `/v1/forge/*` endpoints already implemented
4. **Add missing admin APIs** where gaps exist (warm pool status, agent instances, platform health)
5. **Same tech stack** as `volund-desktop` for consistency (React 19, Tailwind v4, shadcn/ui, Vite)

---

## What Exists Today

### Existing Admin API Endpoints (gateway)

| Method | Path | Purpose |
|--------|------|---------|
| `POST` | `/v1/admin/agents` | Create system-scoped agent (visible to all tenant users) |
| `POST` | `/v1/admin/skills/{id}/install` | Install a skill for the tenant |
| `DELETE` | `/v1/admin/skills/{id}/install` | Uninstall a skill |
| `GET` | `/v1/admin/skills/installed` | List installed skills |
| `PUT` | `/v1/admin/providers/{id}/credentials` | Set OAuth client_id/secret for a provider |
| `GET` | `/v1/admin/providers` | List all providers (including unconfigured) |
| `DELETE` | `/v1/admin/providers/{id}` | Remove a provider |

### Existing Tenant/Platform Endpoints (admin-accessible)

| Method | Path | Purpose |
|--------|------|---------|
| `POST/GET/PUT/DELETE` | `/v1/tenants[/{id}]` | Full tenant CRUD (`platform_admin` sees all) |
| `GET` | `/v1/tenants/{id}/members` | List tenant members |
| `POST` | `/v1/tenants/{id}/invites` | Send invite |
| `GET` | `/v1/usage/summary` | Usage summary (currently tenant-scoped) |
| `GET` | `/v1/usage/breakdown` | Usage breakdown by model |
| `GET` | `/v1/usage/quota` | Quota status |
| `GET` | `/v1/forge/skills` | Browse skill catalog |
| `GET` | `/v1/healthz` | Gateway health |

### Auth Roles

- `platform_admin` — sees all tenants, full platform access
- `owner` — tenant owner, can create tenants
- `admin` — tenant admin, can manage system agents & providers
- `member` — regular user

---

## New Admin API Endpoints Needed

These don't exist yet and need to be added to the gateway:

| Method | Path | Purpose | Repo |
|--------|------|---------|------|
| `GET` | `/v1/admin/instances` | List all agent instances across tenants (pod name, state, tenant, profile, claimed_at) | `volund` |
| `GET` | `/v1/admin/instances/{id}` | Get instance detail | `volund` |
| `DELETE` | `/v1/admin/instances/{id}` | Force-release an instance | `volund` |
| `GET` | `/v1/admin/warmpool` | Warm pool status (total, available, claimed, by profile) | `volund` |
| `GET` | `/v1/admin/usage/platform` | Cross-tenant usage summary (all tenants aggregated) | `volund` |
| `GET` | `/v1/admin/usage/tenants` | Per-tenant usage breakdown | `volund` |
| `GET` | `/v1/admin/health` | Platform health (gateway, NATS, Postgres, Redis, operator) | `volund` |
| `GET` | `/v1/admin/events` | Recent platform events (agent claims, releases, errors) | `volund` |
| `PUT` | `/v1/admin/tenants/{id}/quota` | Set/update tenant quotas | `volund` |
| `GET` | `/v1/admin/agents` | List all system agents across tenants | `volund` |
| `PUT` | `/v1/admin/agents/{id}` | Update a system agent | `volund` |
| `DELETE` | `/v1/admin/agents/{id}` | Delete a system agent | `volund` |

---

## Admin UI Pages

### 1. Dashboard (`/admin`)
- Platform health status (green/yellow/red for each component)
- Key metrics: total tenants, active agents, warm pool utilization, requests today
- Recent events feed (agent claims, errors, deploys)
- Quick links to common admin actions

### 2. Tenants (`/admin/tenants`)
- Table: all tenants with name, slug, plan, member count, usage, created date
- Click into tenant detail: members, agents, usage, quota settings
- Create tenant, update plan, set quotas
- Invite management

### 3. Agents & Instances (`/admin/agents`)
- **System Agents tab**: CRUD for system-scoped agent profiles
- **Live Instances tab**: all running agent pods — state, tenant, profile, pod name, uptime
- Force-release stuck instances
- Warm pool overview: total capacity, available, claimed, utilization %

### 4. Skills & Forge (`/admin/skills`)
- Installed skills per tenant
- Install/uninstall skills
- Forge catalog browser (search, filter by type/tag)
- Skill detail view with readme

### 5. Providers & Connections (`/admin/providers`)
- List all OAuth providers (configured + unconfigured)
- Set client_id / client_secret for each provider
- Delete provider configurations
- Status indicators (configured/unconfigured, last used)

### 6. Usage & Billing (`/admin/usage`)
- Platform-wide usage: total tokens, requests, cost estimate
- Per-tenant breakdown table
- Per-model breakdown
- Quota management: view/set per-tenant quotas
- Time range selector

### 7. Settings (`/admin/settings`)
- LLM provider configuration (which providers are enabled, API keys)
- Platform configuration (OIDC providers, storage backend)
- Audit log viewer

---

## Technical Architecture

### Project: `volund-admin`

```
volund-admin/
├── index.html
├── package.json
├── vite.config.ts
├── tsconfig.json
├── tsconfig.app.json
├── tsconfig.node.json
├── src/
│   ├── main.tsx
│   ├── App.tsx
│   ├── index.css              # Tailwind v4 + shadcn theme (shared from desktop)
│   ├── lib/
│   │   ├── admin-api.ts       # API client for all admin + platform endpoints
│   │   ├── utils.ts           # cn() helper
│   │   └── use-theme.ts       # Dark/light theme hook
│   ├── components/
│   │   ├── layout.tsx         # Admin shell: sidebar + header + outlet
│   │   ├── admin-sidebar.tsx  # Navigation sidebar
│   │   ├── login.tsx          # Admin login (reuses /v1/auth/login)
│   │   ├── error-boundary.tsx
│   │   ├── stat-card.tsx      # Dashboard metric card
│   │   ├── status-badge.tsx   # Health status indicator
│   │   ├── data-table.tsx     # Reusable sortable/filterable table
│   │   └── ui/               # shadcn/ui primitives (copied from desktop)
│   └── pages/
│       ├── dashboard.tsx
│       ├── tenants.tsx
│       ├── tenant-detail.tsx
│       ├── agents.tsx
│       ├── instances.tsx
│       ├── skills.tsx
│       ├── providers.tsx
│       ├── usage.tsx
│       └── settings.tsx
```

### Dependencies (same as desktop minus Tauri)

- `react` + `react-dom` 19.x
- `react-router-dom` 7.x
- `tailwindcss` v4 + `@tailwindcss/vite`
- `shadcn` (button, card, input, dialog, dropdown-menu, table, badge, skeleton, tooltip, separator, tabs, select)
- `lucide-react` for icons
- `class-variance-authority` + `clsx` + `tailwind-merge`
- `recharts` for usage charts (new — desktop doesn't have charts yet)

### Serving Strategy

The built admin SPA is served by the gateway as static files under `/admin`:

```go
// In volund/internal/gateway/server.go
// Serve admin UI static files from embedded or mounted directory
mux.Handle("/admin/", http.StripPrefix("/admin", spaHandler(adminDistDir)))
```

For local dev, Vite dev server runs on `:5174` and proxies API calls to the gateway on `:8080`.

### Tiltfile Integration

```python
# volund-admin — admin UI dev server
local_resource(
    'volund-admin',
    serve_cmd='cd volund-admin && npm run dev',
    deps=['volund-admin/src'],
    labels=['ui'],
)
```

---

## Implementation Phases

### Phase A: Scaffold + Auth + Dashboard
1. Create `volund-admin` project (Vite + React + Tailwind + shadcn)
2. Copy shared UI primitives from `volund-desktop`
3. Admin login page (reuses `/v1/auth/login`, checks for admin/owner role)
4. Layout shell with sidebar navigation
5. Dashboard page with health check + basic stats

### Phase B: Tenant & Agent Management
1. Tenants list + detail pages
2. System agents CRUD
3. Add `GET /v1/admin/agents` endpoint to gateway
4. Live instances page (requires new `GET /v1/admin/instances` endpoint)
5. Warm pool status (requires new `GET /v1/admin/warmpool` endpoint)

### Phase C: Skills, Providers, Usage
1. Skills management page (install/uninstall, forge browser)
2. Providers page (list, configure credentials, delete)
3. Usage page with cross-tenant breakdown (requires new platform-wide usage endpoints)
4. Quota management UI

### Phase D: Gateway Integration + Polish
1. Embed built admin SPA in gateway Docker image
2. Add SPA handler to gateway server
3. Add to Tiltfile for live dev
4. Audit log viewer
5. Platform events feed on dashboard
6. Dark mode, responsive polish

---

## Repos Affected

| Repo | Changes |
|------|---------|
| `volund-admin` | **New repo** — entire admin SPA |
| `volund` | New admin API endpoints, SPA static file serving, Dockerfile update |
| `volund-docs` | This document + architecture updates |

`volund-agent`, `volund-operator`, `volund-proto`, `volund-forge`, `volund-skills`, `volund-agents`, `volund-desktop` — **no changes needed** for the admin UI itself (the `feature/admin-ui` branch on those repos can stay empty or be deleted).

---

## Open Questions

1. **Separate repo vs subdirectory?** — Could live as `volund-admin/` at the workspace root (separate repo) or as `volund/web/admin/` inside the platform core. Separate repo is cleaner for CI but adds complexity. **Recommendation: separate repo** matching the pattern of `volund-desktop`.
2. **Embed in gateway binary or mount at deploy?** — Embedding via `go:embed` is simpler for single-binary deployment. Mounting as a volume allows updating the UI without rebuilding Go. **Recommendation: embed for production, volume mount for dev**.
3. **RBAC enforcement** — Current admin endpoints use `RequireAuth` but not all check for admin role. Need to add a `RequireAdmin` middleware. Which roles get admin UI access? **Recommendation: `platform_admin` + `owner` + `admin`**.
