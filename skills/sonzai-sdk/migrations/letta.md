---
name: migration-letta
description: Use when migrating from Letta (formerly MemGPT) to Sonzai. Letta core memory blocks → priming + personality_prompt; Letta tools → Sonzai custom tools; Letta passages → memory.bulk_create_facts.
---

# Migrating from Letta / MemGPT

## What Letta provides

Letta (formerly MemGPT) is a memory-augmented agent framework with explicit core memory blocks (human + persona), recall memory (chat history), and archival memory (vector store + passages). Tool-using agents with paging memory.

## Field mapping

| Letta concept | Sonzai equivalent |
|---|---|
| Letta agent | Sonzai agent (`agents.create`) |
| Letta persona block | `personality_prompt` |
| Letta human block | `priming.metadata` + content_blocks (per-user context) |
| Letta recall memory | Sonzai sessions (automatic history) |
| Letta archival memory (passages) | `agents.memory.bulk_create_facts` or `priming.batch_import` |
| Letta tools | `agents.create_custom_tool` |
| Letta function calling | Sonzai's built-in via `side_effects.external_tool_calls` |
| `letta.send_message(agent_id, message)` | `client.agents.chat(agent_id, messages=[...])` or session-based |
| `letta.list_agents()` | `client.agents.list()` |

## Migration order

1. **Replicate Letta personas as Sonzai agents.** One Letta persona → one Sonzai `agents.create(name, personality_prompt=...)`.
2. **Per-user data:** Letta's "human block" → `priming.prime_user` metadata + a content_block summarizing that user.
3. **Migrate archival passages:** for each user, dump Letta passages, insert into Sonzai as facts:
   ```python
   for user_id, passages in letta_archival_by_user.items():
       client.agents.memory.bulk_create_facts(
           agent_id=sonzai_agent_id,
           user_id=user_id,
           facts=[{"content": p.text} for p in passages],
       )
   ```
4. **Re-register tools:** Letta function signatures map to Sonzai custom-tool `parameters` JSON Schema:
   ```python
   for letta_tool in letta_tools:
       client.agents.create_custom_tool(
           agent_id=sonzai_agent_id,
           name=letta_tool.name,
           description=letta_tool.description,
           parameters=letta_tool.json_schema,
       )
   ```
5. **Switch send_message calls** to `agents.chat` or `session.turn`. Decommission Letta client.

## Code shape before → after

**Before (Letta):**
```python
from letta import create_client
letta = create_client()
agent = letta.create_agent(name="Compass", persona="A patient coach...", human="User: Alex")
response = letta.send_message(agent.id, "Hello").messages[-1].content
```

**After (Sonzai):**
```python
from sonzai import Sonzai
client = Sonzai()
agent = client.agents.create(name="Compass", personality_prompt="A patient coach...")
client.priming.prime_user(agent_id=agent.agent_id, user_id="alex", metadata={"display_name": "Alex"})
session = client.agents.sessions.start(agent.agent_id, user_id="alex", session_id="s1")
result = session.turn(messages=[{"role": "user", "content": "Hello"}])
```

## Gotchas specific to Letta

- **Letta's "human" block is self-described user info.** Sonzai uses `user_id` to scope state, not a free-text block. Move human-block content to priming metadata + content_blocks.
- **Letta function calling**: parameter schemas migrate 1:1 to Sonzai custom-tool `parameters`. Validate JSON Schema compatibility.
- **Letta passages**: stored with embeddings — Sonzai re-embeds on insert; performance should be comparable or better.
- **Memory paging (MemGPT's claim to fame)** — Sonzai handles context engine + retrieval differently. No need to replicate the paging mechanism; Sonzai's per-turn enriched context covers the use case.
- **Letta core memory edit tools** (`core_memory_append`, `core_memory_replace`) — translate to `agents.update_profile` (personality_prompt) or your app updating priming.
- **Letta server vs cloud** — if you're self-hosting Letta, you may need direct DB access to enumerate users/passages for migration.

## Verification

- Sample 20 conversations. Re-prime users in Sonzai with their archival passages. Send the same messages; compare responses.
- Memory recall test: insert known facts; query Sonzai; verify retrieval.

## Cross-references

- `features/priming.md` — human-block migration
- `features/custom-tools.md` — function call migration
- `archetypes/coach-therapist.md` / `archetypes/companion.md` — typical destinations
- `migrations/overview.md`
