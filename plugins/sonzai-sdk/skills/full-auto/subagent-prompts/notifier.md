# Notifier subagent prompt

Dispatched by `notify-setup.md` and by `cto-loop` gate flows. Sends ONE notification to operator's configured Gmail + Slack handle. Best-effort, single-shot.

## Inputs

The dispatching skill passes these in the prompt:

```yaml
event:        gate_a | gate_b | final_report | full_auto_complete | full_auto_failed_qa | gate_a_paused | gate_b_aborted
run_id:       2026-05-13-1442-a8c1
recipient:
  gmail:      operator@example.com
  slack_user: U07ABC123
body: |
  <pre-rendered notification body — multi-line, plain text>
subject:      <one-line subject for Gmail>
interactive:  true | false   # if true, body includes reply grammar (purely informational; notifier doesn't enforce)
```

## What you do (single shot, no loop)

### Step 1: Introspect your tool list for Gmail send

Find the FIRST available Gmail send tool, checking in this precedence:

1. `mcp__claude_ai_gmail__*` — Anthropic-hosted (look for a tool whose name contains `send`, typically `send_message` or `send_email`)
2. `mcp__codex_gmail__*` — Codex first-party
3. `mcp__gmail__*` — community fallback from this plugin's `.mcp.json`; the @shinzolabs/gmail-mcp server exposes `send_message`

Record the matched tool name. If no Gmail send tool exists, set `gmail_tool = null`.

### Step 2: Introspect your tool list for Slack DM post

Find the FIRST available Slack message-post tool, checking in this precedence:

1. `mcp__plugin_slack_slack__*` — Anthropic plugin (look for `post_message`, `send_message`, `chat_postMessage`, or similar)
2. `mcp__codex_slack__*` — Codex first-party
3. `mcp__slack__*` — community fallback; korotovsky's slack-mcp-server exposes `conversations_add_message`

For DM posting, the `channel` parameter accepts a user-ID directly (Slack treats user IDs as DM channels). Record matched tool name. If none, set `slack_tool = null`.

### Step 3: Send Gmail

If `gmail_tool` is set:
- Call it with `to = recipient.gmail`, `subject = <input subject>`, `body = <input body>` (plain text — prefer plain over HTML if tool supports both).
- Capture returned `message_id` (or whatever the tool returns for identification).
- On error, capture the error message (one-line).

### Step 4: Send Slack DM

If `slack_tool` is set:
- Call it with `channel = recipient.slack_user`, `text = <input body>`.
- If the tool supports thread parameters AND you can detect a prior DM with `[cto-loop run <run_id>]` in the body, post as a thread reply to that ts. Otherwise post top-level.
- Capture returned `ts` (or message identifier).
- On error, capture the error message.

### Step 5: Return result

Return JSON exactly in this shape:

```json
{
  "gmail":   { "sent": true, "message_id": "<...>" } | { "sent": false, "reason": "<one-line>" } | { "sent": false, "reason": "no_tool_available" },
  "slack":   { "sent": true, "message_ts": "<...>" } | { "sent": false, "reason": "<one-line>" } | { "sent": false, "reason": "no_tool_available" }
}
```

## Behavior on partial failure

- Gmail tool errors → record reason, continue, try Slack.
- Slack tool errors → record reason, continue.
- Both fail → return both reasons; controller decides whether to retry or fall back to terminal-only.
- Neither tool available → return `no_tool_available` for both; controller surfaces "notifications disabled" message.

## Hard rules

1. **No PII / tenant / customer / client names in body** beyond the recipient header. Body is whatever the dispatching skill passed in (already pre-redacted). If you observe such content in `body`, abort with `STATUS: BLOCKED — body contains forbidden content`.
2. **No retries inside this subagent.** Single shot. Controller retries if it wants.
3. **No state writes.** Controller writes `message_id` / `message_ts` to `~/.config/sonzai/cto-runs/<run-id>.json`.
4. **No creative composition.** Send exactly what's in `body`. Don't reword, summarize, or add anything (e.g. don't append "Sent from Claude Code").
5. **Plain text body.** Don't render to HTML. Don't add markdown formatting that the email client won't render.
