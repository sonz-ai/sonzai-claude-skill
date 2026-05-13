---
name: migration-mem0
description: Use when migrating from Mem0 (mem0.ai) to Sonzai. Maps mem0.add / mem0.search / mem0.get_all to Sonzai equivalents; bulk-imports historical facts.
---

# Migrating from Mem0

## What Mem0 provides

Mem0 is a memory layer for AI apps — facts stored per-user, semantic search, optional graph variant. Common in 2024-2025 LLM apps. Pythonic, hosted or self-hosted.

## Field mapping

| Mem0 concept | Sonzai equivalent |
|---|---|
| `mem0.add(messages, user_id=...)` | `agents.memory.bulk_create_facts(agent_id, user_id, facts=[...])` (manual) OR `priming.batch_import` (auto-extract) |
| `mem0.search(query, user_id=...)` | `agents.memory.search(agent_id, query, user_id=...)` |
| `mem0.get_all(user_id=...)` | `agents.memory.list(agent_id, user_id=...)` |
| `mem0.update(memory_id, data)` | `agents.memory.update_fact(agent_id, fact_id, ...)` |
| `mem0.delete(memory_id)` | `agents.memory.delete_fact(agent_id, fact_id)` |
| `mem0.history(memory_id)` | `agents.memory.get_fact_history(agent_id, fact_id)` |
| Memory metadata | `priming.metadata` or `custom_states` |
| Mem0 graph (entities + relations) | `client.knowledge.insert_facts(project_id, entities=[...], relationships=[...])` |

## Migration order (strangler)

1. **Wrap your Mem0 search calls** with a fallback: call Sonzai first, fall back to Mem0 if empty. Validate Sonzai retrieval quality in shadow mode.
2. **Stop writing new memories to Mem0**; write to Sonzai via `agents.memory.bulk_create_facts` (if facts are already extracted) or via `session.turn` (auto-extracts).
3. **Bulk-import historical Mem0 data**:

```python
import mem0
old_client = mem0.MemoryClient(api_key="...")

for user_id in your_user_list:
    facts = old_client.get_all(user_id=user_id)
    # Option A — facts are pre-extracted; insert directly
    client.agents.memory.bulk_create_facts(
        agent_id=sonzai_agent_id,
        user_id=user_id,
        facts=[{"content": f["text"]} for f in facts],
    )
    # Option B — facts are raw turns; let Sonzai re-extract
    client.priming.prime_user(
        agent_id=sonzai_agent_id,
        user_id=user_id,
        content_blocks=[
            {"type": "chat", "content": [...turns...]},
        ],
    )
```

4. **Switch reads** to Sonzai exclusively. Decommission Mem0 client.
5. **(Optional) Migrate graph data** to Sonzai KB via `client.knowledge.insert_facts`.

## Code shape before → after

**Before (Mem0):**
```python
import mem0
m = mem0.MemoryClient(api_key="...")
m.add(messages=[{"role": "user", "content": "I love espresso"}], user_id="u1")
results = m.search("favorite coffee", user_id="u1")
```

**After (Sonzai):**
```python
from sonzai import Sonzai
client = Sonzai()
# Single fact
client.agents.memory.bulk_create_facts(
    agent_id="agent-id",
    user_id="u1",
    facts=[{"content": "loves espresso"}],
)
results = client.agents.memory.search("agent-id", query="favorite coffee", user_id="u1")
```

## Gotchas specific to Mem0

- **Mem0 stores extracted text** — Sonzai stores typed facts. Re-extract if your Mem0 records are summaries (use priming with `content_blocks`); use `bulk_create_facts` if they're already atomic.
- **`memory_id` doesn't map directly** — Sonzai's `fact_id` is different. If your code references mem0 IDs, migrate those references.
- **Mem0 graph variant** — entities + relations land in Sonzai KB (`knowledge.insert_facts`), not in agent memory.
- **Mem0 v1 vs v2 API** — verify which version your code uses; field names differ slightly.
- **User_id scheme** — pass Mem0's `user_id` as Sonzai's `user_id` directly; both expect strings.
- **Self-hosted Mem0** — your facts may be in Qdrant / Chroma; you may need to query the vector store directly to enumerate users / facts for import.

## Verification

Before cutover:
- Sample 50 (query, user) pairs from production. Compare Sonzai retrieval results vs Mem0.
- Score-threshold expected — Sonzai may return more or fewer results depending on memory_mode and recall settings.
- For LoCoMo-style benchmarks Sonzai exceeds Mem0 on multi-hop questions specifically; pretest accordingly.

## Cross-references

- `features/priming.md` — bulk import surface
- `features/knowledge-base.md` — graph migration
- `migrations/overview.md` — common principles
- `existing-codebase-audit.md` — audit step 3 detects Mem0 usage
