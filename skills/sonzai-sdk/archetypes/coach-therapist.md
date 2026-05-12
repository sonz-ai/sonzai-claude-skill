---
name: archetype-coach-therapist
description: Use when building a long-session coach / wellness journaler / therapist-adjacent agent. Sessions are long, sync memory, slow personality drift, diary visible to user, mood tracked across sessions. NOT a medical device replacement.
---

# Coach / therapist / journaler archetype

A wellness coach or journaling companion for long, infrequent sessions. Every fact matters; memory is sync; the agent maintains a diary (user-facing) and a tracked mood across sessions. Personality evolves slowly with rapport.

**Important boundary:** This archetype produces a wellness-shaped product. It is **not** a regulated medical device. Do not market it as therapy replacement. Comply with applicable regulations (HIPAA-adjacent if in the US; GDPR everywhere; specific consent and crisis-escalation requirements).

## 1. When this archetype fits

**Strong signals:**
- Weekly / bi-weekly sessions of 30-60 minutes
- User-facing journal / mood timeline / goal tracking
- High-touch personalization (user expects the agent to remember everything)
- Intake assessment at signup (PHQ-9, GAD-7, custom Likert)
- Slow rollout — no rapid drift surprise

**Anti-signals:**
- Short transactional chat → `archetypes/companion.md` (drift fast, low recall)
- Group setting / team → `archetypes/enterprise-assistant.md`
- Crisis-intervention / 24/7 hotline → out of scope; not a Sonzai archetype

## 2. Prescribed stack

| Decision | Value | Why |
|---|---|---|
| `memory_mode` | `sync` | Every fact lands same-turn — can't afford to lose one mid-session |
| Personality drift | on but slow (via prompt shaping) | Evolves with the relationship without surprising the user |
| `shared_memory` | off | Per-user privacy; never share across users |
| `wisdom` | on (default, k-anonymized) | Cross-user patterns without identifying anyone |
| `remember_name` | on | Foundational |
| `knowledge_base` | optional | If you upload coping-skills / self-help corpus |
| `web_search` | off | Compliance + clinical concerns |
| `image_generation` | off | Off-domain |
| `inventory` | off | Use `custom_states` for goal tracking instead |
| Priming | yes | Intake assessment seeds memory |
| Agent insights (diary, mood, goals) | exposed | User-facing dashboard |
| Voice (`voice_generation`) | optional (tier-gated) | Voice journaling is high-engagement |

## 3. Required SDK functions (in order)

### Step 1 — Create the agent

```python
from sonzai import Sonzai
client = Sonzai()

agent = client.agents.create(
    name="Compass",
    personality_prompt=(
        "Compass, a warm, patient wellness coach. Listens without judgment. "
        "Asks clarifying questions before offering reframes. Validates feelings "
        "before suggesting actions. Never diagnoses. Defers to professional "
        "help for crisis topics (self-harm, suicide, abuse) — surfaces hotline "
        "info immediately. Tracks user goals and mood patterns across sessions; "
        "references them naturally. Avoids therapy jargon."
    ),
    language="en",
)
```

### Step 2 — Set capabilities (sync memory, no web search)

```python
client.agents.update_capabilities(
    agent.agent_id,
    memory_mode="sync",
    remember_name=True,
    knowledge_base=False,                 # or True if you upload self-help corpus
    web_search=False,
    shared_memory=False,
)
```

### Step 3 — Intake assessment via priming

After signup, run the intake (PHQ-9, GAD-7, custom Likert) in your UI, then prime the user with the results:

```python
# Your UI collects intake_results = {"phq9_score": 8, "stressors": [...], ...}
client.priming.prime_user(
    agent_id=agent.agent_id,
    user_id=user_id,
    metadata={
        "display_name": user.first_name,
        "intake_phq9_score": str(intake_results["phq9_score"]),
        "intake_gad7_score": str(intake_results["gad7_score"]),
        "primary_goal": user.primary_goal,
    },
    content_blocks=[
        {
            "type": "text",
            "content": (
                f"User completed intake on {intake_date}. PHQ-9 score: "
                f"{intake_results['phq9_score']} ({severity_band}). "
                f"Reported stressors: {', '.join(intake_results['stressors'])}. "
                f"Stated primary goal: {user.primary_goal}."
            ),
        },
    ],
)
```

The agent now has the intake context available from turn 1.

### Step 4 — Session lifecycle (sync, wait=True at end)

```python
def start_coaching_session(user_id):
    session = client.agents.sessions.start(
        agent.agent_id,
        user_id=user_id,
        session_id=f"session-{user_id}-{date.today().isoformat()}",
        provider="gemini",
    )
    return session

def end_coaching_session(session, total_messages, duration_seconds):
    # wait=True forces server-side consolidation to complete before this call returns.
    # Critical: next session might be a week away, but if we re-query memory immediately
    # (e.g. for a summary email), we need consolidated state.
    session.end(
        total_messages=total_messages,
        duration_seconds=duration_seconds,
        wait=True,
    )
```

### Step 5 — Per-turn loop

```python
ctx = session.context(query="what should this turn focus on?")
# ... your prompt building + your LLM call (or let Sonzai handle it) ...

result = session.turn(
    messages=[
        {"role": "user", "content": user_message},
        {"role": "assistant", "content": assistant_reply},
    ],
)
# Mood update is automatic — accessible via result.mood and longer-term via get_mood()
```

### Step 6 — Diary (user-facing summary)

After each session, the agent writes a diary entry reflecting on the conversation. Surface this to the user (with care — see anti-pattern #2 below).

```python
diary = client.agents.get_diary(
    agent.agent_id,
    user_id=user_id,
    limit=10,
)
for entry in diary.entries:
    # entry.content — the agent's reflection
    # entry.timestamp — when this entry was written
    # Surface in YOUR formatted view; do NOT show raw verbatim
    user_facing_summary = your_app_summarizer(entry.content)
```

### Step 7 — Mood timeline

```python
mood_history = client.agents.get_mood_history(
    agent.agent_id,
    user_id=user_id,
    start="2026-01-01",
    end="2026-05-13",
)
# Surface as chart in your UI
```

Or current mood snapshot:

```python
mood = client.agents.get_mood(agent.agent_id, user_id=user_id)
```

### Step 8 — Goals tracking (custom_states or insights API)

```python
# Goals via custom_states (you control the schema)
client.agents.custom_states.upsert(
    agent.agent_id,
    key="weekly_goals",
    value={"goals": [...], "set_at": "..."},
    scope="user",
    user_id=user_id,
)

# OR pull auto-extracted goals from agent insights
goals = client.agents.list_goals(agent.agent_id, user_id=user_id)
```

### Step 9 — Weekly check-in wakeup (optional)

```python
client.schedules.create(
    agent_id=agent.agent_id,
    user_id=user_id,
    cadence={"cron": "0 18 * * SUN", "timezone": user.timezone},
    check_type="general",
    intent="Gentle Sunday-evening prompt to reflect on the week.",
)
```

### Step 10 — Crisis escalation (NON-NEGOTIABLE)

```python
# Detect crisis keywords + LLM classification in your handler
CRISIS_KEYWORDS = ["suicide", "self-harm", "kill myself", "end it", ...]

def detect_crisis(message_text):
    if any(kw in message_text.lower() for kw in CRISIS_KEYWORDS):
        return True
    # Plus an LLM classifier on ambiguous phrasing
    return False

def handle_message(user_id, message_text):
    if detect_crisis(message_text):
        return CRISIS_RESPONSE  # immediate hotline info + de-escalation script
    # ... else proceed with normal session.turn
```

The crisis response is a static, human-vetted message with hotline info (988 in US, etc.). Do not delegate this to the agent.

## 4. Archetype-specific wizard intake questions

After Q2 picks `coach-therapist`, the wizard asks:

1. **Session length (typical)?** 15-30 min / 30-60 min / 60+ min. Affects context budget.
2. **Diary visibility?** User-facing (recommended — but you summarize, never expose raw) / internal-only (you read it; user doesn't see).
3. **Intake assessment shape?** PHQ-9 / GAD-7 / custom Likert / structured questionnaire. Drives priming content.
4. **Weekly / bi-weekly cadence?** Sets up scheduled wakeup.
5. **Voice variant?** Off (default) / yes (tier-gated). Voice journaling is high-engagement.
6. **Crisis-detection coverage?** Keyword filter only / keyword + LLM classifier / external service.
7. **Goal tracking surface?** custom_states (you control) / insights API auto-extraction / both.

## 5. Spec template fields

```markdown
## Agent

- name: <coach name>
- personality_prompt: <directive voice; includes crisis-escalation rule>

## Capabilities

- memory_mode: sync
- web_search: false
- knowledge_base: <true if you upload self-help corpus>

## Intake

- Assessment: PHQ-9 | GAD-7 | custom (specify)
- Priming: metadata + one content block summarizing intake
- Run priming after signup completes

## Session lifecycle

- session_id strategy: per-session UUID (e.g. session-{user_id}-{date})
- session.end wait=True (mandatory — next session may be days away)

## Diary

- Visibility: user-facing (summarized) | internal-only
- Refresh cadence: after each session.end
- Summarizer: <your function — never expose raw>

## Mood

- Polling endpoint: <your /api/mood handler hits get_mood_history>
- UI: chart over time

## Goals

- Storage: custom_states (keys: weekly_goals, monthly_goals) OR insights API
- Refresh: per session

## Crisis escalation

- Detection: keywords + LLM classifier
- Response: static hotline message (HUMAN-VETTED — never agent-generated)
- Hotlines per locale: <list>

## Proactive

- Weekly check-in: schedules.create with user's timezone

## Compliance

- Consent screen at signup (data use, drift, agent boundaries)
- HIPAA-adjacent: <region-specific stance>
- Data export / deletion: <your runbook>
```

## 6. Plan template

```markdown
1. Create agent with directive personality_prompt (including crisis-escalation rule). Verify exists.
2. Set capabilities (sync, no web_search). Verify get_capabilities.
3. Build intake UI (PHQ-9 / GAD-7 / custom). Verify scoring algorithm.
4. Implement priming after signup: priming.prime_user with metadata + content_block. Verify with memory.search.
5. Build session lifecycle handler (start, turn-loop, end with wait=True). Verify session_id consistent across turns.
6. Implement crisis detection (keywords + LLM classifier). Test with sample sentences — must intercept BEFORE session.turn.
7. Implement diary surfacing (your summarizer wrapping get_diary). Verify the user-facing version does not expose raw agent thoughts.
8. Implement mood timeline endpoint. Verify get_mood_history returns expected shape.
9. Implement goals tracking (custom_states OR insights API). Verify retrieval.
10. Set up weekly schedules.create with user timezone. Verify cadence in staging by advancing time.
11. Integration test: 3 sessions over 3 weeks. Verify personality drift is gradual, mood timeline reflects session tone, goals are tracked.
12. Compliance review: consent screen, data-export runbook, hotline list, crisis-detection coverage.
13. Production: BYOK if your provider requires HIPAA-shaped BAA; audit logs to your SIEM.
```

## 7. Anti-patterns

| Anti-pattern | Why it's wrong | Fix |
|---|---|---|
| Using `memory_mode=async` | Sessions can't afford to lose a fact mid-conversation; async lets facts spill to next turn | Use sync always for coaching |
| Skipping `wait=True` on session.end | Next session might be days later; if you re-query memory before consolidation finishes, you get stale state | Always `wait=True` on session.end for this archetype |
| Exposing raw diary entries verbatim to user | Diary contains the agent's reflective thoughts; not all are user-appropriate; some surface internal reasoning | Always summarize via your own function before showing |
| Marketing as therapy / medical device | Regulatory exposure (FDA, MDR), liability | Position as wellness / journaling / coaching; explicit "not a medical device" disclaimer |
| Letting the agent handle crisis topics | Hallucinated hotline numbers, wrong de-escalation, liability | Static crisis response, human-vetted; intercept BEFORE session.turn |
| Enabling `web_search` | External sources can include misinformation; clinical risk | Off; if you need self-help references, upload curated KB |
| Sharing diary or mood data across users | Privacy bug | shared_memory off; per-user only |
| Showing mood timeline as causal explanation ("your mood dropped because you worked late") | False inference from correlation; user backlash | Visualize the data; let user interpret; agent comments stay open-ended |
| Skipping intake priming for returning users | Old context lost on app re-install | Re-prime if you have new intake data; otherwise rely on existing memory |

## Cross-references

- `decisions/memory-mode.md` — why sync is mandatory here
- `decisions/capabilities-matrix.md` — coach column
- `features/priming.md` — intake priming details
- `features/agent-insights.md` — diary, mood, goals API
- `features/proactive.md` — weekly schedule setup
- `features/sessions-vs-conversations.md` — see decisions/sessions-vs-conversations.md (use sessions, not chat)
- `archetypes/companion.md` — adjacent archetype; coach is a more rigorous variant
