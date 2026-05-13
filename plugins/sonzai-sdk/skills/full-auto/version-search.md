# Always-search-current-state (Hard Rule)

**Never recommend a package version, install command, docker image tag, or library API from training-data memory.** Verify at runtime, every time.

## Why

LLM training data lags real-world state by months to over a year. Recommending "Next.js 14" when 16 is current, or "postgres:14" when 17 is the current LTS, or a deprecated install flag, propagates stale state into every greenfield project the skill bootstraps. A skill that scaffolds production code MUST search before recommending.

This applies to:

- Tech-stack intake (Phase 0c) — option lists and pinned versions
- Masterplan tech-stack section (Phase 1) — every version + image tag
- Builder subagent writes (Phase 2) — `package.json`, `requirements.txt`, `go.mod`, `Dockerfile`, `docker-compose.yml`, install scripts
- Skill content authoring — if this skill cites a version, it must be the current one at write-time AND say "as of <date>"

## Verification commands

Run the right one for the ecosystem.

| Ecosystem | Command | Output to extract |
|---|---|---|
| npm | `npm view <pkg> version` | `latest` dist-tag version |
| npm (alternate) | `npm view <pkg> dist-tags --json` | full dist-tags object |
| pip / PyPI | `pip index versions <pkg>` (newer pip) OR `curl -s https://pypi.org/pypi/<pkg>/json \| jq -r .info.version` | latest stable |
| Go modules | `go list -m -versions <module>` | space-separated version list, take last |
| Cargo | `cargo search <crate> --limit 1` | parse the first line's version |
| RubyGems | `gem list <pkg> -r` | latest |
| Docker Hub (official images) | `curl -s "https://hub.docker.com/v2/repositories/library/<image>/tags?page_size=20" \| jq -r '.results[].name' \| head -10` | recent tags; pick stable (not `latest`, not `-alpha`, not `-rc`) |
| Docker Hub (org images) | `curl -s "https://hub.docker.com/v2/repositories/<org>/<image>/tags?page_size=20" \| jq -r '.results[].name'` | same |
| GHCR / ECR / private | `docker manifest inspect <image>:<tag>` (requires login) | manifest |
| Framework getting-started | `WebFetch` the framework's official docs page | install command, current version mentioned |

## Resolving "current stable"

For docker images, "latest" tag is often not the right choice (it can move under you). Prefer:

- A **specific major.minor** tag (e.g., `postgres:17` over `postgres:latest`)
- An **alpine variant** when image size matters (`postgres:17-alpine`)
- A **digest pin** for reproducibility in production (`postgres:17@sha256:...`) — but for local docker-compose, the major.minor tag is enough

When you call the version-checker subagent, ask for the "current stable major.minor for production use" — not "latest".

## When to use `version-checker` subagent vs direct command

- **In Phase 0c (greenfield intake)**: dispatching `version-checker` for the 7 question option lists is overkill. Use direct `npm view` / `pip index versions` calls inline; cache the results in-memory for the masterplan.
- **In Phase 2 (builder)**: the builder subagent SHOULD invoke `version-checker` because (a) it dispatches multiple at once efficiently and (b) it returns structured output the builder can parse.
- **Documentation steps** (skill content, README, this file): use `WebFetch` directly when authoring; cite the date.

## Caching within a single `cto-loop` run

Once a version is fetched, cache it in in-memory state with a key like:

```
version_cache:
  "npm:hono":          { version: "4.X.Y", fetched_at: "2026-MM-DD" }
  "docker:postgres":   { tag: "17-alpine", fetched_at: "2026-MM-DD" }
  "pypi:fastapi":      { version: "0.X.Y", fetched_at: "2026-MM-DD" }
```

Subsequent references during the same run reuse the cache. Don't re-query.

Cache lifetime: the duration of one `cto-loop` invocation. Do NOT persist across runs.

## What NOT to do

- ❌ `"next": "^14.0.0"` written from memory because you "remember Next 14 is current"
- ❌ `postgres:14` in docker-compose because you "remember 14 is the LTS"
- ❌ `pip install fastapi==0.100.0` because that's a version you saw in training
- ❌ Recommending a deprecated CLI flag because the install page changed but you didn't fetch
- ❌ Skipping the search when you're "confident" — confidence ≠ recency

## Failure mode if skipped

The masterplan ships with `"next": "^14.0.0"` and the builder writes that into `package.json`. The operator at Gate A doesn't notice. Phase 3 boots, but the app is using a Next version 2 years stale. CTO reviews at Gate B, doesn't catch it (live app works), but in 3 months the project has a CVE and the team blames the skill.

This rule is the difference between a tool that bootstraps current state and a tool that's a stale-state factory.

## Audit trail

Every version-pinned entry in the masterplan should carry the verification date in a `Verified` column (see `templates/masterplan.md.template` §3). At review time, the operator can spot-check: if today's date is months past the verification date and no re-run has happened, treat with suspicion.

## Hard rules

1. **No version from memory.** Period.
2. **Verify at the time of write**, not the time of design.
3. **Pin the version + the date** in artifacts (masterplan, generated manifests).
4. **Prefer major.minor for docker tags**, not `latest`.
5. **Cache within a run; never across runs.**
6. **If the registry is unreachable**, halt and tell the operator — do NOT fall back to memory. Network failure is operator-visible; silent staleness is not.
