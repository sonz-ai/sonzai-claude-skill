---
name: decision-post-processing-model
description: Use when configuring the model that runs post-session processing (fact extraction, mood update, personality drift, diary writing).
---

# Decision: post-processing model

## The rule

**Default: Gemini Flash Lite (or your project's configured fast tier).** Override only if your domain needs richer extraction (medical, legal, high-stakes) — and the latency/cost trade-off matters less than quality.

## How to apply

Post-processing runs after each `session.end()` (or after consolidation triggers). It does: fact extraction + dedup, mood update, personality drift, diary writing, per-pair RL tuning. The model running this is configurable per chat-model in your project's post-processing model map.

```
Chat model → Post-processing model
gemini-3.1-flash-lite → gemini-3.1-flash-lite (default)
gpt-5.5 → gemini-3.1-flash-lite (cheaper than running 5.5 on every extraction)
*  → gemini-3.1-flash-lite (wildcard fallback)
```

Configure via project config — verify the exact API shape (`client.project_config.set(...)` or dashboard) in your installed SDK.

## Why

Post-processing runs on **every turn**. Using a frontier model here is 10-20× more expensive than Flash Lite and rarely improves outcomes for general chat.

Cheap fast tier handles:
- Fact extraction from conversational text — very high accuracy at Flash Lite quality
- Mood scoring — robust for the 4-dim space (valence, arousal, tension, affiliation)
- Personality drift — incremental updates, well-bounded
- Diary writing — short reflective summaries, model quality matters less

## When to upgrade the post-processing model

- **Medical / legal / clinical domain** — extraction quality matters more; pay the cost
- **Multi-lingual non-English** — verify Flash Lite handles your language well; some cheap models are English-biased
- **Specialized vocabulary** (game lore, jargon-heavy enterprise) — domain-specific fact extraction may need a stronger model

## Exceptions

- **Compliance regime requires specific provider for all calls** — the post-processing map must use that provider's cheap-tier model (e.g. OpenAI gpt-5-mini if you're locked to OpenAI).
- **Cost is irrelevant** — sure, use the same model as chat. Most teams won't notice the lift.

## Cross-references

- `features/models.md` — full models surface including post-processing map syntax
- `features/self-improvement.md` — what post-processing actually does
- `decisions/byok-vs-customllm.md` — provider-level routing (separate decision)
- `references/api_change_pipeline` (in your platform memory) — when default extraction model changes
