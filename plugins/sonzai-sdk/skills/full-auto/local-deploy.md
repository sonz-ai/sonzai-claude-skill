# Local deploy (Phase 3)

After the builder reports `status: done`, generate (or use the builder-generated) `Dockerfile` + `docker-compose.yml`, boot the stack, and run smoke + QA.

## Pre-flight

1. **Docker installed?** `docker --version` and `docker compose version`. Should already have been checked in Phase 0 pre-flight, but re-check here.
2. **Port available?** Check the masterplan's app port (e.g., 3000, 8080). If something is already listening on that port, ask operator: "Port 3000 is in use by `<process>`. Pick another port or stop the existing process."
3. **`.env` present?** If `.env` doesn't exist but `.env.example` does, copy it (`cp .env.example .env`).
4. **`SONZAI_API_KEY` set?** Check `$SONZAI_API_KEY` env var OR a populated `SONZAI_API_KEY=` line in `.env`. If missing or empty:
   - **`full-auto` mode:** abort. Print "Set `SONZAI_API_KEY` in your environment (or in `.env`) before running full-auto. The autonomous pipeline cannot prompt you interactively." Exit.
   - **`cto-loop` mode:** ask the operator inline ("Paste your `SONZAI_API_KEY` — it goes into `.env`, never committed — or type `skip` to halt."). Wait for response.

Double-check `.gitignore` includes `.env` before any commit. Never commit a populated `.env`.

## Generate Dockerfile (if builder didn't)

Choose the template based on `tech_stack.backend_lang`:

- TypeScript / JavaScript → `templates/dockerfile-ts.template`
- Python → `templates/dockerfile-py.template`
- Go → `templates/dockerfile-go.template`

Resolve placeholders via `version-checker`:

- `BASE_IMAGE_TAG` — current stable for the language (e.g., `node:22-alpine`, `python:3.13-slim`, `golang:1.24-alpine`)
- `APP_PORT` — from masterplan
- `ENTRY` — from builder output (`entry_files[0]`)

Write to `<cwd>/Dockerfile`.

## Generate docker-compose.yml

Choose the template based on the masterplan's stack:

| Condition | Template |
|---|---|
| `tech_stack.db == "postgres"` AND no cache | `templates/docker-compose-postgres.template` |
| `tech_stack.db == "postgres"` AND cache (redis) declared | `templates/docker-compose-postgres-redis.template` |
| `tech_stack.db == "none"` | `templates/docker-compose-stateless.template` |
| `tech_stack.db == "mysql"` | adapt postgres template; swap `image:` to `mysql:<tag>` (use version-checker for current tag) |
| `tech_stack.db == "sqlite"` | use stateless template; sqlite is in-app, no compose service needed |

Resolve placeholders via `version-checker` (always-search):

- `POSTGRES_TAG` — current stable (e.g., `17-alpine`)
- `REDIS_TAG` — current stable (e.g., `7-alpine`)
- `APP_PORT` — from masterplan
- `DB_PORT` — typically `5432` for postgres, `3306` for mysql
- `DB_VOLUME_NAME` — e.g., `pgdata`

Write to `<cwd>/docker-compose.yml`. If a `docker-compose.yml` already exists (brownfield), **DO NOT overwrite** — print a diff and ask: "Existing `docker-compose.yml` found. Merge changes (recommended) or replace?"

## Boot

```bash
docker compose up -d --wait
```

The `--wait` flag blocks until all healthchecks pass. If your services don't have healthchecks (most templates here include one for `db`; the `app` service often doesn't), `--wait` may exit early. Fallback:

```bash
docker compose up -d
# then poll for app readiness:
for i in $(seq 1 30); do
  curl -sf http://localhost:${APP_PORT}/ > /dev/null && break
  sleep 2
done
```

If the boot fails (compose returns non-zero, or app never responds):

1. Run `docker compose logs --tail 100` and capture output
2. Dispatch fixer subagent with the failure logs as input (see `feedback-iteration.md` — different from feedback iteration, but same fixer)
3. Fixer reads logs, fixes the issue (often: missing env var, typo in compose, port conflict, package install failure), commits
4. `docker compose down && docker compose up -d --build --wait` (force rebuild)
5. Retry the boot check. Bounded 3 retries total before surfacing to operator with `status: deploy_failed`

## Migrations

After successful boot, if the masterplan declares a DB:

```bash
# choose based on ORM:
docker compose exec app pnpm drizzle-kit push        # Drizzle
docker compose exec app pnpm prisma migrate deploy   # Prisma
docker compose exec app alembic upgrade head         # SQLAlchemy
docker compose exec app go run ./cmd/migrate up      # custom Go
# raw SQL:
docker compose exec db psql -U postgres -f /migrations/init.sql
```

The exact command depends on what the builder set up. The builder's `entry_files` list usually surfaces the migration entrypoint.

If migrations fail: dispatch fixer with the migration logs. Bounded 3 retries.

## Hand off to QA

Once `docker compose up` is healthy AND migrations applied:

1. Save in-memory state: `deploy_status = "up"`, `app_url = "http://localhost:${APP_PORT}"`, `compose_project = "<dir-name>"`
2. Read `qa-loop.md` next.

## Hard rules

1. **`docker compose up` is the deploy.** No `kubectl`, no remote `fly deploy`, no `gcloud run`. Local Docker only.
2. **Never overwrite an existing `docker-compose.yml`** in brownfield without operator confirmation.
3. **`.env` must exist** before booting. Skill copies from `.env.example` if needed but ALWAYS asks operator to fill it before running compose.
4. **Version-search before writing any tag.** This file references templates; templates contain `${VAR}` placeholders; you fill the placeholders via version-checker at deploy time.
5. **Bounded retries.** 3 attempts for boot, 3 for migrations. After that, surface to operator — don't grind.
6. **Don't `docker compose down -v`** on failure (the `-v` removes volumes). On retry, `docker compose down` is fine, but volumes (postgres data) should survive across retries within one cto-loop run.
