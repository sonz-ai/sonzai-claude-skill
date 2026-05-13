# Wizard-answer derivation

Map `.full-auto/signals.md` → exact answers to all 7 wizard questions in `../sonzai-sdk/intake.md` + archetype-specific follow-ups + a concrete acceptance checklist Phase 5 will test against.

Output: `.full-auto/wizard-answers.md` — this is the builder subagent's single source of truth.

## Determinism principle

For every signal that's silent: pick the **lowest-risk default** and **add a line to the Documented Assumptions section**. Never improvise capabilities, scale, or compliance posture from thin air. The operator must be able to read `wizard-answers.md` and understand exactly why each decision was made.

## The 7 answers — rules

### Q1 — Project state
- Operator passed `--target` flag pointing to an existing directory with code → `existing`
- Signals contain "we already have a codebase / add to our existing repo / our app is at X" → `existing` (record path if mentioned)
- Otherwise → `greenfield`

### Q2 — Archetype
Take the strongest archetype signal from `signals.md`.

Tie-break order (when transcript has multiple weak signals):
1. `companion`
2. `coach-therapist`
3. `customer-support`
4. `enterprise-assistant`
5. `game-npc`
6. `guide-router`
7. `hybrid-custom`

This order favors archetypes with more deterministic SDK wiring (companion is the most prescribed; hybrid-custom requires most decisions).

### Q3 — User identification model
Default per archetype, override if transcript specifies:

| Archetype | Default Q3 |
|---|---|
| companion | `stable-user-id` |
| coach-therapist | `stable-user-id` |
| enterprise-assistant | `team-shared` (multi-user-per-agent default) |
| customer-support | `stable-user-id` (ticket owner) |
| game-npc | `stable-user-id` (player_id) |
| guide-router | `anonymous-at-intake` (typical funnel) |
| hybrid-custom | `stable-user-id` (safest) |

Override examples:
- Transcript says "user signs up before talking" → `stable-user-id`
- Transcript says "users land cold from a campaign" → `anonymous-at-intake`

### Q4 — Personality behavior
- Brand-lock signal present → `brand-locked` (prompt-shaping; remember there is no flag — apply via `personality_prompt` + `compiled_system_prompt`)
- Drift signal present → `drift on`
- Both / neither → `drift on` (platform default)

### Q5 — Proactive features
Direct map from signals:
- scheduled-reminders signal → `scheduled-reminders`
- backend-events signal → `backend-events`
- both → `both`
- neither → `none`

### Q6 — Latency budget
Direct map from signals' latency hint. If silent → `500ms-2s` (default).

Resolves memory_mode:
- `<500ms` → `memory_mode: async`, `skip_context_build: true` on chat calls
- `500ms-2s` → `memory_mode: async`
- `2s+` → `memory_mode: sync`

### Q7 — Integration path
Direct map from signals' integration-path hint. If silent → `typescript-sdk` (broadest reach for greenfield demos).

---

## Archetype-specific follow-ups

After Q2, look up the archetype's Section 4 follow-up questions from `../sonzai-sdk/intake.md` cheat sheet. For each follow-up:
1. Transcript answers it → use that
2. Silent → use the default below + add to Documented Assumptions

### companion defaults
- voice: off (tier-gated; enable later via dashboard if transcript mentions it)
- image: off (unless transcript explicitly asks)
- check-ins cadence: daily 09:00 user-local (if Q5 = scheduled-reminders or both)
- per-user agent: true (companion is per-user by definition; one agent per user_id)

### guide-router defaults
- number of specialists: 16 (MBTI) — only if framework signal unclear
- framework: MBTI (most common when "personality routing" mentioned without framework)
- specialist personalities: pre-defined (don't auto-generate unless transcript says "auto generate" or "from scratch")
- re-assessment allowed: true (low-cost flexibility)

### enterprise-assistant defaults
- users per agent: 1000 (placeholder; actual scaling is operator's concern)
- KB scope: `project_only` (lowest-friction; cascade requires org setup)
- privacy floor: disabled (unless transcript says compliance/SOC2)
- audit destination: `stdout` (writes to logs; operator can wire downstream later)

### customer-support defaults
- ticketing system: null (caller wires later — record as TODO)
- escalation channel: email (lowest-friction; Slack/PagerDuty require webhooks)
- KB docs: empty starter (place a sample doc; operator uploads real corpus later)

### game-npc defaults
- single NPC vs cast: single (cast=true only if transcript names multiple characters)
- inventory schema: empty starter (operator defines items)
- player-vs-shared state: `player-state` (per-player by default)
- sharding: off
- events: disabled (operator wires when ready)

### coach-therapist defaults
- session length: 30 min
- diary visibility: internal (safer default; user-facing requires separate UI)
- intake assessment shape: PHQ-9 (most common; document if transcript suggests other)
- reminder cadence: daily (if Q5 = scheduled-reminders)

### hybrid-custom defaults
- strongest-signal archetype as base
- list explicit feature borrows from other archetypes in Documented Assumptions

---

## Capability resolution

Build the final capabilities object using ONLY flags in `UpdateCapabilitiesInputBody` from `.full-auto/openapi.live.json`.

Canonical list (verify against drift artifact — schema may have changed):
- `autoLearnSkills`, `composio`, `imageGeneration`, `inventory`, `knowledgeBase`
- `knowledgeBaseScopeMode` (enum: `project_only` / `org_only` / `cascade` / `union`)
- `knowledgeBaseWrite`, `mcpEnabled`, `memoryMode` (enum: `sync` / `async`)
- `rememberName`, `sharedMemory`, `skills`, `webSearch`, `wisdom`

Pre-conditions (enforce):
- `sharedMemory: true` requires `wisdom: true` (see `../sonzai-sdk/decisions/sharedmemory-vs-wisdom.md`)
- `knowledgeBaseWrite: true` requires `knowledgeBase: true`
- `composio: true` requires `mcpEnabled: true`

NEVER include these in the capabilities object (read-only / tier-gated):
- `voiceGeneration`, `voiceId`, `voiceTier`, `voiceUnlockedAt`
- `customTools`, `pendingCapabilities`, `imageUnlockedAt`
- `musicGeneration`, `musicUnlockedAt`, `videoGeneration`, `videoUnlockedAt`
- `knowledgeBaseProjectId`

If transcript asks for voice/music/video → document under "Tier-gated capabilities (requires dashboard upgrade)" in assumptions.

---

## Target repo path

Default: `./sonzai-auto-<Q2>-<YYYY-MM-DD>` (e.g., `./sonzai-auto-companion-2026-05-13`).

Override:
- Operator passed `--target <path>` → use that
- Transcript names a repo / project ("call it Aria" / "name it support-bot") → use kebab-case of that name

If a directory at the target path already exists AND is non-empty AND has a `.git`, treat as `existing` for Q1 purposes (even if we previously said greenfield) — the builder will adapt.

---

## Acceptance checklist (what Phase 5 will test)

Generate this from the archetype's required endpoints + transcript-specific asks. Every line must be something Phase 5 can mechanically verify.

Examples per archetype:

### companion checklist
- [ ] `POST /chat` (or `client.chat.completions.create` server-side wrapper) returns streamed reply
- [ ] First token latency < {{Q6_BUDGET}} (measured)
- [ ] Agent created at startup (one per user_id) via `client.agents.create`
- [ ] Memory persists across two chats with same `user_id`
- [ ] If Q5=scheduled-reminders: `client.schedules.create` registers a daily cron
- [ ] If voice mentioned: `.env.example` documents tier-gated capability (no live voice tests)

### guide-router checklist
- [ ] Intake guide agent exists and responds
- [ ] After N intake turns, router determines specialist (assertion: returned `specialist_id` is one of the N)
- [ ] User can re-chat with assigned specialist; context carries (memory smoke test)
- [ ] Specialist personalities differ measurably (smoke: two specialists give different responses to same input)

### enterprise-assistant checklist
- [ ] Two different `user_id`s chatting same agent → both turns succeed
- [ ] Shared memory: a fact stored by user A is retrievable when user B asks (if sharedMemory=true)
- [ ] KB query returns matching docs (smoke against starter KB content)

### customer-support checklist
- [ ] Chat endpoint responds with KB-grounded reply (assertion: response references KB doc)
- [ ] Custom tools registered: `client.agents.tools.update` returned 200
- [ ] Escalation webhook fires on configured trigger (mock receiver)

### game-npc checklist
- [ ] NPC agent created with inventory + custom_states schema
- [ ] Item add/remove via `client.agents.inventory` mutates state
- [ ] Dialogue event triggers via `agents.triggerBackendEvent` (smoke)

### coach-therapist checklist
- [ ] Long session (10+ turns) maintains context
- [ ] Mood retrieval via `client.agents.mood.get` returns valid shape
- [ ] Diary entry exists post-session (if exposed by SDK)
- [ ] Intake assessment recorded as `custom_states`

### hybrid-custom checklist
- [ ] Generate from base archetype's checklist
- [ ] Add one assertion per borrowed feature

---

## Output template: `.full-auto/wizard-answers.md`

```markdown
# Wizard answers (derived autonomously)

**Transcript**: `.full-auto/transcript.txt`
**Signals**: `.full-auto/signals.md`
**Derived**: {{ISO_TIMESTAMP}}
**Drift artifact**: `.full-auto/openapi.live.json` (fetched at {{ISO}})

## Q1-Q7 answers
- **Q1** (project state): `{{Q1}}`
- **Q2** (archetype): `{{Q2}}`
- **Q3** (user id): `{{Q3}}`
- **Q4** (personality): `{{Q4}}`
- **Q5** (proactive): `{{Q5}}`
- **Q6** (latency): `{{Q6}}` → `memory_mode: {{MEMORY_MODE}}`
- **Q7** (integration): `{{Q7}}`

## Archetype follow-ups ({{Q2}})
{{FOLLOWUPS_AS_BULLETS}}

## Target
- **Repo path**: `{{TARGET_PATH}}`
- **Language**: {{LANG}}
- **Runtime**: {{RUNTIME_REQUIREMENTS}}

## Capabilities (UpdateCapabilitiesInputBody-shaped)
```json
{{CAPABILITIES_JSON}}
```

## Tier-gated capabilities (not enabled here)
{{TIER_GATED_LIST_OR_NONE}}

## Documented assumptions
1. {{ASSUMPTION_1}}
2. {{ASSUMPTION_2}}
...

## Out-of-scope mentions from transcript (operator should review)
{{OOS_LIST_OR_NONE}}

## Acceptance checklist (Phase 5 will verify each)
{{CHECKLIST_AS_BULLETS}}

## Reading order for the builder subagent
1. This file (`wizard-answers.md`)
2. Transcript: `.full-auto/transcript.txt`
3. Archetype playbook: `sonzai-claude-skill/skills/sonzai-sdk/archetypes/{{Q2}}.md`
4. Feature files referenced in Section 5 of that archetype
5. Spec template:
   - greenfield → `sonzai-claude-skill/skills/sonzai-sdk/spec-templates/archetype-spec.md.template`
   - existing-codebase → `sonzai-claude-skill/skills/sonzai-sdk/spec-templates/existing-codebase-spec.md.template`
6. Drift artifact: `.full-auto/openapi.live.json` (verify every Sonzai symbol against this before writing it)
```

## Self-check before handing to builder

Before writing `wizard-answers.md`, verify:

- Every capability in the JSON is in the live OpenAPI's `UpdateCapabilitiesInputBody` (grep the drift artifact)
- Pre-conditions hold (sharedMemory→wisdom; knowledgeBaseWrite→knowledgeBase)
- Every Documented Assumption corresponds to a transcript silence (not made up)
- Every Acceptance Checklist item is mechanically verifiable (a curl, a script, a screenshot)
- Target repo path is absolute or unambiguous relative to CWD
- No tenant-specific names (Razer, Eragon, etc.) leaked into the spec
