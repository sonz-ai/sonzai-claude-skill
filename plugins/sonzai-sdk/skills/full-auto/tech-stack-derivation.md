# Tech-stack derivation (Phase 0c — greenfield, autonomous)

Used by `full-auto` when project-type-detection returns `greenfield`. Derives the 7 tech-stack fields autonomously from the transcript + sensible defaults. **No operator prompts.** If you want the interactive 7-question variant, the operator must invoke `cto-loop` instead — see `../cto-loop/tech-stack-intake.md`.

## Inputs

- `transcript_analysis` (from Phase 0a)

## The 7 fields

Same 7 fields as `cto-loop/tech-stack-intake.md`. Derive in this order.

### 1. backend_lang

| Transcript hint | Choice |
|---|---|
| "TypeScript", "TS", "Node", "Bun", "Deno" | `typescript` |
| "Python", "Py", "FastAPI", "Django", "Flask" | `python` |
| "Go", "Golang" | `go` |
| Mentions a TS-only framework (Next.js, Vite, SvelteKit) but no backend lang | `typescript` |
| Mentions a Python-only framework (FastAPI, Litestar) | `python` |
| Mentions a Go-only framework (Echo, Fiber, Gin) | `go` |
| No signal | `typescript` (default) |

### 2. backend_framework

Run `version-checker` to confirm the current stable version, then pick:

| Backend lang | Default framework |
|---|---|
| `typescript` | `hono` — small, fast, current popular choice for SDK-shaped backends |
| `python` | `fastapi` — current popular choice |
| `go` | `echo` — current popular choice |

Override defaults if the transcript explicitly named a different framework in that ecosystem.

### 3. frontend

| Transcript hint | Choice |
|---|---|
| "web app", "dashboard", "SaaS", "UI" | `next` |
| "mobile", "iOS", "Android" | `embedded` |
| "Telegram", "Discord", "Slack" + "bot" | `embedded` |
| "webhook", "API only", "no UI", "backend" | `api-only` |
| Transcript names a framework explicitly | use that |
| No signal | `api-only` (conservative — fewer moving parts) |

For frontend frameworks, run `version-checker` to pin the current version.

### 4. database

Decision tree:

```
Does the vertical need to STORE state that's NOT in Sonzai's memory layer?
  Examples of "yes": user accounts, conversation logs duplicated for analytics,
                     business records (orders, tickets, KB docs), audit trail
  Examples of "no":  stateless passthrough, webhook bridge, kiosk-style demo

  yes → postgres (default)
  no  → none (stateless)
```

Apply:

| Transcript signal | Choice |
|---|---|
| "auth", "users", "accounts", "login", "signup" | `postgres` |
| "store conversations", "history", "logs", "tickets", "orders" | `postgres` |
| "KB", "documents", "knowledge base" | `postgres` |
| "webhook handler", "passthrough", "bridge", "relay" | `none` |
| "stateless" | `none` |
| Explicit framework named (`mysql`, `sqlite`) | use that |
| No signal but archetype is one of {`companion`, `enterprise-assistant`, `customer-support`, `coach-therapist`} | `postgres` |
| No signal but archetype is `game-npc` or `hybrid-custom` | `postgres` (state-bearing by default) |
| No signal but archetype is `guide-router` (could be stateless) | `none` |

If `postgres`: `version-checker` → current `postgres:<tag>` from Docker Hub.

### 5. auth

| Transcript signal | Choice |
|---|---|
| Names a library (Clerk, Auth.js, Better-Auth, etc.) | use that |
| "no auth", "public", "anonymous" | `none` |
| Mentions auth but no library | by ecosystem: TS → `better-auth`; Python → `fastapi-users`; Go → custom JWT |
| No signal AND database = `postgres` | by ecosystem default (assume auth is needed if DB is needed) |
| No signal AND database = `none` | `none` |

If a library is chosen: `version-checker` → current version.

### 6. orm

Skip entirely if `database == "none"`.

| Backend lang × DB | Default ORM |
|---|---|
| `typescript` × postgres | `drizzle-orm` |
| `typescript` × mysql | `drizzle-orm` |
| `typescript` × sqlite | `drizzle-orm` |
| `python` × postgres | `sqlalchemy` |
| `python` × mysql | `sqlalchemy` |
| `python` × sqlite | `sqlalchemy` |
| `go` × postgres | `sqlc` (compile-time safety) or `bun` |
| `go` × mysql | `sqlc` |

Override if transcript named a different ORM in that ecosystem.

Run `version-checker` for the chosen ORM.

### 7. deploy_target

| Transcript signal | Choice |
|---|---|
| "Fly.io", "Railway", "Cloud Run", "AWS", "Azure", "GCP", "Hetzner", "DO" | use that |
| "on-prem", "customer's own infra", "self-hosted" | `on-prem` |
| "Kubernetes", "k8s" | `k8s-generic` |
| No signal | `tbd` (flag in masterplan risks) |

This field doesn't gate anything — local docker-compose deploy is what `full-auto` does regardless. The field captures intent for the masterplan.

## Output

In-memory state populated identically to `cto-loop/tech-stack-intake.md`:

```yaml
tech_stack:
  backend_lang:           ...
  backend_framework:      ...
  backend_framework_version: ...
  frontend:               ...
  frontend_version:       ...
  db:                     ...
  db_image_tag:           ...
  auth:                   ...
  auth_version:           ...
  orm:                    ...
  orm_version:            ...
  deploy_target:          ...
  verified_at:            <ISO date when version-checker last ran>
  derivation_source:      autonomous   # vs. "interactive" in cto-loop mode
  derivation_notes:       <one line per field on which signal won>
```

## Print

```
Tech stack derived (autonomous, from transcript + defaults; versions verified <date>):
  Backend:   <framework> v<X.Y> on <lang>             [from: <signal>]
  Frontend:  <framework or "api-only" or "embedded"> [from: <signal>]
  Database:  <db> <tag>                                [from: <signal>]
  Auth:      <library> v<X.Y> or "none"                [from: <signal>]
  ORM:       <library> v<X.Y> or "n/a"                 [from: <signal>]
  Deploy:    <target>                                  [from: <signal>]
```

Note the `[from:]` annotations — they help the final report explain what got chosen and why.

## Handoff

Proceed to `answer-derivation.md` (Phase 0d).

## Hard rules

1. **Always-search.** Every version pin must come from a runtime call to `version-checker` — never hardcoded in this file.
2. **Defaults are documented.** If the transcript was silent, the default kicks in AND is annotated as `[from: default]` in the print + `derivation_notes`.
3. **No operator prompts.** This is full-auto. If a field genuinely cannot be derived, use the documented default and flag in `derivation_notes` so the final report surfaces it.
4. **No tenant names in operator hints.** Already redacted by transcript-analysis; verify before saving.
5. **Database default is `none` for unclear cases involving stateless archetypes.** `postgres` is not the default for everything — it's the default when the vertical clearly needs to store its own state.
