# Notify-setup (Phase 0-pre — autonomous variant)

Runs at the very start of `full-auto`, before Phase 0a transcript analysis. Detects whether outbound notifications are possible. Does NOT ask the operator anything — full-auto is unattended.

## When to use

First step of `pipeline.md` for full-auto. Before any transcript work.

This is also the shared core for cto-loop. `cto-loop/notify-setup.md` runs Steps 1-3 from this file before adding its own interactive ask.

## What you do

### Step 1: Load recipient config

Check in this precedence:

1. **Env vars**: `SONZAI_NOTIFY_GMAIL`, `SONZAI_NOTIFY_SLACK_HANDLE`, `SONZAI_NOTIFY_SLACK_USER_ID`. If all three are set, use them.
2. **File**: `~/.config/sonzai/cto.json`. If exists, load JSON: `{ "gmail": "...", "slack_handle": "...", "slack_user_id": "..." }`.
3. **Neither**: notifications DISABLED for this run. Print ONE line:

```
Notifications disabled — no recipient configured.
Set SONZAI_NOTIFY_GMAIL + SONZAI_NOTIFY_SLACK_USER_ID, or run /cto-loop once to set ~/.config/sonzai/cto.json.
Proceeding in unattended mode.
```

Save `notify_enabled = false` to in-memory state, return.

### Step 2: Detect Gmail MCP

Introspect available tools (via your tool list — agents on Claude Code / Codex / Gemini CLI all have access to their own tool list).

Check in precedence:
1. `mcp__claude_ai_gmail__*` (Anthropic-hosted)
2. `mcp__codex_gmail__*` (Codex first-party)
3. `mcp__gmail__*` (community fallback from this plugin's `.mcp.json`)

Record which prefix matched (or `null`). If matched: call ONE no-op / profile tool to verify auth works (e.g. `get_profile` on shinzolabs, or whatever profile-equivalent exists on the matched prefix). If the tool returns "needs authentication" / 401 / similar auth error: treat as not-available, record reason `gmail_auth_missing`.

### Step 3: Detect Slack MCP

Same pattern:
1. `mcp__plugin_slack_slack__*` (Anthropic plugin)
2. `mcp__codex_slack__*` (Codex first-party)
3. `mcp__slack__*` (community fallback)

Verify auth via a profile-equivalent or `users_search` call. Record matched prefix.

### Step 4: Resolve slack_user_id if missing

If recipient config has `slack_handle` but not `slack_user_id`, AND Slack MCP is available: call `users_search` (or equivalent on the matched prefix) to resolve handle → user_id. If matched prefix supports `users_lookupByEmail` and the recipient's email is at a Slack-workspace-domain match, prefer that. Cache the resolved `slack_user_id` back to `~/.config/sonzai/cto.json` (atomic write: `<file>.tmp`, then rename).

### Step 5: Summarize state

Save to in-memory state for the rest of the pipeline:

```yaml
notify_enabled:    true | false
gmail_tool_prefix: mcp__claude_ai_gmail__ | mcp__codex_gmail__ | mcp__gmail__ | null
slack_tool_prefix: mcp__plugin_slack_slack__ | mcp__codex_slack__ | mcp__slack__ | null
recipient:
  gmail:         <email-or-null>
  slack_user_id: <id-or-null>
```

Print one line to terminal:

```
Notifications: Slack ✓ (<prefix>), Gmail ✓ (<prefix>). Will notify on build complete / failing QA.
```

OR (if either is disabled):
```
Notifications: Slack ✓, Gmail ✗ (no MCP). Will notify via Slack only.
```

OR (both disabled):
```
Notifications disabled — no MCP available or no recipient configured. Build proceeds silently.
```

Continue to Phase 0a.

## Hard rules

1. **Never block on missing MCPs.** This is autonomous mode; degrade silently.
2. **Never ask the operator.** That's cto-loop's job (it overlays an interactive ask).
3. **No retries on detection.** One pass, record state, move on.
4. **Atomic config write.** Use `<file>.tmp` + rename pattern when caching `slack_user_id`. Avoid corrupt config on crash.
