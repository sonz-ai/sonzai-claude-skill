---
name: migration-raw-json
description: Use when bulk-importing arbitrary JSON data (chat logs, custom user records, exported NoSQL dumps) into Sonzai via priming. Schema-less source needs categorization first.
---

# Migrating from raw JSON

## What you have

Arbitrary JSON — heterogeneous user records, exported NoSQL dumps, chat-log archives without a structured schema. May contain a mix of structured fields and free-text content.

## Approach

Heterogeneous JSON usually needs **categorization** before import. For each record, decide:

1. **Structured attributes** → `priming.metadata` (queryable, indexed)
2. **Free-text content** → `priming.content_blocks` (type=text)
3. **Chat transcripts** → `priming.content_blocks` (type=chat)
4. **Pre-extracted facts** → `agents.memory.bulk_create_facts` (no LLM extraction)
5. **Reference / docs** → `client.knowledge.upload_document` or `client.knowledge.insert_facts`

## Migration order

1. **Inspect a sample.** Print 5-10 records. Identify the field categories above.

2. **Write a categorization function:**

   ```python
   def categorize(record):
       """Maps a raw JSON record to priming args."""
       meta = {}
       text_blocks = []
       chat_blocks = []

       # Move structured fields to metadata
       for key in ("name", "email", "company", "role", "tier", "joined_at"):
           if key in record:
               meta[key] = str(record[key])

       # Free-text fields → text content_block
       for key in ("notes", "summary", "description"):
           if key in record and record[key]:
               text_blocks.append({"type": "text", "content": f"{key}: {record[key]}"})

       # Chat transcripts → chat content_block
       if "messages" in record:
           chat_blocks.append({
               "type": "chat",
               "content": [
                   {"role": m["role"], "content": m["text"]}
                   for m in record["messages"]
               ],
           })

       return {
           "user_id": record["id"],  # YOUR stable identifier
           "metadata": meta,
           "content_blocks": text_blocks + chat_blocks,
       }
   ```

3. **Handle weird cases:**

   - **Deeply nested structured data** — flatten to dotted keys (`profile.location.city → "profile_location_city"`).
   - **Lists of structured items** — if they're items (game inventory, holdings), use `agents.inventory.batch_import` instead of priming.
   - **LLM-assisted extraction** — if the JSON is too heterogeneous to categorize manually, run a pass with a small LLM that emits Sonzai-friendly facts. Then use `agents.memory.bulk_create_facts`.

4. **Bulk import:**

   ```python
   import json

   with open("export.json") as f:
       records = json.load(f)

   CHUNK = 500
   buf = []
   for r in records:
       buf.append(categorize(r))
       if len(buf) >= CHUNK:
           ref = client.priming.batch_import(agent_id=agent_id, users=buf)
           wait_for(ref.import_id)
           buf = []
   if buf:
       ref = client.priming.batch_import(agent_id=agent_id, users=buf)
       wait_for(ref.import_id)
   ```

5. **Verify** with `memory.search` on a sample.

## Pre-extracted facts shortcut

If your JSON already contains extracted atomic facts (not raw text), skip priming entirely:

```python
for user_id, facts in extracted_facts_by_user.items():
    client.agents.memory.bulk_create_facts(
        agent_id=agent_id,
        user_id=user_id,
        facts=[{"content": f["text"], "fact_type": f.get("type")} for f in facts],
    )
```

This skips LLM extraction (cheaper, faster) but you need to have done the extraction yourself first.

## Gotchas specific to raw JSON

- **Schema variance** — write a categorizer that defaults gracefully on missing fields. Don't crash on the first weird record.
- **Encoding** — UTF-8 throughout; reject other encodings explicitly.
- **Field name normalization** — JSON keys may be camelCase, snake_case, or mixed. Pick one for your metadata keys (snake_case recommended); transform on the way in.
- **Lists of items vs lists of users** — clarify which. Items go to inventory; users go to priming.
- **PII handling** — strip secret/sensitive fields before importing (passwords, SSN, financial info — even into priming).
- **LLM extraction cost** — if you use an extraction pass, batch chunks of 50-100 records per LLM call to amortize.
- **Quotas** — large imports may exceed daily quota; chunk and pace.

## Cross-references

- `features/priming.md`
- `features/inventory.md` — for item-shaped lists
- `migrations/crm-csv.md` — structured tabular variant
- `migrations/overview.md`
