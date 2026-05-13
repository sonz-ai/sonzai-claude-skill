---
name: migration-zep
description: Use when migrating from Zep to Sonzai. Zep memory + sessions + summaries map to Sonzai's priming + sessions + auto-diary.
---

# Migrating from Zep

## What Zep provides

Zep is a memory + session management layer for LLM apps. Stores conversation history per session, generates summaries per session, supports vector search across history. Session-first model.

## Field mapping

| Zep concept | Sonzai equivalent |
|---|---|
| Zep session | Sonzai session (`agents.sessions.start`) |
| Zep memory (per session) | Automatic via Sonzai sessions |
| Zep summaries | Sonzai diary entries (`agents.get_diary`) — auto-generated |
| Zep facts | Sonzai facts (`agents.memory.bulk_create_facts`) |
| `zep.memory.add(session_id, messages)` | `session.turn(messages=[...])` |
| `zep.memory.search(session_id, query)` | `session.context(query=...)` or `agents.memory.search` |
| Zep user metadata | `priming.metadata` |
| `zep.user.get(user_id)` | (not a 1:1 — Sonzai infers per-user state from priming + interactions) |

## Migration order

1. **Export Zep data** — per user: sessions, summaries, facts. Zep CLI or API.
2. **Create Sonzai agent(s)** to mirror Zep's per-app persona (often one agent serves all users).
3. **Prime users with metadata** — Zep user.metadata → `priming.metadata`.
4. **Bulk import historical sessions** — for each user, prime with a chat content_block per session:
   ```python
   for user_id, sessions in zep_sessions_by_user.items():
       content_blocks = []
       for session in sessions:
           content_blocks.append({
               "type": "chat",
               "content": [{"role": m.role, "content": m.content} for m in session.messages],
           })
       client.priming.prime_user(
           agent_id=sonzai_agent_id,
           user_id=user_id,
           metadata=zep_user_metadata[user_id],
           content_blocks=content_blocks,
       )
   ```
5. **Switch session/memory calls** to Sonzai. Match Zep `session_id` to Sonzai `session_id` for continuity.
6. **Decommission Zep client** after parallel-run validation.

## Code shape before → after

**Before (Zep):**
```python
from zep_python import ZepClient
zep = ZepClient(api_key="...")
zep.memory.add_memory(session_id="s1", messages=[Message(role="user", content="Hi")])
recent = zep.memory.get_memory(session_id="s1")
```

**After (Sonzai):**
```python
from sonzai import Sonzai
client = Sonzai()
session = client.agents.sessions.start(agent_id="a1", user_id="u1", session_id="s1")
result = session.turn(messages=[{"role": "user", "content": "Hi"}])
# Recent context is automatic via session.context() or just trust the turn loop
```

## Gotchas specific to Zep

- **Zep summarizes per-session**; Sonzai writes a diary entry per session via the self-improvement pipeline. Surface diaries via `agents.get_diary`.
- **Zep facts vs Zep messages** — Zep has both. Map facts to `memory.bulk_create_facts`; map messages to `priming` content_blocks.
- **Session ID continuity** — pass the Zep `session_id` as Sonzai `session_id`. They're both your choice; preserve for traceability.
- **Zep cloud vs self-hosted** — if self-hosting, you'll dump from Postgres or your storage layer.
- **Long-running sessions** — Zep allows very long sessions; Sonzai sessions are usually shorter (per-conversation). Consider splitting Zep super-sessions into multiple Sonzai sessions if natural breakpoints exist.

## Verification

- Sample 20 user histories. Prime, then ask Sonzai about facts that were in the Zep history. Verify recall quality.
- Compare session summarization quality (Zep summary vs Sonzai diary entry).

## Cross-references

- `features/priming.md`
- `features/agent-insights.md` — diary
- `migrations/overview.md`
