---
name: feature-generation
description: Use when creating an agent from a natural-language description. Auto-derives personality (Big5), bio, and seed memories. Fastest cold-start.
---

# Agent generation

## What it is

`agents.generation.generate_and_create(name, description, language)` creates a full agent — personality (Big5), bio, seed memories — from a free-text description in seconds. Idempotent on `agent_id`.

## When to use

- Cold-start: you have a 50-200 word description, not Big5 scores
- Demos, prototypes, hackathons
- User-generated agents ("let users design their own NPC")
- The N specialists in a guide-router archetype — when each type can be described in language
- Onboarding flows where the agent personality is generated from user-provided text

## When NOT to use

- Production with pinned personality (use `agents.create` with explicit Big5 — see `decisions/generation-vs-manual-create.md`)
- A/B testing personality variations (you want clean control over Big5)
- Re-creating the same agent across deploys (use explicit `agents.create` with deterministic `agent_id`)

## SDK surface

```python
# Generate and create in one call
agent = client.agents.generation.generate_and_create(
    name="Luna",
    description=(
        "A warm, curious AI companion who remembers what matters to you. "
        "Asks open-ended questions; never therapy-jargon."
    ),
    language="en",
)
print(agent.agent_id)
print(agent.big5)              # auto-derived
print(agent.personality)       # full profile

# Preview without committing (verify the surface in your SDK version)
preview = client.agents.generation.generate_character(
    description="...",
    language="en",
)
# returns generated Big5, personality_prompt, etc. — doesn't create the agent
```

```typescript
const agent = await client.agents.generation.generateAndCreate({
  name: "Luna",
  description: "...",
  language: "en",
});
```

```go
agent, err := client.Agents.Generation.GenerateAndCreate(ctx, sonzai.GenerateAndCreateOptions{
    Name:        "Luna",
    Description: "...",
    Language:    "en",
})
```

## Best practices for `description`

- **50-200 words.** Too short = generic personality; too long = noisy signal.
- **Cover:** tone, characteristic behaviors, language style, boundaries.
- **Avoid:** narrative back-story (handled by seed memories), prescriptive instructions (handled by `personality_prompt`).
- **Language:** the `language` field affects the derivation. Multi-lingual variants generate language-aware personalities.

Example:

```
A cheerful and curious AI assistant who loves helping developers debug
code. She's patient, witty, and always encouraging. Asks clarifying
questions before suggesting fixes. Speaks plainly; uses humor when
tension rises.
```

## Decisions linked

- `decisions/generation-vs-manual-create.md` — when to use this vs `agents.create`
- `archetypes/companion.md` — typical use case
- `archetypes/guide-router.md` — generation for N specialists

## Common gotchas

- **Idempotent on `agent_id`** — repeat calls with the same name update the same agent. Don't accidentally call this on every request (you'll re-generate personality and surprise users).
- **`regenerate=True` (if exposed in your SDK)** forces fresh personality on an existing `agent_id`. Use during design, never in production unless you intend personality reset.
- **Description is the canonical source** — if you want to update personality, update the description and regenerate (with intent). Don't try to patch Big5 incrementally — that's `agents.update_profile`.
- **Multi-language drift** — if you change `language` later, expect personality shifts as the model re-derives.
- **`generate_character()` (preview)** returns derivations without commit. Use to iterate before settling.
