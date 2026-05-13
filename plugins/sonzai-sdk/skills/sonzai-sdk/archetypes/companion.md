---
name: archetype-companion
description: Use when building a 1:1 persistent AI companion (Replika-shaped) where personality evolves with each user, conversations are long-running, and the value is rapport that compounds over time.
---

# Companion archetype

A persistent AI companion that one user owns and that evolves with them. Memory builds over weeks; personality drifts toward the user; voice and image generation are common add-ons. Privacy-critical: each user gets their own context — never cross-bleed.

## 1. When this archetype fits

**Strong signals:**
- "Like Replika / Pi / Character.AI but for X"
- 1:1 relationship, no team sharing
- Sessions are long (10+ turns typical) and span weeks/months
- Personality should feel different per user (their companion vs your companion)
- Voice or image generation as engagement add-ons

**Anti-signals (use a different archetype):**
- Multiple users share one agent → `archetypes/enterprise-assistant.md`
- Routing by personality type → `archetypes/guide-router.md`
- Short transactional flows → not Sonzai; raw chat is fine

## 2. Prescribed stack

| Decision | Value | Why |
|---|---|---|
| `memory_mode` | `async` | TTFC matters; companion chat is interactive |
| Runtime mode (Q8) | **A — full-chat** | Chat-first product; `agents.chat` / `chatStream` gives the whole pipeline in one call. See `decisions/runtime-mode.md`. |
| LLM provider (production) | **BYOK** or Custom LLM | Platform credit is dev/eval only; production should use your own key. See `decisions/byok-vs-customllm.md`. |
| Personality drift | on (default — automatic) | The whole point: rapport that compounds (SOTOPIA s30 lift) |
| Per-user personality overlay | on (default — automatic) | Each user sees a different version of the agent |
| `shared_memory` | off | 1:1 — sharing is a privacy bug |
| `wisdom` | on (default) | K-anonymized cross-user patterns, no identity leak |
| `knowledge_base` | off | Companion is about the user, not a corpus |
| `remember_name` | on | Foundational — companion knows who you are |
| `web_search` | usually off | Internal-facing; only enable if your companion answers external questions |
| `image_generation` | optional (tier-gated) | Common engagement add-on |
| Voice (`voiceGeneration`, tier-gated) | optional | High-engagement; requires tier unlock + `memory_mode=async` |
| `inventory` | off | Companion has no items to track |

**Memory mode override:** If latency budget is generous (>2s TTFC acceptable, e.g. async/email-shaped UX), `sync` is fine and gives every fact same-turn — slight quality bump for cold-start.

## 3. Required SDK functions (in order)

### Step 1 — Create the agent

```python
# Python — generate from description (fastest cold-start)
from sonzai import Sonzai
client = Sonzai()

agent = client.agents.generation.generate_and_create(
    name="Luna",
    description="A warm, curious AI companion who remembers what matters to you. Asks open-ended questions, celebrates milestones, gently notices when you're off. Speaks in plain language; never therapy-jargon.",
    language="en",
)
print(agent.agent_id)
```

```typescript
// TypeScript
import { Sonzai } from "@sonzai-labs/agents";
const client = new Sonzai();

const agent = await client.agents.generation.generateAndCreate({
  name: "Luna",
  description: "A warm, curious AI companion who remembers what matters to you. Asks open-ended questions, celebrates milestones, gently notices when you're off. Speaks in plain language; never therapy-jargon.",
  language: "en",
});
```

```go
// Go
import sonzai "github.com/sonz-ai/sonzai-go"
client, _ := sonzai.NewClient("")

agent, err := client.Agents.Generation.GenerateAndCreate(ctx, sonzai.GenerateAndCreateOptions{
    Name:        "Luna",
    Description: "A warm, curious AI companion who remembers what matters to you. Asks open-ended questions, celebrates milestones, gently notices when you're off. Speaks in plain language; never therapy-jargon.",
    Language:    "en",
})
```

For explicit personality tuning instead, use `agents.create(name, big5={...}, personality_prompt=...)` — see `decisions/generation-vs-manual-create.md`.

### Step 2 — Set capabilities

```python
client.agents.update_capabilities(
    agent.agent_id,
    memory_mode="async",
    remember_name=True,
    image_generation=True,         # only if tier-unlocked — check first
)
```

Verify image was actually unlocked:

```python
caps = client.agents.get_capabilities(agent.agent_id)
if not caps.image_unlocked_at:
    print("image_generation flag set but tier not unlocked — contact admin")
```

### Step 3 — Run the per-turn session loop

```python
# Each user gets a Session handle. user_id is YOUR stable user identifier
# (Clerk JWT sub, NextAuth user.id, Supabase auth.uid()).
session = client.agents.sessions.start(
    agent.agent_id,
    user_id="user-123",
    session_id="session-456",  # your choice; persistent across turns of one conversation
    provider="gemini",
    model="gemini-3.1-flash-lite",
)

# Per-turn loop:
ctx = session.context(query="what's the user likely about to say?")
# ... build your prompt with ctx, call your LLM (or let Sonzai), get assistant_reply ...

result = session.turn(
    messages=[
        {"role": "user", "content": "I had a great day hiking"},
        {"role": "assistant", "content": assistant_reply},
    ],
    fetch_next_context={"query": "anticipated next user message"},
)
# result.mood (auto-updated), result.extraction_id (deferred fact extraction)
```

### Step 4 — End the session

```python
# wait=False for production (consolidation runs async — UX cost zero)
session.end(total_messages=10, duration_seconds=300, wait=False)

# wait=True only in tests/benchmarks that query memory immediately after
```

### Step 5 (optional) — Scheduled check-in

For **recurring** check-ins use `client.schedules.create`; for **one-off** wakeups use `client.agents.schedule_wakeup`. The companion's typical pattern is recurring.

```python
# Weekly check-in. cadence is {"cron": "..."} or {"simple": {...}};
# both forms require timezone. The agent generates a notification at fire
# time, surfaced via your delivery channel (SSE / polling / webhook —
# see decisions/proactive-channel.md).
client.schedules.create(
    agent_id=agent.agent_id,
    user_id="user-123",
    cadence={"cron": "0 18 * * SUN", "timezone": "America/New_York"},
    check_type="general",                     # check_type taxonomy is in features/proactive.md
    intent="Check in on the user after the weekend.",
)

# OR for a single one-off (e.g. "remind about the doctor appointment in 24h"):
client.agents.schedule_wakeup(
    agent.agent_id,
    check_type="reminder",
    delay_hours=24,
    intent="Remind about the doctor appointment tomorrow.",
)
```

### Step 6 (optional) — Voice

```python
# Verify tier first
caps = client.agents.get_capabilities(agent.agent_id)
if caps.voice_generation:
    voices = client.voices.list()
    selected = voices[0]            # pick from catalog
    # Then in chat handler — request live duplex token
    token = client.agents.voice.get_token(
        agent.agent_id,
        voice_name=selected.name,
        language="en-US",
        user_id="user-123",
    )
    stream = client.agents.voice.stream(token)
    # ... handle bidirectional audio
```

See `features/voice.md` for the full live-stream event loop.

## 4. Archetype-specific wizard intake questions

After Q2 picks `companion`, the wizard asks:

1. **Voice needed?** yes / no — if yes, verify tier and follow Section 3 Step 6
2. **Image generation needed?** yes / no — if yes, verify `imageUnlockedAt` after capability update
3. **Scheduled check-ins?** none / daily / weekly / custom cron — if any, see Section 3 Step 5
4. **One agent for all users, or per-user agent?**
   - One-agent-serves-all: simplest. Same `agent_id` for everyone; per-user overlay is automatic. Recommended.
   - Per-user agent: extra wiring (you create+store agent_id per user); usually unnecessary because Sonzai already personalizes per user via overlays.

## 5. Spec template fields (filled by wizard into `sonzai-implementation-spec.md`)

```markdown
## Agent

- **Strategy:** generate-from-description (recommended) OR explicit big5
- **Name:** <user choice>
- **Description (for generation):** <50-200 words>
- **Language:** en|...
- **One-agent-serves-all (recommended)** OR per-user agent

## Capabilities

- memory_mode: async (or sync if 2s+ TTFC budget)
- remember_name: true
- image_generation: <yes/no>
- voice_generation: <tier-check — yes if unlocked>
- web_search: <yes/no — default no>

## Session lifecycle

- session_id strategy: <e.g. one per conversation, rotated daily>
- session.end wait flag: false in production, true in tests
- fetch_next_context usage: <yes/no, helpful when latency matters>

## Proactive

- Scheduled wakeups: <none|cadence>
- Delivery channel: <SSE|polling|webhook> (see decisions/proactive-channel.md)

## Voice (if enabled)

- voice_id: <from voices.list()>
- language: en-US|...
- Live duplex vs TTS-only?
```

## 6. Plan template (typical step breakdown for `superpowers:writing-plans`)

The wizard writes these as atomic steps into `sonzai-implementation-plan.md`. Each step has a verify gate.

```markdown
1. Generate or create the agent — verify: `agents.list()` includes it
2. Set capabilities (memory_mode, remember_name, optional image/voice) — verify: `get_capabilities` returns expected values
3. Wire your server-side chat handler — verify: POST /chat returns 200 with content
4. Implement session lifecycle (start on connect, end on disconnect with wait=False) — verify: same session_id across turns
5. (Optional) Add scheduled wakeup — verify: `agents.notifications.list` returns it after the cadence
6. (Optional) Wire voice via `agents.voice.stream` — verify: PCM frames arriving
7. Integration test: 10-message session — verify: `agents.memory.search` finds facts from messages 1-3
8. Production checklist: SONZAI_API_KEY in secret manager, error boundaries on chat handler, BYOK if billing isolation required (see decisions/byok-vs-customllm.md)
```

## 7. Anti-patterns

| Anti-pattern | Why it's wrong | Fix |
|---|---|---|
| Enabling `shared_memory` for a companion | 1:1 — sharing across users is a privacy bug | Leave `shared_memory=false`; per-user overlay is what you want |
| Using `memory_mode=sync` + voice | Sync blocks the audio loop; first-token blows the audio TTFC budget | Force `async` when voice is on |
| Disabling personality drift (or trying to) | Defeats the compounding behavior — the s30 SOTOPIA lift disappears | Leave drift on (default); use prompt shaping if you need tone constraints |
| Per-user `agent_id` without justification | Fragments retrieval data; per-user overlay already exists — you're paying complexity for nothing | Use one agent for all users (per-user state is automatic via overlays) |
| Polling memory immediately after `session.end(wait=False)` | Consolidation hasn't run yet; you'll see stale state | Use `wait=True` in tests; in prod, accept eventual consistency |
| Exposing `SONZAI_API_KEY` to the browser (Expo/Next.js `NEXT_PUBLIC_*`, Vite `VITE_*`) | Server-side only — full account access if leaked | Proxy through your backend; the browser only talks to your server |

## Cross-references

- `decisions/memory-mode.md` — sync vs async detail
- `decisions/capabilities-matrix.md` — full capability grid
- `decisions/generation-vs-manual-create.md` — `generate_and_create` vs explicit `agents.create`
- `decisions/proactive-channel.md` — SSE vs polling vs webhook for scheduled check-ins
- `features/generation.md` — agent generation surface
- `features/voice.md` — TTS/STT/live duplex details
- `features/proactive.md` — schedules + wakeups + events + notifications
- `features/agent-insights.md` — diary / habits / goals (companion dashboards)
- `references/python.md` / `references/typescript.md` / `references/go.md` — per-language SDK surface
- `references/streaming-chat.md` — SSE vs polling chat patterns
