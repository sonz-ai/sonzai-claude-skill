# Brownfield audit (Phase 0c — brownfield, autonomous)

Used by `full-auto` when project-type-detection returns `brownfield`. The auditor subagent reads the existing repo, detects the stack, and writes findings directly to `docs/cto-review/<date>-brownfield-context.md`. **No operator prompts.** If you want detect-and-confirm UX, the operator must invoke `cto-loop` — see `../cto-loop/brownfield-audit.md`.

## How to run

Dispatch the auditor subagent (see `subagent-prompts/auditor.md`). Pass:

- Operator's CWD
- The transcript analysis output (so the auditor knows what kind of integration is coming and can prioritize relevant areas)

## What the auditor reads (read-only)

| File / dir | What to extract |
|---|---|
| `package.json` | name, scripts, dependencies, devDependencies, engines |
| `requirements.txt` / `pyproject.toml` / `Pipfile` | Python deps + Python version |
| `go.mod` | Go version + modules |
| `Cargo.toml` | Rust crate deps |
| `Gemfile` / `pom.xml` | Ruby / Java deps |
| `docker-compose.yml` / `compose.yml` | services, images, ports, env |
| `Dockerfile` | base image, build steps |
| `.env.example` / `.env.local` (read keys only, never values) | which env vars are expected |
| `prisma/schema.prisma` | Prisma ORM + schema |
| `drizzle.config.{ts,js}` + `**/schema.ts` | Drizzle ORM + schema |
| `alembic.ini` + `alembic/versions/*` | SQLAlchemy migrations |
| `migrations/*.sql` | raw SQL migrations (any ORM) |
| `next.config.{js,ts}` | Next.js + version |
| `vite.config.{js,ts}` | Vite + framework hint |
| `astro.config.mjs` | Astro |
| `svelte.config.js` | SvelteKit |
| `tailwind.config.{js,ts}` | Tailwind in use |
| `jest.config*` / `vitest.config*` / `pytest.ini` / `*_test.go` | test framework |
| `README.md` | human-written context, intent, install instructions |
| `**/*.{ts,tsx,js,py,go}` (grep, not full read) | search for `sonzai`, `@sonzai-labs/agents`, `SONZAI_API_KEY`, sdk init patterns |

## Output schema

The auditor returns YAML:

```yaml
audit:
  backend:
    lang:          typescript
    framework:     hono
    version:       <verified at audit time>
    confidence:    high
  frontend:
    framework:     next
    version:       <verified at audit time>
    confidence:    high
  database:
    type:          postgres
    image_tag:     <from compose file, if present>
    confidence:    high
  orm:
    type:          drizzle
    version:       <verified at audit time>
    confidence:    medium
  auth:
    type:          better-auth
    confidence:    low
  test_framework:
    type:          vitest
    confidence:    high
  sonzai_sdk:
    installed:     true
    version:       <if pinned in deps>
    integration:   partial   # not-installed | installed-unused | partial | full
    notes:         "Found chat handler but no memory layer wired"
  notable_patterns:
    - "Monorepo with apps/web + apps/api"
    - "API uses tRPC for internal contract"
  risks:
    - "Existing auth uses deprecated next-auth v4"
```

Write the YAML output to `docs/cto-review/<date>-brownfield-context.md` (wrapped in a markdown ` ```yaml ` block).

## Autonomous handling of low-confidence detections

In autonomous mode, low-confidence detections do NOT pause for operator confirmation. Instead:

1. The detection still goes into the audit YAML with `confidence: low`
2. Each low-confidence row also goes into `risks:` with text like "Auth detected as `better-auth` with low confidence (only inferred from imports). Builder should verify against the actual auth flow before adding code that depends on it."
3. The final-report at end of full-auto run prominently lists these risks so the operator sees what the autonomous pipeline guessed

The builder subagent reads the audit (including risks) and prefers fail-safe interpretations on low-confidence rows. E.g., if `auth: better-auth (low)`, the builder uses Better-Auth's public API and avoids deep-integrating with auth internals it can't verify.

## Print to operator

```
Brownfield audit complete (autonomous; full-auto does not confirm):

  Layer        Detected                              Confidence
  ─────        ────────                              ──────────
  Backend      <fwk> v<X> on <lang>                  high
  Frontend     <fwk> v<X>                            high
  Database     <db> <tag>                            high
  ORM          <orm> v<X>                            medium
  Auth         <auth>                                LOW    ⚠️
  Tests        <test fwk>                            high
  Sonzai SDK   v<X> (<integration>)                  high

Risks (will appear in masterplan + final-report):
  - Auth detected as <auth> with low confidence
  - <other risks>

Written to: docs/cto-review/<date>-brownfield-context.md
```

## Handoff

Proceed to `answer-derivation.md` (Phase 0d). The audit feeds the wizard answer derivation (especially capabilities and integration_path).

## Hard rules

1. **Read-only.** The auditor never edits, deletes, or creates files (other than the `brownfield-context.md` it writes).
2. **No secret values read.** `.env*` files are parsed for keys only.
3. **No tenant-named patterns.** Redact tenant names in `notable_patterns` and `risks`.
4. **Low confidence → risk row.** Every `confidence: low` detection MUST also appear in `risks:` with mitigation notes.
5. **Confidence is honest.** Don't round up to `high` for terseness. The final report uses confidence to explain post-hoc what went wrong if the build doesn't match operator expectations.
6. **No operator prompts in full-auto.** The audit output is what it is; the masterplan + final report surface the uncertainty.
