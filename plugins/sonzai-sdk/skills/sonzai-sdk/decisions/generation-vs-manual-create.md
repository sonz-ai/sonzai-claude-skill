---
name: decision-generation-vs-manual-create
description: Use when choosing how to create an agent — agents.generation.generate_and_create (auto-derive personality from a description) vs agents.create (explicit Big5 and personality_prompt).
---

# Decision: generate_and_create vs explicit agents.create

## The rule

Use **`agents.generation.generate_and_create(name, description, language)`** when you have a natural-language description and want personality + bio + seed memories auto-derived.

Use **`agents.create(name, big5={...}, personality_prompt=...)`** when you have explicit Big5 scores or need exact control.

## How to apply

```python
# Fastest cold-start: describe the agent in plain language
agent = client.agents.generation.generate_and_create(
    name="Luna",
    description=(
        "A cheerful and curious AI assistant who loves helping developers "
        "debug code. She's patient, witty, and always encouraging."
    ),
    language="en",
)
# Returns: agent with auto-derived Big5, personality_prompt, bio, seed memories
```

vs.

```python
# Production: explicit control
agent = client.agents.create(
    agent_id="...",                       # optional deterministic uuid5
    name="Luna",
    big5={
        "openness": 0.75,
        "conscientiousness": 0.60,
        "extraversion": 0.80,
        "agreeableness": 0.70,
        "neuroticism": 0.30,
    },
    personality_prompt="A cheerful, curious assistant...",
    language="en",
)
```

Both calls are idempotent on `agent_id` — repeat calls update (don't error or duplicate).

## When to pick which

| Situation | Recommended |
|---|---|
| Demo, prototype, hackathon | `generate_and_create` |
| First-time cold-start (you don't know Big5 yet) | `generate_and_create` |
| Production deploy with pinned personality | `agents.create` with explicit Big5 |
| A/B test personality variations | `agents.create` (consistent control) |
| Multi-language agent | `generate_and_create` (language-aware derivation) |
| Pre-defined framework specialists (MBTI's 16, etc.) | `agents.create` (you write each personality_prompt) |
| User-generated agents (e.g. "let users design their own NPC") | `generate_and_create` (use user's text directly) |
| Need to re-derive personality from updated description | `generate_and_create` with `regenerate=True` (verify exact flag shape in your SDK) |

## Why

- **`generate_and_create`** is great for cold-start. A 50-200 word description produces a coherent agent without you needing to tune Big5 by hand. The derived personality is consistent because it comes from one description.

- **`agents.create`** is for production where you want deterministic personality. Pinned Big5 scores survive regenerations; the personality is stable across deploys. Best when you've measured / iterated on what personality works for your domain.

## Exceptions

- **Mixed approach:** prototype with `generate_and_create`, capture the resulting Big5 values from the response, then switch to explicit `agents.create(big5=...)` for production stability.
- **Personality variation per user** is automatic via per-user overlays regardless of which creation API you used. You don't need separate agents per user.

## Cross-references

- `features/generation.md` — full generation surface including `generate_character()` (preview without commit)
- `archetypes/companion.md` — uses `generate_and_create` typically
- `archetypes/guide-router.md` — uses `agents.create` with explicit `personality_prompt` for the specialists (one per framework type)
- `archetypes/enterprise-assistant.md`, `archetypes/customer-support.md` — explicit `agents.create` with brand-locked `personality_prompt`
