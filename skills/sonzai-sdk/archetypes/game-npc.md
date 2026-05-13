---
name: archetype-game-npc
description: Use when building an in-game NPC (single character or cast) for a multiplayer game. NPCs have personalities that evolve per player, manage typed inventory items, carry per-player faction/quest state via custom states, react to backend events (level_up, boss_defeated), optionally engage in multi-NPC dialogue scenes.
---

# Game NPC archetype

A character living inside a game, talking to many players. NPCs have personality that drifts per player, hold a per-player inventory (weapons, currency, consumables), carry per-player state (faction, quest progress), and react to backend triggers ("the boss has been defeated").

## 1. When this archetype fits

**Strong signals:**
- NPC inside a game (mobile, console, web)
- Per-player game state matters (level, faction, quest flags, inventory)
- Backend triggers feed the NPC ("player just leveled up — react in character")
- Multi-NPC dialogue scenes (two NPCs banter while player watches)
- Game has shards / regions / dev/staging/prod that should isolate state

**Anti-signals:**
- Pure narrative companion (no game state) → `archetypes/companion.md`
- Routing by player personality → `archetypes/guide-router.md`
- Customer-facing support bot → `archetypes/customer-support.md`

## 2. Prescribed stack

| Decision | Value | Why |
|---|---|---|
| `memory_mode` | `async` | Game chat is interactive; TTFC matters |
| Runtime mode (Q8) | **B — full-chat with explicit sessions** | Per-session tool injection (swap toolsets between quests / scenes / dialogue trees) is the killer feature. `sessions.start` → `sessions.set_tools` → `sessions.turn` per dialogue step → `sessions.end` on scene change. A works but you lose explicit boundaries. See `decisions/runtime-mode.md`. |
| LLM provider (production) | **BYOK** or Custom LLM | Tight per-game model control; BYOK lets you pick the cheapest model that hits your tone target. Custom LLM if you fine-tuned on game-specific dialogue. See `decisions/byok-vs-customllm.md`. |
| Personality drift | on (default) | NPCs evolve with their assigned players — feels alive |
| `shared_memory` | off | Each player's relationship with the NPC is private |
| `wisdom` | on (default) | Cross-player patterns without identifying anyone |
| `inventory` | `on` | Items have schemas, quantities, properties |
| `knowledge_base` | optional (game lore) | If you upload lore docs, the NPC can reference them |
| `knowledge_base_scope_mode` | `project_only` | Lore is project-specific |
| Custom tools | yes (`spend_currency`, `give_item`, `award_xp`, etc.) | Backend integration for game mechanics |
| Custom states | per-player (`level`, `faction_standing`, `quest_flags`) | Per-player state |
| Events | `triggerBackendEvent` for game moments | NPC reactivity |
| Dialogue (multi-NPC) | optional | NPC-to-NPC scenes |
| Instances | one per shard/region/environment | State isolation between deploys |
| `web_search` | off | Lore leakage risk; KB or game-server is authority |
| `image_generation` | optional (tier-gated) | If NPCs send generated illustrations |

## 3. Required SDK functions (in order)

### Step 1 — Create the NPC agent

```python
from sonzai import Sonzai
client = Sonzai()

npc = client.agents.create(
    name="Grommash",
    personality_prompt=(
        "Grommash, an old orc shopkeeper in the trade district. Gruff, "
        "loyal to repeat customers, secretly soft for desperate adventurers. "
        "Speaks plainly; never spoils main-quest details. References player's "
        "faction standing — kind to allies, terse with rivals."
    ),
    language="en",
)
```

### Step 2 — Set capabilities

```python
client.agents.update_capabilities(
    npc.agent_id,
    memory_mode="async",
    inventory=True,
    remember_name=True,
    web_search=False,
)
```

### Step 3 — Create inventory schema in KB

Inventory items go through a KB schema. Define it once at deploy.

```python
# Example schema for game items
client.knowledge.create_schema(
    project_id="your-project-id",
    entity_type="GameItem",
    properties={
        "name": {"type": "string", "required": True},
        "rarity": {"type": "string", "enum": ["common", "rare", "epic", "legendary"]},
        "damage": {"type": "integer"},
        "durability": {"type": "integer"},
        "stack_size": {"type": "integer"},
    },
)
```

Verify the exact `create_schema` shape against your installed SDK — schema syntax may differ slightly. See `features/inventory.md` and `features/knowledge-base.md`.

### Step 4 — Create instances per shard / environment

```python
# At deploy time per region/shard
for shard in ["us-east", "eu-west", "ap-south"]:
    client.agents.instances.create(
        npc.agent_id,
        instance_id=shard,                  # YOUR choice; deterministic
        name=f"Grommash-{shard}",
    )
```

`default` instance always exists. Per-instance custom states isolate state between shards.

### Step 5 — Per-player chat handler (server-side from your game)

```python
def npc_say(player_id, shard, player_message):
    session = client.agents.sessions.start(
        npc.agent_id,
        user_id=player_id,
        session_id=f"npc-grommash-{player_id}",
        instance_id=shard,                  # state isolation
    )
    result = session.turn(
        messages=[{"role": "user", "content": player_message}],
    )
    # Dispatch tool calls (see Step 6)
    return result.response, result.side_effects
```

### Step 6 — Register and dispatch custom tools

```python
# Currency
client.agents.create_custom_tool(
    npc.agent_id,
    name="spend_currency",
    description="Deduct gold from the player's purse when they buy something.",
    parameters={
        "type": "object",
        "properties": {
            "amount": {"type": "integer", "minimum": 1},
            "reason": {"type": "string"},
        },
        "required": ["amount", "reason"],
    },
)

# Item give
client.agents.create_custom_tool(
    npc.agent_id,
    name="give_item",
    description="Add an item to the player's inventory.",
    parameters={
        "type": "object",
        "properties": {
            "item_type": {"type": "string"},
            "label": {"type": "string"},
            "properties": {"type": "object"},
            "quantity": {"type": "integer", "minimum": 1},
        },
        "required": ["item_type", "label"],
    },
)

# XP
client.agents.create_custom_tool(
    npc.agent_id,
    name="award_xp",
    description="Award experience to the player.",
    parameters={
        "type": "object",
        "properties": {"amount": {"type": "integer", "minimum": 1}},
        "required": ["amount"],
    },
)
```

Dispatch in your `npc_say` handler:

```python
for tool_call in (result.side_effects.external_tool_calls or []):
    if tool_call.name == "spend_currency":
        game_db.deduct_gold(player_id, tool_call.arguments["amount"])
    elif tool_call.name == "give_item":
        client.agents.inventory.create(
            npc.agent_id,
            user_id=player_id,
            item_type=tool_call.arguments["item_type"],
            label=tool_call.arguments["label"],
            properties=tool_call.arguments.get("properties", {}),
            instance_id=shard,
        )
    elif tool_call.name == "award_xp":
        game_db.add_xp(player_id, tool_call.arguments["amount"])
```

### Step 7 — Carry per-player game state

```python
# Set when faction standing changes (e.g. after a quest)
client.agents.custom_states.upsert(
    npc.agent_id,
    key="faction_standing",
    value={"alliance": 75, "horde": -20},
    scope="user",
    user_id=player_id,
    instance_id=shard,
)

# Set quest flags
client.agents.custom_states.upsert(
    npc.agent_id,
    key="quest_flags",
    value={"main_quest_act1": "complete", "side_quest_grommash": "started"},
    scope="user",
    user_id=player_id,
    instance_id=shard,
)
```

The NPC reads these automatically during context build — its responses adapt to faction/quest state without you injecting it into prompts.

### Step 8 — Backend-event triggers

```python
# Player leveled up — fire from your game server
client.agents.trigger_backend_event(
    npc.agent_id,
    user_id=player_id,
    event_type="level_up",
    event_description="Player just reached level 25",
    metadata={"new_level": "25", "shard": shard},
    instance_id=shard,
)

# Boss defeated
client.agents.trigger_backend_event(
    npc.agent_id,
    user_id=player_id,
    event_type="boss_defeated",
    event_description="Player just defeated the Dragon of Ember",
    metadata={"boss": "DragonOfEmber"},
    instance_id=shard,
)
```

The NPC's next conversation with this player includes the event as soft context — they'll naturally reference it.

### Step 9 — (Optional) Multi-NPC dialogue scene

```python
# Two NPCs converse autonomously while the player watches
result = client.agents.dialogue(
    npc.agent_id,
    user_id=player_id,
    other_agent_id=other_npc_id,
    turn_messages=[
        {"role": "user", "content": "[scene: marketplace, player observing]"},
    ],
    instance_id=shard,
)
# Verify exact dialogue API shape against your installed SDK
```

You orchestrate turns (each `dialogue()` call is one turn); both NPCs draw on their memory + personality + mood.

## 4. Archetype-specific wizard intake questions

After Q2 picks `game-npc`, the wizard asks:

1. **Single NPC or cast of N?** Affects whether you generate or hand-craft.
2. **Inventory schema?** Define item types and their properties (rarity, damage, durability, stack_size, etc.). Required before `inventory.create`.
3. **Per-player or shared NPC state?** Typically per-player (the standard pattern).
4. **Sharding strategy?** One global instance / per-region instances / dev+staging+prod instances. Use instances to isolate state.
5. **Events fired?** level_up / boss_defeated / quest_complete / player_offline / custom. List them.
6. **Multi-NPC dialogue scenes?** Off / yes (limited) / yes (frequent).
7. **Game lore in KB?** Off / project KB only / org-level lore shared across multiple games (rare).

## 5. Spec template fields

```markdown
## Agents (NPCs)

- Single NPC | cast of N
- Per-NPC: name, personality_prompt, agent_id strategy (deterministic uuid5)
- Capabilities (per NPC): memory_mode=async, inventory=true, web_search=false, knowledge_base=<true if lore>

## Inventory schema

- Item types and properties: <table>
- KB schema entity_type: GameItem (or your taxonomy)

## Per-player state model

- custom_states keys: level, faction_standing, quest_flags, ... (per game design)
- All scoped to (agent_id, user_id, instance_id)

## Instances (sharding)

- Strategy: one global | per-region | per-environment
- Instance IDs: <list>

## Custom tools

- spend_currency, give_item, award_xp, <plus your game's verbs>
- Each: parameters schema + backend handler

## Events

- Event types: level_up, boss_defeated, quest_complete, <custom>
- Trigger source: game server (which file / service)

## Multi-NPC dialogue (if applicable)

- Scenes: <list of scripted moments>
- Trigger source: game scripting layer
```

## 6. Plan template

```markdown
1. Define inventory KB schema. Verify: knowledge.list_schemas shows it.
2. Create NPC agent(s) with personality_prompts. Verify: agents.list shows them.
3. Set capabilities (async, inventory=true, no web_search). Verify get_capabilities.
4. Create instances per shard. Verify agents.instances.list returns expected.
5. Register custom tools (spend_currency, give_item, award_xp, custom). Verify each appears in get_capabilities().customTools.
6. Implement npc_say chat handler in game server with tool-call dispatch. Verify a "buy item" message triggers spend_currency + give_item.
7. Implement per-player state writes (faction, quest flags) tied to game-event hooks. Verify custom_states.get_by_key returns the right value.
8. Wire backend event triggers in game loop (level_up etc.). Verify next NPC chat references the event.
9. (Optional) Implement multi-NPC dialogue scenes via dialogue(). Verify both NPCs speak in character.
10. Per-shard integration test: player on us-east has faction=alliance; same player_id on eu-west has different state. Verify instance isolation.
11. Production checklist: API key in game-server secrets, never in client; rate-limit headroom for player population; BYOK if billing isolation matters.
```

## 7. Anti-patterns

| Anti-pattern | Why it's wrong | Fix |
|---|---|---|
| Using `custom_states` for inventory items | custom_states are schema-less primitives; inventory items need typed validation, queryable properties, KB integration | Use `agents.inventory.create` with a KB schema; reserve custom_states for scalar flags |
| Triggering backend events for every routine action | Event firehose; NPC chat gets noisy; quota burn | Reserve events for moments (level_up, boss kill, quest start/end). Routine actions go through tool calls or are inferred from inventory/state changes. |
| Sharing `instance_id` across shards | Player state bleeds across regions | Always pass instance_id matching the player's shard on every call |
| Exposing API key to game client | Full account access leaked to anyone who can sniff the binary | Game server holds the key; client talks only to game server, game server talks to Sonzai |
| Forgetting `instance_id` on chat (defaults to `default`) | All players' state lands in the `default` instance; sharding silently breaks | Pass `instance_id` explicitly on every chat / tool / state write |
| Enabling `web_search` on NPCs | Lore leakage risk (NPC could reference real-world unrelated facts) | Off; rely on KB + personality |
| Generating NPCs ad-hoc per player | Quota burn, fragmented retrieval, inconsistent character | Create NPCs once at deploy; player-specific behavior is via per-player overlay + custom_states |
| Per-player `agent_id` for the same NPC | Same character splits into N different agents; defeats per-player overlay | One agent_id per NPC; differentiate per-player via user_id + custom_states + instance_id |
| Hardcoding inventory item properties without a schema | No validation; "the red sword" can mean anything | Define KB schema first; `inventory.create` will validate properties |

## Cross-references

- `decisions/state-vs-inventory.md` — when to use which (the canonical primitive-vs-schema rule)
- `decisions/instances-vs-multitenant.md` — sharding via instances vs separate projects
- `decisions/memory-mode.md` — async is right for games
- `decisions/capabilities-matrix.md` — game NPC capability column
- `features/inventory.md` — full inventory surface + schema details
- `features/custom-states.md` — per-player flag storage
- `features/custom-tools.md` — tool registration + dispatch
- `features/events-and-dialogue.md` — backend event triggers + multi-NPC dialogue
- `features/instances.md` — per-shard isolation
- `features/knowledge-base.md` — inventory KB schema + lore docs
