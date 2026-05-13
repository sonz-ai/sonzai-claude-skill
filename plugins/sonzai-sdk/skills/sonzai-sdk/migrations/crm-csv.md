---
name: migration-crm-csv
description: Use when bulk-importing user data from a CSV / CRM (Salesforce export, HubSpot, custom spreadsheet) into Sonzai via priming.
---

# Migrating from CSV / CRM

## What you have

A spreadsheet of users with structured attributes (name, email, company, role, etc.) and optionally free-text notes (last interaction, sales notes, support history). Common output of Salesforce export, HubSpot, Pipedrive, or rolled-your-own CRM.

## Field mapping

| CSV column | Sonzai destination |
|---|---|
| stable identifier (id, email, slack_id) | `user_id` (pass as string; derive UUID if needed) |
| name / display_name | `priming.metadata.display_name` |
| company | `priming.metadata.company` |
| role / title | `priming.metadata.role` |
| custom structured fields | `priming.metadata.<key>` |
| free-text notes | `priming.content_blocks` (type=text) |
| prior chat / email transcripts | `priming.content_blocks` (type=chat) |
| timestamps | `priming.metadata.<key>_at` (preserve ISO 8601 format) |

## Migration order

1. **Pre-clean the CSV:**
   - UTF-8 encoding, no BOM
   - Normalize column names (snake_case recommended)
   - Validate stable identifier — must be unique per row
   - Strip / quote fields containing commas

2. **Map columns:**
   ```python
   import csv

   def row_to_priming_args(row):
       return {
           "user_id": row["email"].lower(),
           "metadata": {
               "display_name": row["name"],
               "company": row["company"],
               "role": row["title"],
           },
           "content_blocks": [
               {"type": "text", "content": f"Internal notes: {row['notes']}"} if row.get("notes") else None,
           ],
       }
   ```

3. **Bulk import in chunks:**
   ```python
   CHUNK = 500
   users_buffer = []
   with open("users.csv") as f:
       reader = csv.DictReader(f)
       for row in reader:
           args = row_to_priming_args(row)
           args["content_blocks"] = [b for b in args["content_blocks"] if b]
           users_buffer.append(args)
           if len(users_buffer) >= CHUNK:
               ref = client.priming.batch_import(agent_id=agent_id, users=users_buffer)
               wait_for_completion(client, ref.import_id)
               users_buffer = []
   if users_buffer:
       ref = client.priming.batch_import(agent_id=agent_id, users=users_buffer)
       wait_for_completion(client, ref.import_id)
   ```

4. **Verify per chunk:**
   ```python
   def wait_for_completion(client, import_id):
       import time
       while True:
           status = client.priming.get_import_status(import_id)
           if status.state in ("completed", "failed"):
               return status
           time.sleep(5)
   ```

5. **Smoke test:** pick 5 random users; ask the agent something they should know.

## Gotchas specific to CRM CSVs

- **Encoding** — Excel often saves CSV as windows-1252 or UTF-8 with BOM. Re-save as plain UTF-8.
- **Large CSVs** — chunk at 500-1000 rows per `batch_import`. Don't load 100K rows into memory.
- **Quoted commas** — use `csv.DictReader` (Python) / `csv-parse` (Node) — not split-on-comma.
- **Stable identifiers** — emails change; use a stable internal ID if you have one. Otherwise, lower-case + trim emails for consistency.
- **Multi-value fields** — if a column has comma-separated tags, parse them into a JSON array in metadata.
- **Date formats** — preserve ISO 8601. US date format (MM/DD/YYYY) without explicit format string causes misparsing.
- **Idempotency** — re-running the same CSV updates existing users (idempotent on `user_id`). Safe to retry.
- **Quota** — bulk imports use your project's import quota. Check before importing > 10K rows.

## Verification

- After import: `client.agents.memory.search(agent_id, query=expected_fact, user_id=sample_user_id)` should return the imported content.
- Spot-check display_name on 10 users: chat handler should greet them by name.

## Cross-references

- `features/priming.md` — full batch_import surface
- `migrations/overview.md`
- `migrations/raw-json.md` — for non-CSV structured data
