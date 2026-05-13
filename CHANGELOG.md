# Changelog

All notable changes to `sonzai-claude-skill` are documented here. The project follows [Semantic Versioning](https://semver.org/). Dates are `YYYY-MM-DD`.

## v1.6.0 — 2026-05-13

### Added: `cto-loop` skill (semi-autonomous, two-gate variant of `full-auto`)

The `sonzai-sdk` plugin now ships THREE public skills (was two). All three are SDK-agnostic, public, and tenant-free.

```
plugins/sonzai-sdk/skills/
├── sonzai-sdk/          # interactive wizard → spec + plan (no build)
├── full-auto/           # autonomous transcript → running app (no operator prompts)
└── cto-loop/            # NEW — same pipeline + 2 operator gates + interactive tech-stack intake
```

| Skill | Operator prompts | Output | Use when |
|---|---|---|---|
| `sonzai-sdk` | wizard Q&A | spec + plan documents | scoping meeting → spec → plan; no auto-build |
| `full-auto` | none | running local app + final report | transcript + no human available |
| `cto-loop` | 2 gates + 7 intake Qs (greenfield) or audit confirm (brownfield) + per-cycle feedback | running local app + final report w/ approval + feedback log | transcript + tech-lead available to review |

### Added: Shared core in `full-auto/` (used by both `full-auto` and `cto-loop`)

The transcript → running-app pipeline now exists once as a complete autonomous flow in `full-auto/`. `cto-loop` is a thin overlay that inserts two operator gates and swaps two phases for interactive variants (tech-stack intake, brownfield audit confirm). Everything else is shared via `../full-auto/<file>.md` references.

New files in `full-auto/`:

- `project-type-detection.md` — greenfield vs brownfield detection
- `tech-stack-derivation.md` — autonomous greenfield stack derivation (transcript + sensible defaults)
- `brownfield-audit.md` — autonomous codebase audit
- `masterplan-assembly.md` — assembles single review-able masterplan doc
- `version-search.md` — **HARD RULE: always-search-current-state** (no version, install command, docker tag, or library API from training memory)
- `local-deploy.md` — Dockerfile + docker-compose generation; `docker compose up -d --wait` + healthcheck-aware boot
- `feedback-iteration.md` — auto-fixer loop on QA failure
- `subagent-prompts/builder.md`, `fixer.md`, `auditor.md`, `version-checker.md`
- `templates/dockerfile-{ts,py,go}.template`
- `templates/docker-compose-{postgres,postgres-redis,stateless}.template`
- `templates/masterplan.md.template`

Existing files in `full-auto/` were upgraded (transcript-analysis, answer-derivation, builder-dispatch, qa-loop, SKILL.md, pipeline.md, final-report.md.template) to reflect the new pipeline.

New files in `cto-loop/`:

- `SKILL.md` — thin router
- `pipeline.md` — overlay map showing which phases are shared vs cto-loop-local
- `tech-stack-intake.md` — interactive 7-question greenfield stack intake (replaces full-auto's autonomous derivation when cto-loop is active)
- `brownfield-audit.md` — operator-confirm wrapper around full-auto's autonomous auditor
- `masterplan-gate.md` — Gate A (approve / edit / reject)
- `cto-review-gate.md` — Gate B (live URL review, approve / feedback / abort)
- `feedback-iteration.md` — operator-feedback-driven fixer loop (5 cycle hard cap + 1 explicit "one more")

### Added: Always-search-current-state hard rule

A new behavioral rule across all three public skills: no package version, install command, docker image tag, or library API recommendation may come from training memory. Every recommendation must be verified at write-time via:

- `npm view <pkg> version` / `pip index versions <pkg>` / `go list -m -versions <module>` / etc.
- Docker Hub API for current stable image tags
- WebFetch of canonical framework docs for install commands

Implementation: dedicated `version-checker` subagent in `full-auto/subagent-prompts/`. The wizard skill also gets a smaller back-port at `sonzai-sdk/references/version-search.md`.

Rationale: training-data lag (often >12 months) was silently propagating stale package versions into every generated scaffold. The skill now refuses to recommend anything time-sensitive without a runtime verification.

### Added: Local docker-compose deploy + functional QA against running app

Both `full-auto` and `cto-loop` now produce a *running* local stack (not just code + a smoke test). After build:

1. Generate `Dockerfile` from per-language template (postgres / postgres+redis / stateless variants)
2. Resolve image tag placeholders via `version-checker` (no hardcoded versions in templates)
3. `docker compose up -d --wait`
4. Run migrations
5. Functional QA against `http://localhost:<port>` — smoke + archetype-specific checks (auth signup → chat → memory verification for `companion`; routing decision for `guide-router`; etc.)
6. Fail → fixer subagent → rebuild → re-QA (bounded 5 cycles)

### Added: Greenfield tech-stack intake (7 questions, `cto-loop` only)

When `cto-loop` is invoked on an empty CWD:

```
Q1. Backend language?            → TypeScript / Python / Go
Q2. Backend framework?           → top-3 current popular for chosen lang (verified live)
Q3. Frontend?                    → Next.js / Vite+React / SvelteKit / Astro / api-only / embedded
Q4. Database?                    → postgres (default if stateful) / mysql / sqlite-dev / no-DB
Q5. Auth library?                → Clerk / Auth.js / Better-Auth / Lucia / Supabase / custom JWT / no auth
Q6. ORM (skipped if no DB)?      → Drizzle / Prisma / SQLAlchemy / GORM / sqlc / raw SQL
Q7. Production deploy target?    → Fly / Railway / Cloud Run / AWS / VPS / on-prem / TBD
```

Defaults document fallbacks (used in `full-auto`'s autonomous variant). Postgres is NEVER mandatory — Q4 offers "no DB" and that branch skips Q6.

### Added: Brownfield audit (autonomous in `full-auto`, detect-and-confirm in `cto-loop`)

When the operator's CWD has prior signals (`package.json`, `go.mod`, `docker-compose.yml`, source files, etc.):

- The `auditor` subagent reads the existing repo (read-only) and detects: backend lang + framework + version, frontend, database, ORM, auth, test framework, sonzai SDK install status, notable patterns, risks
- Confidence-scored per layer (`high` / `medium` / `low`)
- Low-confidence detections become risk entries the builder treats conservatively downstream
- `full-auto` writes findings to `docs/cto-review/<date>-brownfield-context.md` without confirmation
- `cto-loop` prints findings as a table and asks the operator to confirm or `fix <row>` each line before proceeding

Brownfield builds **integrate** into the existing repo (never recreate alongside). Hard Rule 7 in both skills.

### Added: Masterplan as a single review-able artifact

Both modes now produce `docs/cto-review/<date>-masterplan.md` covering:

- Client goals (from transcript)
- Sonzai integration (8 wizard answers + rationales)
- Tech stack (with verified-at dates)
- Architecture diagram + service boundaries
- File structure (NEW vs MODIFY for brownfield)
- docker-compose plan (services, ports, volumes, env vars)
- Scope (in/out)
- Risks
- (cto-loop) Operator review notes + approval checkbox

In `full-auto` the masterplan is the build's input artifact; the approval section is emitted but never marked. In `cto-loop` Gate A blocks until the operator approves it.

### Removed / replaced

- `full-auto/subagent-prompts/builder-prompt.md.template` → replaced by `builder.md`
- `full-auto/subagent-prompts/fixer-prompt.md.template` → replaced by `fixer.md`
- `full-auto/`'s old 6-phase pipeline (drift / ingest / signals / wizard-answers / build / QA / report) → re-numbered as Phase 0a-d + 1-5 to align with `cto-loop`

### Migration from v1.5.0

No action required. Both plugin manifests bump in lockstep (`sonzai-sdk` 1.5.0 → 1.6.0; `sonzai-internal-staff` 1.5.0 → 1.6.0). Existing `full-auto` invocations work — the skill content has expanded, but the trigger phrases and inputs are unchanged. `cto-loop` is purely additive.

To use `cto-loop`:

```bash
/plugin install sonzai-sdk@sonz-ai     # already installed if you're on v1.5.0
# then in chat:
"use cto-loop on this transcript: <paste or path>"
```

### Architecture notes

This release corrects a v1.5.0 design oversight: docker-compose deploy, version-search, fixer subagents, tech-stack intake, brownfield audit etc. were originally drafted for `cto-loop` only. The user (correctly) pushed back: "CTO loop is just semi-auto (with gates), full-auto should be the exact same thing." Implementation refactored mid-flight to put shared core in `full-auto/`, leaving `cto-loop/` as a 6-file thin overlay (SKILL.md, pipeline.md, tech-stack-intake, brownfield-audit confirm wrapper, two gate files, operator-feedback iteration).

---

## v1.5.0 — 2026-05-13

### Breaking: repo split into two plugins (install-time gating)

The repo now ships **two plugins** instead of one. The runtime gate (`SONZAI_INTERNAL_STAFF=1` env var / `--sonzai-internal-staff` flag) is **removed**. Gating now happens at install time, not runtime — the plugin marketplace model decides who gets which files on disk.

**Before (v1.4.0)** — single plugin, three skills, runtime gate:

```
sonzai-claude-skill/
├── .claude-plugin/plugin.json
└── skills/
    ├── sonzai-sdk/
    ├── full-auto/
    └── sonzai-internal-staff/    ← copied to every public disk; STOP gate at runtime
```

**After (v1.5.0)** — two plugins, three skills, install-time gate:

```
sonzai-claude-skill/
├── .claude-plugin/marketplace.json                 ← lists both plugins; sonz-ai marketplace
├── plugins/
│   ├── sonzai-sdk/                                 ← PUBLIC plugin (auto-installed)
│   │   ├── .claude-plugin/plugin.json
│   │   ├── .codex-plugin/plugin.json
│   │   └── skills/{sonzai-sdk, full-auto}
│   └── sonzai-internal-staff/                      ← INTERNAL plugin (opt-in install only)
│       ├── .claude-plugin/plugin.json
│       ├── .codex-plugin/plugin.json
│       └── skills/sonzai-internal-staff
```

**Public-user UX (Claude Code):**

```bash
/plugin marketplace add sonz-ai/sonzai-claude-skill
/plugin install sonzai-sdk@sonz-ai
```

→ only public files copied to disk. Internal-staff skill is NOT installed.

**Staff UX (one extra line):**

```bash
/plugin install sonzai-internal-staff@sonz-ai
```

→ adds the internal plugin alongside.

**Codex** gets the same shape via `.codex-plugin/plugin.json` in each plugin folder. Manual install (no marketplace) uses per-plugin symlinks — see `README.md`.

### Why this matters

Earlier versions still copied the internal SKILL.md (with its `$SONZAI_WORKSPACE/sonzai-ai-monolith-ts/...` paths and monolith-aware augmentations) onto every public user's disk, even though a runtime gate refused to load it. That was wasteful and conceptually messy.

The new shape mirrors Anthropic's own `claude-plugins-official` marketplace pattern (200+ plugins in one repo, each independently installable by name via `git-subdir`/relative-path source). It also matches Codex's plugin model 1:1.

### Removed

- `SONZAI_INTERNAL_STAFF=1` env var gate (no longer needed)
- `--sonzai-internal-staff` invocation flag (no longer needed)
- STOP / gate-check block at the top of the internal SKILL.md
- Description-frontmatter language in the internal SKILL.md requiring runtime opt-in
- Root-level `.claude-plugin/plugin.json` (replaced by per-plugin manifests + marketplace.json)

### Added

- `.claude-plugin/marketplace.json` — sonz-ai marketplace listing both plugins
- `plugins/sonzai-sdk/.claude-plugin/plugin.json` + `.codex-plugin/plugin.json` — public plugin manifests
- `plugins/sonzai-internal-staff/.claude-plugin/plugin.json` + `.codex-plugin/plugin.json` — internal plugin manifests

### Migration for existing v1.4.0 installs

```bash
# Claude Code
/plugin uninstall sonzai-sdk          # the old single plugin
/plugin marketplace add sonz-ai/sonzai-claude-skill
/plugin install sonzai-sdk@sonz-ai    # the new public plugin
# Optional, staff only:
/plugin install sonzai-internal-staff@sonz-ai

# Manual symlink users
rm ~/.claude/skills/sonzai-sdk
ln -s ~/sonzai-claude-skill/plugins/sonzai-sdk/skills/sonzai-sdk ~/.claude/skills/sonzai-sdk
# Repeat for full-auto and (if staff) sonzai-internal-staff
```

### Unchanged

- All skill contents (intake.md, archetypes, decisions, runtime-mode.md, full-auto pipeline, internal workspace-pointers.md) carry forward as-is. Only paths and manifests changed.
- Wizard answers Q1–Q8, BYOK-as-production-default, and the 7 archetype playbooks remain identical to v1.4.0.
- `CLAUDE.md` maintenance rules unchanged: public-plugin content must remain tenant-agnostic; internal pointers live only in `plugins/sonzai-internal-staff/`.

---

## v1.4.0 — 2026-05-13

### Added: Q8 (Runtime mode) — the wizard's biggest architectural decision

The wizard now asks **who calls the chat LLM — Sonzai or you?** Four modes, each verified against the live SDK source (`sonzai-go`, `sonzai-typescript`, `sonzai-python`) and the live OpenAPI:

- **A. full-chat** (recommended default) — `agents.chat` / `chatStream` / `chatAsync`. Sonzai orchestrates context build → LLM call → response → memory consolidation in one round trip. ~80% of builds default here.
- **B. full-chat with explicit sessions** — `sessions.start` → `sessions.turn` → `sessions.end`. Sonzai owns the chat LLM; you own the session lifecycle. Use when you need per-session tool injection (game NPCs swapping toolsets between scenes), deferred-turn semantics, or end-of-session consolidation hooks.
- **C. memory-layer via sessions** — `sessions.start` → `agents.process` → `sessions.end`. **You** call your own LLM; Sonzai handles memory + session boundaries. Use when you already have an LLM stack (Anthropic, OpenAI, vLLM, internal) you don't want to replace.
- **D. memory-layer via /process only** — `agents.process` per event. No session lifecycle. Use for non-chat ingestion: email reads, doc memory, telemetry, passive learning surfaces.

**Verification:** every endpoint name and method shape grepped against the three SDK source repos + live OpenAPI. Notable findings:
- `sessions.turn()` does invoke the chat LLM (has `provider` / `model` fields). Sessions are NOT a BYO-LLM mode — they're an explicit lifecycle wrapper around modes A or C.
- `agents.process()` (POST `/api/v1/agents/{id}/process`) is the BYO-LLM ingestion path. Returns `MemoriesCreated`, `FactsExtracted`, `SideEffects` — no assistant reply.
- BYOK is NOT a separate runtime mode; it's a sub-configuration of A and B.

### Added: `decisions/runtime-mode.md`

Full decision aid with per-language code samples (Python / TypeScript / Go) for all four modes, latency comparison, capability implications table, billing posture per mode, and a quick-picker matrix.

### Changed: BYOK is now the recommended production posture

Updated `decisions/byok-vs-customllm.md`:

- **Production**: BYOK (or Custom LLM). Provides rate-limit isolation, cleaner audit, provider-region compliance, separates customer billing.
- **Platform default**: dev / eval / prototyping only. Convenient for getting started but routes all token cost through Sonzai's billing and shares platform-wide rate limits.
- Picker table reordered to lead with BYOK for any production scenario.

### Changed: every archetype playbook now prescribes a default runtime mode

Each of the 7 archetype playbooks gained two rows in its "Prescribed stack" table:

| Archetype | Default runtime mode | LLM provider (production) |
|---|---|---|
| companion | **A** | BYOK or Custom LLM |
| coach-therapist | **A** (with optional **D** for diary side flow) | BYOK or Custom LLM |
| customer-support | **A** (default) or **B** (per-ticket session lifecycle) | BYOK |
| enterprise-assistant | **A** (default) or **C** (existing internal chat infra) | BYOK (often required) or Custom LLM |
| game-npc | **B** (per-session tool swap is the killer feature) | BYOK or Custom LLM |
| guide-router | **A** (both guide and specialists; sessions add overhead) | BYOK |
| hybrid-custom | depends on dominant pattern; explicit per-tenant for multi-tenant | BYOK |

### Changed: `intake.md`

- **Q8 added** between Q7 and Section 3 (archetype follow-ups). Four-option fork (A/B/C/D) plus a production-posture note pointing at BYOK.
- **Section 1 (pre-question inference) expanded** with 4 new signals: existing chat handler in repo → memory-layer mode; "we have our own LLM" → C/D; "ingest emails / no chat" → D; "per-session tools" → B.

### Changed: spec templates

- `spec-templates/archetype-spec.md.template` — top-of-doc fields now include **Runtime mode (Q8)** and **LLM provider (production)**. Integration-points table's Chat handler row now distinguishes mode A/B/C/D wiring.
- `spec-templates/existing-codebase-spec.md.template` — same two top-of-doc fields. Incumbent-systems table's Chat handler and LLM provider rows now describe the per-mode migration target.

### Changed: full-auto pipeline

- `full-auto/transcript-analysis.md` — added signal categories **6b (Runtime mode)** and **6c (LLM provider posture)**. 5+ trigger phrases per Q8 option; explicit defaults when transcript is silent.
- `full-auto/answer-derivation.md` — added Q8 and Q8a derivation rules. Tie-break order: A > B > D > C. Production-bound defaults to BYOK with `openai` as starter provider. Self-check now verifies Q8 ↔ archetype consistency and BYOK posture for modes A/B.

### Rationale

The wizard previously stopped at Q7 (which SDK language?) and assumed everyone was using mode A. That mismatched reality:

- Enterprise customers with existing LLM contracts wanted mode C and were getting mode-A recommendations
- Game NPC builders needed mode B for tool injection but the wizard didn't surface it
- Data-ingestion verticals (email memory, doc ingest) didn't fit any chat-shaped recommendation at all — they need mode D
- Production users were being implicitly told to ship on platform credit, which is wrong — BYOK is the right posture for anyone past prototyping

This release closes those gaps with one new wizard question, one new decision aid, archetype defaults, spec-template fields, and full-auto signal extraction.

## v1.3.0 — 2026-05-13

### Added: `sonzai-internal-staff` skill (gated, internal-only)

Third skill in the plugin: `skills/sonzai-internal-staff/`. Layers Sonzai workspace awareness (SDK source repos + monolith) onto `sonzai-sdk` and `full-auto` for internal dogfooding. **Does not load** unless the operator opts in.

**Triple gate** (all three layered for defense-in-depth):

1. **Soft description gate** — the skill's frontmatter description says "Use ONLY when SONZAI_INTERNAL_STAFF=1 OR --sonzai-internal-staff is in the invocation". Without an opt-in phrase in the operator's prompt or env, the agent won't auto-load it.
2. **Hard runtime gate** — first section of `SKILL.md` is a STOP gate. The agent runs `echo "$SONZAI_INTERNAL_STAFF"` and checks the invocation for the flag. If neither is present, prints "internal-only, not loaded" and exits without reading the rest.
3. **Natural path gate** — the skill references `$SONZAI_WORKSPACE/sonzai-sdk/` and `$SONZAI_WORKSPACE/sonzai-ai-monolith-ts/`. External users without these repos hit useful 404s.

**Opt-in mechanisms** (operator activates either):

- Set env var: `export SONZAI_INTERNAL_STAFF=1`
- Pass flag: `--sonzai-internal-staff` (or `/sonzai-internal-staff`) literally in the invocation

**Files:**

- `SKILL.md` — gate + purpose + cross-refs to public skills + hard rules (no tenant names, no secrets, no monolith pushes)
- `workspace-pointers.md` — workspace resolution (env var → CWD ancestor walk → `$HOME/code/sonzai/` candidates → fail), SDK source map (`sonzai-python`, `sonzai-typescript`, `sonzai-go`, `sonzai-openclaw`), monolith map (contextengine, platform/api, ai-character-service, deploy), SDK↔monolith request mapping, OpenAPI source-of-truth ordering

**Workspace resolution** — never hardcodes a path. Resolves via:

1. `$SONZAI_WORKSPACE` env var
2. Walk up from CWD for ancestor containing both `sonzai-sdk/` and `sonzai-ai-monolith-ts/`
3. Try `$HOME/code/sonzai/`, `$HOME/work/sonzai/`, `$HOME/dev/sonzai/`, `$HOME/src/sonzai/`
4. None → print resolution instructions, exit

**What it does (after gate passes):**

- Phase 0 (drift check) — also reads the locally generated OpenAPI in `services/platform/api/docs/` and the SDK source for the most-current schema (live API may lag one deploy)
- Phase 5 (QA loop) — optionally tail local monolith server logs to diagnose failures (read-only)
- Wizard derivation — verify capability flags against canonical Go source in `services/contextengine/domain/entity/agent.go` instead of just the snapshot

### Changed

- **`sonzai-sdk/SKILL.md`** — adds a "Sonzai internal staff (gated)" section pointing to the new skill
- **`full-auto/SKILL.md`** — adds "Optional sibling: sonzai-internal-staff" under cross-skill dependencies
- **`.claude-plugin/plugin.json`** — version 1.3.0; description updated to describe three skills
- **`package.json`** — version 1.3.0

### Rationale

Sales / engineering staff frequently want to dogfood `full-auto` and the wizard with full server-side context — tracing SDK calls into platform/api handlers, verifying drift against contextengine source, tailing local logs during QA cycles. The public skills must stay tenant-agnostic and free of platform internals. This new skill is the safe place to put internal pointers. The triple gate ensures it stays dormant for external users; the user's quote: "I dont mind open sourcing the sonzai internal staff part because people cant access our codebase anyways, so at least we can dogfood our own skills easier."

## v1.2.0 — 2026-05-13

### Added: `full-auto` skill

A second skill in the plugin: `skills/full-auto/`. Autonomous closed-loop Sonzai implementer. Transcript in → working repo out. Zero operator prompts.

**Pipeline (6 phases):**

1. **Drift check** — fetch live OpenAPI from `https://api.sonz.ai/docs/openapi.json` into `.full-auto/openapi.live.json`
2. **Transcript ingest** — file arg, prior message, or inline; persist to `.full-auto/transcript.txt`
3. **Signal extraction** (`transcript-analysis.md`) — direct-quote evidence for archetype, integration path, latency, capabilities, brand/persona, proactive features, scope → `.full-auto/signals.md`
4. **Wizard-answer derivation** (`answer-derivation.md`) — deterministic mapping to all 7 wizard answers + archetype follow-ups + capabilities (resolved against UpdateCapabilitiesInputBody) + acceptance checklist → `.full-auto/wizard-answers.md`
5. **Builder subagent dispatch** (`builder-dispatch.md`) — Agent with name `sonzai-builder` reads wizard-answers, runs `sonzai-sdk` skill non-interactively, fills spec template, invokes `superpowers:writing-plans` and `superpowers:subagent-driven-development`, commits locally, returns structured JSON
6. **QA loop** (`qa-loop.md`) — outer session starts servers in background, exercises every endpoint in the acceptance checklist, drives the UI via browser MCP (chrome-devtools or playwright; falls back to API-contract-only if absent), aggregates failures, `SendMessage`s `sonzai-builder` with the fixer template, repeats up to 5 cycles
7. **Final report** (`final-report.md.template`) — `.full-auto/REPORT.md` with what was built, what was tested, all assumptions, all cycle reports, operator next steps

**Subagent prompts:**

- `subagent-prompts/builder-prompt.md.template` — initial dispatch with JSON return contract, hard rules (no `git push`, no mocks, no invented SDK symbols, no questions back), self-check checklist
- `subagent-prompts/fixer-prompt.md.template` — re-dispatch via SendMessage with QA cycle report; minimum-change discipline; per-failure hypothesis; PARTIAL status for operator-actionable items

**Hard rules:**

1. Never ask the operator a question — ambiguity becomes a documented assumption
2. Bounded 5-cycle fix loop; cycle 5 with failures → `.full-auto/BLOCKED.md`, exit
3. Local commits only — never `git push`, never `gh pr create`
4. Drift check first — never proceed without a live OpenAPI snapshot
5. No invented SDK symbols — every capability flag / method must verify against the drift artifact
6. No mock SDK call paths — real SDK, real API key, or BLOCKED

### Changed

- **`sonzai-sdk/SKILL.md`** — adds a "Full-auto (no human in the loop)" section pointing operators to the new skill with trigger phrases
- **`.claude-plugin/plugin.json`** — version 1.2.0; description updated to describe both skills
- **`package.json`** — version 1.2.0

### Rationale

v1.0.0 and v1.1.0 assumed a human in the loop answering wizard questions. Sales engineering reality: meetings produce voice transcripts, and the time between meeting-end and "can we see a demo?" is small. full-auto closes that loop. The skill is fully autonomous; the operator gets a working repo + a written audit of every decision the agent made.

## v1.1.0 — 2026-05-13

### Added

- **`features/mcp-integration.md`** — consuming Sonzai via the hosted MCP server at `https://api.sonz.ai/mcp/memory/{agent_id}`. Setup snippets for Claude Code, Cursor, VS Code, ChatGPT (Developer Mode), Claude Desktop, and the local stdio binary fallback. Covers the 34-tool surface and trade-offs vs SDK.
- **`features/openclaw-integration.md`** — using the `@sonzai-labs/openclaw-context` plugin to register Sonzai as OpenClaw's `contextEngine` slot. One-shot install, manual install, openclaw.json shape, B2B provisioning patterns for TS / Python / Go.

### Changed

- **`intake.md`** Q7 renamed from "Language" to "Integration path"; now offers 5 options (Python / TS / Go SDKs + MCP-only + OpenClaw). Pre-question inference detects MCP / OpenClaw signals in the workspace.
- **`SKILL.md`** trigger list extended to recognize MCP / OpenClaw integration intents.

### Rationale

v1.0.0 framed MCP and OpenClaw as "out of scope" (integration paths, not development surfaces). Reality: your quickstart docs show MCP and OpenClaw alongside Python / TS / Go SDKs as install paths — a developer wiring Sonzai into Claude Desktop or OpenClaw IS doing integration work. Closing the gap.

Cross-reference integrity: 398 internal references verified.

## v1.0.0 — 2026-05-13

Wizard-driven release. Diagnoses what the developer is building, prescribes archetype + memory mode + capabilities, drives them through spec → review → plan.

### Added

- **`intake.md`** — 7-question wizard interview with pre-question workspace-signal inference. Forks on greenfield vs existing-codebase. Maps answers to archetype + capabilities deterministically.
- **`existing-codebase-audit.md`** — 8-step insertion-point checklist for migrating existing apps. Routes to migration playbooks based on detected incumbent.
- **7 archetype playbooks** (`archetypes/`):
  - `companion.md` — 1:1 persistent companion
  - `guide-router.md` — MBTI / personality-routed (intake guide + N specialists)
  - `enterprise-assistant.md` — team-shared with shared memory + audit
  - `customer-support.md` — KB + custom tools + webhooks
  - `game-npc.md` — inventory + custom states + dialogue + events
  - `coach-therapist.md` — long sessions, sync memory, diary, mood
  - `hybrid-custom.md` — combinations / multi-tenant
- **20 feature references** (`features/`): full SDK surface — generation, inventory, custom-tools, custom-states, capabilities, voice, knowledge-base, org-knowledge-base, priming, personas, proactive, shared-memory, multiplayer-memory, instances, events-and-dialogue, agent-insights, self-improvement, models, eval-and-simulation, webhooks.
- **10 decision aids** (`decisions/`): memory-mode, state-vs-inventory, capabilities-matrix, sharedmemory-vs-wisdom, byok-vs-customllm, instances-vs-multitenant, sessions-vs-conversations, proactive-channel, post-processing-model, generation-vs-manual-create.
- **9 migration playbooks** (`migrations/`): overview, mem0, langchain, letta, zep, openai-assistants, character-ai, crm-csv, raw-json.
- **4 spec/plan templates** (`spec-templates/`) the wizard fills in: archetype-spec, archetype-plan, existing-codebase-spec, migration-spec.

### Changed

- **`SKILL.md`** tightened to <200 words narrative; routes to `intake.md` by default. Skip-wizard path falls through to per-language references.

### Kept from v0.1.0

- `references/drift-detection.md` — drift check against live OpenAPI (run as Step 0 of every interaction)
- `references/auth-and-setup.md`, `references/streaming-chat.md`, `references/migration-from-http.md`, `references/troubleshooting.md`
- `references/python.md`, `references/typescript.md`, `references/go.md` — per-language syntax lookups

### Source-of-truth verification

All capability flags, method names, and field references verified against the live OpenAPI spec at `https://api.sonz.ai/docs/openapi.json` and the three public SDK repos. Invented symbol caught and removed during verification: `personality_drift_disabled` (does not exist; replaced with prompt-shaping guidance).

### Cross-reference integrity

All 383 internal cross-references between skill files verified to resolve.

## v0.1.0 — 2026-05-12

Initial release. Reference library for SDK install, auth, chat, basic memory, sessions, BYOK, drift detection, troubleshooting. Per-language references for Python, TypeScript, Go.
