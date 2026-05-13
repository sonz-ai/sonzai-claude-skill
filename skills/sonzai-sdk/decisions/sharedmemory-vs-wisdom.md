---
name: decision-sharedmemory-vs-wisdom
description: Use when picking between wisdom (default-on, k-anonymized) and shared_memory (opt-in, attributed) for cross-user learning on a multi-user agent.
---

# Decision: shared_memory vs wisdom

## The rule

**Wisdom is default-on, k-anonymized, attribution-stripped.** Cross-user patterns without identifying anyone.

**`shared_memory` is opt-in, attributed, requires the privacy floor configured.** Cross-user **facts** with attribution ("Alice owns the auth domain").

## How to apply

| Goal | Setting |
|---|---|
| Agent learns general patterns across users without identifying anyone (e.g. "most users find onboarding confusing") | `wisdom=on` (default), `shared_memory=off` |
| Agent learns attributed cross-user facts (team coordinator: "Alice owns auth, Bob owns billing") | `wisdom=on` (precondition), `shared_memory=on`, **privacy floor configured** |
| Strict per-user privacy with no cross-user inference whatsoever | `wisdom=off`, `shared_memory=off` (rare; loses platform value) |

Set via `agents.update_capabilities`:

```python
client.agents.update_capabilities(
    agent_id,
    wisdom=True,            # precondition; default is on
    shared_memory=True,     # opt-in; only enable with privacy floor
)
```

`shared_memory=true` without `wisdom=true` is rejected server-side.

## Why

- **Wisdom** uses k-anonymized patterns — the agent learns "users in this category tend to X" without surfacing any single user's identity. Compliance-friendly by default.
- **Shared memory** lets the agent surface specific cross-user facts with attribution. Powerful for team coordination, customer support institutional memory, and B2B internal bots. But it raises a privacy bar: the server-side **privacy floor** must be configured to refuse surfacing protected categories (compensation, health, performance, PII).
- The **disclosure audit** logs every surfaced cross-user fact — review it for compliance evidence.

## Exceptions

- **B2B compliance regimes** that prohibit any cross-user attribution: both `wisdom=off`, `shared_memory=off`. Loses cross-user pattern learning entirely.
- **Solo apps** (one user per agent / 1:1 companion): both are irrelevant; leave defaults.
- **Multi-tenant SaaS** where tenants must be fully isolated: never share `shared_memory` across tenants — use separate Sonzai projects per tenant. See `decisions/instances-vs-multitenant.md`.

## Cross-references

- `features/shared-memory.md` — privacy floor configuration, disclosure audit
- `features/multiplayer-memory.md` — the broader cross-user / cross-agent learning model
- `archetypes/enterprise-assistant.md` — shared_memory=on is the prescribed default
- `archetypes/customer-support.md` — shared_memory=on for CS team learning
- `archetypes/companion.md`, `archetypes/coach-therapist.md`, `archetypes/game-npc.md` — shared_memory=off
- `decisions/capabilities-matrix.md` — the full grid
