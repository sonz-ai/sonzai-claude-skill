# Notify-setup (Phase 0-pre — interactive variant for cto-loop)

Runs at the very start of `/cto-loop`, before Phase 0a transcript analysis. Overlay on `../full-auto/notify-setup.md` — uses the same detection logic, adds an interactive ask if recipient config is missing AND at least one MCP is available.

## When to use

First step of `cto-loop/pipeline.md`. Before any transcript work.

## What you do

### Step 1: Run autonomous detection from `../full-auto/notify-setup.md`

Follow Steps 1-3 from `../full-auto/notify-setup.md` exactly — load env vars, check `~/.config/sonzai/cto.json`, detect Gmail + Slack MCP tool prefixes + verify auth.

### Step 2: If recipient config is missing, ASK

This is the cto-loop-specific behavior. If the autonomous flow reached "notifications disabled" via the no-recipient-config path AND at least one of Gmail / Slack MCP was detected as available, prompt the operator:

```
─────────────────────────────────────────────────────
NOTIFY-SETUP

I detected <Gmail / Slack / Gmail + Slack> MCPs are available.
For async gate replies (approve / feedback / abort via your inbox
or Slack DM), I need:

  Gmail address (where to email you):    _______________________
  Slack handle (or user ID if known):    _______________________

Reply with both on one line, comma-separated. Examples:
  operator@example.com, @operator
  operator@example.com, U07ABC123

Or type `skip` to run in terminal-only mode (no notifications;
you must come back to this terminal to reply at gates).
─────────────────────────────────────────────────────
```

If only ONE MCP is available, ask for only the relevant handle (and note "Slack notifications disabled" / "Gmail notifications disabled" accordingly).

### Step 3: Capture + persist

If operator replied with values:

1. **Validate Gmail**: must look like an email (regex `^[^@\s]+@[^@\s]+\.[^@\s]+$`).
2. **Slack handle resolution**:
   - If input starts with `@`: treat as handle. Resolve via `users_search` on the available Slack MCP.
   - If input starts with `U` followed by uppercase alphanumerics: treat as user_id directly.
   - Otherwise: re-ask.
3. **Save** to `~/.config/sonzai/cto.json`:

```bash
mkdir -p ~/.config/sonzai
cat > ~/.config/sonzai/cto.json.tmp <<EOF
{
  "gmail": "<email>",
  "slack_handle": "<handle>",
  "slack_user_id": "<resolved-id>"
}
EOF
mv ~/.config/sonzai/cto.json.tmp ~/.config/sonzai/cto.json
chmod 600 ~/.config/sonzai/cto.json
```

4. Print confirmation:

```
Saved to ~/.config/sonzai/cto.json. Future cto-loop / full-auto runs will reuse this.
Override with env vars SONZAI_NOTIFY_GMAIL / SONZAI_NOTIFY_SLACK_USER_ID if needed.
```

If operator replied `skip`:
1. Set `notify_enabled = false` in in-memory state.
2. Print: `Skipped. Gates will only accept terminal replies.`
3. Continue.

If invalid input (e.g. one field missing, malformed email): re-ask ONCE. Second invalid → set `notify_enabled = false`, print one-liner, continue.

### Step 4: Resume autonomous flow at Step 4

Continue with `../full-auto/notify-setup.md` Step 4 (slack_user_id resolution if needed — though likely already done in Step 3 above) and Step 5 (summarize state).

## Hard rules

1. **Ask at most once per run.** Don't re-ask if config already exists.
2. **`skip` is explicit.** Empty input / whitespace / "no" / "n" all re-ask. Only literal `skip` short-circuits.
3. **`chmod 600`** on `~/.config/sonzai/cto.json` — per-user, treat as sensitive (email + Slack handle).
4. **No ask if no MCPs.** If detection found NEITHER Gmail nor Slack tools available, do NOT ask — there's no channel to send through. Print an enablement guide and skip:

```
No Gmail or Slack MCP enabled. Skipping notify-setup.
To enable in Claude Code:  /mcp → enable claude.ai Gmail and plugin:slack:slack.
To enable in Codex:        codex mcp enable codex_gmail codex_slack.
Notifications disabled. Gates will accept terminal replies only.
```

5. **`gate_a` reply grammar applies later, not here.** Step 2's reply ("operator@example.com, @nas" or "skip") is parsed by THIS step's own logic, not by `reply-poller.md`'s gate grammars.
