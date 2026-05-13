---
name: feature-inventory
description: Use when storing per-user typed items (game items, medications, holdings, pets) with schema validation, queryability, and KB integration. Distinct from custom_states (primitives).
---

# Inventory

## What it is

Per-user typed item storage on an agent. Items have a schema (defined in the KB), validation, properties, and queryable interface. The agent reads inventory automatically during context build — references items naturally in chat without you prompting.

## When to use

- Game items (weapons, currency, consumables)
- Medications, holdings, pets, vehicles — anything with structured properties
- User collections that the agent should reference
- Items that need disambiguation ("the red sword" vs "the heavy one")

## When NOT to use

- Primitives (counters, flags, scalar strings) → `features/custom-states.md`
- Cross-user shared resources → KB or `features/shared-memory.md`
- Ephemeral session state → in-memory or `custom_states`

See `decisions/state-vs-inventory.md` for the canonical split.

## SDK surface

```python
# Mount: client.agents.inventory (per-user when user_id is passed)

# 1. Define the KB schema (once, at deploy)
client.knowledge.create_schema(
    project_id="your-project-id",
    entity_type="GameItem",
    properties={
        "name": {"type": "string", "required": True},
        "rarity": {"type": "string", "enum": ["common", "rare", "epic", "legendary"]},
        "damage": {"type": "integer"},
        "durability": {"type": "integer"},
    },
)
# Verify exact create_schema shape in your SDK version

# 2. Create an item (agent + user scoped)
client.agents.inventory.create(
    agent_id,
    user_id="player-123",
    item_type="GameItem",
    label="Sword of Embers",
    properties={"rarity": "epic", "damage": 50, "durability": 100},
    instance_id="us-east",          # optional — for sharded games
)

# 3. Update an item (action-based — e.g. consume, transfer, repair)
client.agents.inventory.update(
    agent_id,
    user_id="player-123",
    action="consume",                # e.g. consume / drop / transfer
    item_type="Potion",
    properties={"quantity": 1},
)

# 4. Query inventory
results = client.agents.inventory.query(
    agent_id,
    user_id="player-123",
    item_type="GameItem",
    query="legendary",               # natural-language or filter
    sort_by="rarity",
    sort_order="desc",
)
for item in results.items:
    print(item.label, item.properties)

# 5. Bulk import (up to 1000 items)
client.agents.inventory.batch_import(
    agent_id,
    user_id="player-123",
    items=[
        {"item_type": "GameItem", "label": "Sword", "properties": {...}},
        # ...
    ],
)

# 6. Direct update / delete by fact_id
client.agents.inventory.direct_update(
    agent_id,
    user_id="player-123",
    fact_id="...",
    properties={"durability": 50},
)
client.agents.inventory.direct_delete(agent_id, user_id="player-123", fact_id="...")
```

```typescript
// Mount: client.agents.inventory
await client.agents.inventory.create(agentId, "player-123", {
  itemType: "GameItem",
  label: "Sword of Embers",
  properties: { rarity: "epic", damage: 50 },
  instanceId: "us-east",
});
```

```go
err := client.Agents.Inventory.Create(ctx, agentID, "player-123", sonzai.InventoryCreateOptions{
    ItemType:   "GameItem",
    Label:      "Sword of Embers",
    Properties: map[string]any{"rarity": "epic", "damage": 50},
})
```

## Disambiguation

When the user references an item ambiguously ("the red one"), the query API may return a `disambiguation_needed` shape rather than a single result. Your code surfaces options to the user:

```python
result = client.agents.inventory.query(
    agent_id, user_id="player-123", query="the red one",
)
if result.disambiguation_needed:
    # present result.candidates to the user, get their pick
    ...
```

## Decisions linked

- `decisions/state-vs-inventory.md` — when to use which
- `decisions/capabilities-matrix.md` — `inventory` capability flag required

## Common gotchas

- **KB schema must exist BEFORE `inventory.create`** — define entity types in the KB first.
- **`inventory` capability must be on** — `client.agents.update_capabilities(agent_id, inventory=True)`.
- **Per-user scope** — items are keyed by `(agent_id, user_id, instance_id)`. Sharing requires explicit promotion (e.g. to KB).
- **Schema evolution** — adding a property is forward-compatible; removing or renaming requires a migration plan.
- **`instance_id`** isolates per game shard. Don't share across shards.
- **Soft delete** — deleted items are recoverable for a retention window; hard-delete via admin.
- **Quantity stacking** — if your items are stackable, model `quantity` in properties; use `action="consume"` to decrement.
