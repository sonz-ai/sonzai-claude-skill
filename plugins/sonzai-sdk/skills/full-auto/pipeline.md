# Pipeline (autonomous)

Six phases. Phase 4 is the fixer loop. No operator gates — `full-auto` is fully autonomous.

For the gated semi-auto variant of this same pipeline, see `../cto-loop/pipeline.md`.

Artifacts go under `docs/cto-review/<date>-*.md` (committed to the repo, so the final report has an audit trail) and under `.full-auto/` (transient, gitignored — add to `.gitignore` at the start of Phase 0).

---

## Pre-flight

Before Phase 0a:

1. **Drift check / live OpenAPI fetch:**
   ```bash
   mkdir -p .full-auto
   curl -sSfL https://api.sonz.ai/docs/openapi.json -o .full-auto/openapi.live.json
   ```
   Retry once on failure. Two failures → halt + write `.full-auto/BLOCKED.md`.

2. **Docker check:**
   ```bash
   docker --version && docker compose version
   ```
   Missing → halt + tell operator to install Docker.

3. **`SONZAI_API_KEY` check:**
   ```bash
   test -n "$SONZAI_API_KEY" || grep -q '^SONZAI_API_KEY=..*' .env 2>/dev/null
   ```
   Missing → halt + tell operator to set the key. Autonomous pipeline cannot prompt.

4. **`.gitignore`:** add `.full-auto/` and `.env` if missing.

---

## Phase 0a — Transcript analysis

→ Read `transcript-analysis.md`.

Source resolution order:
1. Arg given (`/full-auto path/to/transcript.txt`) → read that file
2. Prior message contains a transcript-shaped block → use it
3. Operator's current prompt has the transcript inline → use it
4. None of the above → halt

Save to `.full-auto/transcript.txt` for later phases.

Output: in-memory `transcript_analysis` (client goal, archetype hint, scale, capabilities, constraints).

---

## Phase 0b — Project type detection

→ Read `project-type-detection.md`.

Decide: greenfield vs brownfield based on CWD signals (existing manifest, source files, compose, etc.).

Output: `project_type` in {greenfield, brownfield}.

---

## Phase 0c — Tech-stack derivation (greenfield) OR Brownfield audit (brownfield)

Branch on `project_type`:

### Greenfield → `tech-stack-derivation.md`

Derive 7 tech-stack fields autonomously from transcript + sensible defaults. Run `version-checker` subagent to pin current versions / docker tags. **No operator prompts.** Defaults documented in `tech-stack-derivation.md`.

Output: in-memory `tech_stack` with versions + verification date.

### Brownfield → `brownfield-audit.md`

Dispatch auditor subagent (`subagent-prompts/auditor.md`). Reads existing repo files (read-only), produces structured YAML with confidence ratings. Low-confidence detections become risks that the builder downstream treats conservatively.

Output: `docs/cto-review/<date>-brownfield-context.md` (YAML in markdown block).

---

## Phase 0d — Wizard answer derivation

→ Read `answer-derivation.md`.

Derive the 8 sonzai-sdk wizard answers (archetype, integration_path, runtime_mode, capabilities, brand_persona, proactive, scope, byok_posture) from transcript + tech-stack/audit. Unclear values fall back to documented defaults (autonomous mode); cto-loop surfaces them at Gate A instead.

Output: in-memory `sonzai_wizard` with rationales.

---

## Phase 1 — Masterplan assembly

→ Read `masterplan-assembly.md` and `templates/masterplan.md.template`.

Combine Phase 0 outputs into a single doc at `docs/cto-review/<date>-masterplan.md`. Every section filled; verification date embedded; risks listed.

In `full-auto` mode the approval section is emitted but left unmarked (audit-trail uniformity with `cto-loop`).

---

## Phase 2 — Build

→ Read `builder-dispatch.md`. Subagent prompts in `subagent-prompts/builder.md` and `subagent-prompts/version-checker.md`.

Dispatch ONE builder subagent (model: sonnet by default, opus for complex archetypes). It:

1. Reads the masterplan
2. Scaffolds the project per the file-structure section
3. Dispatches `version-checker` before writing any manifest (HARD RULE: always-search)
4. Writes code (backend + frontend if applicable + tests)
5. Commits locally with `feat:` / `chore:` prefixes
6. Returns JSON: `{status, commits, entry_files, smoke_command, version_pins, open_questions}`

Handle `status: needs_context` by re-dispatching with the missing context (filled from in-memory state). Handle `status: blocked` by halting + writing BLOCKED.md.

---

## Phase 3 — Local deploy + QA

→ Read `local-deploy.md` then `qa-loop.md`.

### Local deploy

1. Generate Dockerfile from `templates/dockerfile-<lang>.template` (placeholders resolved via `version-checker`)
2. Generate `docker-compose.yml` from `templates/docker-compose-{postgres,postgres-redis,stateless}.template` (placeholders resolved)
3. Brownfield: if `docker-compose.yml` already exists, merge carefully; never overwrite
4. `docker compose up -d --wait`
5. Run migrations (`drizzle-kit push` / `prisma migrate deploy` / `alembic upgrade head` / etc.)

If boot fails: dispatch fixer (Phase 4) immediately — don't wait for QA. Bounded 3 retries for boot.

### QA

→ `qa-loop.md`. Smoke checks (root URL, /health, DB connectivity) + archetype-specific functional checks (auth signup → chat → memory persistence for `companion`, etc.).

QA failures → Phase 4.

QA passes → Phase 5.

---

## Phase 4 — Feedback iteration (auto-fixer loop)

→ Read `feedback-iteration.md`. Subagent prompt `subagent-prompts/fixer.md`.

When QA fails:

```
cycle = 0
while qa.overall == fail and cycle < 5:
  dispatch fixer with QA report + logs + masterplan
  fixer commits
  docker compose up -d --build --wait
  re-run qa-loop
  cycle += 1
```

After 5 cycles: stop; final report says `qa-failing-after-5-fixes`.

If fixer returns `status: blocked`: stop immediately; final report says `fixer-blocked`.

---

## Phase 5 — Final report

→ Use `final-report.md.template`.

Write `docs/cto-review/<date>-final-report.md` summarizing:

- Archetype, integration path, runtime mode, capabilities
- Tech stack with verified-at dates
- Build commits (range)
- QA outcome (passing / failing-after-5 / fixer-blocked)
- Auto-derived defaults that were used (so operator sees what was guessed)
- Risks from brownfield audit (if applicable)
- Live URL (still running on `docker compose`)
- Operator's next steps (review, push, deploy to prod)

Then exit. App stays running; operator owns `docker compose down` and the push decision.

---

## Failure surface table

| Phase | Halt condition | Output |
|---|---|---|
| Pre-flight | OpenAPI unreachable (twice) | `.full-auto/BLOCKED.md` |
| Pre-flight | Docker missing | halt with install instructions |
| Pre-flight | `SONZAI_API_KEY` missing | halt with "set the key, then re-run" |
| 0a | No transcript anywhere | halt with one-line reason |
| 0a | Transcript is non-Sonzai-shaped (trading/payments/etc.) | `.full-auto/BLOCKED.md` |
| 2 | Builder returns `blocked` | `.full-auto/BLOCKED.md` with builder's blocker |
| 3 | docker compose boot fails after 3 retries | `.full-auto/BLOCKED.md` with logs |
| 4 | Cycle 5 with QA failing | proceed to Phase 5 with status `qa-failing-after-5-fixes` |
| 4 | Fixer returns `blocked` | proceed to Phase 5 with status `fixer-blocked` |

In every halt: leave the working tree as-is. Do not auto-rollback or auto-tear-down.
