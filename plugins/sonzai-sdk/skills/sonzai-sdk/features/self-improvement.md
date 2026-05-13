---
name: feature-self-improvement
description: Use when understanding the automatic post-session pipeline — fact extraction, dedup, personality drift, mood update, diary writing, per-pair RL/bandits. No direct API; triggered by sessions.end.
---

# Self-improvement (the post-session pipeline)

## What it is

The asynchronous pipeline that runs after `sessions.end()` (or on consolidation triggers). It does:

- **Fact extraction** — pulls facts from the session's conversation turns
- **Deduplication** — merges new facts with existing memory
- **Personality drift** — incremental updates to the agent's Big5 per the conversation
- **Mood update** — 4-dim mood shift based on conversation tone
- **Diary writing** — agent writes a reflective entry per session
- **Per-pair RL tuning** — online learning + bandits per `(agent_id, user_id)` pair to optimize retrieval and response quality
- **Significant moment detection** — flags breakthroughs

No direct API. You don't call self-improvement; it runs server-side. The post-processing model that handles it is configurable.

## When to use

This is automatic. The user-facing surfaces of self-improvement are in `features/agent-insights.md` (habits, goals, diary, mood, constellation).

You interact with self-improvement via:
- `sessions.end(wait=True)` — forces synchronous run (for tests / benchmarks)
- Configuring the post-processing model — see `decisions/post-processing-model.md`
- Reading the outputs via `features/agent-insights.md`

## How to control timing

```python
# Production — async (wait=False, default)
session.end(total_messages=10, duration_seconds=300, wait=False)
# Self-improvement runs eventually; query memory may show stale state briefly

# Tests / benchmarks — synchronous
session.end(total_messages=10, duration_seconds=300, wait=True)
# Blocks until consolidation finishes; memory queries immediately after see fresh state
```

## How to configure the post-processing model

```python
# Per-project — verify exact API shape against your SDK
client.project_config.set(
    project_id="your-project-id",
    post_processing_model_map={
        "gemini-3.1-flash-lite": "gemini-3.1-flash-lite",
        "gpt-5.5": "gemini-3.1-flash-lite",
        "*": "gemini-3.1-flash-lite",          # wildcard fallback
    },
)
```

See `features/models.md` for the full models surface.

## What you observe

```python
# After session.end completes (or after a wait), these surfaces reflect the run:
client.agents.get_diary(agent_id, user_id=user_id)              # new entry appears
client.agents.get_mood(agent_id, user_id=user_id)               # updated mood
client.agents.memory.search(agent_id, query="...", user_id=user_id)  # new facts findable
client.agents.personality.get_recent_shifts(agent_id, user_id=user_id)  # drift events logged
```

## Decisions linked

- `decisions/post-processing-model.md` — model choice
- `features/agent-insights.md` — the outputs you can read
- `archetypes/coach-therapist.md` — `wait=True` is mandatory here
- `archetypes/companion.md` — `wait=False` typical

## Common gotchas

- **`wait=True` blocks the calling thread** — only use in tests/benchmarks. Production should use `wait=False`.
- **Per-pair RL/bandits** — the agent's retrieval and response quality improves for *this user* over time; this is the SOTOPIA s30 lift mechanism.
- **Auto-tune behind the scenes** — no override for the RL/bandit hyperparameters; this is intentional.
- **Post-processing cost** — running on every turn (not just session end) for some sub-steps. Use a cheap fast model unless you have a domain reason to upgrade.
- **Eventual consistency** — async runs mean memory queries right after `session.end(wait=False)` may be stale. If your test needs fresh state, use `wait=True`.
- **Per-(agent, user) isolation** — RL/bandits learn per-pair. Same agent across users doesn't share tuning (good for privacy).
- **No way to inspect bandits/RL state** in the SDK — opaque. If you need that level of introspection, contact support / use platform analytics.
