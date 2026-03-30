# better-auth Migration — Platform-Wide Auth Unification

## Current State

Auth is custom Go code in `volund/internal/auth/`:
- `service.go` — Register, Login, RefreshToken, LoginOIDC, Logout
- `jwt.go` — HS256 JWT issue/validate (claims: `sub`, `tenant_id`, `role`)
- `oidc.go` — OIDCManager wrapping `go-oidc/v3` + `golang.org/x/oauth2`
- `middleware.go` — HTTP + gRPC middleware extracts JWT → `Claims` in context
- `password.go` — bcrypt hash/verify

The desktop app (`volund-desktop`) has a hand-rolled `VolundAPI` class that calls `/v1/auth/login`, `/v1/auth/register`, `/v1/auth/refresh`, stores JWT in localStorage, and manages auto-refresh with timers.

### What Gets Replaced

| Current | Replaced By |
|---------|------------|
| `auth/service.go` (Register, Login, Refresh, Logout, LoginOIDC) | better-auth server |
| `auth/oidc.go` (OIDCManager, go-oidc/v3) | better-auth OIDC plugin |
| `auth/password.go` (bcrypt) | better-auth email+password plugin |
| `auth/jwt.go` (issue tokens) | better-auth JWT plugin issues tokens |
| Gateway endpoints: `/v1/auth/*` | better-auth handles all auth routes |
| Desktop `volund-api.ts` auth methods | better-auth React client |

### What Stays

| Keep | Why |
|------|-----|
| `auth/jwt.go` — `Validate()` only | Gateway still validates JWTs on every request |
| `auth/middleware.go` | HTTP + gRPC middleware still extracts claims from JWTs |
| `Claims` struct (`sub`, `tenant_id`, `role`) | All API handlers depend on this contract |
| Database tables: `users`, `tenants`, `tenant_members` | better-auth user table bridges to existing tenant model |

---

## Architecture

```
┌─────────────────┐  ┌─────────────────┐  ┌──────────────────┐
│   Admin UI      │  │   Desktop App   │  │   Future Web UI  │
│   (React SPA)   │  │   (Tauri+React) │  │                  │
└────────┬────────┘  └────────┬────────┘  └────────┬─────────┘
         │                    │                     │
         │  better-auth       │  better-auth        │  better-auth
         │  React client      │  React client       │  React client
         │                    │                     │
         └────────────────────┼─────────────────────┘
                              │
                              ▼
                 ┌────────────────────────┐
                 │   better-auth server   │
                 │   (Hono middleware)     │
                 │                        │
                 │  • email/password      │
                 │  • generic OIDC        │
                 │  • session management  │
                 │  • JWT issuance        │
                 │                        │
                 │  Routes: /auth/**      │
                 │  DB: PostgreSQL        │
                 └───────────┬────────────┘
                             │ issues JWT with
                             │ { sub, tenant_id, role }
                             ▼
                 ┌────────────────────────┐
                 │   Gateway (Go)         │
                 │                        │
                 │  JWT validation only   │
                 │  (middleware.go stays)  │
                 │                        │
                 │  /v1/auth/* removed    │
                 │  All other APIs stay   │
                 └────────────────────────┘
```

### Where Does better-auth Run?

**New service: `volund-auth`** — a lightweight Hono server that:
1. Hosts better-auth at `/auth/**`
2. Connects to the same PostgreSQL instance
3. Manages its own tables (`user`, `session`, `account`, `verification`) alongside existing Volund tables
4. Issues JWTs signed with the same `VOLUND_JWT_SECRET` so the gateway accepts them
5. On sign-up/sign-in, hooks into the existing `users` + `tenants` + `tenant_members` tables to populate `tenant_id` and `role` in the JWT

Runs as a separate K8s deployment, exposed on `:3456`.

---

## better-auth Configuration

### Server (`volund-auth/src/auth.ts`)

```typescript
import { betterAuth } from "better-auth";
import { jwt } from "better-auth/plugins";
import { genericOAuth } from "better-auth/plugins";
import { Pool } from "pg";

export const auth = betterAuth({
  database: new Pool({
    connectionString: process.env.DATABASE_URL,
  }),

  emailAndPassword: {
    enabled: true,
    minPasswordLength: 8,
  },

  plugins: [
    // Issue JWTs compatible with the gateway
    jwt({
      jwt: {
        definePayload: async ({ user }) => {
          // Query tenant_members to get tenant_id + role
          // This bridges better-auth users to the Volund tenant model
          return {
            sub: user.id,
            email: user.email,
            tenant_id: user.tenantId,   // from hook
            role: user.role,            // from hook
            iss: "volund",
          };
        },
      },
      jwks: { disabled: true },           // we use HS256 shared secret
    }),

    // Generic OIDC — configured at runtime via env vars
    genericOAuth({
      config: JSON.parse(process.env.OIDC_PROVIDERS || "[]"),
    }),
  ],

  // Hook: after sign-up, create a tenant + tenant_member row
  // Hook: after sign-in, resolve tenant_id + role for JWT
});
```

### Client (shared by admin UI + desktop app)

```typescript
// packages/auth-client/src/index.ts  (or inline in each app)
import { createAuthClient } from "better-auth/react";
import { genericOAuthClient } from "better-auth/client/plugins";
import { jwtClient } from "better-auth/client/plugins";

export const authClient = createAuthClient({
  baseURL: import.meta.env.VITE_AUTH_URL || "http://localhost:3456",
  plugins: [
    genericOAuthClient(),
    jwtClient(),
  ],
});

// Usage:
// const { data: session } = authClient.useSession();
// await authClient.signIn.email({ email, password });
// await authClient.signUp.email({ email, password, name });
// await authClient.signIn.social({ provider: "google" });
// const { token } = await authClient.token();  // JWT for gateway
```

---

## Tenant Bridge

better-auth manages its own `user` table. We need to bridge to Volund's tenant model:

**Option: better-auth hooks + existing tables**

On **sign-up**:
1. better-auth creates the user in its `user` table
2. An `afterSignUp` hook also inserts into Volund's `users` table (or we make better-auth USE the existing `users` table with field mapping)
3. Creates a default tenant in `tenants`
4. Adds `tenant_members` row with role `owner`

On **sign-in**:
1. better-auth authenticates the user
2. A hook queries `tenant_members` to resolve `tenant_id` + `role`
3. These are included in the JWT payload

**Recommended: configure better-auth to use the existing `users` table** by mapping fields, so we don't have two user tables. better-auth supports custom table/field mapping.

---

## Migration Plan

### Phase 1: volund-auth Service (new)

1. Create `volund-auth/` repo
2. Hono server + better-auth with PostgreSQL
3. email/password + generic OIDC
4. JWT plugin configured with `VOLUND_JWT_SECRET`
5. Hooks to bridge tenant model (sign-up creates tenant, sign-in resolves tenant_id+role)
6. Deploy as K8s service in Tiltfile
7. **Test**: sign up → get JWT → call gateway API → works

### Phase 2: Admin UI (new)

1. Create `volund-admin/` with Vite + React + Tailwind + shadcn
2. Uses better-auth React client for auth
3. Gets JWT from better-auth, calls gateway admin APIs directly
4. Pages: dashboard, tenants, agents, skills, providers, usage, settings

### Phase 3: Desktop App Migration

1. Replace `volund-api.ts` auth methods with better-auth client
2. Remove: `login()`, `register()`, `getOIDCProviders()`, `oidcRedirectUrl()`, token auto-refresh logic
3. Add: `authClient.useSession()`, `authClient.signIn.email()`, `authClient.signUp.email()`, `authClient.signIn.social()`
4. Use `authClient.token()` to get JWT for gateway API calls
5. Login component switches from manual form to better-auth hooks
6. **Test**: existing flows still work

### Phase 4: Gateway Cleanup

1. Remove `/v1/auth/register`, `/v1/auth/login`, `/v1/auth/refresh`, `/v1/auth/logout`, `/v1/auth/oidc/*` routes
2. Remove `auth/service.go`, `auth/oidc.go`, `auth/password.go`
3. Keep `auth/jwt.go` (Validate only), `auth/middleware.go`
4. Remove Go dependencies: `go-oidc/v3`, `golang.org/x/oauth2`, `golang.org/x/crypto` (bcrypt)
5. Add env var `VOLUND_AUTH_URL` for health checks / service discovery
6. **Test**: all API calls still authenticate via JWT

---

## Repos Affected

| Repo | Changes |
|------|---------|
| **`volund-auth`** | **New** — Hono + better-auth server |
| **`volund-admin`** | **New** — Admin SPA using better-auth client |
| **`volund`** | Remove auth endpoints + Go auth code, keep JWT validation |
| **`volund-desktop`** | Replace auth code with better-auth client |
| `volund-docs` | This doc + updates to architecture docs |

No changes to: `volund-agent`, `volund-operator`, `volund-proto`, `volund-forge`, `volund-skills`, `volund-agents`

---

## Deployment (Tiltfile)

```python
# volund-auth — better-auth service
docker_build(
    'ghcr.io/ai-volund/volund-auth',
    context='volund-auth',
    dockerfile='volund-auth/Dockerfile',
    live_update=[
        sync('volund-auth/src', '/app/src'),
        run('cd /app && npm run build', trigger=['volund-auth/src/**/*.ts']),
    ],
)
k8s_yaml('volund-auth/deploy/local/auth.yaml')
k8s_resource(
    'volund-auth',
    port_forwards=['3456:3456'],
    resource_deps=['postgres'],
    labels=['platform'],
)
```

---

## Open Questions

1. **Single user table or two?** — better-auth can be configured to use custom table names and field mappings. Can we point it at the existing `users` table? If the schema is incompatible, we use better-auth's tables and bridge via hooks.

2. **Session cookies vs JWT-only?** — better-auth defaults to session cookies. For the desktop app (Tauri), cookies may not work cleanly across origins. The JWT plugin lets us get a bearer token. **Recommendation: use JWT plugin for API auth, cookies for the web UIs.**

3. **CORS** — The auth service runs on `:3456`, gateway on `:8080`, admin UI on `:5174`, desktop on `:5173`. Need CORS configured on the auth service for all origins in dev.

4. **Agent pods** — Agent runtime currently calls the LLM router gRPC without user JWTs (skipped in middleware). This doesn't change.
