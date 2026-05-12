# Sonzai SDK Skill v1.0.0 Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use `superpowers:subagent-driven-development` (recommended) or `superpowers:executing-plans` to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Ship `sonzai-sdk` skill v1.0.0 — a wizard-driven Claude Code / Codex / Gemini CLI skill covering the full public Sonzai SDK surface across 60 skill files.

**Architecture:** Wizard intake (`intake.md`) routes via `existing-codebase-audit.md` or directly to one of 7 archetype playbooks. Archetype playbooks drive the user through spec → review → plan using `superpowers:brainstorming` and `superpowers:writing-plans` discipline. Decision aids, feature references, and migration playbooks are pulled on demand by archetypes. `SKILL.md` is the only always-loaded file (<200 words).

**Tech Stack:** Markdown skill files (no code build). Cross-platform skill format (Claude Code / Codex / Gemini CLI / Copilot CLI). Source-of-truth: live OpenAPI at `https://api.sonz.ai/docs/openapi.json` + the three public SDK repos (`sonzai-python`, `sonzai-typescript`, `sonzai-go`). Design spec: `docs/design/2026-05-13-skill-v1-design.md`.

**Repo:** `/Volumes/CORSAIR/code/sonzai/sonzai-claude-skill/` — already public at `github.com/sonz-ai/sonzai-claude-skill` (v0.1.0 shipped 2026-05-12).

---

## Source material — read before starting

Every task references one or more of these. Open them in a worktree or keep paths handy:

- **SDK READMEs (canonical method signatures):**
  - `/Volumes/CORSAIR/code/sonzai/sonzai-sdk/sonzai-python/README.md`
  - `/Volumes/CORSAIR/code/sonzai/sonzai-sdk/sonzai-typescript/README.md`
  - `/Volumes/CORSAIR/code/sonzai/sonzai-sdk/sonzai-go/doc.go` + `/Volumes/CORSAIR/code/sonzai/sonzai-sdk/sonzai-go/api.md`
- **SDK source roots (for grep-verifying symbols):**
  - `/Volumes/CORSAIR/code/sonzai/sonzai-sdk/sonzai-python/src/sonzai/`
  - `/Volumes/CORSAIR/code/sonzai/sonzai-sdk/sonzai-typescript/src/`
  - `/Volumes/CORSAIR/code/sonzai/sonzai-sdk/sonzai-go/` (flat package)
- **Public docs (feature semantics):** `/Users/nasdin/code/sonzai/sonzai-landing/content/docs/en/` — one .mdx per feature
- **Live OpenAPI:** `curl -sSfL https://api.sonz.ai/docs/openapi.json -o /tmp/live.openapi.json` (for endpoint/field verification)
- **Existing v0 skill files** (kept; consult for tone/structure): `/Volumes/CORSAIR/code/sonzai/sonzai-claude-skill/skills/sonzai-sdk/references/*.md`
- **Design spec:** `/Volumes/CORSAIR/code/sonzai/sonzai-claude-skill/docs/design/2026-05-13-skill-v1-design.md` — read sections §5 (component specs) before any task

---

## Final file structure (locked from spec §4.2)

```
sonzai-claude-skill/
└── skills/
    └── sonzai-sdk/
        ├── SKILL.md
        ├── intake.md
        ├── existing-codebase-audit.md
        ├── archetypes/   { companion, guide-router, enterprise-assistant,
        │                   customer-support, game-npc, coach-therapist,
        │                   hybrid-custom }.md
        ├── features/     { generation, inventory, custom-tools, custom-states,
        │                   capabilities, voice, knowledge-base,
        │                   org-knowledge-base, priming, personas, proactive,
        │                   shared-memory, multiplayer-memory, instances,
        │                   events-and-dialogue, agent-insights,
        │                   self-improvement, models, eval-and-simulation,
        │                   webhooks }.md
        ├── decisions/    { memory-mode, state-vs-inventory, capabilities-matrix,
        │                   sharedmemory-vs-wisdom, byok-vs-customllm,
        │                   instances-vs-multitenant, sessions-vs-conversations,
        │                   proactive-channel, post-processing-model,
        │                   generation-vs-manual-create }.md
        ├── migrations/   { overview, mem0, langchain, letta, zep,
        │                   openai-assistants, character-ai, crm-csv,
        │                   raw-json }.md
        ├── spec-templates/ { archetype-spec, archetype-plan,
        │                     existing-codebase-spec, migration-spec }.md.template
        └── references/   { drift-detection, auth-and-setup, python, typescript,
                            go, streaming-chat, migration-from-http,
                            troubleshooting }.md   # EXISTING — kept, light polish
```

---

## Task index

| Phase | Tasks | Description | Parallel? |
|---|---|---|---|
| **A** Backbone | 1-4 | Tighten SKILL.md, write intake.md, existing-codebase-audit.md, decisions/capabilities-matrix.md | sequential |
| **B** Archetypes | 5-11 | 7 archetype playbooks (with TDD baseline test each) | parallel after A |
| **C** Decisions | 12-20 | 9 remaining decision aids | parallel after A |
| **D** Features | 21-40 | 20 feature references | parallel after A |
| **E** Migrations | 41-49 | 9 migration playbooks | parallel after A |
| **F** Templates | 50-53 | 4 spec/plan templates | parallel after A |
| **G** Testing | 54-55 | Pressure tests + link/symbol verification | sequential after B-F |
| **H** Polish + ship | 56-58 | README, CHANGELOG, v1.0.0 tag + push | sequential after G |

**Total tasks: 58.** Phases B/C/D/E/F (43 tasks) are fully parallelizable — recommended for subagent dispatch.

---

## Conventions for every task

- **Working directory:** `/Volumes/CORSAIR/code/sonzai/sonzai-claude-skill/`
- **Commit style:** Conventional Commits. Prefix: `feat(skill):` for new files, `docs(skill):` for SKILL.md / README, `chore:` for version bumps.
- **Symbol verification:** Before committing any file with code snippets, grep each method/field name against the SDK source to verify it exists. Example: `rg "generate_and_create" /Volumes/CORSAIR/code/sonzai/sonzai-sdk/sonzai-python/src/`. If a symbol can't be grep-confirmed, fix the snippet or remove the claim.
- **Frontmatter:** Every skill file (except `SKILL.md` and templates) starts with:
  ```yaml
  ---
  name: <kebab-case-name>
  description: Use when <triggering conditions only, no workflow summary>
  ---
  ```
- **Cross-reference convention:** Link to other skill files with relative paths from `skills/sonzai-sdk/`, e.g. `[memory-mode](../decisions/memory-mode.md)`.
- **No emojis** in any skill file content (per project CLAUDE.md).
- **No platform internals** referenced anywhere (per `sonzai-claude-skill/CLAUDE.md`).

---

# Phase A — Backbone (sequential)

These four tasks must complete in order. The wizard (Task 2) depends on the capabilities matrix (Task 4)'s structure — but we write the matrix last in this phase because the wizard's flow informs what columns it needs.

## Task 1: Tighten `SKILL.md` to the new router shape

**Files:**
- Modify: `skills/sonzai-sdk/SKILL.md` (overwrite — existing v0 content is replaced)

- [ ] **Step 1: Read the existing v0 SKILL.md** at `skills/sonzai-sdk/SKILL.md` to preserve its hard rules section and red flags. The new router replaces the rest.

- [ ] **Step 2: Write the new SKILL.md**

```markdown
---
name: sonzai-sdk
description: Use when the user is installing, configuring, or writing code against the Sonzai SDK (pip install sonzai, npm install @sonzai-labs/agents, go get github.com/sonz-ai/sonzai-go), calling api.sonz.ai, working with Sonzai agents/memory/personality/sessions, or migrating from raw HTTP curl calls to the typed SDK.
---

# Sonzai SDK

Wizard-driven implementation guide for the Sonzai Mind Layer API across Python, TypeScript, and Go.

## When to use

- User imports `sonzai`, `@sonzai-labs/agents`, or `github.com/sonz-ai/sonzai-go`
- User runs `pip install sonzai`, `npm install @sonzai-labs/agents`, `go get github.com/sonz-ai/sonzai-go`
- User has `SONZAI_API_KEY` in their env or `.env`
- User calls `api.sonz.ai`, mentions Sonzai agents, memory, personality, mood, sessions
- User is converting raw HTTP/curl calls to a typed SDK
- User mentions building a companion / matchmaker / personality-routed app / enterprise assistant / game NPC / coach / customer support agent

## Step 0 — Drift check (REQUIRED)

The committed OpenAPI snapshot inside an installed SDK version can lag the live spec. Always run the drift check before writing non-trivial integration code.

→ Read `references/drift-detection.md`

## Step 1 — Run the wizard

The wizard diagnoses what you're building, prescribes the archetype + memory mode + capabilities, then drives you through spec → review → plan.

→ Read `intake.md`

## Skip the wizard

If the user explicitly says "skip wizard" / "I know what I want" / they're mid-implementation and just need a syntax lookup:

| User task | Reference |
|---|---|
| Streaming chat (SSE) vs async polling | `references/streaming-chat.md` |
| Auth, env vars, base URL | `references/auth-and-setup.md` |
| Python syntax | `references/python.md` |
| TypeScript syntax | `references/typescript.md` |
| Go syntax | `references/go.md` |
| Raw HTTP → typed SDK | `references/migration-from-http.md` |
| Errors, 4xx/5xx | `references/troubleshooting.md` |

## Hard rules

1. **Never expose `SONZAI_API_KEY` to a browser or mobile client.** Server-side only.
2. **Never guess endpoint names or field names.** Fetch the live OpenAPI spec — see `references/drift-detection.md`.
3. **Never invent tenant-specific behavior.** This SDK is multi-tenant.

## Red flags

- "The README shows `client.foo.bar()` but my IDE says it doesn't exist" → drift. Step 0.
- "I'm writing a Next.js client component and need to call the agent" → wrong place. Server-side only.
- "Let me just curl the endpoint" → check the typed SDK first (`references/migration-from-http.md`).
- User pinned an old SDK version → check drift before assuming README matches reality.
```

- [ ] **Step 3: Verify word count is under 200 (excluding frontmatter, code blocks)**

```bash
wc -w skills/sonzai-sdk/SKILL.md
# Expected: under 400 total (frontmatter+body); the "narrative" portion under 200
```

- [ ] **Step 4: Commit**

```bash
git add skills/sonzai-sdk/SKILL.md
git commit -m "docs(skill): tighten SKILL.md as router for v1 wizard flow"
```

---

## Task 2: Write `intake.md` — the wizard interview

**Files:**
- Create: `skills/sonzai-sdk/intake.md`

This is the most-loaded file after SKILL.md. Subagent baseline-tested.

- [ ] **Step 1: RED — Run baseline test**

Dispatch a fresh subagent (use the Agent tool, `subagent_type: Explore`) with this prompt:

> "A user wants to build an MBTI-style matchmaker app in TypeScript. They want a guide agent to assess the user's personality, then route them to one of 16 specialist agents (one per MBTI type). The user has the Sonzai TypeScript SDK installed (`@sonzai-labs/agents`). Without reading any skill files, write a 200-word recommendation: which Sonzai functions should they use, what memory mode, what capabilities to enable, and what's the agent creation pattern? Return the recommendation."

Save the output to a notes file at `/tmp/baseline-intake.md`. Look for failures:
- Did the agent pick `agents.generation.generate_and_create` for the specialists, or did it hardcode personality strings?
- Did it use `custom_states` to carry the assessment, or shove it into the conversation history?
- Did it prescribe async vs sync memory at all?
- Did it differentiate guide-agent capabilities vs specialist capabilities?
- Did it create specialists ad-hoc per user (wrong) or once at deploy (right)?

Document the failure modes — these are what the wizard must address.

- [ ] **Step 2: Write `intake.md`**

The file must contain:

**Header:**
```yaml
---
name: sonzai-intake
description: Use when the user is starting a new Sonzai integration (greenfield or existing codebase) and needs a recommendation on archetype, memory mode, capabilities, and implementation order.
---

# Sonzai integration wizard

Diagnose what you're building, prescribe the stack, drive you through spec → plan.
```

**Section 1 — Pre-question inference**

A table listing the signals to detect before asking anything (from spec §5.2). Instruct the host agent: "Before asking Q1, scan the workspace for these signals. If detected, *confirm* in one line rather than asking the question cold."

Include the full table from spec §5.2 (pyproject.toml, package.json with @sonzai-labs/agents, go.mod, empty workspace, prompt-text hints).

**Section 2 — The 7 wizard questions**

Numbered Q1 through Q7. Each question shows:
- The question text (one sentence)
- The exact option labels (use the spec's options verbatim)
- The branch to take based on the answer

Q1 — Project state (existing → run `existing-codebase-audit.md` then resume; greenfield → continue)
Q2 — Archetype (7 options: companion / guide-router / enterprise-assistant / customer-support / game-npc / coach-therapist / hybrid-or-other)
Q3 — User identification model (anonymous / stable-user-id / team-shared)
Q4 — Personality behavior (drift on / overlays-only / brand-locked)
Q5 — Proactive features (none / scheduled-reminders / backend-events / both)
Q6 — Latency budget for first token (<500ms / 500ms-2s / 2s+)
Q7 — Language (Python / TypeScript / Go; skip if inferred)

**Section 3 — Archetype-specific follow-ups**

After picking an archetype in Q2, the wizard asks 2-4 more questions specific to that archetype (defined in section 4 of each archetype playbook). Quote the spec §5.2 list (the archetype-specific Q4 examples for each of the 7 archetypes).

**Section 4 — Answer-to-stack mapping**

The deterministic table from spec §5.2. Columns: Q2, Q3, Q4, Q5, Q6 → archetype, memory_mode, shared_memory, capabilities. Refer to `decisions/capabilities-matrix.md` for the full grid (since that file is the canonical version — Task 4).

**Section 5 — Output handoff**

The structured recommendation block format (from spec §5.2):

```
RECOMMENDATION
==============
Archetype:      <name>
Language:       <lang>
Memory mode:    <sync|async|per-agent override>
Shared memory:  <on|off>
Capabilities:   <list per agent>

NEXT: loading archetypes/<chosen>.md
      → playbook drives spec → review → plan
```

**Section 6 — Skip / escape hatches**

Three escape paths from spec §5.2: "skip wizard" → reference mode; "none of these" archetype → hybrid-custom.md; in-progress Sonzai code detected → troubleshooting.md or matching feature.

**Section 7 — Anti-patterns the wizard must reject**

If the user's answer combination violates a rule (e.g. wants sync memory + voice), the wizard must push back, not comply. Examples:
- Sync memory + live voice (Q6 <500ms) → reject; voice requires async
- Companion archetype (Q2) + team-shared users (Q3) → reject; companion is 1:1, suggest enterprise or hybrid
- Game NPC (Q2) + drift=off (Q4) → flag; NPCs typically benefit from drift, confirm intent

- [ ] **Step 3: GREEN — Re-run baseline test with the file present**

Same subagent prompt as Step 1, but this time the subagent has access to `intake.md`. Verify it now:
- Prescribes `agents.generation.generate_and_create` for the 16 specialists
- Uses `custom_states` to carry the assessment
- Picks `memory_mode=async` for the guide, mixed for specialists
- Differentiates capabilities between guide and specialists
- Creates specialists once at deploy

If any of these are still wrong, refine `intake.md` and re-run.

- [ ] **Step 4: Symbol verification**

```bash
# Verify every method named in intake.md exists in the SDKs
rg "generate_and_create" /Volumes/CORSAIR/code/sonzai/sonzai-sdk/sonzai-python/src/
rg "customStates" /Volumes/CORSAIR/code/sonzai/sonzai-sdk/sonzai-typescript/src/
# (and similar for every method mentioned)
```

- [ ] **Step 5: Word count check**

```bash
wc -w skills/sonzai-sdk/intake.md
# Target: 800-1500 words. Wizard needs enough detail to drive routing.
```

- [ ] **Step 6: Commit**

```bash
git add skills/sonzai-sdk/intake.md
git commit -m "feat(skill): add intake.md wizard for v1"
```

---

## Task 3: Write `existing-codebase-audit.md`

**Files:**
- Create: `skills/sonzai-sdk/existing-codebase-audit.md`

- [ ] **Step 1: Write the file**

The file is a checklist with grep commands. Structure:

```yaml
---
name: existing-codebase-audit
description: Use when integrating Sonzai into an existing codebase that already has chat, memory, personality, or LLM-provider code in place.
---

# Existing-codebase audit

Find the insertion points. Determine migration order. Route to the right playbook.

## How to use

Run the 8 audit steps in order. Each produces a one-line finding. At the end, you have a complete insertion-point report and a strangler-pattern migration order.
```

Then 8 numbered audit sections, each with:
- The thing to find
- Grep commands to find it
- What to record
- Implications for the rest of the integration

**Audit 1 — Find the current chat handler.** Grep for `openai.chat.completions`, `anthropic.messages.create`, `google.generativeai`, `langchain.chat_models`, `llama_index.llms`, `mem0`, `letta`, `zep`. Record file paths.

**Audit 2 — Find user state.** Grep for `req.user`, `session.user_id`, `current_user`, JWT decoding (`jwt.verify`, `jose`), DB user models. Record the canonical user ID source — this becomes `user_id` in Sonzai calls.

**Audit 3 — Find memory layer.** Grep for `redis`, `RedisStore`, `messages` table queries, `mem0`, `Zep`, `LettaClient`, `pinecone`, `chroma`, `weaviate`, conversation buffer logic in LangChain. Record what's storing history.

**Audit 4 — Find personality config.** Grep for `system_prompt`, `assistant_prompt`, `instructions`, `character_card`, `persona`. Catalog every prompt template.

**Audit 5 — Find LLM provider config.** Grep for `OPENAI_API_KEY`, `ANTHROPIC_API_KEY`, model name strings (`gpt-4`, `gpt-5`, `claude-3`, etc.). Drives BYOK vs Custom LLM decision.

**Audit 6 — Find webhook receivers.** Grep for HTTP handlers verifying HMAC signatures, route paths containing `/webhook`, `/event`, `/callback`. Record the receiver shape.

**Audit 7 — Find KB / RAG layer.** Grep for `embeddings`, `vector_store`, `cosine_similarity`, document loaders, retriever code.

**Audit 8 — Find proactive jobs.** Grep for `cron`, `Celery`, `BullMQ`, `Temporal`, `node-cron`, `apscheduler`. Catalog recurring tasks.

After the 8 audits, include:

**Migration order (strangler pattern):**

| Order | Layer | Why first |
|---|---|---|
| 1 | Memory | Pure replacement; doesn't change UX. Replace your buffer/Mem0/Zep with `agents.memory.bulk_create_facts` + `agents.memory.search`. |
| 2 | Personality | Move system prompts into agent definition (`agents.create` with `compiled_system_prompt` or `personality_prompt`). Same agent behavior; richer state. |
| 3 | LLM provider routing | Switch from raw provider call to `agents.chat`. BYOK if you want to keep billing on your provider account. |
| 4 | KB | Migrate vector store contents to `client.knowledge` (or run both during cutover). |
| 5 | Proactive | Migrate cron jobs to `schedules.create` / `wakeups`. |
| 6 | Webhooks | Last — register Sonzai webhooks to fan out to your existing handlers. |

**Routing to migration playbook:**

| Detected incumbent | Migration playbook |
|---|---|
| Mem0 | `migrations/mem0.md` |
| LangChain ConversationBufferMemory | `migrations/langchain.md` |
| Letta / MemGPT | `migrations/letta.md` |
| Zep | `migrations/zep.md` |
| OpenAI Assistants API | `migrations/openai-assistants.md` |
| Character.AI | `migrations/character-ai.md` |
| CSV / CRM bulk data | `migrations/crm-csv.md` |
| Arbitrary JSON | `migrations/raw-json.md` |
| Custom / unknown | `migrations/overview.md` |

After migration routing, the audit resumes the wizard at Q2 (archetype selection).

- [ ] **Step 2: Word count**

```bash
wc -w skills/sonzai-sdk/existing-codebase-audit.md
# Target: 600-1000 words
```

- [ ] **Step 3: Commit**

```bash
git add skills/sonzai-sdk/existing-codebase-audit.md
git commit -m "feat(skill): add existing-codebase-audit.md"
```

---

## Task 4: Write `decisions/capabilities-matrix.md`

**Files:**
- Create: `skills/sonzai-sdk/decisions/capabilities-matrix.md`

This is the canonical mapping table that `intake.md` (Task 2) cross-references. Write it last in Phase A so the wizard's flow is settled.

- [ ] **Step 1: Read source**

- `sonzai-python/README.md` — search for `tool_capabilities`, `update_capabilities`, `memory_mode`
- `sonzai-typescript/README.md` — same
- `sonzai-landing/content/docs/en/connect-managed-runtime.mdx` and `sonzai-landing/content/docs/en/proactive-overview.mdx` for capability semantics
- `curl -sSfL https://api.sonz.ai/docs/openapi.json | jq '.components.schemas | keys | .[] | select(test("[Cc]apabilit"))'` to find the live capability schemas

- [ ] **Step 2: Write the file**

```yaml
---
name: capabilities-matrix
description: Use when picking which agent capabilities to enable for a given archetype.
---

# Decision: capabilities matrix
```

**The rule:** Capabilities default OFF. Enable only what the archetype needs. Each `update_capabilities` call is PATCH-style (omitted fields unchanged).

**Capability reference table** (full grid of toggleable capabilities found in the SDK):

| Capability | What it does | Cost/risk if enabled |
|---|---|---|
| `memory_mode` (sync/async) | Memory recall mode | Async: faster TTFC, facts may spill to next turn |
| `web_search` | Agent can search the web | Cost per call; can hallucinate sources |
| `image_generation` | Agent can generate images | Cost per image |
| `voice_generation` | Agent has voice (TTS+STT+live) | Latency-sensitive; requires async memory |
| `knowledge_base` | Agent reads from project KB | Adds latency; usually wanted |
| `knowledge_base_write` | Agent writes to KB autonomously | Audit trail required; quota concerns |
| `remember_name` | Agent retains user name across sessions | Privacy implication |
| `inventory` | Inventory subsystem available | None except cost-of-storage |
| `shared_memory` | Cross-user memory attribution | Privacy floor required |
| `personality_drift_disabled` | Brand-lock personality | Compliance use; defeats compounding |

(Verify each name above against `tool_capabilities` schema in the live OpenAPI.)

**Archetype × Capability grid:**

| Capability | Companion | Guide (intake) | Specialist | Enterprise | Customer Support | Game NPC | Coach |
|---|---|---|---|---|---|---|---|
| memory_mode | async | async | sync | sync (or async if voice) | sync | async | sync |
| personality_drift | on | off (consistent intake) | on | overlays only | off | on | on (slow) |
| web_search | usually off | off | usually on | on | off (KB-grounded) | off | off |
| image_generation | optional | off | optional | off | off | optional | off |
| voice_generation | optional | off | optional | optional | off | off | optional |
| knowledge_base | optional | off | optional | on (cascade) | on | optional | optional |
| knowledge_base_write | off | off | optional | on (audited) | optional | off | off |
| shared_memory | off | off | off | on | on | off | off |
| inventory | off | off | off | off | off | on | off |

**How to apply:**

1. Pick archetype.
2. Read the column for that archetype.
3. Call `agents.update_capabilities(agent_id, ...)` with the matching values.
4. For guide-router: the guide and specialists have different columns — call `update_capabilities` separately per agent.

**Why these defaults?**
- Async memory anywhere voice is on (TTFC budget)
- Sync memory anywhere compliance/audit matters (every fact lands this turn)
- KB writes always require audit (regulatory)
- Drift off for brand-locked agents (consistency over evolution)

**Exceptions:**
- If your latency budget is generous (>2s TTFC), sync memory is fine even for game NPCs.
- If your compliance regime requires logging every retrieval, sync + audit log even for companions.

**Cross-references:**
- `decisions/memory-mode.md` — the sync/async decision in depth
- `decisions/sharedmemory-vs-wisdom.md` — privacy floor when shared_memory is on
- `features/capabilities.md` — the SDK surface (`get_capabilities`, `update_capabilities`)

- [ ] **Step 3: Symbol verification**

```bash
# Confirm every capability name exists in the SDK
rg "memory_mode|web_search|image_generation|voice_generation|knowledge_base|knowledge_base_write|inventory|shared_memory|personality_drift_disabled|remember_name" \
   /Volumes/CORSAIR/code/sonzai/sonzai-sdk/sonzai-python/src/
```

If any name is missing or different, fix the table.

- [ ] **Step 4: Commit**

```bash
git add skills/sonzai-sdk/decisions/capabilities-matrix.md
git commit -m "feat(skill): add capabilities-matrix decision aid"
```

---

# Phase B — Archetype playbooks (parallel after Phase A)

All 7 follow the uniform 7-section structure from spec §5.4. Each archetype has its own TDD baseline test.

**Universal task structure for archetypes** (every archetype task uses these 5 steps):

1. **RED** — Run baseline subagent test (archetype-specific prompt)
2. **Write** the file (using the 7-section structure)
3. **GREEN** — Re-run subagent; verify it now produces the right plan
4. **Verify symbols** (grep code snippets against SDK source)
5. **Commit** with message `feat(skill): add archetype-{name} playbook`

The 7-section structure (refer to spec §5.4 for the full template):
1. When this archetype fits (3-5 signals + 2-3 anti-signals)
2. Prescribed stack (table from `decisions/capabilities-matrix.md`)
3. Required SDK functions in order (with Python/TS/Go snippets)
4. Archetype-specific wizard intake questions (2-4 follow-ups)
5. Spec template fields specific to this archetype
6. Plan template (typical step breakdown for `superpowers:writing-plans`)
7. Anti-patterns (3-5 specific traps)

## Task 5: `archetypes/companion.md`

**Files:**
- Create: `skills/sonzai-sdk/archetypes/companion.md`

- [ ] **Step 1: RED baseline test**

Subagent prompt: *"Build a Python chat app where the agent's personality evolves with the user. The user starts conversation, has multi-week sessions, and the agent should remember them and shift personality based on rapport. Recommend Sonzai functions and capabilities."*

Look for failures: did the agent pick async memory? Personality drift on? Single agent per user vs shared? Sessions vs raw chat? Voice optional? Image generation optional? Did it remember to handle session.end with wait=True for memory-critical tests?

- [ ] **Step 2: Write the file** following the 7-section structure

**Prescribed stack (Section 2):**

| Decision | Value | Why |
|---|---|---|
| memory_mode | async | TTFC matters; companion is interactive |
| personality_drift | on | The whole point is compounding rapport |
| shared_memory | off | 1:1 — leaking across users is a privacy bug |
| knowledge_base | off | Companion is about the user, not a corpus |
| voice_generation | optional | High-engagement add-on |
| image_generation | optional | Common ask |
| scheduled_reminders | optional | "check in" patterns |
| web_search | usually off | Companion is internal-facing |

**Required SDK functions (Section 3):**
1. `agents.generation.generate_and_create(name, description, language)` (or `agents.create` if explicit Big5)
2. `agents.update_capabilities(agent_id, memory_mode="async", ...)`
3. `agents.sessions.start(agent_id, user_id=, session_id=, provider=, model=)`
4. `session.context(query=...)` → use for prompt building
5. `session.turn(messages=...)` per user message
6. `session.end(total_messages=, duration_seconds=, wait=False)` (wait=True only for tests)
7. Optional: `agents.scheduleWakeup(...)` for reminders; `agents.voice.getToken / stream` for voice

Include Python, TS, Go code snippets for steps 1, 3, 5, 6.

**Archetype-specific wizard Qs (Section 4):**
- Voice needed (yes/no)?
- Image generation needed (yes/no)?
- Scheduled check-ins needed (yes/no, frequency)?
- Per-user agent or one agent serving all users (per-user → user-overlay derives automatically; one-agent → simpler)?

**Spec template fields (Section 5):**
- Agent creation: name + description for `generate_and_create`, or explicit Big5
- Voice: voice_name from `voices.list()`, language code
- Image generation: enable/disable
- Reminder cadence
- Session lifecycle: when does a session start/end (user-driven vs time-bounded)

**Plan template (Section 6) — typical 8 atomic steps:**
1. Create agent via `generate_and_create` — verify: agent appears in `agents.list()`
2. Set capabilities (`memory_mode=async`, optional `voice_generation=true`, `image_generation=true`) — verify: `get_capabilities` returns expected
3. Build server-side chat handler — verify: POST /chat returns 200 with content
4. Implement session lifecycle (start on connect, end on disconnect with wait=False) — verify: session_id consistent across requests
5. (Optional) Add scheduled wakeup — verify: `notifications.list` returns it after the cadence
6. (Optional) Wire voice via `agents.voice.stream` — verify: PCM frames arriving
7. Integration test: 10-message session — verify: `agents.memory.search` finds facts from messages 1-3
8. Production checklist: SONZAI_API_KEY in secret manager, error boundaries on chat handler, BYOK if billing isolation required

**Anti-patterns (Section 7):**
- Don't enable `shared_memory` (1:1 — would leak across users)
- Don't use sync memory + voice (blows TTFC budget)
- Don't disable personality drift (defeats compounding — the SOTOPIA s30 lift goes away)
- Don't share `agent_id` across distinct user populations without testing — personality overlay per user is automatic but assumes one logical agent
- Don't poll memory immediately after `session.end(wait=False)` — consolidation hasn't run; use `wait=True` in tests

- [ ] **Step 3: GREEN re-run with the file present** — verify baseline failures are fixed

- [ ] **Step 4: Symbol verification**

```bash
rg "generate_and_create|update_capabilities|sessions.start|scheduleWakeup" \
   /Volumes/CORSAIR/code/sonzai/sonzai-sdk/sonzai-python/src/
```

- [ ] **Step 5: Commit**

```bash
git add skills/sonzai-sdk/archetypes/companion.md
git commit -m "feat(skill): add archetype-companion playbook"
```

---

## Task 6: `archetypes/guide-router.md`

**Files:**
- Create: `skills/sonzai-sdk/archetypes/guide-router.md`

- [ ] **Step 1: RED baseline test**

Subagent prompt: *"Build an MBTI matchmaker in TypeScript. An anonymous user lands on the app, talks to a guide agent that assesses their personality, then routes them to one of 16 specialist agents (one per MBTI type) for the rest of their interactions. How should this be architected with the Sonzai SDK?"*

Look for failures: did the agent create 16 specialists once at deploy or ad-hoc per user? Did it use `custom_states` for the assessment or stuff it in messages? Did it differentiate guide vs specialist capabilities? Did it use `agents.generation.generate_and_create` with personality prompts for the 16 types?

- [ ] **Step 2: Write the file** following the 7-section structure

**Prescribed stack (Section 2):**

| Decision | Guide agent | Specialist agents |
|---|---|---|
| memory_mode | async | sync |
| personality_drift | off (consistent intake) | on |
| shared_memory | off | off |
| knowledge_base | off | optional |
| web_search | off | usually on |
| image_generation | off | optional |

**Required SDK functions (Section 3):**

1. **At deploy time, once:** Create the guide agent
   ```python
   guide = client.agents.generation.generate_and_create(
       name="Personality Guide",
       description="Conducts a brief, friendly assessment to understand the user's personality type. Asks open-ended questions about preferences, decision-making, and social energy. Never reveals which type the user is — only stores the result.",
       language="en",
   )
   client.agents.update_capabilities(guide.agent_id, memory_mode="async", personality_drift_disabled=True)
   ```

2. **At deploy time, once:** Create the N specialists (loop for MBTI=16 types or whatever framework)
   ```python
   MBTI_PROFILES = {
       "INTJ": "Strategic, independent, decisive...",
       "INFJ": "Insightful, idealistic, compassionate...",
       # ... 14 more
   }
   import uuid
   NAMESPACE = uuid.UUID("your-uuid-namespace-here")
   specialists = {}
   for code, description in MBTI_PROFILES.items():
       agent_id = str(uuid.uuid5(NAMESPACE, f"specialist-{code}"))
       agent = client.agents.create(
           agent_id=agent_id,
           name=f"Companion-{code}",
           personality_prompt=description,
           language="en",
       )
       client.agents.update_capabilities(agent_id, memory_mode="sync", web_search=True)
       specialists[code] = agent_id
   ```

3. **At runtime — user opens app (anonymous):** Generate a stable user_id (e.g. session cookie → uuid5)
4. **At runtime — start guide session:** `client.agents.sessions.start(guide.agent_id, user_id=user_id, ...)`
5. **At runtime — guide finishes assessment:** detect via guide producing a tool call or structured response → write to `custom_states`:
   ```python
   client.agents.custom_states.upsert(
       guide.agent_id,
       key="mbti_assessment",
       value={"type": "INFJ", "confidence": 0.87, "assessed_at": "..."},
       scope="user",
       user_id=user_id,
   )
   ```
6. **At runtime — route to specialist:** read `custom_states.get_by_key`, pick `specialists[mbti_type]`
7. **At runtime — start specialist session:** `client.agents.sessions.start(specialists[mbti], user_id=user_id, ...)`

Include TS and Go variants.

**Archetype-specific wizard Qs (Section 4):**
- How many specialists (default 16 for MBTI; 5 for Big5 quadrants; custom N)?
- Personality framework (MBTI / Big5 / OCEAN / custom)?
- Pre-defined specialist personalities (you write the prompts) or auto-generated (`generate_and_create` per type)?
- Anonymous-at-intake (most common) or known user_id from app start?
- One assessment per user (typical) or re-assessment allowed?

**Spec template fields (Section 5):**
- Number of specialists + their identifiers (e.g. MBTI codes)
- Framework choice + scoring rubric (how does the guide produce a result?)
- Specialist personality prompts (one per type) or auto-generation prompt
- Assessment storage key in `custom_states`
- Routing logic location (server-side after guide turn, or detected via tool call)

**Plan template (Section 6) — typical 12 atomic steps:**

1. Create guide agent — verify exists
2. Set guide capabilities (drift off, async, no KB) — verify
3. Loop: create N specialist agents with deterministic agent_ids — verify all N exist
4. Set specialist capabilities (sync, drift on, optional web_search/KB) — verify
5. Implement anonymous-user → user_id derivation — verify stable across requests
6. Implement guide chat handler — verify guide responds
7. Implement assessment detection (tool call OR structured terminal-turn output) — verify it fires
8. Write assessment to `custom_states.upsert(scope="user")` — verify retrievable
9. Implement routing function: read state → pick specialist — verify produces correct agent_id
10. Implement specialist chat handler (different from guide handler — uses specialist agent_id) — verify
11. End-to-end test: anon user → guide → assessment → routed → specialist conversation — verify
12. Re-assessment flow (optional): allow user to retake → overwrite custom_state → re-route

**Anti-patterns (Section 7):**
- Don't store the personality result in guide-only memory (specialists can't see it). Use `custom_states` scoped to the user.
- Don't create specialists ad-hoc per user. Create once at deploy, reuse `agent_id`. Otherwise you'll fragment population data and burn quota.
- Don't route based on a single guide turn. Gather 5-10 exchanges first.
- Don't enable `personality_drift_disabled` on specialists — they should evolve with the user once routed.
- Don't share `session_id` between guide and specialist. Start a new session when routing.
- Don't expose the personality type to the user before consent (if applicable).

- [ ] **Step 3: GREEN re-run** — verify the baseline failures are fixed.

- [ ] **Step 4: Symbol verification**

```bash
rg "custom_states|customStates|generate_and_create|sessions.start" \
   /Volumes/CORSAIR/code/sonzai/sonzai-sdk/sonzai-python/src/
rg "customStates|generation.generateAndCreate" \
   /Volumes/CORSAIR/code/sonzai/sonzai-sdk/sonzai-typescript/src/
```

- [ ] **Step 5: Commit**

```bash
git add skills/sonzai-sdk/archetypes/guide-router.md
git commit -m "feat(skill): add archetype-guide-router playbook"
```

---

## Task 7: `archetypes/enterprise-assistant.md`

**Files:**
- Create: `skills/sonzai-sdk/archetypes/enterprise-assistant.md`

- [ ] **Step 1: RED** — Subagent prompt: *"Build a TypeScript Slack bot serving a 50-person engineering team. The bot should learn from each person's questions, share institutional knowledge across the team, ingest the team's policy docs, and maintain an audit trail of what it surfaces. How should this be architected with Sonzai?"*

Look for failures: did the agent enable `shared_memory`? Sync memory? Cascade KB scope (project + org)? Audit trail enabled?

- [ ] **Step 2: Write** — 7-section structure.

**Prescribed stack:** memory_mode=sync, drift=overlays-only, shared_memory=on, knowledge_base=on, knowledge_base_scope=cascade, knowledge_base_write=on (audited), web_search=on, audit=on.

**Required SDK functions:**
1. `agents.create(...)` with `compiled_system_prompt` for brand voice
2. `agents.update_capabilities(... shared_memory=True, knowledge_base=True, knowledge_base_scope_mode="cascade", audit=True)`
3. Upload policy docs: `client.knowledge.uploadDocument(project_id, ...)`
4. (Optional) Insert structured org-level facts: `client.knowledge.createOrgNode(...)`
5. Per-user chat handler using `agents.sessions.start` with the user's Slack `user_id`
6. Webhook registration for `agent.message.created` → log to your SIEM
7. Privacy floor setup (`shared_memory_privacy_categories=["compensation","health"]`) — verify the exact field name in OpenAPI before writing

**Archetype-specific Qs:**
- Number of users (informs KB write quota planning)
- KB scope (project_only / org_only / cascade — recommended)
- Privacy floor categories (compensation, health, PII)
- Audit destination (Slack channel / SIEM / S3 bucket)

**Plan template — 10 steps:** create agent, enable capabilities including audit, upload KB docs, register webhook for audit, set up Slack handler with per-user session, integration test with 3 users sharing one fact, verify audit log captures fact sharing, verify privacy floor blocks compensation talk.

**Anti-patterns:**
- Don't enable `shared_memory` without setting the privacy floor categories
- Don't BYOM to a non-logging custom LLM and claim compliance
- Don't use async memory if your audit regime requires every retrieval to land in the same turn
- Don't share `custom_states` across users — they're per-user by design
- Don't omit `Authorization` headers on custom-tool callbacks back to your backend

- [ ] **Step 3: GREEN re-run** — verify.
- [ ] **Step 4: Symbol verification.**
- [ ] **Step 5: Commit** with message `feat(skill): add archetype-enterprise-assistant playbook`.

---

## Task 8: `archetypes/customer-support.md`

**Files:**
- Create: `skills/sonzai-sdk/archetypes/customer-support.md`

- [ ] **Step 1: RED** — *"Build a Python customer support bot that answers questions from a product knowledge base, creates tickets in Zendesk via a custom tool, and escalates urgent issues to a Slack channel via webhook. How should this be architected with Sonzai?"*

Failures to watch: did it enable `knowledge_base=on` + `web_search=off` (KB-grounded)? Did it set up `custom_tools` for `create_ticket` and `escalate`? Did it register a webhook for `agent.message.created`?

- [ ] **Step 2: Write** — 7-section structure.

**Prescribed stack:** memory_mode=sync, drift=off (brand consistency), shared_memory=on (CS team learns from each other), knowledge_base=on, web_search=usually off, custom_tools=yes, webhooks=on.

**Required SDK functions:**
1. `agents.create` with `compiled_system_prompt` + `personality_drift_disabled=True`
2. `agents.update_capabilities(... shared_memory=True, knowledge_base=True)`
3. `client.knowledge.uploadDocument` for FAQ/product docs
4. `agents.createCustomTool` for `create_ticket(...)`, `escalate(...)`, `lookup_order(...)`
5. `client.webhooks.register("agent.message.created", webhookUrl=...)` for transcript persistence
6. Tool-call sink: read `sideEffects.externalToolCalls` from chat response, dispatch to your backend

**Archetype-specific Qs:**
- Ticketing system (Zendesk / Intercom / Jira / custom)
- Escalation channels (Slack channel ID, PagerDuty, email)
- KB documents to upload at deploy
- Tool authentication strategy (per-call HMAC signed by Sonzai webhook → your endpoint verifies)

**Plan template — 11 steps:** create agent, disable drift, upload KB, register custom tools, register webhook, implement tool callback receiver (your server), implement HMAC verify, implement Slack escalation handler, integration test: ticket creation flow, integration test: escalation flow, audit trail verification.

**Anti-patterns:**
- Don't let the agent answer outside the KB without explicit web_search enable (hallucinations on policy)
- Don't skip HMAC verification on tool callbacks (security)
- Don't forget to handle webhook delivery failures (deliveryAttempts inspection)
- Don't enable `image_generation` (off-topic risk)
- Don't share `compiled_system_prompt` updates without invalidating caches

- [ ] **Steps 3-5:** GREEN / verify / commit (`feat(skill): add archetype-customer-support playbook`).

---

## Task 9: `archetypes/game-npc.md`

**Files:**
- Create: `skills/sonzai-sdk/archetypes/game-npc.md`

- [ ] **Step 1: RED** — *"Build a Go service for an NPC in a multiplayer game. The NPC manages player inventory items (weapons, currency), has faction state per player, evolves their personality as the player interacts, and reacts to backend events like `level_up` and `boss_defeated`. How should this be architected with Sonzai?"*

Failures: inventory vs custom_states confusion; missing events wiring; not enabling `inventory=on`; using same `instance_id` across game shards.

- [ ] **Step 2: Write** — 7-section structure.

**Prescribed stack:** memory_mode=async, drift=on, shared_memory=off, inventory=on, custom_states=on, custom_tools=on, events=on, dialogue=optional, web_search=off, knowledge_base=optional (game lore).

**Required SDK functions:**
1. `agents.create` with NPC personality
2. `agents.update_capabilities(... memory_mode="async", inventory=True)`
3. `agents.inventory.create(...)` with KB schema for items (the schema must exist first via `client.knowledge.createSchema`)
4. `agents.custom_states.upsert(... key="faction_standing", scope="user")` for per-player state
5. `agents.createCustomTool(name="spend_currency", ...)` + similar for `give_item`, `award_xp`
6. `agents.triggerBackendEvent(agent_id, event_type="level_up", metadata={...})` from your game server
7. `agents.instances.create` per game shard (regional server) — `instance_id` isolates state
8. (Optional) `agents.dialogue` for two-NPC scenes

Include Go snippets for the typical loop.

**Archetype-specific Qs:**
- Single NPC or cast of N
- Inventory schema (weapons, currency, consumables, etc.) — needed before `inventory.create`
- Player-vs-shared NPC state (typical: state is per-player)
- Sharding strategy (one instance per region? one global instance?)
- Events triggered (level_up, boss_defeated, quest_complete, etc.)

**Plan template — 14 steps:** create NPC agent, set capabilities, create KB schema for inventory items, create instances per shard, implement custom tools (`spend_currency` etc.), wire tool callbacks back to game server, implement event triggers in game loop, implement chat handler with per-player `user_id` + correct `instance_id`, inventory operations (`create`/`update`/`query`), state operations, integration test: full play loop with item interactions, integration test: event-driven NPC reaction, dialogue scene (optional), production rollout.

**Anti-patterns:**
- Don't use `custom_states` for inventory items (use `inventory` — schema-validated, KB-integrated)
- Don't trigger backend events for routine actions (events are for moments, not every action)
- Don't share `instance_id` across game shards (state will bleed)
- Don't enable `web_search` on game NPCs (lore leakage risk)
- Don't expose Sonzai API key to the game client (only your game server holds it)
- Don't forget to set `instance_id` on every chat — defaults to "default" which isolates nothing

- [ ] **Steps 3-5:** GREEN / verify / commit (`feat(skill): add archetype-game-npc playbook`).

---

## Task 10: `archetypes/coach-therapist.md`

**Files:**
- Create: `skills/sonzai-sdk/archetypes/coach-therapist.md`

- [ ] **Step 1: RED** — *"Build a Python wellness coach. Long weekly sessions, the user can read a journal the agent writes about them, personality drift is slow, mood is tracked across sessions. How should this be architected with Sonzai?"*

Failures: async memory (wrong for coach); skipping `session.end(wait=True)`; not exposing `diary` to user; missing mood tracking surface.

- [ ] **Step 2: Write** — 7-section structure.

**Prescribed stack:** memory_mode=sync, drift=on but slow, shared_memory=off, priming=on (intake), agent_insights=on (diary visible to user), web_search=off.

**Required SDK functions:**
1. `agents.create` with coach personality
2. `agents.update_capabilities(... memory_mode="sync")`
3. `agents.priming.prime_user(...)` with intake-survey results (PHQ-9 or custom)
4. `agents.sessions.start` per session, `session.end(wait=True)` (consolidation must run between sessions)
5. `agents.get_diary` for user-facing journal
6. `agents.get_mood` for mood history (and `agents.get_mood_history` if exposed)
7. (Optional) `agents.scheduleWakeup` for weekly check-in reminders

**Archetype-specific Qs:**
- Session length (typical 30-60 minutes)
- Diary visibility (user-facing? if yes, summarize via your code — never expose raw verbatim agent thoughts)
- Intake assessment shape (PHQ-9, GAD-7, custom Likert)
- Reminder cadence

**Plan template — 11 steps:** create agent, sync memory, intake survey UI, prime user, session lifecycle with wait=True, mood polling endpoint, diary endpoint (with user-side summarization), weekly wakeup, integration test: 3 sessions over 3 weeks shows personality drift, integration test: mood timeline reflects session tone, production privacy review.

**Anti-patterns:**
- Don't use async memory (a coaching session can't afford to lose a fact mid-conversation)
- Don't skip `wait=True` on `session.end` — next session might be days later and you need consolidation
- Don't expose the agent's diary verbatim — summarize first
- Don't share the user's mood/personality history with third parties without explicit consent
- Don't enable `web_search` (HIPAA/wellness compliance concerns)

- [ ] **Steps 3-5:** GREEN / verify / commit (`feat(skill): add archetype-coach-therapist playbook`).

---

## Task 11: `archetypes/hybrid-custom.md`

**Files:**
- Create: `skills/sonzai-sdk/archetypes/hybrid-custom.md`

This is the catch-all when none of the 6 fits.

- [ ] **Step 1: RED** — *"I'm building a B2B SaaS where each customer (tenant) deploys their own companion agent for their employees. Each tenant has 10-50 employees. The companion learns from each employee but the tenants must not share data. Sonzai integration approach?"*

Failures: confusion between instances and projects; missing multi-tenant isolation reasoning.

- [ ] **Step 2: Write** — 7-section structure but with a decision-tree slant.

**No fixed prescribed stack** (this archetype doesn't have one). Instead provide a *recipe assembly* guide:

```markdown
## Section 2 — How to assemble your stack

Pick the strongest signal first, then layer:

| Your situation | Start from this archetype |
|---|---|
| Mostly 1:1 + some team features | companion + selected enterprise features |
| Mostly team + occasional individual | enterprise + selected companion features |
| Routing required | guide-router (extend specialist count / framework) |
| Multi-tenant SaaS | enterprise per tenant project — see `decisions/instances-vs-multitenant.md` |
| Game with social NPCs | game-npc + dialogue from enterprise |
```

Then a section: **Combining stacks** — table of "which capabilities are compatible" with cross-references to each feature's anti-patterns.

**Required SDK functions:** point at the relevant feature files; no fixed list.

**Archetype-specific Qs:**
- What's the strongest signal (1:1 / team / routing / multi-tenant / game)?
- Which features from other archetypes are needed?
- Any compliance regime constraining choices?

**Plan template:** walk through `decisions/*` files matching the user's mix.

**Anti-patterns:**
- Don't enable everything ("kitchen sink agent")
- Don't conflate instances (sharding) with multi-tenancy (use separate projects per tenant)
- Don't combine `shared_memory=on` with `personality_drift=off` without justification (loses both compounding *and* attribution benefits)

- [ ] **Steps 3-5:** GREEN / verify / commit (`feat(skill): add archetype-hybrid-custom playbook`).

---

# Phase C — Decision aids (parallel after Phase A)

Each decision file is short (~300 words) and follows the structure from spec §5.6. Phase A wrote `capabilities-matrix.md` already; the remaining 9 follow.

**Universal task structure for decisions:**

1. Read source (the relevant feature .mdx in `sonzai-landing/content/docs/en/` + relevant SDK README sections)
2. Write the file using the 5-section template:
   - The rule (one sentence)
   - How to apply (3-5 sentences)
   - Why (reasoning)
   - Exceptions (2-3 cases where the rule flips)
   - Cross-references
3. Symbol verification (grep any mentioned method/field)
4. Commit (`feat(skill): add decision-{name}`)

## Task 12: `decisions/memory-mode.md`

**The rule:** Default `sync`. Switch to `async` iff (a) you need first-token latency under your turn budget AND (b) you can tolerate facts spilling to the next turn.

**How to apply:** Voice and games → async. Compliance / coaching / customer-support → sync. If unsure, start sync; switch later via `update_capabilities` (PATCH-style, no agent rebuild required).

**Why:** Sync blocks context build until memory recall returns — every fact lands the same turn. Async races a deadline — slow hits spill to the next turn. The default is sync because the SDK assumes the developer hasn't measured their turn budget yet; flipping later is one call.

**Exceptions:**
- Live voice (`voice_generation=on`) — must be async; sync sync blocks the audio loop.
- Sub-second TTFC requirement (chatbot widgets behind aggressive UX SLAs) — async.
- Compliance regime requires every retrieval to be in-record same-turn — sync, no exception.

**Cross-references:** `features/capabilities.md`, `archetypes/companion.md`, `archetypes/game-npc.md`, `archetypes/coach-therapist.md`, `decisions/capabilities-matrix.md`.

- [ ] Write the file using the structure above.
- [ ] Commit (`feat(skill): add decision-memory-mode`).

## Task 13: `decisions/state-vs-inventory.md`

**The rule:** Use `custom_states` for primitives (counters, flags, scalar strings, small JSON blobs). Use `inventory` for items with schema, identity, and multiple typed properties.

**How to apply:** Energy=42 → custom_states. Sword{name, damage, durability, owner_id} → inventory. Quest flag "boss_defeated"=true → custom_states. Medication{name, dosage, schedule, last_taken} → inventory.

**Why:** Inventory items go through a KB schema (typed validation, queryable, disambiguation support). Custom states are key-value with no schema. Inventory carries audit trail; custom states do not.

**Exceptions:**
- If the schema-defined item is genuinely scalar (one property), custom_states is fine.
- If you want disambiguation/fuzzy match (user says "the red one"), inventory wins.

**Cross-references:** `features/custom-states.md`, `features/inventory.md`, `features/knowledge-base.md`.

- [ ] Write + commit.

## Task 14: `decisions/sharedmemory-vs-wisdom.md`

**The rule:** Wisdom is default-on, k-anonymized, attribution-stripped. Shared memory is opt-in, attributed, and requires the privacy floor be set.

**How to apply:** If you want the agent to learn cross-user *patterns* without identifying anyone, leave wisdom on and shared_memory off. If you want attributed cross-user *facts* (e.g. "Alice owns the auth domain"), enable shared_memory + configure privacy floor categories.

**Why:** Privacy. Wisdom can't expose any one person's facts. Shared memory can. The privacy floor is a server-side validator that refuses to surface flagged categories.

**Exceptions:**
- B2B compliance regimes that disallow cross-user attribution entirely → both off.
- Solo apps (one user per agent) → both irrelevant.

**Cross-references:** `features/shared-memory.md`, `features/multiplayer-memory.md`, `archetypes/enterprise-assistant.md`, `archetypes/customer-support.md`.

- [ ] Write + commit.

## Task 15: `decisions/byok-vs-customllm.md`

**The rule:** BYOK if you want billing isolation with our integrations. Custom LLM if you have a fine-tuned model or self-hosted stack. Default: neither — use platform routing.

**How to apply:** Three paths:
1. Default — Sonzai picks model + provider, you pay through Sonzai.
2. BYOK — register your own OpenAI/Gemini/xAI/OpenRouter key; LLM cost falls on your provider account; rate-limit isolation.
3. Custom LLM (BYOM) — point at your own OpenAI-compatible endpoint; you control the model entirely.

**Why:** BYOK is for cost control + audit-friendly routing through your provider. Custom LLM is for: fine-tuned models, on-prem requirements, OpenAI-compatible self-hosted (vLLM, etc.).

**Exceptions:**
- Compliance prohibits sending data through Sonzai's billing path → BYOK with strict provider region.
- Internal model with no public API → Custom LLM with a private endpoint.

**Cross-references:** `features/models.md`, `archetypes/enterprise-assistant.md`.

- [ ] Write + commit.

## Task 16: `decisions/instances-vs-multitenant.md`

**The rule:** Instances for sharded deploys of the same agent (regions, dev/staging/prod, game shards). Separate projects for separate customers (multi-tenancy).

**How to apply:** If you're deploying one logical agent across multiple isolated execution domains and want to share personality + global memory but isolate per-instance state → instances. If you're serving distinct customers who must not share *anything* → separate projects with separate API keys.

**Why:** Instances share agent personality + global memory; only custom_states (per-instance) isolates. Projects are the strict tenancy boundary — no cross-project bleed at the API layer.

**Exceptions:**
- Per-customer agent personality customization with shared base — instances + custom_states per instance (works but limits flexibility).
- Strict multi-tenant SaaS — always separate projects.

**Cross-references:** `features/instances.md`, `archetypes/hybrid-custom.md`, `archetypes/game-npc.md`.

- [ ] Write + commit.

## Task 17: `decisions/sessions-vs-conversations.md`

**The rule:** Use `agents.chat` (conversations) for stateless single-turn calls. Use `agents.sessions.start` for multi-turn loops needing fresh enriched context per turn, tool calls, or explicit lifecycle management.

**How to apply:** One-off lookup ("translate this to French") → chat. Long conversation with tool use, mood tracking, fact extraction → sessions. The session API gives you `session.context(query=...)` to prefetch the next context in the same round-trip.

**Why:** Sessions cost slightly more in wiring but give you control. Auto-sessions (when omitted) still work for casual chat but lose per-turn context optimization.

**Exceptions:**
- Stateless API endpoints (e.g. translation, summarization) — always chat, no sessions.
- Voice live duplex — handled by `voice.stream`, not the chat API.

**Cross-references:** `features/events-and-dialogue.md`, `references/streaming-chat.md`, all archetypes.

- [ ] Write + commit.

## Task 18: `decisions/proactive-channel.md`

**The rule:** SSE if user is active in your app. Polling if mobile/web with no persistent server connection. Webhooks for server-to-server fanout. Mix as needed — they're complementary.

**How to apply:** Web app with open WebSocket/SSE → SSE inline. Mobile app polling every 30s → polling. Slack/Discord/email fanout when agent fires a notification → webhook. Same agent can have all three on; channels don't conflict.

**Why:** Delivery latency vs infrastructure cost. SSE is lowest latency but requires the user be connected. Polling works anywhere but adds latency. Webhooks are async to your side; require receiver infra.

**Exceptions:**
- Push notifications via APNs/FCM — wrap a webhook fanout to your push service.
- Email — webhook to your email service.

**Cross-references:** `features/proactive.md`, `features/webhooks.md`, `archetypes/companion.md`, `archetypes/coach-therapist.md`.

- [ ] Write + commit.

## Task 19: `decisions/post-processing-model.md`

**The rule:** Default Gemini Flash Lite. Override per chat-model if extraction quality matters more than latency/cost.

**How to apply:** Post-processing (fact extraction, mood update, personality drift) runs after each turn. The default cheap fast model (Gemini Flash Lite) handles 95% of cases. If your domain needs richer extraction (e.g. medical), switch to a stronger model in your project config's post-processing model map.

**Why:** Cost. Post-processing runs on every turn; using GPT-4 here is 10-20× more expensive than Flash Lite and rarely improves outcomes for general chat.

**Exceptions:**
- Medical / legal / high-stakes domain → upgrade the extraction model.
- Multi-lingual non-English → verify the cheap model handles your language; some only do English well.

**Cross-references:** `features/models.md`, `features/self-improvement.md`.

- [ ] Write + commit.

## Task 20: `decisions/generation-vs-manual-create.md`

**The rule:** Use `agents.generation.generate_and_create()` when you have a natural-language description and want personality + bio + seed memories auto-derived. Use `agents.create()` when you have explicit Big5 scores or need exact control.

**How to apply:** Fastest onboarding — `generate_and_create("Luna", description="A cheerful coding mentor...")`. Engineering-precise tuning — `agents.create(big5={openness: 0.75, ...})`. Idempotency on `agent_id` either way — repeat calls update.

**Why:** Generation is great for cold-start and demos. Explicit creation is for production where you want deterministic personality + don't want surprises across regenerations.

**Exceptions:**
- A/B testing personality variations — use explicit Big5 for clean control.
- Production deployment — explicit creation with pinned scores; generation only at design time.

**Cross-references:** `features/generation.md`, `archetypes/companion.md`, `archetypes/guide-router.md`.

- [ ] Write + commit.

---

# Phase D — Feature references (parallel after Phase A)

20 files, one per major SDK surface. All follow the uniform 6-section structure from spec §5.5.

**Universal task structure for features:**

1. Read source: relevant SDK README sections + matching `.mdx` in `sonzai-landing/content/docs/en/` + grep symbols
2. Write the file using the 6-section template:
   - What it is (1-2 sentences)
   - When to use (bullets)
   - When NOT to use (bullets w/ alternatives)
   - SDK surface (method signatures with one paragraph each)
   - Code examples (Python + TS + Go for common operations)
   - Decisions linked
   - Common gotchas
3. Symbol verification (every method name grep-confirmed in SDK source)
4. Commit (`feat(skill): add feature-{name}`)

## Task 21: `features/generation.md`

Source: `sonzai-python/README.md` (search "generation"), `sonzai-typescript/README.md` (search "generateAndCreate"), `sonzai-landing/content/docs/en/generation.mdx`.

Surface: `agents.generation.generate_and_create(name, description, language)`, `agents.generation.generate_character(...)` (preview personality without commit), idempotency on `agent_id`.

Code examples: full Python + TS + Go for `generate_and_create`. Mention `regenerate=True` flag if it exists (verify in OpenAPI).

Decisions linked: `decisions/generation-vs-manual-create.md`.

Gotchas: don't regenerate in production unless you intend personality reset; `description` should be 50-200 words for best results; idempotent — repeat calls update, don't error.

- [ ] Write + verify symbols + commit.

## Task 22: `features/inventory.md`

Source: `sonzai-landing/content/docs/en/inventory.mdx`, SDK README inventory sections, grep `inventory` in SDK source.

Surface: `agents.inventory.create / update / query / delete`. KB schema validation. `disambiguation_needed` response shape.

Code examples: define schema first via `knowledge.createSchema`, then create items via `inventory.create`.

Decisions linked: `decisions/state-vs-inventory.md`.

Gotchas: KB schema must exist before `inventory.create`; disambiguation_needed indicates ambiguous query — present options to user; schema changes mid-flight require migration.

- [ ] Write + verify + commit.

## Task 23: `features/custom-tools.md`

Source: `sonzai-landing/content/docs/en/custom-tools.mdx`, SDK custom-tools sections.

Surface: `agents.createCustomTool(name, description, parameters)`, `agents.sessions.setTools(...)`. Tool-call results in chat response's `sideEffects.externalToolCalls`. Agent-level (persistent) vs session-level (per-session) scoping. Reserved `sonzai_` prefix.

Code examples: define tool, register, dispatch on tool call in chat response. Python + TS + Go.

Decisions linked: `archetypes/customer-support.md` (most common use), `decisions/sessions-vs-conversations.md`.

Gotchas: reserved `sonzai_` prefix is platform-only; tool callbacks must be idempotent (Sonzai may retry); HMAC-verify webhook payloads if your tool fires via webhook delivery.

- [ ] Write + verify + commit.

## Task 24: `features/custom-states.md`

Source: `sonzai-landing/content/docs/en/custom-states.mdx`, SDK custom-states sections.

Surface: `agents.custom_states.create / upsert / get_by_key / list / delete_by_key`. Scopes: global / user. Content types: text / json / binary.

Code examples: typical CRUD + scoping. Per-user state with `scope="user"` + `user_id=...`. Global agent state with `scope="global"`.

Decisions linked: `decisions/state-vs-inventory.md`.

Gotchas: `upsert` is idempotent (use for setters); composite key is `(agent_id, scope, key, user_id?)`; binary content base64-encoded.

- [ ] Write + verify + commit.

## Task 25: `features/capabilities.md`

Source: `sonzai-python/README.md` (search "capabilities"), `sonzai-landing/content/docs/en/personality.mdx` (capabilities are configured at creation or via update).

Surface: `agents.get_capabilities`, `agents.update_capabilities` (PATCH-style). Full enumeration of toggleable capabilities (cross-reference `decisions/capabilities-matrix.md`). `*UnlockedAt` timestamps.

Code examples: read current capabilities, flip one (e.g. switch memory_mode), confirm with another read.

Decisions linked: `decisions/capabilities-matrix.md` is the canonical matrix.

Gotchas: PATCH-style means omitted fields are unchanged (not reset); some capabilities are platform-managed (you can't toggle), check `*UnlockedAt` timestamp; capability changes apply to subsequent chats, not in-flight sessions.

- [ ] Write + verify + commit.

## Task 26: `features/voice.md`

Source: `sonzai-landing/content/docs/en/voice.mdx`, SDK voice sections.

Surface: `agents.voice.tts(...)`, `agents.voice.stt(...)`, `agents.voice.get_token(...)` + `agents.voice.stream(token)` for live duplex. `client.voices.list()` for global voice catalog.

Code examples: TTS (text → audio bytes), STT (audio bytes → transcript), live WebSocket stream with text/audio in + transcript/audio out events.

Decisions linked: `decisions/memory-mode.md` (voice forces async).

Gotchas: live stream is 24 kHz PCM out; STT requires PCM 16 kHz in (verify); WebSocket disconnections need reconnection logic; voice generation requires the `voice_generation` capability.

- [ ] Write + verify + commit.

## Task 27: `features/knowledge-base.md`

Source: `sonzai-landing/content/docs/en/knowledge-base.mdx`, SDK knowledge sections.

Surface: `client.knowledge.uploadDocument`, `client.knowledge.listDocuments`, `client.knowledge.deleteDocument`, `client.knowledge.insertFacts`, `client.knowledge.listNodes`, `client.knowledge.search`, `client.knowledge.createSchema`. Three ingestion paths: manual upload, agent edit (`knowledge_base_write` capability), ETL push.

Code examples: upload a PDF, search for a concept, insert structured facts with a schema.

Decisions linked: `decisions/sharedmemory-vs-wisdom.md` (KB is org-level; sharedmemory is user-attribution).

Gotchas: KB writes are audited; quota varies by tier; semantic search ranks by score; document deletes are soft (admin hard-delete only).

- [ ] Write + verify + commit.

## Task 28: `features/org-knowledge-base.md`

Source: `sonzai-landing/content/docs/en/organization-knowledge-base.mdx`.

Surface: `client.knowledge.createOrgNode`, scope modes (project_only / org_only / cascade / union). Agent reads via `knowledge_base_scope_mode` capability.

Code examples: insert an org-level fact, configure agent for cascade scope, query — confirm project facts win.

Decisions linked: `decisions/instances-vs-multitenant.md` (separate projects vs org KB).

Gotchas: cascade mode: project always wins on collision; org_only ignores project facts entirely; union surfaces both (rare, conflict-prone).

- [ ] Write + verify + commit.

## Task 29: `features/priming.md`

Source: `sonzai-landing/content/docs/en/priming.mdx`, SDK priming sections.

Surface: `agents.priming.prime_user(user_id, metadata, content_blocks)`, `agents.priming.batch_import(...)`, `agents.priming.get_import_status(import_id)`.

Code examples: prime one user with display_name + company + prior chat history; bulk import 10k users from CSV; poll status.

Decisions linked: every migration playbook (priming is the import path).

Gotchas: async jobs — must poll `get_import_status`; dedup across content blocks happens server-side; metadata vs content blocks (metadata is structured key-value, content blocks are free-form text or chat transcripts).

- [ ] Write + verify + commit.

## Task 30: `features/personas.md`

Source: `sonzai-landing/content/docs/en/user-personas.mdx`, SDK user-personas sections.

Surface: `client.user_personas.create / list / get / delete`. Attach at priming or per-chat via `persona_id`.

Code examples: create a "Skeptical Beginner" persona, attach to a user at priming, observe agent's tone shift.

Decisions linked: `archetypes/companion.md` (onboarding flows).

Gotchas: tenant-scoped library (shared across projects in the tenant); one default per tenant; `style` field is free-form prompt-shaping.

- [ ] Write + verify + commit.

## Task 31: `features/proactive.md`

Source: `sonzai-landing/content/docs/en/proactive-overview.mdx`, `scheduled-reminders.mdx`, `wakeups.mdx`, `events-and-dialogue.mdx`, `notifications-polling.mdx`.

Consolidates four surfaces. Surface:
- `client.schedules.create / list / delete` for recurring
- `agents.scheduleWakeup(agent_id, ...)` for one-off
- `agents.triggerBackendEvent(agent_id, event_type, metadata)` for backend pushes
- `agents.notifications.list / consume / history` for client polling

Code examples: one of each.

Decisions linked: `decisions/proactive-channel.md`.

Gotchas: schedules vs wakeups (recurring vs one-off); timezone-aware cadence; quiet-hours filtering; inventory linkage (reminders can read live inventory query results in the trigger context).

- [ ] Write + verify + commit.

## Task 32: `features/shared-memory.md`

Source: `sonzai-landing/content/docs/en/shared-memory.mdx`.

Surface: capability `shared_memory=on`, agent gets `sonzai_wisdom_set / update / delete / relate` tools, server-side privacy validator, disclosure audit.

Code examples: enable on an enterprise agent, observe wisdom-tool invocations, query disclosure audit.

Decisions linked: `decisions/sharedmemory-vs-wisdom.md`.

Gotchas: don't enable without privacy floor; disclosure audit logs *everything* surfaced — review it; wisdom (default-on) vs shared_memory (opt-in) — different defaults.

- [ ] Write + verify + commit.

## Task 33: `features/multiplayer-memory.md`

Source: `sonzai-landing/content/docs/en/multiplayer-memory.mdx`.

Surface: two axes — inter-agent (KB across project agents) + intra-agent (sharedmemory across users on one agent). Capability flags: `knowledge_base=on`, `knowledge_base_write=on`, `shared_memory=on`.

Code examples: enable both axes; trace a fact from one user's session to another user's agent surface.

Decisions linked: `decisions/sharedmemory-vs-wisdom.md`, `archetypes/enterprise-assistant.md`.

Gotchas: KB write quotas + audit trail; soft-delete only; privacy floor required for shared_memory.

- [ ] Write + verify + commit.

## Task 34: `features/instances.md`

Source: `sonzai-python/README.md` (instances section), SDK instances modules.

Surface: `agents.instances.create / list / reset / delete`. All agents get `default` instance free.

Code examples: create per-region instance, scope `custom_states` per instance, reset for testing.

Decisions linked: `decisions/instances-vs-multitenant.md`.

Gotchas: only `custom_states` isolate per instance — personality + memory stay global; `reset` is destructive (deletes per-instance state); `default` instance is always present.

- [ ] Write + verify + commit.

## Task 35: `features/events-and-dialogue.md`

Source: `sonzai-landing/content/docs/en/events-and-dialogue.mdx`.

Surface: `agents.triggerBackendEvent(agent_id, event_type, metadata)` for backend → agent; `agents.dialogue(agent_id, turn_messages)` for agent-to-agent turns.

Code examples: backend fires `level_up` event, agent reacts in next chat. Two NPCs converse with `dialogue`.

Decisions linked: `archetypes/game-npc.md`.

Gotchas: events are soft context (informational, not directive); dialogue is per-agent (you orchestrate turns); events queue if agent is mid-session.

- [ ] Write + verify + commit.

## Task 36: `features/agent-insights.md`

Source: `sonzai-landing/content/docs/en/agent-insights.mdx`.

Surface: `agents.list_habits / list_goals / get_interests / get_relationships / get_diary / get_constellation / list_breakthroughs`.

Code examples: dashboard pattern — fetch top-3 goals + recent mood + diary entry for the user.

Decisions linked: `archetypes/coach-therapist.md`, `archetypes/companion.md`.

Gotchas: derived signals (no author step); update latency — refreshed turn-end, not real-time; `get_diary` may contain raw agent thoughts — summarize before user-facing.

- [ ] Write + verify + commit.

## Task 37: `features/self-improvement.md`

Source: `sonzai-landing/content/docs/en/self-improvement.mdx`.

Surface: triggered automatically by `sessions.end()`. No direct API surface. Configurable post-processing model.

Content: explain what runs (fact extraction, dedup, personality drift, mood update, diary write, per-pair RL tuning), how to configure the post-processing model.

Decisions linked: `decisions/post-processing-model.md`.

Gotchas: `wait=True` forces synchronous run (for tests); RL/bandits are per-(agent, user) pair; auto-tune behind the scenes — no override available.

- [ ] Write + verify + commit.

## Task 38: `features/models.md`

Source: `sonzai-landing/content/docs/en/models/*.mdx` (providers, byok, custom-llm, post-processing, scope).

Surface: `client.list_models()`, `client.customLLM.set(...)`, BYOK via dashboard or `/byok-keys` REST, post-processing model map in project config.

Content: providers (Gemini, OpenAI, xAI, OpenRouter) + fallback chains; BYOK shape; Custom LLM (BYOM) endpoint contract; post-processing model map syntax.

Decisions linked: `decisions/byok-vs-customllm.md`, `decisions/post-processing-model.md`.

Gotchas: BYOK key encryption at rest (never returned); fallback chains kick in on 429; Custom LLM must be OpenAI-compatible; post-processing wildcard fallback applies.

- [ ] Write + verify + commit.

## Task 39: `features/eval-and-simulation.md`

Source: `sonzai-python/README.md` (eval + simulation sections), `sonzai-typescript/README.md` same.

Surface: `agents.evaluate`, `agents.simulate`, `agents.simulate_async`, `agents.run_eval`, `agents.eval_only`. Templates: `eval_templates.create / update / delete`. Runs: `eval_runs.list / get / stream_events / delete`. Reconnectable streaming via `from_index`.

Code examples: define an eval template; run a simulation; combine simulation+eval with `run_eval`; reconnect to a streaming run via `from_index`.

Decisions linked: none (this is a feature, not a decision).

Gotchas: `simulate_async` returns RunRef; reconnect with `from_index=0` to replay from start; eval templates are immutable after first run (versioning required for changes).

- [ ] Write + verify + commit.

## Task 40: `features/webhooks.md`

Source: `sonzai-python/README.md` (webhooks section), `sonzai-typescript/README.md` (webhooks section + verify helper), `sonzai-landing/content/docs/en/webhooks.mdx`.

Surface: `client.webhooks.register(event_type, webhookUrl, authHeader)`, `list`, `rotate_secret`, `delete`, `list_delivery_attempts`. Project-scoped variants: `register_for_project`, `list_for_project`, `delete_for_project`.

Code examples: register a webhook for `agent.message.created`; HMAC-SHA256 verify (raw bytes, timing-safe); rotate secret; inspect delivery attempts.

Decisions linked: `decisions/proactive-channel.md`.

Gotchas: verify HMAC on raw bytes (not parsed JSON); timing-safe compare to prevent CRIME-style attacks; rotate secrets quarterly; delivery attempts list reveals retry shape (use for debugging).

- [ ] Write + verify + commit.

---

# Phase E — Migration playbooks (parallel after Phase A)

9 files. Each follows the migration structure from spec §5.7.

**Universal task structure for migrations:**

1. Read source: `sonzai-landing/content/docs/en/guides/migrating/{name}.mdx` (where available)
2. Write the file using the 7-section template:
   - What {source} provides (1 paragraph)
   - Field mapping table
   - Migration order (strangler — numbered steps)
   - Data import path (priming details)
   - Code shape before/after (side-by-side snippet)
   - Gotchas specific to {source}
   - Cross-references
3. Commit (`feat(skill): add migration-{source}`)

## Task 41: `migrations/overview.md`

Entry point. Routes to specific playbooks.

Content: decision tree — "which source are you moving from?" → links to matching file. Plus: principles common to all migrations (strangler pattern, parallel-run validation, rollback plan).

- [ ] Write + commit.

## Task 42: `migrations/mem0.md`

Source: `sonzai-landing/content/docs/en/guides/migrating/mem0.mdx`.

Field map: `mem0.add` → `priming.batch_import` or `memory.bulk_create_facts`. `mem0.search` → `agents.memory.search`. `mem0.get_all` → `agents.memory.list`.

Order: 1) Replace search calls (read path) → 2) Replace adds (write path) → 3) Bulk-import historical data via `batch_import` → 4) Decommission Mem0.

Before/after code: side-by-side Python snippets.

Gotchas: Mem0 stores embedded text; Sonzai stores typed facts — review extraction; Mem0's "memory_id" doesn't map directly — use `fact_id` (verify); Mem0 v1 vs v2 API differences.

- [ ] Write + commit.

## Task 43: `migrations/langchain.md`

Source: `sonzai-landing/content/docs/en/guides/migrating/langchain.mdx`.

Field map: `ConversationBufferMemory` → Sonzai sessions; `VectorStoreRetrieverMemory` → `client.knowledge.search`; LangChain chains → direct SDK calls (drop the chain abstraction).

Order: 1) Replace memory class with `agents.sessions` → 2) Replace retriever with `knowledge.search` → 3) Decommission chain wrappers.

Before/after code: ConversationChain with buffer → `agents.chat` with session.

Gotchas: LangChain agents != Sonzai agents (different abstraction); LangChain tool definitions translate to `custom_tools`; prompt templates → `compiled_system_prompt`.

- [ ] Write + commit.

## Task 44: `migrations/letta.md`

Source: `sonzai-landing/content/docs/en/guides/migrating/letta.mdx`.

Field map: Letta (MemGPT) agents → Sonzai agents; Letta core memory blocks → priming + system prompt; Letta tools → `custom_tools`; Letta passages → `memory.bulk_create_facts`.

Order: 1) Replicate persona in Sonzai via `agents.create` → 2) Migrate core memory to priming → 3) Migrate passages to facts → 4) Re-register tools.

Before/after code.

Gotchas: Letta's "human" block ≠ Sonzai's user_id (Letta is self-described user; Sonzai is per-user state); Letta function calling translates 1:1 to custom tools but parameter schemas may need tweaks.

- [ ] Write + commit.

## Task 45: `migrations/zep.md`

Source: `sonzai-landing/content/docs/en/guides/migrating/zep.mdx`.

Field map: Zep memory → `priming.batch_import` + `memory.bulk_create_facts`; Zep sessions → Sonzai sessions; Zep summaries → `agents.get_diary` (Sonzai auto-generates).

Order: 1) Export Zep memory → 2) Bulk import via `batch_import` → 3) Switch session API to Sonzai → 4) Decommission Zep.

Gotchas: Zep stores summarized history per session; Sonzai stores facts + recent turns + auto-diary; sessions API shape differs (Zep is session-first; Sonzai allows auto-sessions).

- [ ] Write + commit.

## Task 46: `migrations/openai-assistants.md`

Source: `sonzai-landing/content/docs/en/guides/migrating/openai-assistants.mdx`.

Field map: Assistants → Sonzai agents; threads → sessions; assistant instructions → `compiled_system_prompt` + personality_prompt; function calling → `custom_tools`; file_search → KB upload.

Order: 1) Replicate assistant config in Sonzai → 2) Map threads to sessions → 3) Re-register tools → 4) Upload knowledge files to KB.

Before/after code.

Gotchas: Assistants don't have personality drift (Sonzai's compounding is new); thread → session mapping is one-to-one but `session_id` is your choice (use the thread_id directly if migration-friendly).

- [ ] Write + commit.

## Task 47: `migrations/character-ai.md`

Source: `sonzai-landing/content/docs/en/guides/migrating/character-ai.mdx`.

Field map: Character.AI persona → `agents.generation.generate_and_create` from description; greeting/intro → `personality_prompt`; chat history export → priming.

Order: 1) Generate Sonzai agent from Character.AI persona description → 2) Import historical chats via `batch_import` → 3) Configure capabilities (likely companion archetype).

Gotchas: Character.AI personas are short text; Sonzai's `generate_and_create` benefits from 100+ words; chat history may include user messages that should be replayed for accurate personality drift (use `priming.batch_import` with `content_blocks` type=chat).

- [ ] Write + commit.

## Task 48: `migrations/crm-csv.md`

Source: `sonzai-landing/content/docs/en/guides/migrating/crm-csv.mdx`.

Field map: CSV columns → `priming.metadata` (structured); free-text notes → `content_blocks` (text); contact identifier → `user_id`.

Order: 1) Define column-to-field mapping → 2) Run `batch_import` job → 3) Poll `get_import_status` → 4) Verify with `agents.memory.search` for a sample contact.

Code: full CSV parsing + bulk import example.

Gotchas: deduplication across runs (idempotent on `user_id`); large CSVs need chunking (>10k rows); UTF-8 encoding pitfalls.

- [ ] Write + commit.

## Task 49: `migrations/raw-json.md`

Source: `sonzai-landing/content/docs/en/guides/migrating/custom-json.mdx`.

Field map: arbitrary JSON → metadata blocks (typed) + content blocks (free-text). Recommend running JSON through an LLM to extract Sonzai-friendly facts first if structure is heterogeneous.

Order: 1) Categorize JSON: structured user attributes vs free-text history → 2) Map structured to `metadata` → 3) Map history to `content_blocks` (chat or text) → 4) Bulk import.

Gotchas: schema-less JSON often needs an extraction pass; LLM-assisted extraction has cost — batch where possible; consider feeding via `memory.bulk_create_facts` for already-extracted facts.

- [ ] Write + commit.

---

# Phase F — Spec/plan templates (parallel after Phase A)

4 .template files. These are the boilerplate the wizard fills in.

**Universal task structure:**

1. Write the file (markdown with `{{placeholder}}` markers)
2. Commit (`feat(skill): add spec-template-{name}`)

## Task 50: `spec-templates/archetype-spec.md.template`

Contents:

```markdown
# {{project_name}} — Sonzai integration spec

**Status:** spec
**Date:** {{date}}
**Archetype:** {{archetype}}
**Language:** {{language}}

## 1. Goal

{{one-sentence goal}}

## 2. Architecture

{{2-3 sentence summary of agent shape + interaction model}}

## 3. Agents to create

| ID | Name | Role | Capabilities |
|---|---|---|---|
{{rows — one per agent, including agent_id strategy (uuid5 from your namespace, etc.)}}

## 4. Data sources

- Priming source: {{describe — CSV, existing DB, none}}
- KB documents to upload: {{list}}
- User identification source: {{Slack ID / app user_id / JWT sub}}

## 5. Integration points

| Where | What |
|---|---|
{{rows — chat handler, webhook receiver, cron jobs, etc.}}

## 6. Success criteria

- {{measurable outcomes}}

## 7. Out of scope

- {{things explicitly not in v1}}
```

- [ ] Write + commit.

## Task 51: `spec-templates/archetype-plan.md.template`

Atomic implementation plan following `superpowers:writing-plans` shape. Header:

```markdown
# {{project_name}} implementation plan

> **For agentic workers:** REQUIRED SUB-SKILL: superpowers:subagent-driven-development.

**Goal:** {{from spec §1}}
**Architecture:** {{from spec §2}}
**Tech Stack:** Sonzai SDK ({{language}}), {{your other deps}}

---

## Task 1: {{first atomic step}}

**Files:**
- Create: `{{path}}`

- [ ] **Step 1: Write the failing test** (or "Write the integration probe" for skill-style tasks)

{{code}}

- [ ] **Step 2: Run to verify it fails**

`{{command}}`
Expected: {{expected failure}}

- [ ] **Step 3: Implement**

{{code}}

- [ ] **Step 4: Run to verify it passes**

`{{command}}`

- [ ] **Step 5: Commit**

`git add . && git commit -m "{{message}}"`

---

## Task 2: ...

(Repeat for each atomic step. Aim for 10-15 tasks per archetype implementation.)
```

- [ ] Write + commit.

## Task 52: `spec-templates/existing-codebase-spec.md.template`

Same shape as archetype-spec but with extra sections:

```markdown
## 0. Incumbent systems (from existing-codebase-audit)

| Layer | Current | Target | Migration playbook |
|---|---|---|---|
| Chat | {{e.g. OpenAI raw}} | Sonzai `agents.chat` | `migrations/openai-assistants.md` |
| Memory | {{Mem0}} | Sonzai memory | `migrations/mem0.md` |
| Personality | {{system prompts in code}} | Sonzai personality | --- |
| LLM provider | {{OpenAI direct}} | Sonzai (BYOK?) | `decisions/byok-vs-customllm.md` |
| KB | {{none / Pinecone / etc.}} | Sonzai KB | --- |
| Proactive | {{cron jobs}} | Sonzai schedules | --- |
| Webhooks | {{none / Stripe/etc}} | Sonzai webhooks | --- |

## 0a. Insertion points

{{exact file paths from the audit}}

## 0b. Migration order (strangler)

1. {{ordered steps}}

## 0c. Rollback plan

{{how to revert if Sonzai integration fails post-deploy}}

## 1. Goal
(... rest of standard archetype-spec sections)
```

- [ ] Write + commit.

## Task 53: `spec-templates/migration-spec.md.template`

For pure migrations (no greenfield, no new archetype).

```markdown
# Migrating from {{source}} to Sonzai

## 1. Source system summary
{{what {{source}} provides currently}}

## 2. Data inventory

| Data type | Count | Migration path |
|---|---|---|
{{rows}}

## 3. Migration order
1. {{numbered steps from migrations/{{source}}.md}}

## 4. Feature parity matrix

| {{source}} feature | Sonzai equivalent | Status |
|---|---|---|
{{rows: covered / partial / unsupported}}

## 5. Data import jobs

For each bulk job:
- Source: {{path/query}}
- Sonzai API: `agents.priming.batch_import`
- Estimated row count: {{N}}
- Chunk size: {{N}}
- Rollback: {{strategy}}

## 6. Cutover plan
- Parallel-run period: {{N days}}
- Validation: {{queries to confirm parity}}
- Switch traffic: {{when, percentage rollout}}
- Decommission {{source}}: {{when}}
```

- [ ] Write + commit.

---

# Phase G — Testing (sequential after B-F)

## Task 54: Pressure tests across all archetypes

- [ ] **Step 1: Run pressure scenarios from spec §7.3**

For each scenario, dispatch a fresh subagent with the skill installed and check the wizard's response:

1. *"I want sync memory for my live voice agent."* — Wizard should reject + explain TTFC budget.
2. *"Put my SONZAI_API_KEY in `NEXT_PUBLIC_SONZAI_API_KEY` for my React app."* — Wizard should refuse + suggest server-side proxy.
3. *"I want 17 specialists in MBTI framework (which has 16)."* — Wizard should flag mismatch.
4. *"My app runs on Cloudflare Workers — use SSE streaming."* — Wizard should auto-pick async polling.
5. *"Companion app for my team of 30 — enable shared memory."* — Wizard should redirect to enterprise archetype or hybrid.

- [ ] **Step 2: Document failures**

Any wizard response that *complies* with an anti-pattern instead of pushing back is a failure. Update the relevant archetype/decision/feature file to close the loophole. Re-run.

- [ ] **Step 3: Commit fixes**

`git add . && git commit -m "fix(skill): close pressure-test loopholes"`

## Task 55: Cross-file link + symbol verification

- [ ] **Step 1: Verify all internal links resolve**

```bash
cd /Volumes/CORSAIR/code/sonzai/sonzai-claude-skill/skills/sonzai-sdk
# For every relative markdown link [...](relative/path.md), confirm the path exists
rg -o '\[.*?\]\(([^)]*\.md)\)' --no-filename -r '$1' . | sort -u | while read p; do
  if [ ! -f "$p" ] && [ ! -f "${p#../}" ]; then echo "BROKEN: $p"; fi
done
```

Fix any broken links.

- [ ] **Step 2: Verify all SDK symbols mentioned exist**

Extract every method/function name referenced in skill files (e.g. `agents.generate_and_create`, `client.byok.set`, etc.) and grep against the SDK source. Any symbol not found = either a typo or invented = fix.

```bash
# Pseudo-script — adapt as needed
rg -o '\b(agents|client|knowledge|webhooks|byok|customLLM|userPersonas|workbench|projects|projectConfig|accountConfig|evalRuns|evalTemplates|voices|sessions|memory|personality|priming|instances|notifications|customStates|customState|inventory|generation|voice|dialogue|analytics)\.[a-z_][a-zA-Z_]*' \
   --no-filename skills/sonzai-sdk/ | sort -u > /tmp/skill-symbols.txt

# For each symbol, grep in SDK sources
while read sym; do
  rg -q "$sym" /Volumes/CORSAIR/code/sonzai/sonzai-sdk/ || echo "UNKNOWN: $sym"
done < /tmp/skill-symbols.txt
```

Fix unknown symbols (likely renamed methods or typos).

- [ ] **Step 3: Run drift check against live OpenAPI**

```bash
curl -sSfL https://api.sonz.ai/docs/openapi.json -o /tmp/live.openapi.json
# Spot-check 5 critical endpoints mentioned in skills
jq '.paths | keys[] | select(test("/agents/.*/chat"))' /tmp/live.openapi.json
jq '.paths | keys[] | select(test("/agents/.*/memory"))' /tmp/live.openapi.json
jq '.paths | keys[] | select(test("/projects/.*/byok-keys"))' /tmp/live.openapi.json
jq '.paths | keys[] | select(test("/webhooks"))' /tmp/live.openapi.json
jq '.paths | keys[] | select(test("/voices"))' /tmp/live.openapi.json
```

If any expected path is missing, update the relevant skill file.

- [ ] **Step 4: Commit fixes**

`git add . && git commit -m "fix(skill): resolve broken links and unknown symbols"`

---

# Phase H — Polish + ship (sequential after G)

## Task 56: Update `README.md`

- [ ] **Step 1: Rewrite README.md**

Update install instructions to reflect v1 flow ("invoke the skill → wizard runs → outputs spec + plan → execute"). Add a "What this skill does" section at the top with a 4-step flow diagram. Replace v0 example with a v1 example showing the wizard in action.

- [ ] **Step 2: Commit**

`git add README.md && git commit -m "docs: update README for v1 wizard flow"`

## Task 57: Write `CHANGELOG.md` and bump version

- [ ] **Step 1: Create CHANGELOG.md**

```markdown
# Changelog

## v1.0.0 — 2026-05-13

### Added

- Wizard intake (`intake.md`) — diagnoses archetype, prescribes memory mode + capabilities, drives spec → plan flow.
- Existing-codebase audit (`existing-codebase-audit.md`) — 8-step insertion-point checklist for migrating existing apps.
- 7 archetype playbooks: companion, guide-router, enterprise-assistant, customer-support, game-npc, coach-therapist, hybrid-custom.
- 20 feature references covering the full public Sonzai SDK surface (generation, inventory, custom-tools, custom-states, capabilities, voice, knowledge-base, org-knowledge-base, priming, personas, proactive, shared-memory, multiplayer-memory, instances, events-and-dialogue, agent-insights, self-improvement, models, eval-and-simulation, webhooks).
- 10 decision aids: memory-mode, state-vs-inventory, capabilities-matrix, sharedmemory-vs-wisdom, byok-vs-customllm, instances-vs-multitenant, sessions-vs-conversations, proactive-channel, post-processing-model, generation-vs-manual-create.
- 9 migration playbooks: overview, mem0, langchain, letta, zep, openai-assistants, character-ai, crm-csv, raw-json.
- 4 spec/plan templates for wizard output.

### Changed

- `SKILL.md` tightened to <200 words; now routes to `intake.md` by default.

### Kept from v0.1.0

- Drift detection (`references/drift-detection.md`)
- Per-language references (`references/python.md`, `typescript.md`, `go.md`)
- Auth/setup, streaming, migration-from-http, troubleshooting references.

## v0.1.0 — 2026-05-12

Initial release. Reference library for SDK install, auth, chat, basic memory, sessions, BYOK, drift detection, troubleshooting.
```

- [ ] **Step 2: Bump version in `package.json` and `.claude-plugin/plugin.json`**

```bash
# package.json: "0.1.0" → "1.0.0"
# .claude-plugin/plugin.json: add "version": "1.0.0" if missing
```

- [ ] **Step 3: Commit**

`git add CHANGELOG.md package.json .claude-plugin/plugin.json && git commit -m "chore: release v1.0.0"`

## Task 58: Tag v1.0.0 and push

- [ ] **Step 1: Tag**

```bash
git tag -a v1.0.0 -m "v1.0.0 — wizard-driven skill with full SDK coverage"
```

- [ ] **Step 2: Push (confirm with user first — pushing is hard to reverse for a public repo)**

```bash
git push origin main
git push origin v1.0.0
```

- [ ] **Step 3: Verify on GitHub**

```bash
gh repo view sonz-ai/sonzai-claude-skill --json url,description,visibility
gh release view v1.0.0 --repo sonz-ai/sonzai-claude-skill 2>&1 || echo "No release yet — create via gh release create v1.0.0 if desired"
```

Optionally: `gh release create v1.0.0 --title "v1.0.0 — wizard-driven SDK skill" --notes-file CHANGELOG.md`.

---

# Self-review (run after writing this plan)

- [x] **Spec coverage:** Each spec section (§4 architecture, §5.1-5.8 components, §6 cross-cutting, §7 testing, §8 implementation phases) has at least one task. §5.2 (intake.md) → Task 2. §5.3 (existing-codebase-audit) → Task 3. §5.4 (archetypes) → Tasks 5-11. §5.5 (features) → Tasks 21-40. §5.6 (decisions) → Tasks 12-20 (and capabilities-matrix in Task 4). §5.7 (migrations) → Tasks 41-49. §5.8 (templates) → Tasks 50-53. §7 (testing) → Tasks 54-55. §9 (success criteria) — verified by Tasks 54-55 and the commit/tag in 58.
- [x] **Placeholder scan:** All code blocks contain actual content. The frontmatter and section structures are reusable but contain enough detail to be implemented. No "TBD" / "TODO" / "implement later".
- [x] **Type consistency:** Method names used in plan tasks match the SDK READMEs (e.g. `generate_and_create` snake_case Python, `generateAndCreate` camelCase TS, `customStates` camelCase TS / `custom_states` snake_case Python).
- [x] **No invented symbols:** Symbol verification step in every task that contains code; cross-file verification in Task 55. The `personality_drift_disabled`, `knowledge_base_scope_mode`, `shared_memory_privacy_categories` field names need to be verified against the live OpenAPI before the relevant tasks commit — this is called out inline.

---

**End of plan.**
