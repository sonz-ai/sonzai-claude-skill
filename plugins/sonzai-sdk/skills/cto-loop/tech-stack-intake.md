# Tech-stack intake (Phase 0c — greenfield, interactive)

**Overlay on `../full-auto/tech-stack-derivation.md`.** Same 7 fields, but asks the operator interactively instead of deriving from transcript + defaults.

Used when `cto-loop` is the active skill and project-type-detection returned `greenfield`. In `full-auto`, the equivalent phase runs `../full-auto/tech-stack-derivation.md` autonomously — no operator prompts.

## Inputs

- `transcript_analysis` (from Phase 0a)

## Pre-fill from transcript

Before asking each question, check `transcript_analysis`. If the transcript explicitly named a choice ("build it in Go with Fiber", "use Next.js"), pre-fill that answer. The question then becomes a confirm step: "Transcript said `<X>`. Keep, or change?"

This avoids asking 7 questions when the transcript already answers 4 of them.

## The 7 questions

Same as `../full-auto/tech-stack-derivation.md` table — every option list is regenerated at runtime by `version-checker` (see `../full-auto/version-search.md`).

### Q1. Backend language

```
Backend language for the vertical app?
  1) TypeScript / JavaScript
  2) Python
  3) Go
  4) Other — type the name
```

Save: `tech_stack.backend_lang`.

### Q2. Backend framework

Run `version-checker` for the chosen lang's top-3 current popular frameworks:

```
Backend framework? (top 3 for <lang>, verified <date>)
  1) <framework A> v<X.Y>  — <one-line description>
  2) <framework B> v<X.Y>  — <description>
  3) <framework C> v<X.Y>  — <description>
  4) Other — type name + version
```

Save: `tech_stack.backend_framework` + `tech_stack.backend_framework_version`.

### Q3. Frontend

```
Frontend?
  1) Next.js
  2) Vite + React
  3) SvelteKit
  4) Astro
  5) API only — no frontend (Telegram bot, mobile-only, etc.)
  6) Embedded — caller already has the UI (Slack app, Discord bot, mobile chat)
  7) Other
```

Pin current version via `version-checker` if a framework chosen.

### Q4. Database

```
Database for the vertical's own state?
Note: this is for YOUR app state — Sonzai handles its own memory layer separately.

  1) postgres (recommended for stateful business logic)
  2) mysql
  3) sqlite (dev / single-instance only)
  4) no DB — stateless app (webhook handler, passthrough, etc.)
  5) Other
```

If postgres: pin docker tag via `version-checker`. If "no DB": skip Q6.

### Q5. Auth library

```
Auth for end-users of the vertical?
Note: this is for YOUR app users — the Sonzai SDK uses an API key, separate concern.

  Options depend on ecosystem; for TS:
    1) Clerk
    2) Auth.js
    3) Better-Auth
    4) Lucia
    5) Supabase Auth
    6) Custom JWT
    7) No auth — public app
    8) Other

  For Python: Authlib / FastAPI-Users / custom JWT / no auth / Other
  For Go:     clerk-go / custom JWT / no auth / Other
```

Run `version-checker` for chosen.

### Q6. ORM / DB layer

Skip if Q4 = "no DB".

```
ORM / DB access layer? (options depend on lang + DB)
  TypeScript: Drizzle / Prisma / Kysely / raw pg
  Python:     SQLAlchemy / Tortoise / SQLModel / raw psycopg
  Go:         sqlc / GORM / Bun / raw database/sql
```

### Q7. Deploy target hint

```
Production deploy target hint?
(cto-loop only deploys locally with docker-compose; this captures intent.)
  1) Fly.io
  2) Railway
  3) Google Cloud Run
  4) AWS
  5) VPS (DigitalOcean, Hetzner)
  6) On-prem
  7) TBD
```

## Output

Identical schema to `../full-auto/tech-stack-derivation.md`, with `derivation_source: interactive`:

```yaml
tech_stack:
  backend_lang:              ...
  backend_framework:         ...
  backend_framework_version: ...
  frontend:                  ...
  frontend_version:          ...
  db:                        ...
  db_image_tag:              ...
  auth:                      ...
  auth_version:              ...
  orm:                       ...
  orm_version:               ...
  deploy_target:             ...
  verified_at:               ...
  derivation_source:         interactive
  derivation_notes:          <per-field: which question, what operator picked>
```

## Print after all 7

```
Tech stack chosen (interactive, versions verified <date>):
  Backend:   <framework> v<X.Y> on <lang>
  Frontend:  <framework> v<X.Y>             (or "API only" / "embedded")
  Database:  <db> <tag> + <orm> v<X.Y>      (or "stateless")
  Auth:      <library> v<X.Y>               (or "no auth")
  Production target hint: <target>
```

Then proceed to `../full-auto/answer-derivation.md` (Phase 0d).

## Hard rules

1. **One question at a time.** Don't batch — back-tracking becomes awkward.
2. **Pre-fill from transcript when possible.** Don't ask a question the transcript already answered.
3. **Always-search every version.** Every option list and every pinned version comes from runtime registry / docs queries — never hardcoded here.
4. **No mandatory postgres.** Q4 offers "no DB" and that branch skips Q6.
5. **No tenant names** in operator's freeform "Other" entries; redact when saving.
6. **The output schema MUST match `tech-stack-derivation.md`** so downstream phases (answer-derivation, masterplan-assembly, builder) consume both autonomous and interactive outputs identically.
