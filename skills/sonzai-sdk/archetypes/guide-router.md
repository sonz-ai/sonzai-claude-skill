---
name: archetype-guide-router
description: Use when building a personality-routed app where an intake "guide" agent assesses the user, then routes them to one of N specialist agents (MBTI types, Enneagram, attachment styles, custom frameworks). Common pattern for matchmakers, character-pickers, and onboarding funnels.
---

# Guide-router archetype

An intake **guide** agent profiles the user via conversation, scores them on a personality framework, then routes them to one of N **specialist** agents tuned to that result. After routing, the user interacts exclusively with their specialist — the guide is not in the loop.

## 1. When this archetype fits

**Strong signals:**
- "MBTI matchmaker" / "16 companions" / "Big5 router" / "Enneagram routing"
- Anonymous-at-intake users (no `user_id` yet — they land on the site fresh)
- A short intake phase (5-15 turns) → a long downstream phase
- The specialists have **stable, pre-defined personalities** keyed by framework type (INTJ, INFJ, ...)
- One-time routing decision (re-assessment is optional, not required)

**Anti-signals:**
- Stable user_id from day 1 and only one agent → `archetypes/companion.md`
- Routing based on team/role rather than personality → `archetypes/enterprise-assistant.md`
- Multi-axis matching (personality + skills + availability) → use this archetype as a starting point but expect to extend

## 2. Prescribed stack

This archetype has **two different stacks** — one for the guide, one for the specialists.

### Guide agent (intake only)

| Decision | Value | Why |
|---|---|---|
| `memory_mode` | `async` | Intake is interactive; TTFC matters. The guide doesn't need every prior fact in-turn — the assessment is short-lived. |
| Personality drift | minimize via prompt shaping | Consistent intake across users. Write a directive `personality_prompt` and keep `compiled_system_prompt` on every chat. (No flag disables drift.) |
| `web_search` | off | Intake is internal; no external lookup needed |
| `image_generation` | off | |
| `knowledge_base` | off | Guide doesn't reference a corpus |
| `shared_memory` | off | Each user's assessment is private |
| `remember_name` | on (if you've collected one by then) | |
| `inventory` | off | |

### Specialist agents (one per framework type — e.g. 16 for MBTI)

| Decision | Value | Why |
|---|---|---|
| `memory_mode` | `sync` | Long-running relationship — every fact matters per turn |
| Personality drift | on (default) | Specialists evolve with their assigned users |
| `web_search` | typically on | Specialists may answer external questions |
| `image_generation` | optional (tier-gated) | |
| `knowledge_base` | optional | If you've uploaded type-relevant material |
| `shared_memory` | off | Per-user relationships, not team-shared |
| `remember_name` | on | |
| `inventory` | off | |

## 3. Required SDK functions (in order)

### Step 1 — At deploy time, once: create the guide agent

```python
from sonzai import Sonzai
client = Sonzai()

guide = client.agents.generation.generate_and_create(
    name="Personality Guide",
    description=(
        "Conducts a brief, friendly assessment to understand the user's "
        "personality type. Asks open-ended questions about preferences, "
        "decision-making, and social energy. Listens carefully, asks one "
        "follow-up at a time. Never reveals which type the user is — "
        "stores the result silently and confirms only that the assessment "
        "is complete."
    ),
    language="en",
)
client.agents.update_capabilities(
    guide.agent_id,
    memory_mode="async",
    remember_name=True,
)
```

Idempotent — repeat calls with the same name updates the same agent (or pass an explicit `agent_id`).

### Step 2 — At deploy time, once: create the N specialists

For MBTI (16 types). Use deterministic `agent_id` so the mapping is stable.

```python
import uuid
NAMESPACE = uuid.UUID("00000000-0000-0000-0000-000000000001")  # YOUR own namespace UUID

MBTI_PROFILES = {
    "INTJ": "The Architect — strategic, independent, decisive. Asks why before how. Reserved in early turns; opens up to logical rigor.",
    "INFJ": "The Advocate — insightful, idealistic, compassionate. Listens deeply, returns to themes. Slow to trust but deeply loyal.",
    "INTP": "The Logician — analytical, curious, abstract. Plays with ideas, ambivalent about action. Warms to playful debate.",
    "INFP": "The Mediator — values-driven, empathetic, imaginative. Speaks in metaphors. Sensitive to dismissal.",
    "ENTJ": "The Commander — assertive, organized, strategic. Drives toward outcomes. Frustrated by drift.",
    "ENFJ": "The Protagonist — charismatic, inspiring, attuned to others. Reflects emotion back. Energized by alignment.",
    "ENTP": "The Debater — quick, witty, contrarian. Tests ideas through opposition. Loves a sparring partner.",
    "ENFP": "The Campaigner — enthusiastic, creative, social. Jumps topics. Energized by novelty.",
    "ISTJ": "The Logistician — practical, dutiful, methodical. Prefers concrete plans. Loyal to commitments.",
    "ISFJ": "The Defender — warm, conscientious, protective. Remembers preferences. Quietly attentive.",
    "ISTP": "The Virtuoso — hands-on, observant, calm. Speaks in actions more than words. Direct.",
    "ISFP": "The Adventurer — gentle, aesthetic, present. Notices small beauty. Reluctant to over-explain.",
    "ESTJ": "The Executive — organized, traditional, direct. Values clarity. Pushes for resolution.",
    "ESFJ": "The Consul — warm, social, attentive. Remembers details about people. Gives generously.",
    "ESTP": "The Entrepreneur — bold, perceptive, present. Reads rooms. Acts before discussing.",
    "ESFP": "The Entertainer — spontaneous, playful, warm. Brings energy. Notices when someone fades.",
}

specialists = {}
for code, description in MBTI_PROFILES.items():
    agent_id = str(uuid.uuid5(NAMESPACE, f"mbti-specialist-{code}"))

    agent = client.agents.create(
        agent_id=agent_id,                 # deterministic — safe to re-run
        name=f"Companion-{code}",
        personality_prompt=description,
        language="en",
    )
    client.agents.update_capabilities(
        agent_id,
        memory_mode="sync",
        web_search=True,
        remember_name=True,
    )
    specialists[code] = agent_id

# Persist `specialists` mapping (config file, env, DB) for runtime routing.
```

For non-MBTI frameworks, swap the dict (Big5 quadrants, attachment styles, etc.) and the count.

### Step 3 — At runtime: anonymous user lands; mint a stable `user_id`

```python
import uuid
# Derive from session cookie OR JWT OR a fresh uuid if truly anonymous
USER_NAMESPACE = uuid.UUID("11111111-1111-1111-1111-111111111111")
user_id = str(uuid.uuid5(USER_NAMESPACE, session_cookie_or_email_or_random))
```

Why a derived UUID: stable across the user's reloads; survives auth-later flows when they identify themselves.

### Step 4 — At runtime: start guide session

```python
session = client.agents.sessions.start(
    guide.agent_id,
    user_id=user_id,
    session_id=f"guide-{user_id}",          # one guide session per user
    provider="gemini",
    model="gemini-3.1-flash-lite",
)

# Per-turn loop driving the assessment
for user_message in user_messages:
    result = session.turn(
        messages=[{"role": "user", "content": user_message}],
    )
    # ... show response to user, get next user message ...
```

### Step 5 — Detect assessment completion

You need a deterministic signal that the guide has reached a verdict. Three options (pick one):

**Option A — Structured tool call.** Register a custom tool the guide can fire when it has a verdict:

```python
client.agents.create_custom_tool(
    guide.agent_id,
    name="record_personality_result",
    description="Call when you have a confident assessment of the user's MBTI type.",
    parameters={
        "type": "object",
        "properties": {
            "type": {"type": "string", "enum": list(MBTI_PROFILES.keys())},
            "confidence": {"type": "number", "minimum": 0, "maximum": 1},
            "rationale": {"type": "string"},
        },
        "required": ["type", "confidence", "rationale"],
    },
)
```

On each `session.turn`, inspect `result.side_effects.external_tool_calls` for `record_personality_result` — that's your verdict signal.

**Option B — Backend-side classifier.** After ~5 guide turns, your app classifies the transcript directly (no agent tool); pull turns via the session API and run a classification call.

**Option C — Turn-count gate.** Simpler/cruder: after N turns, your app calls `agents.evaluate` with a small classification template to score the transcript.

Option A is recommended — keeps the verdict in the agent's voice and gives you confidence + rationale to log.

### Step 6 — Write verdict to user-scoped custom state

```python
client.agents.custom_states.upsert(
    guide.agent_id,
    key="mbti_assessment",
    value={
        "type": "INFJ",
        "confidence": 0.87,
        "assessed_at": "2026-05-13T14:32:00Z",
        "rationale": "...",
    },
    scope="user",
    user_id=user_id,
)
```

User-scoped — any agent (including the specialists) can read this for the same `user_id`.

### Step 7 — Read verdict and route

```python
state = client.agents.custom_states.get_by_key(
    guide.agent_id,                 # state was written under guide — still readable by your code
    key="mbti_assessment",
    scope="user",
    user_id=user_id,
)
mbti = state.value["type"]
specialist_agent_id = specialists[mbti]
```

### Step 8 — Start specialist session (new session_id, same user_id)

```python
specialist_session = client.agents.sessions.start(
    specialist_agent_id,
    user_id=user_id,                       # SAME user_id as the guide
    session_id=f"specialist-{user_id}",
    provider="gemini",
    model="gemini-3.1-flash-lite",
)

# Optional: seed the specialist with the user's type via system prompt
# so the specialist's first turn is already MBTI-aware:
specialist_session.turn(
    messages=[{"role": "user", "content": user_first_message}],
    compiled_system_prompt=(
        f"You are this user's MBTI={mbti} companion. Adapt accordingly. "
        f"Their assessed type is: {mbti}."
    ),
)
```

### Step 9 — End guide session

```python
session.end(total_messages=12, duration_seconds=600, wait=False)
```

After this point, the user's interactions go only through the specialist. The guide is done.

### TypeScript / Go variants

The same flow in TS uses `client.agents.generation.generateAndCreate`, `client.agents.updateCapabilities`, `client.agents.sessions.start`, `client.agents.customStates.upsert`. Go uses PascalCase grouped on `client.Agents.Generation.GenerateAndCreate`, etc. See `references/typescript.md` and `references/go.md` for full call shapes.

## 4. Archetype-specific wizard intake questions

After Q2 picks `guide-router`, the wizard asks:

1. **Number of specialists?** 16 (MBTI), 5 (Big5 quadrants), 9 (Enneagram), custom N.
2. **Personality framework?** MBTI / Big5 / OCEAN / attachment-styles / custom. Determines specialist count and personality prompts.
3. **Specialist personalities — pre-defined or auto-generated?**
   - Pre-defined (you write each `personality_prompt`): full control, recommended for known frameworks.
   - Auto-generated (one `generate_and_create` call per type with the framework-name description): faster to scaffold, less control.
4. **Re-assessment allowed?** No (one-time, locked) / Yes (user can retake) / Conditional (retake after N months).
5. **Verdict detection mechanism?** Tool call (recommended) / backend-side classifier / turn-count gate.
6. **Anonymous-at-intake or pre-identified?** Anonymous is the typical guide-router pattern. Pre-identified just means you skip the user_id minting step.

## 5. Spec template fields (filled by wizard into `sonzai-implementation-spec.md`)

```markdown
## Framework

- **Type:** MBTI | Big5 | OCEAN | attachment | custom
- **Specialist count:** N
- **Specialist personalities:** pre-defined (table below) | auto-generated

## Agents

### Guide
- Strategy: generate-from-description
- Description (for generate_and_create): <50-200 words>
- Capabilities: memory_mode=async, remember_name=true

### Specialists (N agents)
- Strategy: explicit agents.create with deterministic agent_ids (`uuid5(NAMESPACE, "specialist-{type}")`)
- Per-type personality_prompt: <one per framework type>
- Capabilities: memory_mode=sync, web_search=true, remember_name=true

## User identification

- Anonymous → derived UUID from session cookie/email (use uuid5 with YOUR namespace)
- Pre-identified → pass auth user_id directly

## Verdict detection

- Mechanism: tool-call (recommended) | backend classifier | turn-count
- Confidence threshold: 0.X (below → ask more questions)
- Storage: custom_states key "mbti_assessment" (or framework-named), scope=user

## Re-assessment policy

- One-time | retake-allowed | conditional (cooldown N months)
```

## 6. Plan template (typical step breakdown for `superpowers:writing-plans`)

Atomic steps the wizard writes into `sonzai-implementation-plan.md`:

```markdown
1. Mint your own UUID namespace; commit it to config. Verify: namespace is stable across deploys.
2. Deploy-time: create the guide agent via generate_and_create. Verify: guide.agent_id is stable.
3. Deploy-time: loop-create the N specialists with deterministic agent_ids. Verify: all N exist via agents.list().
4. Deploy-time: register the `record_personality_result` custom tool on the guide. Verify: appears in get_capabilities().customTools.
5. Implement anonymous-user → user_id derivation. Verify: same session cookie → same user_id across reloads.
6. Implement guide chat handler (session.start, session.turn). Verify: guide responds in character.
7. Implement verdict detection: read result.side_effects.external_tool_calls per turn. Verify: tool fires once after sufficient context.
8. Implement custom_states.upsert with the verdict. Verify: get_by_key returns the value.
9. Implement routing function: read state → map to specialist agent_id. Verify: returns correct agent for each type.
10. Implement specialist chat handler (new session, same user_id, agent_id from mapping). Verify: specialist responds in the correct MBTI character.
11. End-to-end test: anonymous user → guide assessment → custom_state write → routed → 5 specialist turns. Verify: specialist references user's name from the guide phase (memory carries via user_id).
12. (Optional) Re-assessment flow: implement a "retake" route that overwrites custom_state and re-routes. Verify: switching from INFJ to INTJ activates the right specialist.
```

## 7. Anti-patterns

| Anti-pattern | Why it's wrong | Fix |
|---|---|---|
| Storing the personality result in guide's memory only | Specialists can't see it; they'd have to re-assess | Use `custom_states.upsert(scope="user")` — visible to any agent via the same user_id |
| Creating specialists per-user at routing time | Fragments retrieval data; per-type populations get split; quota burn | Create once at deploy with deterministic `agent_id` (`uuid5`); reuse across users |
| Sharing `session_id` between guide and specialist | Conflates two distinct conversation arcs; memory consolidation gets confused | Use different `session_id` per phase (e.g. `guide-{user_id}` and `specialist-{user_id}`) |
| Routing on a single guide turn | Insufficient signal; users will get mis-routed | Require ≥5 turns AND ≥0.7 confidence; ask follow-ups if confidence is low |
| `personality_drift_disabled=true` on specialists (doesn't exist anyway) | Defeats the per-user evolution that makes specialists feel right | Leave drift on; specialists naturally adapt to each user. Drift on the guide is fine to leave at default — prompt-shape it consistent instead. |
| Showing the user their MBTI type without consent | Privacy / surprise factor | Confirm assent before reveal; the platform stores it regardless |
| Hard-coding specialist `agent_id`s as strings | Brittle if you re-deploy; collisions with another developer's namespace | Always derive via `uuid5(YOUR_NAMESPACE, "specialist-<type>")` |
| Specialist count ≠ framework type count (e.g. 17 specialists for MBTI) | Framework has exactly N types; mismatch leaves a routing gap or dead agent | Confirm framework type count before generating specialists; the wizard will flag this |

## Cross-references

- `intake.md` — wizard entry; this archetype is loaded after Q2=guide-router
- `decisions/memory-mode.md` — explains the asymmetric memory mode (async guide, sync specialists)
- `decisions/capabilities-matrix.md` — full capability grid (guide column + specialist column)
- `decisions/generation-vs-manual-create.md` — when to use `generate_and_create` vs `agents.create`
- `features/generation.md` — agent generation surface
- `features/custom-tools.md` — registering `record_personality_result`
- `features/custom-states.md` — assessment storage details
- `decisions/sessions-vs-conversations.md` — which API to use per phase
