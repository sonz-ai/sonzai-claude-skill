---
name: feature-personas
description: Use when defining reusable tenant-level "user personas" (Skeptical Beginner, Power User, Enterprise Buyer) that shape how the agent talks to different user types. Attach at priming or per-chat.
---

# User personas

## What it is

A tenant-scoped library of named personas. Each persona has a `name` and a `style` field (free-form prompt-shaping instruction). When attached to a user (at priming time or per-chat), the agent adapts tone, vocabulary, and pacing accordingly.

## When to use

- Onboarding flows where new users get a "Beginner" persona until they level up
- B2C apps with distinct user types (consumer vs developer)
- A/B testing different tones for the same agent
- Multi-locale tone differences ("formal Japanese", "casual American English")

## When NOT to use

- Personality of the agent itself → use `personality_prompt` on agent creation
- One-off prompt overrides → `compiled_system_prompt` per chat
- Per-user fact storage → priming metadata or `custom_states`

## SDK surface

```python
# Mount: client.user_personas (top-level)

# Create a persona
persona = client.user_personas.create(
    name="Skeptical Beginner",
    style=(
        "User is new to the platform and somewhat wary. Use plain language, "
        "confirm actions before taking them, never assume knowledge of jargon. "
        "Default to brief responses; offer detail only on request."
    ),
)
print(persona.persona_id)

# List
personas = client.user_personas.list(limit=50)

# Get
p = client.user_personas.get(persona.persona_id)

# Delete
client.user_personas.delete(persona.persona_id)
```

```typescript
const persona = await client.userPersonas.create({
  name: "Power User",
  style: "User is technical and prefers terse, direct responses...",
});
```

## Attaching a persona

**At priming time:**

```python
client.priming.prime_user(
    agent_id=agent_id,
    user_id="user-123",
    metadata={"persona_id": persona.persona_id, "display_name": "Alex"},
    content_blocks=[...],
)
```

**Per chat:**

```python
client.agents.chat(
    agent_id,
    messages=[...],
    user_id="user-123",
    persona_id=persona.persona_id,           # verify exact param name in your SDK
)
```

The agent reads the persona's `style` and shapes output accordingly.

## Default persona

Each tenant can mark one persona as default — users without an attached persona inherit it. Verify the exact mechanism in your SDK / dashboard.

## Decisions linked

- `archetypes/companion.md` — onboarding patterns
- `archetypes/enterprise-assistant.md` — per-role personas (executive vs engineer)
- `features/priming.md` — attaching personas at signup

## Common gotchas

- **Tenant-scoped** — shared across projects in the tenant; not per-project
- **`style` is free-form** — keep it directive and specific; vague styles produce uneven results
- **Personas don't override the agent's `personality_prompt`** — they shape *how the agent talks to this user*, not what the agent is. Both apply.
- **Switching personas mid-conversation** — supported but may feel discontinuous to the user; better to switch at session boundaries
- **One default per tenant** — explicit per-call `persona_id` overrides default
- **Localization** — write the persona `style` in the user's language for best results
