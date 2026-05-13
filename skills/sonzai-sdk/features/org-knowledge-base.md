---
name: feature-org-knowledge-base
description: Use when you need knowledge shared across all projects in a tenant (org-wide policies, brand guidelines, multi-game lore). Agent reads via the knowledge_base_scope_mode capability.
---

# Organization knowledge base

## What it is

A tenant-wide KB scope **above** all your projects. Same shape as project KB (documents, entities, schemas) but visible across every project in the tenant when agents are configured with the right scope mode.

## When to use

- Tenant-wide policies (security, code of conduct, brand voice)
- Multi-product / multi-game shared lore
- Cross-project entity catalogs (people, organizations, products)
- Anything that should be authored once and read by many agents

## When NOT to use

- Project-specific docs → project KB (`features/knowledge-base.md`)
- Per-user data → `features/custom-states.md` or `features/inventory.md`
- Tenant-isolated B2B data → separate projects entirely (see `decisions/instances-vs-multitenant.md`)

## SDK surface

```python
# Create org-level entity (no project_id — org-scoped)
client.knowledge.create_org_node(
    type="Policy",
    label="Code review policy",
    properties={
        "approvers_required": 2,
        "blocking_for": ["security/*"],
        "auto_merge_after_hours": 24,
    },
)

# List org-level nodes (verify exact API path against your SDK)
org_nodes = client.knowledge.list_org_nodes(type="Policy", limit=50)

# Update / delete by node_id
client.knowledge.delete_org_node("node-id")
```

```typescript
await client.knowledge.createOrgNode({
  type: "Policy",
  label: "...",
  properties: { ... },
});
```

Verify exact org-KB surface against your installed SDK — APIs may evolve.

## Scope modes (configured per-agent)

```python
client.agents.update_capabilities(
    agent_id,
    knowledge_base=True,
    knowledge_base_scope_mode="cascade",  # see below
)
```

| Scope mode | Behavior |
|---|---|
| `project_only` (default) | Agent reads only its project's KB. Org KB ignored. |
| `cascade` | Reads both project + org. **Project wins on collision** (most common). |
| `org_only` | Reads only org KB. Project KB ignored. (Rare — useful for tenant-wide-only agents.) |
| `union` | Reads both, returns both even on collision. Conflict-prone; not recommended unless you want raw both-sides surfacing. |

**Recommended:** `cascade`. Project-specific values override org defaults, but org provides the baseline.

## Decisions linked

- `decisions/instances-vs-multitenant.md` — org KB ≠ multi-tenancy
- `archetypes/enterprise-assistant.md` — common use of `cascade` scope
- `features/knowledge-base.md` — project KB (the more common surface)

## Common gotchas

- **`cascade` is recommended** — gives you both fallback (org defaults) and override (project specifics).
- **`union` conflicts** — returning both sides for the same query confuses agents. Avoid unless you have a specific reason.
- **Org KB writes** are admin-typically — agents can read but rarely write at org scope.
- **Org KB doesn't break multi-tenancy** — but it does mean tenants see shared org facts. If tenants must not see anything from other tenants, use **separate projects with no org KB**.
- **Schema sharing** — schemas defined at org scope are usable by all projects in the tenant.
- **Quota** — org KB has its own quota separate from project KB.
- **Audit** — org KB writes are audited at org level; review separately from project audit logs.
