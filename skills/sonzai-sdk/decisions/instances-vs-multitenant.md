---
name: decision-instances-vs-multitenant
description: Use when deciding whether to use instances (sharded deploys of the same agent) or separate projects (true multi-tenancy) for an app serving multiple distinct customers or regions.
---

# Decision: instances vs multi-tenant (separate projects)

## The rule

**Instances** for sharded deploys of the **same logical agent**: regions, dev/staging/prod, game shards, A/B test cells.

**Separate Sonzai projects** for **separate customers / true multi-tenancy**: no cross-tenant bleed at the API layer.

## How to apply

| Goal | Use |
|---|---|
| Same agent in multiple regions; share personality + global memory; isolate per-region custom_states | **Instances** (`agents.instances.create(agent_id, instance_id="us-east")`) |
| Game with multiple shards; per-shard inventory/state per player | Instances |
| Dev vs staging vs prod environments of the same agent | Instances |
| A/B test variants of the same agent (same personality, different capability mix) | Instances + capability config per instance (note: capabilities are agent-wide, not per-instance — A/B requires either separate agents or runtime branching) |
| B2B SaaS where Customer A and Customer B must never share data | **Separate projects** (one Sonzai project per customer, separate API key per project) |
| Per-customer customized agent (different personality per tenant) | Separate projects (each project has its own agent with tenant-specific config) |

## Why

- **Instances** share the agent's personality, global memory, and capabilities. Only `custom_states` and `inventory` are per-instance. Personality and per-user memory remain global to the agent. This is right for "same agent, different deployment context."

- **Projects** are the strict tenancy boundary. Separate projects mean: separate API keys, separate KB, separate agent configurations, separate billing scope, no cross-project data visibility. This is the only correct approach for B2B SaaS where customer data must be isolated.

**The misconception trap:** "I'll use instances for multi-tenancy because it's simpler." This bleeds personality and shared memory across tenants. It also burns your project's KB / quota / rate limit on all tenants combined.

## Exceptions

- **B2B SaaS where tenants are allowed to share KB / patterns:** instances might work, but typically you still want separate projects for billing and rate-limit isolation.
- **Multi-region same-tenant** (one customer running their agent in multiple regions): instances is right.
- **Per-customer personality with shared base:** technically possible with instances + custom prompts injected via `compiled_system_prompt` per chat, but fragile — use separate projects for cleaner separation.

## Resource overhead

- **Instances:** very cheap. The `default` instance always exists. Per-instance state is small.
- **Separate projects:** each project gets its own API key, KB quota, rate limits. More setup overhead but clean isolation.

## Cross-references

- `features/instances.md` — instances surface (create, list, reset, delete)
- `archetypes/game-npc.md` — uses instances for game shards
- `archetypes/enterprise-assistant.md` — typical single-project (one team)
- `archetypes/customer-support.md` — multi-tenant variant uses separate projects per business customer
- `archetypes/hybrid-custom.md` — multi-tenant SaaS is the canonical hybrid case
- `features/projects.md` — project management (if you need to programmatically create projects per tenant)
