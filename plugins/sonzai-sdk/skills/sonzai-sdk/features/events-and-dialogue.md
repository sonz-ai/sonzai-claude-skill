---
name: feature-events-and-dialogue
description: Use when backend pushes events to an agent (level_up, order_shipped) or orchestrating multi-agent dialogue scenes (two NPCs converse autonomously).
---

# Events & dialogue

## What it is

Two distinct but related surfaces:

- **`trigger_backend_event`** — server-pushed events. Agent reacts in its next conversation. Soft context (informational, not directive).
- **`dialogue`** — agent-to-agent turns. Two agents converse; you orchestrate the turn-taking.

## When to use

### Events
- Game moments: `level_up`, `boss_defeated`, `quest_complete`
- App lifecycle: `subscription_upgraded`, `order_shipped`, `password_reset`
- External triggers: `slack_message_received`, `pr_merged`
- Anything happening outside the chat that the agent should know about

### Dialogue
- Two NPCs banter in a game (player observes)
- Multi-agent specialists collaborating (rare; advanced)
- Scripted scenes that need agent-driven dialogue

## When NOT to use

- Events for routine actions (firehose risk) — use only for moments
- Dialogue for routing (user → guide → specialist) — that's not dialogue; it's distinct sessions on distinct agents

## SDK surface

### Events

```python
client.agents.trigger_backend_event(
    agent_id,
    user_id="player-123",
    event_type="level_up",
    event_description="Player just reached level 25",
    metadata={"new_level": "25", "previous_level": "24"},
    language="en",                       # optional
    instance_id="us-east",               # optional
    messages=[                           # optional — pre-existing chat context
        {"role": "user", "content": "..."},
    ],
)
```

Returns a `TriggerEventResponse`. The event is queued; the agent's **next conversation** with this user includes the event as soft context. Agent naturally references it.

For immediate notification (not just on next chat), pair with proactive delivery (see `features/proactive.md`).

### Dialogue

```python
# Verify exact API shape in your SDK — dialogue surface may evolve
result = client.agents.dialogue(
    agent_id=npc1_id,
    user_id="player-123",
    other_agent_id=npc2_id,
    turn_messages=[
        {"role": "user", "content": "[scene: marketplace, player observing]"},
    ],
    instance_id="us-east",
)
print(result.response)                  # NPC1's response
# Then call dialogue() on npc2 with NPC1's response, alternating
```

Dialogue is **one turn per call**. You orchestrate turn order.

## Decisions linked

- `archetypes/game-npc.md` — primary user of both
- `features/proactive.md` — events as part of the proactive stack
- `decisions/proactive-channel.md` — how the agent's reaction reaches the user

## Common gotchas

- **Events are soft context.** The agent will reference them but won't force a response. If you need an immediate notification, fire a wakeup OR use SSE inline to deliver a generated message.
- **Don't firehose events.** Routine actions don't need events; reserve for moments.
- **Event metadata is informational** — agent reads but doesn't programmatically act on it. For action, use custom tools.
- **Dialogue requires both agents in the same project.** Cross-project dialogue isn't supported.
- **Dialogue draws on each agent's memory + personality + mood independently.** Both will respond in their own voice.
- **Scene-setting via `messages`** — preface dialogue with a [scene] description; agents will pick up tone.
- **Event ordering** — events fire in queue order; if a player triggers 3 events in rapid succession, the agent sees them in that order on the next chat.
- **Instance-aware** — pass `instance_id` to keep events shard-isolated.
