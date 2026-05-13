---
name: feature-multiplayer-memory
description: Use when understanding the broader cross-user / cross-agent memory model — inter-agent (KB across project agents), intra-agent (shared_memory across users on one agent), and the wisdom layer.
---

# Multiplayer memory

## What it is

Three independent axes for sharing knowledge across users and agents:

1. **Wisdom** (default-on) — k-anonymized cross-user patterns within one agent. No attribution.
2. **Shared memory** (opt-in via `shared_memory=true`) — attributed cross-user facts within one agent.
3. **Knowledge base** (opt-in via `knowledge_base=true`) — cross-agent shared content within one project (and optionally across projects via `cascade` scope).

These compose. A typical enterprise agent enables all three.

## When to use

- Enterprise / customer-support agents where team-shared context matters
- Multi-agent projects where one agent's learning informs another's
- Tenant-wide knowledge (cascade KB scope)

## When NOT to use

- 1:1 companion — everything per-user
- Multi-tenant SaaS — separate projects per tenant (never share across tenant boundaries)

## The decision tree

```
Sharing across USERS on the SAME agent?
├── Attributed ("Alice said this") → shared_memory (opt-in, audited, floor required)
└── Patterns only ("users tend to ask X") → wisdom (default-on, k-anonymized)

Sharing across AGENTS in the same project?
├── Structured content (docs, facts, entities) → knowledge_base (with knowledge_base_write for autonomous authoring)

Sharing across PROJECTS in the same tenant?
└── knowledge_base_scope_mode = cascade (project wins; org fills defaults)

Sharing across TENANTS?
└── DON'T. Use separate Sonzai projects per tenant.
```

## Capability composition

```python
# Full enterprise stack — all three on
client.agents.update_capabilities(
    agent_id,
    wisdom=True,                          # default on; required for shared_memory
    shared_memory=True,                   # attributed cross-user
    knowledge_base=True,                  # cross-agent within project
    knowledge_base_write=True,            # agent can author KB content
    knowledge_base_scope_mode="cascade",  # also read org-level
)
```

## Decisions linked

- `decisions/sharedmemory-vs-wisdom.md` — when to enable shared_memory
- `decisions/instances-vs-multitenant.md` — when separation matters more than sharing
- `features/shared-memory.md` — detailed surface for the attributed-cross-user axis
- `features/knowledge-base.md` + `features/org-knowledge-base.md` — KB axes
- `archetypes/enterprise-assistant.md` — combines all three

## Common gotchas

- **Preconditions:** `shared_memory` requires `wisdom`; `knowledge_base_write` requires `knowledge_base`.
- **Privacy floor required** for `shared_memory`.
- **KB write quotas** — autonomous KB writes use quota; monitor.
- **Soft-delete only** on KB; hard-delete is admin.
- **No cross-tenant** sharing — projects are the boundary.
- **Audit trails** on all of these — review per your compliance regime.
- **Wisdom is unattributed by design** — you can't query "who told me about pattern X" because the attribution was stripped at ingest.
