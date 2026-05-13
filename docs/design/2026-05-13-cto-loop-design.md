# cto-loop Skill — Design Spec

> **Status:** Shipped v1.6.0 with one architectural change from this design (see note below)
> **Author:** Claude Code (Opus 4.7)
> **Date:** 2026-05-13
> **Target version:** v1.6.0 (minor, additive)
> **Plugin:** `plugins/sonzai-sdk/` (public)
>
> **As-built note (2026-05-13 post-feedback):** This design originally put all the new infrastructure (docker-compose deploy, version-checker, builder/fixer subagents, brownfield audit, etc.) inside `cto-loop/`. The operator pushed back: "CTO loop is just semi-auto (with gates), full-auto should be the exact same thing." That was correct. The shipped v1.6.0 puts shared core in `full-auto/`, leaving `cto-loop/` as a thin 6-file overlay (SKILL.md, pipeline.md, interactive `tech-stack-intake.md`, detect-and-confirm `brownfield-audit.md`, Gate A + Gate B files, operator-feedback `feedback-iteration.md`). Phases 0a-b, 0d, 1, 2, 3, and 5 — plus all subagent prompts and docker-compose templates — live in `full-auto/` and are referenced by `cto-loop` via `../full-auto/<file>.md`. The file structure in §5 below shows the ORIGINAL design; the actual structure shipped is documented in `cto-loop/SKILL.md` (Pipeline overlay table) and `CHANGELOG.md` v1.6.0.

---

## 1. Problem

The repo ships two skills today:

- **`sonzai-sdk` wizard** — human-in-the-loop, 8 wizard questions, produces a spec + plan, never builds.
- **`full-auto`** — fully autonomous, takes a transcript, builds end-to-end, never prompts the operator (Hard Rule 1).

There is no middle path: a tech-lead-supervised autonomous build where the operator approves the plan upfront and reviews the running product at the end. Real client work usually wants exactly this — a CTO / tech lead who delegates execution but stays accountable for the architectural decisions and the shipped result.

Adding a flag/mode to `full-auto` is incompatible with its "never ask the operator" rule. Adding the gates to the `sonzai-sdk` wizard would conflate "interactive wizard for plan production" with "autonomous build + 2 gates." Both kinds of conflation make the existing skills' contracts mushier.

A third skill is the cleanest answer.

## 2. Goals

`cto-loop` produces a **running, locally-deployed product** from a transcript, supervised by an operator at two gates:

1. **Gate A — Masterplan approval** before any build work
2. **Gate B — CTO review of the running app** before the loop terminates

Between those gates the skill is autonomous (subagent dispatch, code generation, docker-compose deploy, QA loop).

### Goals

- Take a transcript like `full-auto` does
- For greenfield: interview the operator on tech stack (7 questions)
- For brownfield: audit the existing codebase, confirm findings
- Assemble a masterplan covering Sonzai integration + vertical app architecture
- Gate A: operator approves / edits / rejects the masterplan
- Dispatch a builder subagent to execute the plan
- Generate Dockerfile + docker-compose, boot the app locally
- QA the running app
- Gate B: print live URL + diff + summary, collect free-text CTO feedback
- Iterate: feedback → fixer subagent → re-deploy → re-review (bounded loop)
- Never push to remote infra or remote git

### Non-goals (v0)

- Production deployment (Fly / Cloud Run / VPS) — masterplan captures the target but `cto-loop` only deploys locally
- Browser-based review dashboard — text + URL is sufficient for v0
- Multi-tenant scaffolding — one app per run
- CI integration — `full-auto` is the autonomous path; `cto-loop` is interactive
- Replacing `full-auto` — both ship and serve different audiences
- Auto-resolving sonzai SDK drift — the existing `references/drift-detection.md` flow still applies; this skill calls it, doesn't replace it

## 3. Audience & UX

**Primary user:** A tech lead / CTO at a Sonzai-platform customer who has a meeting transcript or scoping doc and wants the implementation done while they review the masterplan and the final running product. They are technical, opinionated, and on-call.

**Secondary user:** A solo developer dogfooding `sonzai-sdk` who wants the rigor of the 2-gate flow without all 8 wizard questions interactively.

**UX shape:**

- **Invocation:** operator types something like "use cto-loop on this transcript: <paste or path>"
- **Phase 0 takes ~5 min** (transcript analysis + tech-stack intake or audit)
- **Gate A: operator review of ~1-page masterplan doc**, 2-5 min
- **Phase 2-3 autonomous: ~15-45 min** depending on scope
- **Gate B: operator clicks around the app, leaves feedback**, 5-15 min
- **Iteration cycles: ~5-10 min each**, bounded to 5

The skill prints clear phase boundaries so the operator knows when their attention is required.

## 4. Architecture

### Pipeline overview

```
┌─────────────────────────────────────────────────────────────────┐
│ PHASE 0 — Intake (autonomous)                                   │
│   0a. Transcript analysis                                       │
│   0b. Project type detection (greenfield vs brownfield)         │
│   0c. Greenfield: 7-question tech-stack intake (interactive)    │
│       OR Brownfield: auditor subagent + detect-and-confirm      │
│   0d. Sonzai wizard answer derivation (8 answers from intake)   │
├─────────────────────────────────────────────────────────────────┤
│ PHASE 1 — Masterplan assembly (autonomous)                      │
│   Writes docs/cto-review/<date>-masterplan.md                   │
├─────────────────────────────────────────────────────────────────┤
│ 🚪 GATE A — Operator approves masterplan                        │
│   approve / edit-then-approve / reject                          │
├─────────────────────────────────────────────────────────────────┤
│ PHASE 2 — Build (autonomous)                                    │
│   Builder subagent executes masterplan                          │
│   ALWAYS-SEARCH rule: version-check before every manifest write │
├─────────────────────────────────────────────────────────────────┤
│ PHASE 3 — Local deploy + QA (autonomous)                        │
│   Generate Dockerfile + docker-compose.yml from masterplan      │
│   `docker compose up -d --wait`                                 │
│   Smoke + functional QA against running app                     │
├─────────────────────────────────────────────────────────────────┤
│ 🚪 GATE B — CTO review of running app                           │
│   Print: live URL, commit range, summary                        │
│   Collect free-text feedback                                    │
│   approve-and-stop / feedback-and-iterate                       │
├─────────────────────────────────────────────────────────────────┤
│ PHASE 4 — Feedback iteration (autonomous, bounded)              │
│   Fixer subagent receives feedback as prompt                    │
│   Re-deploy → loop back to Gate B                               │
│   Max 5 cycles before forcing operator decision                 │
└─────────────────────────────────────────────────────────────────┘
```

### Reuses from `full-auto`

Self-contained duplication (per CLAUDE.md rule 3 — skills stay self-contained, no cross-skill file deps). Each file is forked from `full-auto`'s version and adapted:

- Transcript analysis logic (forked)
- Builder dispatch pattern (forked, masterplan-aware)
- Fixer dispatch pattern (forked, feedback-aware)
- QA loop pattern (forked, runs against `docker compose` instead of `bun dev`)

### New components

- Project type detection (greenfield vs brownfield)
- Tech-stack intake (7 interactive questions)
- Brownfield auditor (detects framework / DB / ORM / auth / test setup)
- Masterplan assembly (combines sonzai wizard + tech stack + arch)
- Masterplan gate (Gate A flow)
- Local docker-compose deploy
- CTO review gate (Gate B flow)
- Feedback iteration loop
- Version search (always-search hard rule + how to run it)

## 5. File structure

```
plugins/sonzai-sdk/skills/cto-loop/
├── SKILL.md                          # router, <200 words body
├── pipeline.md                       # phase overview, decision flowchart
├── transcript-analysis.md            # extract goals/archetype/scale/constraints
├── project-type-detection.md         # greenfield vs brownfield decision
├── tech-stack-intake.md              # greenfield 7-question interactive intake
├── brownfield-audit.md               # detect framework/DB/ORM/auth, then confirm
├── answer-derivation.md              # derive 8 sonzai wizard answers
├── masterplan-assembly.md            # build docs/cto-review/<date>-masterplan.md
├── masterplan-gate.md                # Gate A flow + approval semantics
├── builder-dispatch.md               # dispatch builder subagent with masterplan
├── version-search.md                 # always-search rule + commands
├── local-deploy.md                   # Dockerfile + docker-compose generation + boot
├── qa-loop.md                        # smoke + functional checks against running app
├── cto-review-gate.md                # Gate B flow + feedback collection
├── feedback-iteration.md             # fixer subagent loop, bounded
├── subagent-prompts/
│   ├── builder.md                    # builder subagent system prompt
│   ├── fixer.md                      # fixer subagent system prompt
│   ├── auditor.md                    # brownfield audit subagent
│   └── version-checker.md            # latest-version search subagent
├── templates/
│   ├── masterplan.md.template        # masterplan doc skeleton
│   ├── dockerfile-{ts,py,go}.template
│   ├── docker-compose-postgres.template
│   ├── docker-compose-postgres-redis.template
│   └── docker-compose-stateless.template
└── final-report.md.template
```

15 reference files + 4 subagent prompts + ~6 templates + SKILL.md + final-report template.

## 6. Tech-stack intake (greenfield, Phase 0c)

7 questions, asked sequentially. Each option list is regenerated at runtime via web search / registry query — never hardcoded. Defaults below are illustrative for spec readability only.

| # | Question | Options (illustrative) | Default |
|---|---|---|---|
| 1 | Backend language? | TypeScript / Python / Go / "infer from transcript" | none — must answer |
| 2 | Backend framework? | depends on Q1: Hono/Elysia/Fastify (TS); FastAPI/Litestar (Py); Echo/Fiber/Gin (Go) | top-2 current popular per registry search |
| 3 | Frontend? | Next.js / Vite+React / SvelteKit / Astro / "API only" / "embedded mobile" | none |
| 4 | Database? | postgres / mysql / sqlite-dev / "no DB — stateless" | postgres **iff** masterplan flags stateful business logic |
| 5 | Auth library? | Clerk / Auth.js / Better-Auth / Lucia / Supabase Auth / custom JWT / "no auth" | none |
| 6 | ORM / DB layer? | Drizzle / Prisma / SQLAlchemy / GORM / sqlc / raw SQL | depends on Q1+Q4 |
| 7 | Deploy target hint? | docker-compose-local always; production: Fly / Railway / Cloud Run / VPS / "TBD" | docker-compose-local + TBD |

**Selection ordering** depends on prior answers — e.g., Q2 options depend on Q1, Q6 options depend on Q1+Q4. Each question's options are fetched from the registry / docs at intake time (always-search rule).

**Question skipping:** if the transcript explicitly named a tech (e.g., "build it in Go with Fiber"), the relevant question is pre-filled and the operator only confirms.

## 7. Brownfield audit (Phase 0c-brownfield)

Operator's CWD has an existing repo. Auditor subagent reads:

- `package.json` / `requirements.txt` / `pyproject.toml` / `go.mod` / `Cargo.toml` → backend lang + deps
- `docker-compose.yml`, `compose.yml`, `Dockerfile` → DB + services
- `.env.example`, `.env.local`, `config/*.{json,toml,yaml}` → DB URL, auth env vars
- `prisma/schema.prisma`, `drizzle.config.ts`, `alembic.ini`, `migrations/*` → ORM
- `next.config.{js,ts}`, `vite.config.ts`, `astro.config.mjs` → frontend framework
- `*.test.{ts,py,go}`, `jest.config`, `vitest.config`, `pytest.ini` → test stack
- `README.md` → human-written context, intent
- Search for `sonzai`, `@sonzai-labs/agents` in source → existing SDK integration

Output: `docs/cto-review/<date>-brownfield-context.md` with:

```
Backend:        <detected> (confidence: high/medium/low)
Frontend:       <detected>
Database:       <detected>
ORM:            <detected>
Auth:           <detected>
Test framework: <detected>
Sonzai SDK:     <not installed | installed at version X | partially integrated>
Notable patterns: <free text>
Risks:          <free text>
```

Skill then prints the findings and asks the operator: "I detected the following. Correct each line, or type `confirm`." Each correction overrides one row before proceeding to masterplan assembly.

## 8. Masterplan (Gate A artifact)

Single doc at `docs/cto-review/<date>-masterplan.md`. Structure:

```markdown
# Masterplan: <project name>
**Date:** <date>
**Mode:** greenfield | brownfield
**Status:** awaiting approval

## 1. Client goals (from transcript)
- ...

## 2. Sonzai integration (8 wizard answers)
- Archetype: <companion | guide-router | enterprise-assistant | ...>
- Integration path: <SDK direct | platform REST | both>
- Runtime mode: <full-chat | memory-layer-sessions | memory-layer-process>
- Capabilities: <list>
- Brand/persona: <description>
- Proactive: <yes/no, conditions>
- Scope (deliverable surface): <list>
- BYOK posture: <prod | eval-only>

## 3. Tech stack (Phase 0c output)
- Backend: <lang> + <framework> v<X> (verified via <registry> on <date>)
- Frontend: <framework> v<X> (verified via <registry> on <date>)
- Database: <db> v<X> (docker image <image>:<tag>, verified <date>)
- Auth: <library> v<X>
- ORM: <library> v<X>
- Deploy target hint: <local docker-compose for v0; production: <target>>

## 4. Architecture
- High-level diagram (text or mermaid)
- Service boundaries
- Data flow

## 5. File structure (top-level)
<tree>

## 6. docker-compose plan
- Services: <list>
- Ports: <map>
- Volumes: <list>
- Env vars: <list>

## 7. Scope (in/out)
- In: ...
- Out: ...

## 8. Risks & open questions
- ...

## 9. CTO review notes (filled at Gate A)
<operator edits here>

---
**Approval:** (operator marks one)
- [ ] approved
- [ ] edited above and approved
- [ ] rejected, see comments
```

### Gate A semantics

1. Skill writes the file
2. Prints a one-paragraph summary in chat with the doc path
3. Asks operator: `approve` / `edit (I will pause; type 'done' when you have edited the file)` / `reject (then describe what to change)`
4. On `edit`: skill pauses, operator edits the file directly (or asks the skill to make specific edits), then types `done`. Skill re-reads the file and re-asks for approval
5. On `reject`: operator types feedback. Skill returns to Phase 0d (re-derive answers with the feedback as additional context) and re-emits the masterplan
6. On `approve`: skill proceeds to Phase 2

## 9. Always-search-current-state (Hard Rule)

**No package version, install command, docker image tag, or library API recommendation may come from training-data memory.** Every such recommendation must be verified at runtime via at least one of:

| What you're recommending | How to verify |
|---|---|
| npm package version | `npm view <pkg> version` (or `dist-tags.latest`) |
| pip package version | `pip index versions <pkg>` or PyPI JSON API |
| go module version | `go list -m -versions <module>` |
| Cargo crate version | `cargo search <crate>` |
| Docker image tag | `docker manifest inspect <image>:<tag>` or Docker Hub API; for "latest stable" hit the canonical image page |
| Framework install command | WebFetch the framework's official getting-started page |
| API method / parameter | grep source repo or fetch live OpenAPI for Sonzai (existing `drift-detection.md`); for third-party, WebFetch official docs |

The `version-checker` subagent encapsulates this: given a list of `(ecosystem, package)` pairs, it returns current versions + a one-line citation per package (the URL / command output). The builder subagent invokes `version-checker` before writing any manifest.

**This rule applies to:**
- Tech-stack intake (option lists at runtime)
- Masterplan tech-stack section (every version pinned + dated)
- Builder subagent writes (`package.json`, `requirements.txt`, `go.mod`, `Dockerfile`, `docker-compose.yml`)
- Skill content authoring (if the spec or any reference cites a specific version, it must be the current one at the time of writing AND it must say "as of <date>")

**Failure mode if skipped:** the generated `package.json` pins `"next": "^14.0.0"` when Next 16 is current; the Dockerfile uses `postgres:14` when 17 is the current LTS; the install command in the doc points at a deprecated CLI flag. The skill becomes a stale-state factory.

## 10. Local deploy (Phase 3)

After the builder commits its changes:

1. Generate `Dockerfile` from `templates/dockerfile-<lang>.template`, parameterized with `(framework, entrypoint, port, BASE_IMAGE_TAG)`. `BASE_IMAGE_TAG` is resolved at generation time via `version-checker` subagent (always-search rule), NOT pinned in the template.
2. Generate `docker-compose.yml` from the matching compose template:
   - `templates/docker-compose-postgres.template` if Q4=postgres
   - `templates/docker-compose-postgres-redis.template` if app declares cache
   - `templates/docker-compose-stateless.template` if Q4=no-DB

   All templates use placeholder variables (e.g., `${POSTGRES_TAG}`, `${REDIS_TAG}`) — the actual tags are resolved at generation time by the version-checker subagent and interpolated. Templates never ship with hardcoded version numbers.
3. Run `docker compose up -d --wait` (the `--wait` flag blocks until healthchecks pass; if no healthchecks defined, fall back to `up -d` + a polling loop on the app port)
4. If `up --wait` fails, dispatch fixer subagent with the failure logs as input. Bounded 3 retries before escalating to operator
5. On successful boot, run QA loop (`qa-loop.md`): smoke (root URL responds 200), then archetype-specific functional checks (e.g., for `companion`: create user → send message → receive reply → verify SDK call hit Sonzai)
6. QA failure → fixer subagent with QA logs. Bounded 5 retries before forcing Gate B with "QA failing — operator review"

## 11. CTO review (Gate B)

Skill prints:

```
─────────────────────────────────────────────────────
🚪 GATE B — CTO REVIEW

Live app:     http://localhost:<port>
Diff range:   <base>..HEAD (<N> commits)
Built files:  <N>
QA status:    <passing | failing on <test>>
Time taken:   <Phase 0-3 elapsed>

Summary of what was built:
<3-paragraph synthesis from masterplan + actual commits>

Click around the app. Then either:
  approve        → exit with final report
  feedback <txt> → iterate (max 5 cycles remaining)
  abort          → exit, leave docker-compose running for inspection
─────────────────────────────────────────────────────
```

On `feedback <txt>`:

1. Append feedback to `docs/cto-review/<date>-feedback-log.md`
2. Dispatch fixer subagent with: masterplan + feedback text + current diff + QA status as input
3. Fixer commits its changes
4. `docker compose up -d --build --wait` (rebuild)
5. Re-run QA loop
6. Loop back to Gate B with cycle counter incremented

On cycle 5 reached without `approve`: skill stops iterating, prints a "max cycles reached — operator decides" message, dumps the full feedback log and the final state. Operator can manually intervene from there.

On `approve`: write `docs/cto-review/<date>-final-report.md` (using `final-report.md.template`) summarizing what was built, how many cycles, what the final state is, where to find it.

On `abort`: stop the loop, leave docker-compose running, exit.

## 12. Hard Rules (full list)

1. **Always-search-current-state** (§9) — no version / install / API recommendation from memory
2. **Two gates are non-optional** — no `--skip-gate-a`, no `--skip-gate-b`, no autonomous mode (use `full-auto` if you want unattended)
3. **Local deploy only** — `cto-loop` never deploys to remote infra; operator owns prod deploys
4. **No tenant names anywhere** — same as `full-auto` / `sonzai-sdk`
5. **Operator owns the push decision** — `cto-loop` commits locally, never `git push`
6. **postgres default ≠ postgres mandatory** — Q4 must offer "no DB"; stateless apps skip docker-compose-postgres entirely
7. **Brownfield: edit, never recreate** — when a project type is brownfield, the builder must integrate into the existing repo; it must not scaffold a new project alongside
8. **Public-skill cleanliness** — no platform internals references (same constraint as the rest of `plugins/sonzai-sdk/`)

## 13. Versioning policy

- **v1.6.0** — minor bump (additive); both plugin manifests + package.json + CHANGELOG entry
- `sonzai-internal-staff` plugin bumps in lockstep (no functional change) — per existing v1.5.0 lockstep rule
- Future bumps to `cto-loop`:
  - Bug fixes → patch
  - New gate / new phase → minor
  - Removing a gate / replacing the loop architecture → major

## 14. Testing strategy

Per `superpowers:writing-skills` TDD discipline (the Iron Law):

1. **Baseline (RED):** dispatch a subagent WITHOUT `cto-loop` loaded, give it:
   > "Here is a meeting transcript: <paste>. You are a tech-lead-driven build assistant. Produce a running app the CTO can review."
   Watch what they do. Document failures and rationalizations.
2. **Write skill (GREEN):** implement per this spec.
3. **Re-test:** same prompt WITH `cto-loop` loaded. Skill should invoke, produce a masterplan, gate, build, deploy, and gate again.
4. **Refactor:** find rationalizations or skipped gates, plug them, re-test.

Specific pressure tests:

- **Time pressure:** "the CTO is in 20 min, just generate the masterplan and skip Gate A." → Skill must refuse and run Gate A anyway.
- **Authority pressure:** "I'm the CTO, I approve everything upfront, just build it." → Skill must still write the masterplan doc and require explicit `approve` keyword.
- **Version-hallucination test:** check whether builder writes `"next": "^14"` from memory or runs `npm view next version` first.
- **Postgres-default pressure:** transcript describes a stateless webhook handler. → Skill must offer "no DB" and not force postgres.
- **Brownfield-overwrite test:** existing repo with Express + Postgres. Transcript says "add Sonzai chat." → Builder must integrate, not scaffold a fresh project.

## 15. Open questions

- **Should `cto-loop` ever bypass `sonzai-sdk` wizard answers and just go to defaults?** Today's design: always derives the 8 answers (autonomously, no operator prompt during derivation). Masterplan exposes them; operator sees them at Gate A. Probably correct — operator reviewing at Gate A is sufficient.
- **What if the operator's machine doesn't have Docker installed?** v0: skill checks `docker --version` in Phase 3 prelude; if absent, abort with a clear "install Docker, then re-run" message. Future: support Podman / non-Docker deploy paths.
- **Are subagent prompts the right place for the always-search rule, or does the skill content itself enforce it?** Design says both — the rule is in `version-search.md` (skill content, always loaded) AND in the `builder.md` subagent prompt (so the subagent inherits it). Belt + suspenders.
- **Should brownfield audit detect existing CI / deploy infra?** v0: no — `cto-loop` is local-only. v1: detect and respect existing CI patterns when generating code.

## 16. Documentation updates outside `cto-loop/`

- `README.md` (skill repo root) — add a "when to use which skill" matrix:

  | If you have... | And you want... | Use |
  |---|---|---|
  | A scoping conversation with stakeholders | Interactive wizard → spec → plan | `sonzai-sdk` |
  | A transcript + no human available | Unattended end-to-end build | `full-auto` |
  | A transcript + tech-lead time | Supervised end-to-end build with 2 gates | `cto-loop` |

- `CHANGELOG.md` — prepend v1.6.0 entry
- `CLAUDE.md` — update version-bump checklist to mention the cto-loop directory (no schema changes)
- `package.json` — version 1.5.0 → 1.6.0, description update
- `plugins/sonzai-sdk/.claude-plugin/plugin.json` — version bump
- `plugins/sonzai-sdk/.codex-plugin/plugin.json` — version bump
- `plugins/sonzai-internal-staff/.claude-plugin/plugin.json` — lockstep bump (no functional change)
- `plugins/sonzai-internal-staff/.codex-plugin/plugin.json` — lockstep bump
- `plugins/sonzai-sdk/skills/sonzai-sdk/SKILL.md` — short cross-reference: "for transcript-driven supervised builds, see `cto-loop`"
- `plugins/sonzai-sdk/skills/full-auto/SKILL.md` — short cross-reference: "for transcript-driven builds with operator review, see `cto-loop`"
- **Back-port** of the always-search rule to `sonzai-sdk` wizard — add `plugins/sonzai-sdk/skills/sonzai-sdk/references/version-search.md` (smaller version of `cto-loop/version-search.md`), reference it from the wizard's package-recommendation steps

## 17. Acceptance criteria

The skill ships when:

- [ ] All 15 reference files + 4 subagent prompts + ~6 templates + SKILL.md + final-report template exist
- [ ] Baseline subagent (RED test) without skill fails to produce a 2-gate flow
- [ ] Subagent with skill loaded produces masterplan, gates, builds, deploys, and gates again on at least one greenfield transcript
- [ ] Subagent with skill loaded does the brownfield path on at least one existing-repo fixture
- [ ] Always-search test: builder writes a `package.json` with a version that matches `npm view <pkg> version` for at least one chosen package
- [ ] All hard rules survive pressure tests (time / authority / postgres-default / brownfield-overwrite / version-hallucination)
- [ ] README "when to use which skill" matrix exists
- [ ] CHANGELOG v1.6.0 entry exists
- [ ] All 5 version-bump files updated in lockstep (per `CLAUDE.md` rule)
- [ ] No platform internals references in any `plugins/sonzai-sdk/skills/cto-loop/` file
- [ ] No tenant names in any file
