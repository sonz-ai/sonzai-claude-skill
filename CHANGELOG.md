# Changelog

All notable changes to `sonzai-claude-skill` are documented here. The project follows [Semantic Versioning](https://semver.org/). Dates are `YYYY-MM-DD`.

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
