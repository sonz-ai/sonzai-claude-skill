---
name: existing-codebase-audit
description: Use when integrating Sonzai into an existing codebase that already has chat, memory, personality, LLM-provider, knowledge-base, webhook, or scheduled-job code in place. Runs before archetype selection — the audit findings often change which archetype fits.
---

# Existing-codebase audit

Find the insertion points. Determine migration order. Route to the right playbook.

## How to use

Run the 8 audit steps in order. Each produces a one-line finding. At the end, you have an insertion-point report and a strangler-pattern migration order.

This audit runs when `intake.md` Q1 = `existing`. After completion, resume `intake.md` at Q2 (archetype selection).

---

## Audit 1 — Find the current chat handler

The path your app currently takes to talk to an LLM.

```bash
rg -l "openai\.(chat|completions|ChatCompletion)|anthropic\.messages|google\.generativeai|@google/generative-ai|GoogleGenerativeAI|langchain.*chat|llama_index.*llms|mem0\.|letta\.|zep\.|llm.invoke|chain\.invoke" \
   --type-add 'all:*.{py,ts,tsx,js,jsx,go}' --type all . 2>/dev/null
```

**Record:** Every file path that surfaces. Note the provider (OpenAI / Anthropic / Gemini / etc.) and whether the call is direct or via a wrapper (LangChain, LlamaIndex).

**Implication:** This is the file you'll replace with `agents.chat` / `agents.sessions.start` + `session.turn`. The wrapper (if any) determines which migration playbook applies.

---

## Audit 2 — Find user state

Where the app identifies users today. The chosen identifier becomes the `user_id` you pass to Sonzai.

```bash
rg "req\.user|session\.user|current_user|jwt\.(verify|decode)|getSession|getServerSession|@auth/|next-auth|clerk\.|supabase\.auth" \
   --type-add 'all:*.{py,ts,tsx,js,jsx,go}' --type all . 2>/dev/null | head -30
```

Also check DB schema files for `users` table / `User` model.

**Record:** The canonical user-id source (e.g. `req.user.id` from Clerk JWT, `session.user.email` from NextAuth, etc.).

**Implication:** Pass that exact identifier as `user_id` to Sonzai calls. Stable across requests = correct memory attribution. If your identifier rotates (e.g. session tokens), derive a stable hash first.

---

## Audit 3 — Find memory layer

What's storing conversation history today?

```bash
rg -l "ConversationBufferMemory|VectorStoreRetrieverMemory|RedisChatMessageHistory|messages.*table|chat_history|mem0|MemoryClient|Zep|ZepMemory|letta|LettaClient|MemGPT|pinecone|chroma|weaviate|qdrant" \
   --type-add 'all:*.{py,ts,tsx,js,jsx,go,sql}' --type all . 2>/dev/null
```

**Record:** Memory abstraction in use (named system or rolled-your-own).

**Implication:** This is the **first thing to replace** (strangler step 1). Migration playbook routes:

| Incumbent | Playbook |
|---|---|
| Mem0 | `migrations/mem0.md` |
| LangChain `ConversationBufferMemory` / `VectorStoreRetrieverMemory` | `migrations/langchain.md` |
| Letta / MemGPT | `migrations/letta.md` |
| Zep | `migrations/zep.md` |
| Pinecone / Chroma / Weaviate (raw vector store) | `migrations/raw-json.md` (or skip if you'll keep using it as your own KB) |
| Custom messages table | `migrations/raw-json.md` |
| None | nothing to migrate — fresh memory loop |

---

## Audit 4 — Find personality config

Where the agent's "voice" is currently defined.

```bash
rg -l "system_prompt|assistant_prompt|instructions|character_card|persona_description|personality" \
   --type-add 'all:*.{py,ts,tsx,js,jsx,go,md,txt,yaml,yml,json}' --type all . 2>/dev/null | head -20
```

**Record:** Every file containing a system prompt or persona definition. Catalog them.

**Implication:** Move into Sonzai via either `agents.create(... personality_prompt="...")` (if you want explicit control) or `agents.generation.generate_and_create(... description="...")` (if you'd rather let Sonzai derive personality from a description). See `decisions/generation-vs-manual-create.md`.

---

## Audit 5 — Find LLM provider config

Which provider/model is in use today? Pinned or dynamic?

```bash
rg "OPENAI_API_KEY|ANTHROPIC_API_KEY|GOOGLE_API_KEY|GEMINI_API_KEY|XAI_API_KEY|OPENROUTER_API_KEY|model.*=.*['\"]gpt|model.*=.*['\"]claude|model.*=.*['\"]gemini" \
   --type-add 'all:*.{py,ts,tsx,js,jsx,go,env}' --type all . 2>/dev/null | head -20
```

**Record:** Provider, model, where the model is pinned.

**Implication:** Drives the BYOK vs Custom LLM decision (see `decisions/byok-vs-customllm.md`):
- Keep billing on your provider account → BYOK (register your provider key with Sonzai)
- Self-hosted / fine-tuned model → Custom LLM (OpenAI-compatible endpoint)
- Don't care, use platform routing → default (Sonzai picks)

---

## Audit 6 — Find webhook receivers

Existing event/webhook handling.

```bash
rg -l "POST.*webhook|/webhook|/event|/callback|hmac|HMAC|verifyWebhook|stripe\.webhooks\.constructEvent" \
   --type-add 'all:*.{py,ts,tsx,js,jsx,go}' --type all . 2>/dev/null
```

**Record:** Receiver routes and their shape (which events, which signature algorithm).

**Implication:** Sonzai webhooks use HMAC-SHA256; register via `client.webhooks.register` (see `features/webhooks.md`). If your app already verifies HMAC, the verify helper pattern is reusable.

---

## Audit 7 — Find KB / RAG layer

Existing vector store / document ingestion pipeline.

```bash
rg -l "embeddings|vector_store|VectorStore|cosine_similarity|TextSplitter|RecursiveCharacterTextSplitter|DocumentLoader|PyPDFLoader|UnstructuredLoader|retriever\.invoke|similarity_search" \
   --type-add 'all:*.{py,ts,tsx,js,jsx,go}' --type all . 2>/dev/null
```

**Record:** Vector store, embedding model, document corpus size.

**Implication:** Migrate to `client.knowledge.uploadDocument` + `client.knowledge.search`. See `features/knowledge-base.md`. Keep both running during cutover for parallel-run validation. If org-wide content exists (policies, brand), also see `features/org-knowledge-base.md`.

---

## Audit 8 — Find proactive jobs

Recurring tasks / scheduled jobs.

```bash
rg -l "cron|schedule_interval|node-cron|apscheduler|celery|BullMQ|Bull\(|Temporal|TemporalClient|setInterval.*60.*1000|@scheduled" \
   --type-add 'all:*.{py,ts,tsx,js,jsx,go}' --type all . 2>/dev/null
```

**Record:** Recurring tasks and their cadence.

**Implication:** Migrate to `client.schedules.create` (recurring) or `agents.scheduleWakeup` (one-off). Delivery channel decision in `decisions/proactive-channel.md`.

---

## Output — insertion-point report

After all 8 audits, produce a summary table:

| Layer | Current incumbent | Target Sonzai surface | Migration playbook |
|---|---|---|---|
| Chat handler | `<from Audit 1>` | `agents.chat` / `agents.sessions.start` | depends on Audit 1 |
| User identification | `<from Audit 2>` | pass as `user_id` to all Sonzai calls | — |
| Memory | `<from Audit 3>` | `agents.memory.*` + sessions | `migrations/<source>.md` |
| Personality | `<from Audit 4>` | `agents.create` + `personality_prompt` | `decisions/generation-vs-manual-create.md` |
| LLM provider | `<from Audit 5>` | platform default / BYOK / Custom LLM | `decisions/byok-vs-customllm.md` |
| Webhooks | `<from Audit 6>` | `client.webhooks.*` | `features/webhooks.md` |
| KB / RAG | `<from Audit 7>` | `client.knowledge.*` | `features/knowledge-base.md` |
| Proactive | `<from Audit 8>` | `client.schedules` / `agents.scheduleWakeup` | `features/proactive.md` |

---

## Migration order (strangler pattern)

Run in this order. Each step is independently shippable and rollback-able.

| Order | Layer | Why first | Verify with |
|---|---|---|---|
| 1 | Memory | Pure replacement; doesn't change UX | Diff retrieval quality on a fixed query set |
| 2 | Personality | System prompts → agent definition | Agent responds in same voice on 10 canonical prompts |
| 3 | LLM provider routing | Direct call → `agents.chat`. BYOK if you want to keep billing on your provider | Token usage flows match expected |
| 4 | KB | Migrate vector store contents to `client.knowledge` (or run both during cutover) | Top-5 retrieval matches incumbent on N test queries |
| 5 | Proactive | Cron / Celery → `client.schedules` | Job fires at expected cadence in staging |
| 6 | Webhooks | Register Sonzai webhooks to fan out to your existing handlers | Verify HMAC on a test delivery |

**Important:** Steps 1-3 in that order. The memory migration is safe to ship first because reading from your old memory + writing to Sonzai means no user-visible change. Steps 4-6 can be done in parallel after 3 is stable.

---

## Resume the wizard

After this audit completes, return to `intake.md` Q2 (archetype selection). The audit findings often clarify which archetype actually fits:

- Existing Mem0 + chat handler + single user_id per conversation → likely **companion** or **guide-router** archetype
- Existing shared chat handler across team users → **enterprise-assistant**
- Existing webhooks for tickets / KB-grounded answers → **customer-support**
- Existing cron jobs for proactive outreach + drift-on persona → likely **companion** or **coach-therapist**
- Existing custom_states-like per-user game data → **game-npc**

---

## Cross-references

- `intake.md` — wizard entry; this audit is Q1's existing-codebase branch
- `migrations/overview.md` — entry point to all migration playbooks
- `migrations/{source}.md` — per-source migration details (mem0, langchain, letta, zep, openai-assistants, character-ai, crm-csv, raw-json)
- `decisions/byok-vs-customllm.md` — LLM provider routing decision
- `decisions/generation-vs-manual-create.md` — personality migration choice
- `features/webhooks.md`, `features/knowledge-base.md`, `features/proactive.md` — target surfaces
