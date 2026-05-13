---
name: decision-state-vs-inventory
description: Use when choosing between custom_states and inventory for storing per-user data on an agent.
---

# Decision: custom_states vs inventory

## The rule

**Use `custom_states` for primitives** (counters, flags, scalar strings, small JSON blobs).
**Use `inventory` for items with schema, identity, and multiple typed properties.**

## How to apply

| Data | Use | Why |
|---|---|---|
| `energy = 42` | custom_states | scalar |
| `mbti_result = {"type": "INFJ", "confidence": 0.87}` | custom_states | small JSON, no per-record identity |
| `faction_standing = {"alliance": 75, "horde": -20}` | custom_states | small JSON |
| `quest_flags = {"main_act1": "complete", ...}` | custom_states | flag bag |
| Sword `{name, damage, durability, owner_id}` | inventory | typed item with schema and identity |
| Medication `{name, dosage_mg, schedule, last_taken_at}` | inventory | typed item, queryable |
| Pets `{species, name, level, mood}` (collection per user) | inventory | typed collection |
| Goals `{title, status, due_date}` | depends — inventory if you want disambiguation ("the marketing goal") + audit; custom_states if just a list of strings |

## Why

**Inventory:**
- KB schema validates properties at insert time (`agents.inventory.create` rejects malformed)
- Items are queryable (`agents.inventory.query` with filters, sort, aggregations)
- Supports **disambiguation** when the user says "the red one" — `disambiguation_needed` response shape
- Auto-summarized in chat context (the agent knows the inventory without you prompting it)
- Carries an audit trail; soft-delete
- KB integration — items can link to knowledge entities (`kb_node_id`)

**custom_states:**
- No schema, no validation
- Composite key: `(agent_id, scope, key, user_id?, instance_id?)`
- Lower overhead; great for flags and counters
- Two scopes (`global` agent-wide vs `user` per-user)
- Three content types (`text`, `json`, `binary`)
- Read with `get_by_key`; bulk read with `list`

## Exceptions

- **If your "item" is genuinely scalar** (one property like `level=15`), custom_states is fine even if you'd loosely call it an item.
- **If you want fuzzy match / disambiguation** (user says "the heavy one"), inventory wins regardless of complexity.
- **High-frequency mutation** (counter incremented per turn) → custom_states (lower overhead than inventory.create + update).
- **Need cross-agent visibility within the same project** → inventory items are agent-scoped; for cross-agent shared data, use the project KB instead.

## Cross-references

- `features/custom-states.md` — full custom_states surface (create / upsert / get_by_key / list / delete_by_key)
- `features/inventory.md` — full inventory surface (create / update / query / batch_import) + KB schema
- `features/knowledge-base.md` — KB schemas back inventory item types
- `archetypes/game-npc.md` — uses both (inventory for items, custom_states for flags)
- `archetypes/guide-router.md` — uses custom_states for assessment result
