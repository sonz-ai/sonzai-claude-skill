# Changelog

All notable changes to `sonzai-claude-skill` are documented here. The project follows [Semantic Versioning](https://semver.org/). Dates are `YYYY-MM-DD`.

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
