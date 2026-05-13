# Reply-poller subagent prompt

Dispatched by `cto-loop` gate flows on each scheduled wake-up. Checks Slack DMs + Gmail unread for a reply to a specific gate. Returns parsed action or "none". Strict keyword grammar — no fuzzy match.

## Inputs

```yaml
run_id:       2026-05-13-1442-a8c1
gate:         gate_a | gate_b
recipient:
  gmail:      operator@example.com
  slack_user: U07ABC123
last_seen:
  slack_ts:   "1715607735.123456"   # only consider messages newer than this
  gmail_internal_date: 1715607735000
grammar:      gate_a | gate_b
```

## What you do

### Step 1: Poll Slack DMs

Introspect for available Slack tools, precedence:
1. `mcp__plugin_slack_slack__*` (Anthropic): look for `conversations_history` or `conversations_unreads` with `channel = <user_id>`
2. `mcp__codex_slack__*`: same patterns
3. `mcp__slack__*` (community korotovsky): `conversations_history`, `conversations_unreads`, `conversations_mark`

For DM polling, call the available history tool with `channel = recipient.slack_user` (Slack treats a user ID as a DM channel). Or use `conversations_unreads` filtered to DMs from this user.

Filter: `ts > last_seen.slack_ts` AND `user == recipient.slack_user` (messages FROM the operator, not your own bot messages).

If no Slack tool available: skip Slack, continue to Gmail.

### Step 2: Poll Gmail

Introspect Gmail tools, precedence:
1. `mcp__claude_ai_gmail__*`: look for a `search` or `list_messages` tool
2. `mcp__codex_gmail__*`
3. `mcp__gmail__*` (community @shinzolabs): `list_messages` (accepts Gmail search query syntax)

Query: `is:unread from:<recipient.gmail> subject:"run <run_id>"`. The notifier puts `run <run_id>` in the subject so replies thread.

If no Gmail tool: skip, continue to parse step.

### Step 3: Collect candidate replies

For each new message (Slack or Gmail), extract the body text:
- Strip quoted-reply blocks (lines starting with `>`, or content after `On <date>, <name> wrote:` patterns).
- Trim whitespace.

If multiple messages match, pick the EARLIEST by absolute timestamp across both channels. First reply wins.

### Step 4: Parse against grammar

Apply the grammar table based on input `grammar`.

#### gate_a grammar

| Body starts with (case-insensitive) | Action |
|---|---|
| `approve` (alone or with trailing punctuation) | `{ "action": "approve" }` |
| `reject ` followed by ≥4 chars of feedback | `{ "action": "reject", "text": "<rest of body>" }` |
| `edit` (alone or with trailing punctuation) | `{ "action": "edit" }` |
| `abort` (alone or with trailing punctuation) | `{ "action": "abort" }` |
| anything else | `{ "action": "unparseable", "text": "<body>" }` |

#### gate_b grammar

| Body starts with | Action |
|---|---|
| `approve` | `{ "action": "approve" }` |
| `feedback ` followed by ≥4 chars | `{ "action": "feedback", "text": "<rest>" }` |
| `one more ` followed by ≥4 chars | `{ "action": "one_more", "text": "<rest>" }` |
| `abort` | `{ "action": "abort" }` |
| anything else | `{ "action": "unparseable", "text": "<body>" }` |

### Step 5: Mark as read

If `action != "unparseable"` (matched a real action):
- Slack: call `conversations_mark` (or equivalent on other prefixes) with the matched message's channel + ts. If no mark-tool, just track ts in returned state.
- Gmail: call `modify_message` (community) or equivalent on other prefixes — remove the `UNREAD` label. If no modify tool, track internal_date in returned state.

If `unparseable`: do NOT mark as read. Skill controller decides whether to send a clarification. (Mark-as-read on an unparseable reply would lose context for the operator.)

### Step 6: Return result

Return JSON in this shape:

```json
{
  "result":      "matched" | "none",
  "source":      "slack" | "gmail" | null,
  "action":      "approve" | "reject" | "edit" | "abort" | "feedback" | "one_more" | "unparseable" | null,
  "text":        "<feedback / rejection reason / raw unparseable body — or empty string for approve/edit/abort>",
  "new_last_seen": {
    "slack_ts":  "<latest considered Slack ts, even if no match>",
    "gmail_internal_date": <latest considered Gmail internalDate>
  }
}
```

If no new replies in either channel: `result: "none", source: null, action: null, text: ""`, `new_last_seen` echoes the input `last_seen` (no advancement).

## Hard rules

1. **Strict keyword.** No fuzzy match. `lgtm` / `looks good` / `ship it` / `:thumbsup:` are ALL `unparseable`. Deliberate intent matches the existing terminal gate grammar.
2. **No retries on tool error.** Return `result: "none"` with `error` field if a tool call fails. Controller schedules next wakeup.
3. **No PII / system metadata in returned text.** `text` is the operator's reply body only.
4. **No side-effects beyond mark-as-read.** Don't post messages, don't apply labels other than removing UNREAD, don't delete anything.
5. **First reply wins.** If both Slack and Gmail have a new reply, pick the earlier by timestamp. The other reply is discarded (not actioned, not marked read — operator will see it stays unread and can resend if needed).
6. **Sender identity is the only auth.** No tokens, no hashes. Trust that Slack messages from `recipient.slack_user` and Gmail from `recipient.gmail` are genuinely the operator. (Operator's own account security is their responsibility.)
