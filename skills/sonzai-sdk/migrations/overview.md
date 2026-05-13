---
name: migration-overview
description: Use as the entry point when migrating an existing AI app to Sonzai. Routes to source-specific playbooks (Mem0, LangChain, Letta, Zep, OpenAI Assistants, Character.AI, CRM CSV, raw JSON) and explains the strangler-pattern common to all.
---

# Migration overview

Entry point for any migration. After `existing-codebase-audit.md` identifies the incumbent system, this file routes you to the matching playbook and explains the common principles.

## Source → playbook

| Detected incumbent | Playbook |
|---|---|
| Mem0 (`mem0` import / `MemoryClient`) | `migrations/mem0.md` |
| LangChain (`ConversationBufferMemory`, `VectorStoreRetrieverMemory`, `langchain.chains`) | `migrations/langchain.md` |
| Letta / MemGPT | `migrations/letta.md` |
| Zep | `migrations/zep.md` |
| OpenAI Assistants API | `migrations/openai-assistants.md` |
| Character.AI export | `migrations/character-ai.md` |
| CSV / CRM bulk data | `migrations/crm-csv.md` |
| Custom JSON / unknown shape | `migrations/raw-json.md` |

## Principles common to all migrations

### 1. Strangler pattern (always)

Replace one layer at a time. Each step is independently shippable and rollback-able.

| Order | Layer | Why first |
|---|---|---|
| 1 | Memory (read path) | Read your existing system, also read Sonzai. Compare retrieval quality on a fixed query set. Switch reads to Sonzai when parity confirmed. |
| 2 | Memory (write path) | Stop writing to the old system; write to Sonzai via `memory.bulk_create_facts` or `priming.batch_import`. Backfill historical data. |
| 3 | Personality / system prompts | Move into `agents.create(..., personality_prompt=...)` or `generate_and_create(..., description=...)`. |
| 4 | LLM provider | Switch from your direct provider call to `agents.chat` / `agents.sessions`. BYOK if billing isolation matters. |
| 5 | KB / RAG | Migrate vector store contents to `client.knowledge`. Parallel-run until parity. |
| 6 | Proactive (cron jobs) | Move to `client.schedules.create` / `agents.scheduleWakeup`. |
| 7 | Webhooks (if any) | Last — register Sonzai webhooks to fan out to your existing handlers. |

### 2. Parallel-run validation

Don't cut over blind. Run old + new in parallel:

- Old system handles production traffic
- New system shadows requests (read-only)
- Compare outputs on a sample set
- Cut over when parity is acceptable

### 3. Bulk-import for historical data

Use `priming.batch_import` for users with prior history. Verify with `priming.get_import_status` before declaring done. Idempotent on `user_id` — safe to retry.

### 4. Rollback plan

For each step, document how to revert. Common pattern:

- Feature flag the new code path
- Keep old system warm for N days post-cutover
- Monitor user-facing metrics; rollback if degraded

### 5. Data shape compatibility

Sonzai stores **facts** (extracted from text or directly inserted). Old systems may store:
- Raw transcripts → use `priming` content_blocks (type=chat) — agent extracts facts
- Summarized memory → use `priming` content_blocks (type=text) — agent re-extracts
- Pre-extracted facts → use `agents.memory.bulk_create_facts(... source_type="manual")` — no extraction

### 6. Personality continuity

Existing system prompts → Sonzai `personality_prompt`. If you have natural-language descriptions, use `generate_and_create` to auto-derive Big5. If you have specific traits, use `agents.create` with explicit Big5.

### 7. Audit & compliance

If your incumbent has audit obligations (HIPAA, SOC2), confirm Sonzai's audit trail covers them before cutover:
- `agent.message.created` webhook for chat audit
- `disclosure_audit` for shared_memory surfacing
- KB write audit on `knowledge_base_write`

## Common gotchas across all migrations

- **User ID continuity** — preserve your existing user identifiers when bulk-importing. Pass them as `user_id` to Sonzai. Don't mint new IDs unless you have a migration table.
- **Encoding** — UTF-8 everywhere. CSV files with BOM, latin-1, or weird encodings cause silent failures.
- **Quotas** — bulk imports may exceed per-day quotas; chunk if needed.
- **Idempotency** — re-runs should be safe. Use deterministic `user_id`, `agent_id`, `session_id`.
- **Memory mode at cutover** — pick `sync` initially (safer, fact-completeness); switch to `async` after measuring latency in production.
- **Cost during parallel-run** — you're paying both systems. Set a deadline for cutover.

## Cross-references

- `existing-codebase-audit.md` — the audit that identified the incumbent
- `features/priming.md` — bulk import surface
- `features/memory.md` (covered in references/python.md etc.) — bulk_create_facts
- `decisions/byok-vs-customllm.md` — LLM provider migration
- All `migrations/*.md` source-specific playbooks
