# sonzai-sdk skill — v1.0.0 design

**Status:** spec (pre-implementation)
**Date:** 2026-05-13
**Replaces:** v0.1.0 (initial shipped 2026-05-12)
**Target repo:** `github.com/sonz-ai/sonzai-claude-skill`

---

## 1. Problem statement

The v0 skill ships a reference library — install commands, client init, chat, basic memory, sessions, BYOK, drift detection. It is a **syntax manual**. It fails as a **consultant**.

Concrete failure modes observed in the docs audit (covering 23 Sonzai feature areas across 123 docs):

1. v0 covers ~20% of the SDK surface. Missing: agent generation, inventory, custom tools, custom states, capabilities, the entire proactive stack (schedules / wakeups / events / notifications), shared/multiplayer memory, instances, voice, knowledge base, organization KB, priming, user personas, agent insights, events & dialogue, self-improvement, custom LLM, post-processing models, eval & simulation, webhooks.
2. Memory mode (`sync` vs `async`) is mentioned but never prescribed. Developers can't tell which to pick.
3. No vertical archetypes. A companion app, an enterprise assistant, a game NPC, and a personality-routed matchmaker need **different stacks**. v0 treats them identically.
4. No "guide → specialist" routing pattern (e.g. an intake agent that profiles the user via MBTI/Big5, then routes to one of N specialists). This is a real, repeatable architecture.
5. No existing-codebase audit. Developers replacing an existing chat/memory layer get zero guidance on insertion points or migration order.
6. No combinatorial guidance. The real power of Sonzai is combinations (Generation + Priming + Personas; Sessions + Inventory + Custom Tools; Shared Memory + KB + Wisdom). v0 documents primitives, not combinations.
7. No migration playbooks. The docs have rich migration paths (Mem0, LangChain, Letta, Zep, OpenAI Assistants, Character.AI, CRM CSV) that v0 ignores.
8. No "are you sure" gate. Some integrations don't need Sonzai's full stack; v0 will happily over-prescribe.

## 2. Goals (v1.0.0)

1. **Diagnose** what the developer is building before writing code (wizard intake).
2. **Prescribe** archetype + memory mode + capabilities deterministically from intake answers.
3. **Cover the full public SDK surface** so the wizard's recommendation can route to a real reference.
4. **Drive the user through a spec → review → plan flow** (delegate to `superpowers:brainstorming` and `superpowers:writing-plans` rather than reinventing them).
5. **Fork on greenfield vs existing codebase.** Both flows are first-class.
6. **Stay platform-agnostic.** Skill must work in Claude Code, Codex, Gemini CLI, Copilot CLI without per-platform branches in the skill body.
7. **Preserve v0's wins:** drift detection, per-language references, platform-leak safety, small router.

## 3. Non-goals

- Webhook **receiver** server code (developer's infra).
- Hosting recommendations (Vercel/Cloud Run/Fly).
- Deep MCP authoring (covered by separate `superpowers:mcp-builder`).
- OpenClaw integration internals (covered by separate plugin).
- Multi-tenant infrastructure for the developer's own app.
- Codex/Gemini-specific tool-wrapping (added later only if real divergence emerges; for now skill uses platform-agnostic instructions).

## 4. Architecture

### 4.1 Top-level flow

```
User invokes sonzai-sdk skill
        │
        ▼
   SKILL.md (router, <200 words)
   ── routes to intake.md unless user is mid-implementation
   ── always-loaded; the only file that lives in context
        │
        ▼
   intake.md (wizard)
   ── runs pre-question signal inference
   ── asks up to 7 questions (skips inferred ones)
   ── produces deterministic recommendation
        │
   ┌────┴────┐
   │         │
   ▼         ▼
 existing   greenfield
 audit       (skip audit)
   │         │
   └────┬────┘
        ▼
   load archetypes/{chosen}.md
   ── archetype playbook drives:
       (1) write sonzai-implementation-spec.md
           ── using spec-templates/archetype-spec.md.template
           ── following superpowers:brainstorming discipline
       (2) self-review the spec (placeholders, contradictions, ambiguity)
       (3) ask the user to review the spec
       (4) write sonzai-implementation-plan.md
           ── using spec-templates/archetype-plan.md.template
           ── following superpowers:writing-plans (atomic steps with verify gates)
        │
        ▼
   Hand off to user / superpowers:executing-plans for execution
```

### 4.2 File layout (~60 files)

```
sonzai-claude-skill/
├── README.md                            # (existing — refreshed)
├── CLAUDE.md                            # (existing — repo maintenance rules)
├── LICENSE                              # (existing)
├── package.json                         # (existing — version bump 0.1.0 → 1.0.0)
├── .claude-plugin/plugin.json           # (existing)
├── docs/
│   └── design/
│       └── 2026-05-13-skill-v1-design.md   # this document
└── skills/
    └── sonzai-sdk/
        ├── SKILL.md                     # router (tightened, <200 words)
        ├── intake.md                    # NEW — wizard interview
        ├── existing-codebase-audit.md   # NEW — insertion-point finder
        ├── archetypes/                  # 7 playbooks
        │   ├── companion.md
        │   ├── guide-router.md
        │   ├── enterprise-assistant.md
        │   ├── customer-support.md
        │   ├── game-npc.md
        │   ├── coach-therapist.md
        │   └── hybrid-custom.md
        ├── features/                    # 19 SDK-surface refs
        │   ├── generation.md
        │   ├── inventory.md
        │   ├── custom-tools.md
        │   ├── custom-states.md
        │   ├── capabilities.md
        │   ├── voice.md
        │   ├── knowledge-base.md
        │   ├── org-knowledge-base.md
        │   ├── priming.md
        │   ├── personas.md
        │   ├── proactive.md
        │   ├── shared-memory.md
        │   ├── multiplayer-memory.md
        │   ├── instances.md
        │   ├── events-and-dialogue.md
        │   ├── agent-insights.md
        │   ├── self-improvement.md
        │   ├── models.md
        │   ├── eval-and-simulation.md
        │   └── webhooks.md
        ├── decisions/                   # 10 decision aids
        │   ├── memory-mode.md
        │   ├── state-vs-inventory.md
        │   ├── capabilities-matrix.md
        │   ├── sharedmemory-vs-wisdom.md
        │   ├── byok-vs-customllm.md
        │   ├── instances-vs-multitenant.md
        │   ├── sessions-vs-conversations.md
        │   ├── proactive-channel.md
        │   ├── post-processing-model.md
        │   └── generation-vs-manual-create.md
        ├── migrations/                  # 9 migration playbooks
        │   ├── overview.md
        │   ├── mem0.md
        │   ├── langchain.md
        │   ├── letta.md
        │   ├── zep.md
        │   ├── openai-assistants.md
        │   ├── character-ai.md
        │   ├── crm-csv.md
        │   └── raw-json.md
        ├── spec-templates/              # 4 templates
        │   ├── archetype-spec.md.template
        │   ├── archetype-plan.md.template
        │   ├── existing-codebase-spec.md.template
        │   └── migration-spec.md.template
        └── references/                  # 8 EXISTING (v0)
            ├── drift-detection.md
            ├── auth-and-setup.md
            ├── python.md
            ├── typescript.md
            ├── go.md
            ├── streaming-chat.md
            ├── migration-from-http.md
            └── troubleshooting.md
```

**File counts:**
- Net new: 53 files (2 standalone + 7 archetypes + 20 features + 10 decisions + 9 migrations + 4 templates + 1 design doc *not in skill/*).
- Touched (rewritten): 1 file (`SKILL.md`).
- Kept (lightly polished): 8 existing `references/*.md`.
- **Total files inside `skills/sonzai-sdk/`: 60.**

### 4.3 Always-loaded vs on-demand

- **Always loaded** when skill is active: `SKILL.md` only. Must stay under 200 words.
- **Loaded by wizard:** `intake.md`, then exactly one archetype playbook + the existing-codebase audit if forked there.
- **Loaded on demand by archetype playbook:** specific feature/decision/migration files it references. Archetype playbook should list these explicitly so the wizard knows what to load.

This is the v0 pattern from `references/`. Extending it to ~60 files relies on the same discipline.

## 5. Detailed component specs

### 5.1 `SKILL.md` (tightened router)

**Frontmatter description rule:** describes WHEN to use, never WHAT it does. Per `superpowers:writing-skills` testing, summarizing the workflow in the description causes agents to follow the description and skip the body.

**Body:** ~150 words. Contents:
- Trigger conditions (imports, package names, env vars, mentions of api.sonz.ai, Sonzai agents/memory/personality/sessions, migration from raw HTTP, MBTI/router patterns).
- Step 0: drift check (load `references/drift-detection.md`) — mandatory.
- Step 1: load `intake.md` and run the wizard.
- Hard rules (no client-side API key, no platform internals, never guess endpoint names).
- 4 red flags.

### 5.2 `intake.md` (wizard interview)

#### Pre-question inference (signals)

Before asking anything, the wizard scans the workspace:

| Signal | Inferred answer |
|---|---|
| `pyproject.toml` / `requirements.txt` + source files | language=Python, project=existing |
| `package.json` with `"@sonzai-labs/agents"` in deps | language=TS, project=existing, Sonzai partially wired |
| `package.json` without Sonzai deps | language=TS, project=existing (no Sonzai yet) |
| `go.mod` + `.go` files | language=Go, project=existing |
| Workspace empty / README-only | project=greenfield |
| User prompt contains "companion / Replika-like" | archetype hint=companion (confirm) |
| User prompt contains "MBTI / personality router / matchmaker / route to specialist" | archetype hint=guide-router (confirm) |
| User prompt contains "team / shared / employees / enterprise" | archetype hint=enterprise-assistant (confirm) |
| User prompt contains "NPC / character / game / inventory" | archetype hint=game-npc (confirm) |
| User prompt contains "customer support / ticket / help desk" | archetype hint=customer-support (confirm) |
| User prompt contains "coach / therapist / journal / wellness" | archetype hint=coach-therapist (confirm) |

**Rule:** confirm aggressively in one line ("Detected existing TS project — yes?") rather than asking the question cold.

#### The 7 wizard questions

Asked in order, **skipping any answered by inference**. Each question has a fixed option set (the skill prescribes options; the host agent decides whether to use a structured selector like `AskUserQuestion` or plain prompt — platform-agnostic).

1. **Q1 — Project state**: existing codebase / greenfield. If existing, run `existing-codebase-audit.md` first, then resume.
2. **Q2 — Archetype**: companion / guide-router / enterprise-assistant / customer-support / game-npc / coach-therapist / hybrid-or-other. The last option falls through to `archetypes/hybrid-custom.md`.
3. **Q3 — User identification model**: anonymous-at-intake / stable-user-id / team-shared.
4. **Q4 — Personality behavior**: drift on / overlays-only / brand-locked.
5. **Q5 — Proactive features**: none / scheduled-reminders / backend-events / both.
6. **Q6 — Latency budget for first token**: <500ms / 500ms-2s / 2s+.
7. **Q7 — Language**: Python / TypeScript / Go (skip if inferred).

Plus archetype-specific follow-ups (defined per archetype playbook in section 4 of the playbook):
- guide-router: "How many specialists?" "Personality framework (MBTI / Big5 / OCEAN / custom)?" "Are specialists pre-defined or auto-generated?"
- enterprise: "Number of users sharing one agent?" "KB scope (project-only / org-only / cascade)?"
- customer-support: "Webhook channels needed (ticketing, escalation)?"
- game-npc: "Inventory schema needed?" "Multi-NPC dialogue?"
- coach-therapist: "Session length (typical)?" "Diary visibility (user-facing or internal)?"
- companion: "Voice needed?" "Image generation needed?"

#### Answer-to-stack mapping

A deterministic table mapping (Q2, Q3, Q4, Q5, Q6) → (archetype, memory_mode, shared_memory, capabilities). See `decisions/capabilities-matrix.md` for the full grid. Edge cases not in the table → wizard explicitly tells the user *"This combination isn't in the standard playbook; assembling from feature references — confirm?"*

#### Output handoff

End of intake produces a structured recommendation block (archetype, memory_mode, shared_memory toggle, capabilities to enable per agent), then loads `archetypes/{chosen}.md` and hands off.

#### Skip / escape hatches

- User types "skip wizard" / "I know what I want" → fall through to reference mode (existing v0 behavior).
- User answers Q2 with "none of these" → load `archetypes/hybrid-custom.md`.
- Wizard detects in-progress Sonzai integration (existing Sonzai calls in open file) → short-circuit to `references/troubleshooting.md` or the matching feature ref.

### 5.3 `existing-codebase-audit.md`

Runs when Q1 = existing. Checklist:

1. **Find the current chat handler.** Grep for `openai.chat.completions`, `anthropic.messages.create`, `google.generativeai`, `langchain.*`, `llama_index.*`, `mem0.*`, `letta.*`, `zep.*`. Catalog what exists.
2. **Find user state.** Identify how the app currently identifies users (sessions, JWTs, cookies, DB user_id column). This becomes the `user_id` passed to Sonzai.
3. **Find memory layer.** What's storing conversation history today? In-process? Redis? Postgres `messages` table? Mem0/Zep/Letta? — drives the migration playbook choice.
4. **Find personality config.** System prompts? Agent definitions in a config file? — informs whether to use `generate_and_create` or `agents.create(...)` with explicit Big5.
5. **Find LLM provider config.** Which provider/model? Pinned or dynamic? — drives BYOK vs Custom LLM decision.
6. **Find webhook receivers (if any).** Existing event handling shape.
7. **Find KB / RAG layer (if any).** Existing vector store, document upload pipeline.
8. **Find proactive (scheduled jobs).** Cron, Celery, BullMQ, Temporal — what's running today.

Produces an **insertion-point report** + **suggested migration order** (strangler pattern: replace memory layer first, then personality, then proactive, etc.). Then routes to `migrations/{matched-source}.md` if a known source is detected, else to the standard archetype flow.

### 5.4 Archetype playbooks (`archetypes/*.md`)

All 7 playbooks follow a uniform 7-section structure:

```markdown
---
name: archetype-{name}
description: Use when {triggering conditions}
---

# {Archetype name}

## 1. When this archetype fits
- 3-5 concrete signals
- 2-3 anti-signals (when NOT to use this archetype)

## 2. Prescribed stack
| Decision | Value | Why |
|---|---|---|
| memory_mode | async | <reason> |
| personality drift | on | <reason> |
| shared_memory | off | <reason> |
| knowledge_base | optional | <reason> |
| voice | optional | <reason> |
| ... | ... | ... |

## 3. Required SDK functions (in order)
1. Create agent (with code snippets in Python/TS/Go)
2. Configure capabilities
3. (archetype-specific extra steps)
4. Run chat / session loop
5. (any archetype-specific concluding steps)

## 4. Archetype-specific wizard intake questions
The 2-4 extra questions the wizard asks after picking this archetype.

## 5. Spec template fields
What the wizard fills into sonzai-implementation-spec.md for this archetype.

## 6. Plan template (typical step breakdown)
The atomic steps the wizard writes into sonzai-implementation-plan.md,
following superpowers:writing-plans format (each step has a verify gate).

## 7. Anti-patterns for this archetype
- 3-5 specific traps with explanations
```

#### 5.4.1 `companion.md`

1:1 persistent companion (Replika-shaped).

**Prescribed stack:** memory_mode=async, drift=on, shared_memory=off, voice=optional (often yes), image_generation=optional, scheduled_reminders=optional, web_search=usually off.

**Extra Q4:** Voice? Image generation?

**Anti-patterns:** Don't enable `shared_memory` (leaks across users). Don't use sync memory with voice (blows live-audio turn budget). Don't disable personality drift (defeats the purpose).

#### 5.4.2 `guide-router.md` (MBTI / personality-routed)

Intake guide agent + N specialist agents + routing by personality score.

**Prescribed stack:**
- **Guide agent:** memory_mode=async (intake should be fast), drift=off (consistent intake), capabilities minimal (no web_search, no KB, no image gen).
- **Specialist agents:** memory_mode=sync (each user's session matters for the specialist's quality), drift=on, capabilities per-specialist (web_search likely on, knowledge_base optional).
- Use `custom_states` on the user to carry the assessment result (`{"mbti": "INFJ", "guide_complete": true}`).
- Use `agents.generation.generate_and_create` with personality framework prompts to bootstrap the N specialists (e.g. 16 MBTI types).

**Extra Q4:** Number of specialists? Framework (MBTI / Big5 / OCEAN / custom)? Pre-define each specialist's `compiled_system_prompt` or auto-generate from framework + framework-type label?

**Anti-patterns:** Don't store the personality score in the guide agent's memory only — store it in `custom_states` scoped to the user so any specialist can read it. Don't create specialists ad-hoc per user — create them once at deploy time, reuse `agent_id` across users. Don't route based on a single conversation turn — gather enough signal first.

**Sample architecture:** 1 guide + N specialists, all in one project, all sharing the project-scoped KB. User flow: anonymous → talks to guide → guide writes `custom_state.assessment` → app routes to specialist based on assessment → all subsequent chats target the specialist's `agent_id` with the same `user_id`.

#### 5.4.3 `enterprise-assistant.md`

Team-shared agent serving multiple employees.

**Prescribed stack:** memory_mode=sync (compliance-friendly), drift=overlays-only (consistent brand, per-user perception), shared_memory=on (team context), wisdom=on (k-anonymized cross-user facts), knowledge_base=on, knowledge_base_scope=cascade (project + org), audit trail enabled.

**Extra Q4:** Number of users? KB scope (project_only / org_only / cascade)? Privacy floor categories (compensation, health, etc.)?

**Anti-patterns:** Don't enable `shared_memory` without setting the privacy floor (regulatory risk). Don't use BYOM with audit if your custom LLM isn't logging (loses compliance trail). Don't share custom_states across the team — they're per-user by design.

#### 5.4.4 `customer-support.md`

KB-backed support agent with webhook fanout.

**Prescribed stack:** memory_mode=sync, drift=off (brand consistency), shared_memory=on (CS team learns from each other's tickets), knowledge_base=on (FAQ, product docs), web_search=usually off (KB-grounded), custom_tools=yes (`create_ticket`, `escalate`, `lookup_order`), webhooks=on (event fanout to your ticketing system).

**Extra Q4:** Ticketing system to integrate? Escalation channels (Slack, email, PagerDuty)? KB documents to upload?

**Anti-patterns:** Don't let the agent answer outside the KB (hallucination risk on policy questions). Don't omit `Authorization` headers on custom-tool callbacks. Don't forget to register the `agent.message.created` webhook for transcript persistence.

#### 5.4.5 `game-npc.md`

Game NPC / character with inventory and events.

**Prescribed stack:** memory_mode=async, drift=on (NPCs evolve with players), shared_memory=off, inventory=on, custom_states=on (level, faction, quest flags), custom_tools=on (`spend_currency`, `give_item`, `award_xp`), events=on (`level_up`, `boss_defeated`), dialogue=on (multi-NPC interactions).

**Extra Q4:** Single NPC or cast of N? Inventory schema (typed items)? Player-vs-shared-NPC state?

**Anti-patterns:** Don't use `custom_states` for inventory (use `inventory.create()` — it has schema validation and KB integration). Don't trigger backend events for routine actions (events are for moments, not every action). Don't share the same `instance_id` across game shards — use `instances.create()` per shard.

#### 5.4.6 `coach-therapist.md`

Long-session coach / therapist / journaler.

**Prescribed stack:** memory_mode=sync (every fact matters), drift=slow (cap drift rate via capabilities), shared_memory=off, session.end(wait=True) (must consolidate before next session), priming=on (intake assessment seeds memory), agent_insights=on (habits/goals/diary visible to user), web_search=off.

**Extra Q4:** Session length (typical)? Diary visibility (user-facing / internal-only)? Intake assessment shape (PHQ-9, custom)?

**Anti-patterns:** Don't use async memory (a coaching session can't afford to lose a fact). Don't skip `wait=True` on session end (next session might be hours later but if it's hot you need consolidation). Don't expose the agent's diary verbatim to the user — summarize.

#### 5.4.7 `hybrid-custom.md`

For combinations / unknown shapes.

Walks the developer through assembling from `features/` directly. Provides a decision tree: "your app is X parts companion + Y parts enterprise → take companion's memory_mode rule, enterprise's KB rule, ..." etc.

Anti-patterns: don't try to enable everything ("kitchen sink agent" — slow, expensive, behaviorally inconsistent).

### 5.5 Feature references (`features/*.md`)

One per major SDK surface. Each file follows a uniform structure:

```markdown
---
name: feature-{name}
description: Use when {feature trigger conditions}
---

# {Feature name}

## What it is
1-2 sentences.

## When to use
Bulleted use cases.

## When NOT to use
Bulleted anti-cases (alternatives listed).

## SDK surface
Method signatures with one paragraph each, organized by sub-surface.

## Code examples
Python + TS + Go for the most common operations.

## Decisions linked to this feature
Cross-references to relevant decisions/*.md files.

## Common gotchas
3-5 specific traps.
```

Specific contents per file (one-line summaries for the implementation phase):

- **generation.md** — `agents.generation.generate_and_create()` (full agent from prompt); `generate_character()` (preview personality without committing). Idempotency on `agent_id`. When to use vs explicit `agents.create()`. Cross-ref: `decisions/generation-vs-manual-create.md`.
- **inventory.md** — `agents.inventory.create()` / `update()` / `query()`. KB schema validation. `disambiguation_needed` handling. When to use vs `custom_states`. Cross-ref: `decisions/state-vs-inventory.md`.
- **custom-tools.md** — Agent-level vs session-level tool scoping. `agents.createCustomTool()`, `agents.sessions.setTools()`. Tool-call results in `sideEffects.externalToolCalls`. Reserved `sonzai_` prefix. Webhook delivery on tool fire (optional).
- **custom-states.md** — Two scopes (global / per-user). Three content types (text / json / binary). `create / upsert / get_by_key / delete_by_key / list`. Distinct from `inventory` (no schema).
- **capabilities.md** — `get_capabilities / update_capabilities`. PATCH-style (omitted fields unchanged). Full list of toggleable capabilities. Platform-managed vs developer-managed fields. `*UnlockedAt` timestamps.
- **voice.md** — TTS / STT / live duplex (WebSocket). `voices.list()` for global catalog. `agents.voice.getToken() + stream()` for live. Audio formats (PCM 24kHz on output). Latency budgets.
- **knowledge-base.md** — Upload documents, insert facts, list nodes, semantic search. KB schemas back both KB and Inventory. Three ingestion paths: manual upload, agent edit, ETL push. Audit trail.
- **org-knowledge-base.md** — Org-scoped vs project-scoped KB. Four scope modes (project_only / org_only / cascade / union). Cascade recommended; project wins on collision.
- **priming.md** — `prime_user()`, `batch_import()`, `get_import_status()`. Async jobs. Metadata vs content blocks. Dedup across blocks. Migration source for Mem0/Zep/Letta/Character.AI.
- **personas.md** — `user_personas.create / list / get / delete`. Tenant-scoped library. Attach at priming or chat time. One default per tenant. Free-form `style` instruction.
- **proactive.md** — Three sources (schedules / wakeups / events) × three delivery channels (SSE / polling / webhooks). Decision tree. Timezone-aware cadence. Quiet hours filtering. Inventory linkage (reminder reads live `inventory.query()` results).
- **shared-memory.md** — Wisdom (default-on, k-anonymized) vs shared_memory (opt-in, attributed). Disclosure audit. Server-side privacy validator. Privacy floor categories.
- **multiplayer-memory.md** — Inter-agent (shared KB across project agents) vs intra-agent (shared context on one agent across users). Decision tree. KB write quotas.
- **instances.md** — `agents.instances.create / list / reset / delete`. Per-instance custom state isolation. Personality + memory remain global. Multi-region pattern.
- **events-and-dialogue.md** — `agents.triggerBackendEvent()` (backend → agent). `agents.dialogue()` (agent-to-agent turn). Event metadata as soft context. Orchestrating multi-agent NPC scenes.
- **agent-insights.md** — Read-only derived signals: habits, goals, interests, relationships, diary, constellation, breakthroughs. Update latency. Dashboard patterns.
- **self-improvement.md** — Triggered by `sessions.end()`. Per-pair RL/bandits. Diary writing. Personality drift. Mood update. Fact extraction + dedup. Post-processing model is configurable.
- **models.md** — Providers (Gemini default, OpenAI, xAI, OpenRouter). Fallback chains on 429. BYOK (your key, our integration). Custom LLM / BYOM (your endpoint, OpenAI-compatible). Post-processing model map (cheap fast tier for extraction/mood/personality).
- **eval-and-simulation.md** — `agents.evaluate`, `agents.simulate`, `agents.simulate_async`, `agents.run_eval`, `agents.eval_only`. Templates: `eval_templates.create / update / delete`. Runs: `eval_runs.list / get / stream_events / delete`. Reconnectable streaming.
- **webhooks.md** — Register / list / rotate_secret / delete. Project-scoped variants. HMAC-SHA256 verification (raw bytes, timing-safe compare). Delivery attempts inspection. Event names (`agent.message.created`, `agent.created`, etc.).

### 5.6 Decision aids (`decisions/*.md`)

Each is short (~300 words), structured as:

```markdown
# Decision: {topic}

## The rule
One sentence. State the answer.

## How to apply
3-5 sentences. When to use the rule. Exceptions.

## Why
The reasoning (latency tradeoff / compliance need / cost / drift behavior).

## Exceptions
2-3 cases where the rule flips, with the override criterion.

## Cross-references
Links to feature files and archetypes that depend on this decision.
```

The 10 decisions:

1. **memory-mode.md** — *Default sync. Switch to async iff (a) you need first-token latency under your turn budget AND (b) you can tolerate facts spilling into the next turn. Voice and games → async. Compliance, coaching, support → sync.*
2. **state-vs-inventory.md** — *Use custom_states for primitives, counters, flags. Use inventory for items with schema, identity, multiple typed properties.*
3. **capabilities-matrix.md** — Reference grid: archetype × capability → on/off/optional. Used by the wizard's mapping table.
4. **sharedmemory-vs-wisdom.md** — *Wisdom is default-on, k-anonymized, attribution-stripped. Shared memory is opt-in, attributed, requires privacy floor.*
5. **byok-vs-customllm.md** — *BYOK if you want billing isolation with our integrations. Custom LLM if you have a fine-tuned model or self-hosted stack. Default: neither — use platform routing.*
6. **instances-vs-multitenant.md** — *Instances for sharded deploys of the same agent (regions, dev/staging/prod). Multi-tenant for separate customers — use separate projects + agents instead.*
7. **sessions-vs-conversations.md** — *Conversations (`agents.chat`) for stateless single-turn. Sessions (`agents.sessions.start`) for multi-turn loops needing fresh context per turn, tool calls, or explicit lifecycle.*
8. **proactive-channel.md** — *SSE if user is active. Polling if mobile / web client with no persistent server connection. Webhooks for server-to-server fanout. Mix as needed.*
9. **post-processing-model.md** — *Default Gemini Flash Lite. Override per chat-model if extraction quality matters more than latency/cost. Wildcard fallback applies.*
10. **generation-vs-manual-create.md** — *Use `generate_and_create()` when you have a natural-language description and want personality + bio + seed memories auto-derived. Use `agents.create()` when you have explicit Big5 scores or need exact control.*

### 5.7 Migration playbooks (`migrations/*.md`)

Each migration playbook walks the developer through replacing a specific incumbent with Sonzai. Structure:

```markdown
# Migrating from {source}

## What {source} provides
1-paragraph summary of what the user is moving away from.

## Field mapping
Table: {source} concept → Sonzai concept.

## Migration order (strangler)
Numbered steps. Each step independently shippable.

## Data import path
Use priming.batch_import for bulk; priming.prime_user for one-by-one.

## Code shape before/after
Side-by-side snippet.

## Gotchas specific to {source}
3-5 traps.

## Cross-references
Feature files involved.
```

The 9 migration playbooks:

1. **overview.md** — Entry point. Decision tree: "which source are you moving from?" Routes to the matching file.
2. **mem0.md** — Mem0 → Sonzai. Map `mem0.add` → `priming.batch_import` or `memory.bulk_create_facts`. Map `mem0.search` → `agents.memory.search`.
3. **langchain.md** — LangChain ConversationBufferMemory / VectorStoreRetrieverMemory → Sonzai sessions + memory.
4. **letta.md** — Letta (MemGPT) agents → Sonzai agents. Map Letta tools → Sonzai custom tools.
5. **zep.md** — Zep facts → `agents.memory.bulk_create_facts`. Zep sessions → Sonzai sessions.
6. **openai-assistants.md** — OpenAI Assistants API → Sonzai. Map threads → sessions. Map assistant instructions → `compiled_system_prompt` + `personality`. Map function calling → custom tools.
7. **character-ai.md** — Character.AI personas → Sonzai `generate_and_create` from description. Map chat history → priming.
8. **crm-csv.md** — Bulk CSV import → `priming.batch_import`. Map CRM fields → user metadata.
9. **raw-json.md** — Arbitrary JSON store → priming with manual fact extraction or `memory.bulk_create_facts`.

### 5.8 Spec/plan templates (`spec-templates/*.template`)

Four templates the wizard writes into the user's project.

1. **archetype-spec.md.template** — Standard archetype implementation spec. Fields: project name, archetype, language, agents to create (with capabilities), data sources for priming, integration points, success criteria, out-of-scope.
2. **archetype-plan.md.template** — Atomic implementation steps with verify gates. Follows `superpowers:writing-plans` exactly.
3. **existing-codebase-spec.md.template** — Adds fields: incumbent system (chat/memory/personality/proactive layers), insertion points, migration order, rollback plan.
4. **migration-spec.md.template** — Adds fields: source system, data-import plan (priming batch jobs), feature parity matrix (source feature → Sonzai equivalent → status).

## 6. Cross-cutting concerns

### 6.1 Platform-leak safety (unchanged from v0)

Repo CLAUDE.md prohibits any reference to platform internals (`services/contextengine`, `services/ai-service`, tenant names, deploy/infra). Source of truth for the skill: public SDK repos + live OpenAPI spec at `https://api.sonz.ai/docs/openapi.json` + public docs at `https://sonz.ai/docs`. This rule extends across all 60 files.

### 6.2 Cross-platform compatibility (Claude Code / Codex / Gemini CLI / Copilot CLI)

Skill body uses platform-agnostic instructions:
- "Ask the user X" (not "use AskUserQuestion")
- "Show the user this content" (not "use the Write tool to scaffold a file")
- "Decide based on signals in the workspace" (not "use `mcp__claude-in-chrome__*`")

If a real divergence emerges (e.g. Codex has no equivalent of structured questions), add a `references/codex-tools.md` shim — same as superpowers does. Not adding pre-emptively.

### 6.3 Source-of-truth discipline

Every endpoint name, parameter name, response field referenced in any file must be verifiable in either:
- A current public SDK repo (`sonzai-python`, `sonzai-typescript`, `sonzai-go`)
- The live OpenAPI spec at `https://api.sonz.ai/docs/openapi.json`

Skills must not invent symbols. When unsure, link to the live spec rather than guessing. This is enforced by:
- Drift detection running as Step 0 of every interaction (catches stale references)
- Implementation phase: every snippet sanity-checked against the actual SDK source

### 6.4 Versioning

- Repo follows semver. v1.0.0 ships the full spec.
- Patch releases (1.0.x) are bug fixes only — typos, broken code snippets, fixed link rot.
- Minor releases (1.x.0) reserved for: net-new SDK features that didn't exist at v1.0.0 publication; new migration sources; new archetypes (rare).
- No major release planned. Major bumps only if Sonzai SDK has a breaking redesign requiring a structural skill rewrite.

### 6.5 Maintenance burden

Estimated touchpoints per major SDK release:
- New endpoint → 1 feature file + 0-1 archetype updates + drift-detection snapshot bump
- New capability flag → `capabilities.md` + `capabilities-matrix.md` + relevant archetypes
- New provider → `models.md` + drift detection
- Schema change (renamed field) → grep across all 60 files; touched by drift detection first

Mitigation: feature files **link** to live spec sections rather than inline schemas. Code snippets are minimal and idiomatic, not exhaustive.

## 7. Testing strategy

Per `superpowers:writing-skills` Iron Law: no skill without a failing test first.

### 7.1 Baseline tests (run before writing — RED)

Dispatch a fresh subagent **without the skill** and assign it the canonical scenario for each archetype:

| Archetype | Scenario |
|---|---|
| companion | "Build a Python chat app where the agent's personality evolves with the user." |
| guide-router | "Build an MBTI matchmaker: user talks to a guide, takes assessment, gets routed to one of 16 specialist agents." |
| enterprise-assistant | "Build a TS Slack bot serving a 50-person team, with shared memory and KB ingestion of policy docs." |
| customer-support | "Build a Python support bot that creates tickets and escalates to Slack." |
| game-npc | "Build a Go service for an NPC with inventory and faction state." |
| coach-therapist | "Build a Python wellness coach with weekly journal entries the user can read." |

Document what the agent does wrong without the skill (likely: invents endpoint names, picks sync memory for voice, fails to use generation, hardcodes API key on client side, etc.).

### 7.2 Validation tests (after writing — GREEN)

Re-run the same scenarios with the skill installed. Verify the wizard:
- Detects the archetype correctly
- Prescribes the right memory mode
- Loads the right archetype playbook
- Produces a spec that matches the user's intent
- Produces a plan with atomic verifiable steps

### 7.3 Refactor tests (close loopholes — REFACTOR)

Run pressure scenarios:
- User insists they want sync memory for voice (anti-pattern). Wizard should push back with explanation.
- User asks to put API key in a React client component. Wizard should refuse and explain.
- User has 17 specialist agents but framework=MBTI (which has 16). Wizard should flag mismatch.
- User on Cloudflare Workers (no long-lived connections) → wizard should auto-pick async polling, not SSE.

### 7.4 What we won't test pre-ship (accept user-reported risk)

- Every migration playbook end-to-end (9 sources × 3 languages = 27 paths) — sanity-check the field mappings, ship, rely on real reports.
- Every feature file's code snippet executed against live API — sanity-check syntactically + against SDK signatures, ship, rely on real reports.

## 8. Implementation plan (high level — full plan via `superpowers:writing-plans`)

Rough phasing of the *implementation* work (this is NOT the same as user-facing v0/v1 phasing; we still ship as one v1.0.0):

**Phase A — Backbone (sequential):**
1. Tighten existing `SKILL.md` to the new router shape.
2. Write `intake.md` (the wizard).
3. Write `existing-codebase-audit.md`.
4. Write `decisions/capabilities-matrix.md` (the wizard's mapping table is here).

**Phase B — Archetypes (can parallelize across 7 agents):**
5-11. Write the 7 archetype playbooks. Each follows the uniform 7-section template.

**Phase C — Decisions (parallelizable):**
12-21. Write the 10 decision aids. Short (~300 words each), driven by the archetype-specific calls.

**Phase D — Features (parallelizable in batches of ~5):**
22-41. Write the 20 feature files. Each ~400-700 words.

**Phase E — Migrations (parallelizable):**
42-50. Write the 9 migration playbooks.

**Phase F — Templates:**
51-54. Write the 4 spec/plan templates.

**Phase G — Testing:**
55. Run baseline tests (RED).
56. Run validation tests (GREEN).
57. Run refactor tests (REFACTOR — find loopholes, close them).

**Phase H — Polish + ship:**
58. Update README.md with v1 install + flow description.
59. Bump version (0.1.0 → 1.0.0). Add CHANGELOG.md.
60. Tag and release.

Each item becomes one or more atomic steps in the `superpowers:writing-plans` output. Phases B/C/D/E are parallelizable — `superpowers:subagent-driven-development` can dispatch them to multiple subagents.

## 9. Success criteria

v1.0.0 is "done" when:

1. All 60 files exist and are complete (no TBDs, no TODOs in published content).
2. Every code snippet in every file has been sanity-checked against the actual SDK source (Python, TS, or Go as relevant) — no invented symbols.
3. The wizard correctly routes the 6 canonical scenarios (section 7.1) to the right archetype, memory mode, and capabilities.
4. Pressure tests (section 7.3) pass — wizard pushes back on anti-patterns instead of complying.
5. Drift detection still passes against the current live OpenAPI spec.
6. README.md describes the v1 flow accurately.
7. Published to `github.com/sonz-ai/sonzai-claude-skill` with a v1.0.0 git tag.

## 10. Open questions / explicitly deferred

- **Subagent dispatch in the wizard.** Could the wizard itself dispatch subagents to write the spec + plan in parallel? Deferred — single-agent flow is enough for v1.
- **Visual companion** (mockups, flowcharts rendered in browser). Deferred — skill is text-only.
- **Telemetry hooks** (anonymous usage data on which archetype is picked most). Deferred — skill is local, no phone-home.
- **Localized variants** (Chinese, Japanese skill bodies matching the docs' three languages). Deferred — v1 is English-only; cross-link to localized docs at `sonz.ai/docs/{lang}/...`.
- **MCP-server variant** (the skill exposed as an MCP server other agents can call). Deferred — separate effort.

## 11. Risks

| Risk | Mitigation |
|---|---|
| Drift between skill and SDK accelerates as Sonzai ships | Drift detection runs Step 0; feature files link to live spec sections; quarterly review cycle |
| Wizard feels imposing / users skip it | Explicit skip path (Q2 "none" or "skip wizard"); falls through to v0-style reference mode |
| Code snippets break with SDK upgrades | Snippets are minimal; CI could lint snippets against latest SDK type defs (future) |
| Token cost of loading large archetype files | Always-loaded is SKILL.md only (under 200 words); rest is on-demand |
| 60-file maintenance burden compounds | One feature per file (low coupling); decision aids are short; migrations are isolated |
| Public skill leaks platform internals | Repo-level CLAUDE.md prohibition; review every PR for source paths |
| Sonzai's B2B customers ask for customer-named archetypes (e.g. "Razer mode") | Refuse — repo CLAUDE.md prohibits customer/tenant names. Route unusual combinations through `hybrid-custom.md` instead. |
| Developer building a multi-tenant SaaS doesn't see their pattern covered | `archetypes/hybrid-custom.md` walks them through assembling from `features/`; `decisions/instances-vs-multitenant.md` covers the partitioning question. |

---

**End of design spec.**
