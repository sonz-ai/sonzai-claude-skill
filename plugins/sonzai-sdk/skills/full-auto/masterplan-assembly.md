# Masterplan assembly (Phase 1)

Combine Phase 0 outputs into a single document at `docs/cto-review/<date>-masterplan.md`. This doc is the artifact behind Gate A.

## Inputs (in-memory from Phase 0)

- `transcript_analysis` (Phase 0a output)
- `project_type` (`greenfield` | `brownfield` from 0b)
- `tech_stack` (0c-G output) **OR** `audit` (0c-B output)
- `sonzai_wizard` (0d output, with rationales)
- `open_questions` (anything Phase 0d marked `unclear`)

## Output

A single markdown file at:

```
docs/cto-review/<YYYY-MM-DD>-masterplan.md
```

Use `templates/masterplan.md.template` as the skeleton. Fill in every section from the inputs above. If a section has no content (e.g., brownfield audit section in a greenfield run), include it with `(N/A — greenfield)` so the operator knows it's intentionally empty, not forgotten.

## Required sections (per `templates/masterplan.md.template`)

1. **Header** — date, mode, project name, status (`awaiting approval`)
2. **Client goals** — from transcript
3. **Sonzai integration** — the 8 wizard answers with rationales
4. **Tech stack** — from intake or audit, with version pins + verification date
5. **Architecture** — high-level diagram (text or mermaid), service boundaries, data flow
6. **File structure** — top-level tree of what will be created or modified
7. **docker-compose plan** — services, ports, volumes, env vars (drives Phase 3 deploy)
8. **Scope** — in/out lists, explicit
9. **Risks & open questions** — anything ambiguous; mandatory if Phase 0d produced any
10. **CTO review notes** — empty block for operator to write in at Gate A
11. **Approval** — checkbox row (operator marks at Gate A)

## docker-compose plan section — derivation

This section drives Phase 3. Be explicit:

```yaml
services:
  app:
    build: .
    ports: ["${APP_PORT}:${APP_PORT}"]
    env_from: [.env]
    depends_on: [db]
  db:        # only if tech_stack.db != "none"
    image: postgres:${POSTGRES_TAG}
    volumes: [pgdata:/var/lib/postgresql/data]
    healthcheck: pg_isready
  redis:     # only if app declares cache need
    image: redis:${REDIS_TAG}
```

Cross-check against the chosen tech stack:
- If `tech_stack.db == "none"` → omit the `db` service entirely
- If brownfield and `audit.database.type` was already in existing compose → reuse, don't add a duplicate
- Frontend? If Q3 was `next` / `vite` / `astro` / `sveltekit` and the layout is a separate frontend service, add it; if Next.js with `next start` colocated with backend, single service is fine

## Architecture section — derivation

Write a 2-paragraph high-level description:

- Paragraph 1: data flow at the level of "end user → frontend → backend → sonzai SDK → sonzai platform; vertical state lives in postgres"
- Paragraph 2: any non-obvious decisions worth flagging (e.g., "auth uses Better-Auth with email magic links; sessions stored in postgres alongside business data; Sonzai SDK called per-user with the authenticated user's ID as the agent ID")

A text diagram is enough; mermaid is fine if it clarifies.

## File structure section — derivation

For **greenfield**: emit a top-level tree of what the builder will create. Be concrete:

```
<project-root>/
├── package.json
├── tsconfig.json
├── Dockerfile
├── docker-compose.yml
├── .env.example
├── src/
│   ├── server.ts            # Hono entrypoint, mounts routes
│   ├── routes/
│   │   ├── auth.ts          # Better-Auth integration
│   │   ├── chat.ts          # Sonzai SDK call
│   │   └── webhooks.ts      # if applicable
│   ├── db/
│   │   ├── schema.ts        # Drizzle schema (users, conversations)
│   │   └── client.ts
│   └── lib/
│       └── sonzai.ts        # SDK init
├── app/                      # Next.js if applicable
│   └── ...
└── tests/
    └── chat.test.ts
```

For **brownfield**: emit a diff-style tree showing only what will be added or modified:

```
existing-repo/
├── src/
│   ├── lib/
│   │   └── sonzai.ts        # NEW — SDK init
│   ├── routes/
│   │   └── chat.ts          # NEW — Sonzai-backed chat endpoint
│   └── db/
│       └── schema.ts        # MODIFY — add conversations table
└── docker-compose.yml       # MODIFY — already has postgres, no compose changes
```

Be precise about NEW vs MODIFY — the builder uses this to decide whether to create or edit each file.

## Hard rules

1. **Single doc, single file.** No multi-file masterplans. Operator review surface is one path.
2. **Every version is pinned + dated.** From `tech_stack.verified_at` or `audit.verified_at`. Stale dates surface to the masterplan as a risk.
3. **No tenant names.** Should already be redacted upstream, but verify before writing the file.
4. **Risks section is mandatory.** If you found no risks, write `(no risks identified)` explicitly — don't omit the section.
5. **Approval section at the bottom.** Included in every masterplan regardless of mode (cto-loop marks it; full-auto leaves it unmarked but emits it for audit-trail uniformity).

## Handoff

After writing the file, proceed to `builder-dispatch.md` (Phase 2).

**Mode note:** In `cto-loop` mode, Gate A (operator approval) inserts itself here — see `../cto-loop/masterplan-gate.md`. In pure `full-auto`, the masterplan is the build's input artifact and goes straight to the builder.
