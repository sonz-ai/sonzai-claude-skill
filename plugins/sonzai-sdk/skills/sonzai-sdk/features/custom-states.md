---
name: feature-custom-states
description: Use when storing primitives (counters, flags, scalar strings, small JSON) per-user or globally on an agent. Distinct from inventory (which uses schemas).
---

# Custom states

## What it is

Schema-less key-value storage scoped to an agent. Composite key: `(agent_id, scope, key, user_id?, instance_id?)`. Three content types: text / json / binary. Two scopes: global / user.

## When to use

- Per-user flags ("onboarded": true, "phase": "post-intake")
- Counters (energy, score, daily_login_streak)
- Small JSON blobs ({mbti: "INFJ", confidence: 0.87})
- Server-managed state the agent should be aware of

## When NOT to use

- Typed items with schema (use `features/inventory.md`)
- Cross-user attributed facts (use `features/shared-memory.md`)
- Large blobs (>100KB suggested ceiling — verify in your tier)
- Frequently-mutated counters with race conditions (consider per-call atomicity guarantees in your SDK version)

## SDK surface

```python
# Mount: client.agents.custom_states (Python) / customStates (TS)

# Create or upsert
client.agents.custom_states.upsert(
    agent_id,
    key="weekly_goals",
    value={"goals": ["ship feature X", "fix bug Y"], "set_at": "2026-05-13"},
    scope="user",                     # "user" | "global"
    content_type="json",              # "text" | "json" | "binary"
    user_id="user-123",
    instance_id="us-east",            # optional
)

# Get by composite key
state = client.agents.custom_states.get_by_key(
    agent_id,
    key="weekly_goals",
    scope="user",
    user_id="user-123",
)
print(state.value)

# List all states (with filters)
states = client.agents.custom_states.list(
    agent_id,
    scope="user",
    user_id="user-123",
    limit=50,
)

# Delete
client.agents.custom_states.delete_by_key(
    agent_id,
    key="weekly_goals",
    scope="user",
    user_id="user-123",
)
```

```typescript
await client.agents.customStates.upsert(agentId, {
  key: "weekly_goals",
  value: { goals: ["..."], set_at: "2026-05-13" },
  scope: "user",
  userId: "user-123",
});
```

## Scope semantics

| scope | Visible to | Use case |
|---|---|---|
| `global` | agent-wide, all users see same value | agent-level config, feature flags |
| `user` | one specific (agent, user) pair | per-user flags, counters, JSON blobs |

`scope="user"` + `user_id=X` means: this state is visible to **any agent** querying for user X — including other agents in the same project. Useful for the guide-router pattern where the guide writes assessment data that the specialist reads.

## Content types

- `text` — strings; agent can read in context
- `json` — structured data; agent reads as text representation
- `binary` — base64-encoded; opaque to agent (your backend interprets)

## Decisions linked

- `decisions/state-vs-inventory.md` — primitives vs schema'd items
- `archetypes/guide-router.md` — carries `mbti_assessment` here
- `archetypes/game-npc.md` — faction standing, quest flags

## Common gotchas

- **Composite key includes `user_id`** — same `key` with different `user_id` is a different record
- **`upsert` is idempotent** — use for setters; safe to call repeatedly
- **Binary content base64-encoded** — encode/decode on your side
- **JSON values are stored as JSON strings server-side** — the SDK handles serialization but very deeply nested objects may serialize sub-optimally; flatten if possible
- **State changes don't trigger memory consolidation** — they're not facts; they're metadata. If you want the agent to "remember" the state as a fact, write to memory separately.
- **Atomic updates** — concurrent `upsert` calls overwrite. If you need CAS semantics, implement at your app layer.
- **`instance_id`** for game shards / regional isolation; defaults to `default` if omitted.
