---
name: feature-knowledge-base
description: Use when uploading documents, inserting structured facts, or searching the project-scoped knowledge base. Agent reads from KB during chat when knowledge_base capability is on.
---

# Knowledge base

## What it is

Project-scoped document + entity-graph + fact store. The agent reads from it during chat (when `knowledge_base=true`). Three ingestion paths: document upload (PDF, MD, TXT), structured fact insertion, agent-authored writes (`knowledge_base_write=true`).

## When to use

- Policy / runbook / FAQ docs that agents must reference
- Product knowledge for customer support
- Game lore for NPC archetype
- Structured org facts (entities + relationships)
- Migrating from a vector store

## When NOT to use

- Per-user data → `features/custom-states.md` or `features/inventory.md`
- Cross-project / tenant-wide data → `features/org-knowledge-base.md` (cascade scope)
- Conversation history → memory system; KB is for stable knowledge

## SDK surface

```python
# Mount: client.knowledge (top-level — project-scoped)

# Upload a document
with open("policies/security.pdf", "rb") as f:
    client.knowledge.upload_document(
        project_id="your-project-id",
        file=f,
        filename="security.pdf",
    )

# List documents
docs = client.knowledge.list_documents("your-project-id", limit=50)

# Delete
client.knowledge.delete_document("your-project-id", "doc-id")

# Get document details
client.knowledge.get_document("your-project-id", "doc-id")

# Insert structured facts (entity + relationships)
client.knowledge.insert_facts(
    "your-project-id",
    entities=[
        {"id": "agent-launch-2026", "type": "Event", "label": "Agent Launch", "properties": {...}},
    ],
    relationships=[
        {"from": "user-alice", "to": "agent-launch-2026", "type": "ORGANIZED"},
    ],
)

# List nodes (entity graph)
nodes = client.knowledge.list_nodes("your-project-id", type="Person", limit=100, offset=0)

# Get a specific node
client.knowledge.get_node("your-project-id", "node-id")

# Delete a node
client.knowledge.delete_node("your-project-id", "node-id")

# Semantic search
results = client.knowledge.search(
    "your-project-id",
    query="what is our code review policy?",
    limit=10,
)
for r in results.results:
    print(f"[{r.score:.2f}] {r.label} ({r.type})")

# Define a schema (for typed entities, e.g. backing inventory items)
client.knowledge.create_schema(
    project_id="your-project-id",
    entity_type="GameItem",
    properties={...},
)
```

```typescript
await client.knowledge.uploadDocument("project-id", "doc.pdf", fileData);
const results = await client.knowledge.search("project-id", { query: "...", limit: 10 });
```

```go
err := client.Knowledge.UploadDocument(ctx, "project-id", "doc.pdf", fileData)
```

## Agent capability requirements

```python
client.agents.update_capabilities(
    agent_id,
    knowledge_base=True,                  # agent reads KB
    knowledge_base_write=True,            # agent can author KB content (audited)
    knowledge_base_scope_mode="project_only",  # or cascade / org_only / union
)
```

## Schemas (typed entities)

KB schemas back both KB nodes and `features/inventory.md` items. Defining a schema once enables typed validation across both:

```python
client.knowledge.create_schema(
    project_id="your-project-id",
    entity_type="Person",
    properties={
        "name": {"type": "string", "required": True},
        "email": {"type": "string"},
        "role": {"type": "string"},
    },
)
```

## Decisions linked

- `decisions/sharedmemory-vs-wisdom.md` — KB is not user-attributed; for cross-user attributed facts use shared_memory
- `features/org-knowledge-base.md` — wider scope across all projects in a tenant
- `archetypes/enterprise-assistant.md`, `archetypes/customer-support.md` — primary KB users

## Common gotchas

- **Project-scoped by default** — to share across projects in a tenant, use org KB + `cascade` scope mode
- **Three ingestion paths** — document upload, structured facts, agent autonomous writes (when `knowledge_base_write=true`)
- **Audit trail** on all writes — review for compliance
- **Soft delete** — deleted documents/nodes are recoverable; hard-delete is admin-only
- **Quota** varies by tier; check before bulk imports
- **Schema changes** — adding properties is forward-compat; renaming requires migration
- **Search ranks by relevance score** — set a min threshold for your use case
- **Multi-language KB** — search works across languages but embedding quality varies; consider per-language KBs if you need precision
