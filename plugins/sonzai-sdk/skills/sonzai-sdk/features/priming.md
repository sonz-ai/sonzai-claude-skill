---
name: feature-priming
description: Use when bootstrapping a user's context on an agent at signup, migration, or onboarding — display name, metadata, prior chat transcripts, bulk CSV imports. Async jobs; poll for completion.
---

# Priming

## What it is

The data-loading surface for users. Bootstraps an agent with a user's context **before their first conversation**: display name, metadata fields, content blocks (free-text or chat transcripts). Bulk imports run async — you poll `get_import_status`.

## When to use

- New signup: load display_name, company, role, intake results
- Migration from another system (Mem0, Zep, Letta, OpenAI Assistants, Character.AI): bulk-load existing user history
- CSV import: convert spreadsheet rows to primed users
- Coach archetype: seed the agent with intake assessment results

## When NOT to use

- Mid-session memory writes — use `agents.memory.bulk_create_facts` or auto-extraction via `session.turn`
- Per-turn state changes — use `custom_states`
- Real-time updates — priming is for bootstrap, not continuous sync

## SDK surface

```python
# Mount: client.priming (top-level)

# Prime one user
client.priming.prime_user(
    agent_id=agent_id,
    user_id="user-123",
    metadata={
        "display_name": "Alex",
        "company": "Acme Co",
        "role": "engineer",
        "intake_phq9": "8",
    },
    content_blocks=[
        {"type": "text", "content": "User joined on 2026-01-15. Stated goal: learn Go."},
        {
            "type": "chat",
            "content": [
                {"role": "user", "content": "I want to learn Go."},
                {"role": "assistant", "content": "Great! What's drawing you to Go?"},
                # ... more turns from prior system
            ],
        },
    ],
)

# Bulk import (CSV / migration)
import_ref = client.priming.batch_import(
    agent_id=agent_id,
    users=[
        {"user_id": "user-1", "metadata": {...}, "content_blocks": [...]},
        {"user_id": "user-2", "metadata": {...}, "content_blocks": [...]},
        # up to 1000 users per batch (verify in your SDK)
    ],
)
print(import_ref.import_id)

# Poll status
import time
while True:
    status = client.priming.get_import_status(import_ref.import_id)
    if status.state in ("completed", "failed"):
        break
    time.sleep(5)
print(status.completed_count, status.failed_count)
```

```typescript
await client.priming.primeUser({
  agentId, userId: "user-123",
  metadata: { displayName: "Alex", ... },
  contentBlocks: [{ type: "text", content: "..." }],
});

const ref = await client.priming.batchImport({ agentId, users: [...] });
const status = await client.priming.getImportStatus(ref.importId);
```

## Content block types

| type | Content shape | Use case |
|---|---|---|
| `text` | string | summarized history, intake notes, stated goals |
| `chat` | array of `{role, content}` turns | prior chat history from old system |
| (verify in your SDK for additional types) | | |

## Migration patterns

For each migration source, `migrations/{source}.md` describes the field mapping and ingestion shape. Common pattern:

```python
# Pseudo: Mem0 → Sonzai migration
mem0_facts = mem0.get_all(user_id)
client.priming.prime_user(
    agent_id=agent_id,
    user_id=user_id,
    metadata={"display_name": old_user["name"]},
    content_blocks=[
        {"type": "text", "content": "\n".join(f.text for f in mem0_facts)},
    ],
)
```

For fact-only imports (no transcripts), `agents.memory.bulk_create_facts` is often more direct.

## Decisions linked

- All `migrations/*.md` use priming
- `features/personas.md` — personas attach at priming time
- `archetypes/coach-therapist.md` — primes with intake survey
- `archetypes/enterprise-assistant.md` — primes with employee directory

## Common gotchas

- **Async jobs** — `batch_import` returns immediately with `import_id`; poll for completion. Don't assume sync semantics.
- **Dedup across blocks** — server-side dedup happens but may not be perfect. Consider one canonical pass over your source.
- **Metadata vs content_blocks** — metadata is structured KV (queryable, indexed); content_blocks is free-text or chat (extracted later by the model)
- **Idempotency** — re-priming the same `user_id` updates; doesn't error. Safe to retry.
- **Quota** — large bulk imports may exceed your tier's per-day quota; chunk if needed
- **Encoding** — UTF-8 everywhere; CSVs with BOM or weird encodings cause silent failures
- **Personas** — pass `persona_id` to attach a tenant-level persona at priming (see `features/personas.md`)
