---
name: feature-instances
description: Use when sharding an agent across regions, environments, or game servers. Instances share personality + global memory but isolate per-instance custom_states and inventory.
---

# Instances

## What it is

Per-agent isolation domains. Each agent has at least one instance (`default`). You can create additional instances for regions, environments (dev/staging/prod), game shards, A/B cells. Personality + global memory remain shared; **only `custom_states` and `inventory` isolate per instance**.

## When to use

- Multi-region deploys of the same agent (US-East, EU-West, AP-South)
- Dev / staging / prod separation for the same logical agent
- Game shards where per-shard player state must not bleed
- A/B test cells (same agent, different state populations)

## When NOT to use

- True multi-tenancy (Customer A vs Customer B) → **separate projects**. Instances share personality and global memory; tenants must not. See `decisions/instances-vs-multitenant.md`.
- One-user-per-instance fragmentation → don't; per-user state is automatic via `user_id`
- Per-customer personality customization → either separate agents or rely on per-user overlays

## SDK surface

```python
# Mount: client.agents.instances (per agent)

# Create
client.agents.instances.create(
    agent_id,
    instance_id="us-east",                # YOUR choice; deterministic
    name="Production US-East",
)

# List
instances = client.agents.instances.list(agent_id)
for inst in instances:
    print(inst.instance_id, inst.name)

# Reset (destructive — wipes per-instance state)
client.agents.instances.reset(agent_id, "us-east")

# Delete
client.agents.instances.delete(agent_id, "us-east")
```

```typescript
await client.agents.instances.create(agentId, { instanceId: "us-east", name: "..." });
```

## Using instances in chat / state calls

Pass `instance_id` on every call where state isolation matters:

```python
# Chat
client.agents.chat(agent_id, messages=[...], user_id="u1", instance_id="us-east")

# Session start (preferred)
session = client.agents.sessions.start(
    agent_id,
    user_id="u1",
    session_id="...",
    instance_id="us-east",
)

# Custom states
client.agents.custom_states.upsert(
    agent_id, key="...", value=..., scope="user",
    user_id="u1", instance_id="us-east",
)

# Inventory
client.agents.inventory.create(agent_id, user_id="u1", ..., instance_id="us-east")

# Events
client.agents.trigger_backend_event(agent_id, user_id="u1", event_type="...", instance_id="us-east")
```

If you omit `instance_id`, calls default to `default` instance.

## Decisions linked

- `decisions/instances-vs-multitenant.md` — when to use vs separate projects
- `archetypes/game-npc.md` — typical instance user (game shards)
- `archetypes/hybrid-custom.md` — multi-tenant SaaS clarification

## Common gotchas

- **Personality is global.** Same personality across all instances. If you want different personalities, use different agents.
- **Global memory is global.** Facts not user-scoped are shared across instances.
- **Capabilities are global.** `update_capabilities` applies to all instances.
- **Custom states + inventory are per-instance** when `instance_id` is set. Default is the `default` instance.
- **Resetting an instance is destructive** — wipes per-instance custom_states and inventory. Confirm with user before calling.
- **`default` always exists** — you don't create it.
- **`instance_id` collisions** — be deterministic with your IDs (e.g. `us-east`, `eu-west`, not random UUIDs); makes debugging easier.
- **Mixing instance_id and per-user scope** — both apply; the key is `(agent_id, user_id, instance_id)` for state. Don't conflate.
