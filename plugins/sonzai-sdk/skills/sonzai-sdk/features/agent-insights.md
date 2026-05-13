---
name: feature-agent-insights
description: Use when surfacing derived read-only signals about an agent's user — habits, goals, interests, relationships, diary, constellation (memory clusters), breakthroughs. Powers dashboards.
---

# Agent insights

## What it is

Read-only derived signals the agent extracts as it learns about a user. These power user-facing dashboards (top goals, mood timeline, who you've mentioned) and operator dashboards (drift monitoring, breakthrough detection).

No author step required — extraction happens automatically during the self-improvement pipeline.

## What's available

| Surface | Returns |
|---|---|
| `list_habits` | Recurring patterns the agent noticed (sleep, exercise, work cadence) |
| `list_goals` | Stated and inferred goals |
| `get_interests` | Topics the user engages with |
| `get_relationships` | People/orgs the user mentions and how |
| `get_diary` | Agent's reflective entries about the user (one per session typically) |
| `get_constellation` | Memory cluster graph — nodes + edges + insight pointers |
| `list_breakthroughs` | Moments the agent flagged as significant (mood inflection, goal shift) |
| `get_mood` | Current 4-dim mood (valence, arousal, tension, affiliation) |
| `get_mood_history` | Mood over time |
| `get_recent_shifts` (on personality) | Personality drift events |
| `get_significant_moments` (on personality) | Personality milestone events |

## When to use

- User-facing dashboards (Companion / Coach archetypes)
- Operator monitoring (drift alerts, breakthrough surfacing)
- Weekly digest emails ("here's what your companion noticed about you")
- Compliance review (significant moments + diary for audit)

## When NOT to use

- Real-time per-turn signals — these update at session end / post-processing, not mid-turn
- Authoritative source — these are *the agent's view*, not ground truth. Treat them as soft inputs.

## SDK surface

```python
# All on client.agents.* — per-user (pass user_id)
habits = client.agents.list_habits(agent_id, user_id="user-123")
goals = client.agents.list_goals(agent_id, user_id="user-123")
interests = client.agents.get_interests(agent_id, user_id="user-123")
relationships = client.agents.get_relationships(agent_id, user_id="user-123")
diary = client.agents.get_diary(agent_id, user_id="user-123", limit=10)
constellation = client.agents.get_constellation(agent_id, user_id="user-123")
breakthroughs = client.agents.list_breakthroughs(agent_id, user_id="user-123")

mood = client.agents.get_mood(agent_id, user_id="user-123")
mood_history = client.agents.get_mood_history(
    agent_id, user_id="user-123",
    start="2026-01-01", end="2026-05-13",
)

# Personality history (read via personality sub-resource)
shifts = client.agents.personality.get_recent_shifts(agent_id, user_id="user-123")
moments = client.agents.personality.get_significant_moments(agent_id, user_id="user-123", limit=10)
```

Goals and habits may also be **writable** (for seeding); confirm against your SDK.

## Decisions linked

- `archetypes/coach-therapist.md` — diary, mood, goals are user-facing
- `archetypes/companion.md` — companion dashboards
- `features/self-improvement.md` — the pipeline that produces these signals

## Common gotchas

- **Update latency** — refreshed at session end, not real-time. Don't expect mid-session changes.
- **Diary contains raw agent thoughts** — never show verbatim; summarize before user-facing. See `archetypes/coach-therapist.md` anti-patterns.
- **Goals/habits/interests are derived** — accuracy varies. Treat as suggestions; allow user to correct.
- **Mood is a 4-dim space**: valence, arousal, tension, affiliation. Different from "sentiment" (1-dim).
- **Constellation graph** — nodes and edges plus pointers to underlying facts. Useful for "show me how the agent thinks of me" UI but UX is non-trivial.
- **Breakthroughs are sparse** — don't expect one every session; they're noteworthy moments only.
- **Privacy** — these signals are per-user; never expose user A's diary to user B. Standard scoping rules apply.
- **Reading user-facing diary** — recommend summarizing 3-5 recent entries into a weekly digest rather than per-session verbatim.
