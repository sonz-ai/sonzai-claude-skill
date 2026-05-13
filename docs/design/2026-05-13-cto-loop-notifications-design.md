# cto-loop / full-auto notifications + async reply design

**Status:** Draft (2026-05-13)
**Target version:** v1.7.0
**Scope:** `plugins/sonzai-sdk/` (public plugin only). Codex parity tracked.

## Summary

Add outbound notifications (Slack DM + Gmail) on key pipeline events for both `cto-loop` and `full-auto`. For `cto-loop` gates specifically, the operator can reply via Slack DM or Gmail — first reply (terminal, Slack, or Gmail) wins. The skill polls every 20 minutes for up to 24 hours, then pauses with a resumable state file.

### Goals

1. Operator does not need to babysit the terminal while a long autonomous build runs.
2. Operator can approve / feedback / abort cto-loop gates from their phone (Slack mobile or Gmail mobile).
3. Plugin install (Claude Code or Codex) leaves the operator one `/mcp` enable-click away from working notifications.
4. Graceful degradation: missing MCP, missing config, missing `/loop` wrapper — fall back to current terminal-sync behavior, never block.

### Non-goals

- No phone push (just Slack DM + Gmail).
- No SMS / Discord / Teams.
- No multi-recipient broadcast — single operator only.
- No reply-from-terminal-when-async-mode-on disabling — terminal stdin remains an override path.
- No tokens / signed messages — trust sender identity (Slack user ID, Gmail address).
- No Anthropic-side scheduler — uses Claude Code's `ScheduleWakeup` only.

## Architecture

### Events

| Pipeline | Event | Notify? | Expects reply? |
|---|---|---|---|
| `cto-loop` | Gate A (masterplan ready) | yes | yes (approve / reject / edit) |
| `cto-loop` | Gate B (live app ready) | yes | yes (approve / feedback / abort) |
| `cto-loop` | Final report written | yes | no (informational) |
| `full-auto` | Build complete + healthy | yes | no |
| `full-auto` | Build done with failing QA | yes | no |
| Both | Hard crash / unrecoverable | no (v1.7.0); revisit later | n/a |

### Components

```
plugins/sonzai-sdk/
├── .mcp.json                                   [NEW] community Gmail+Slack fallbacks
├── skills/
│   ├── full-auto/
│   │   ├── notify-setup.md                     [NEW] autonomous variant: load config, detect MCPs
│   │   ├── subagent-prompts/
│   │   │   ├── notifier.md                     [NEW] dispatches a single notification
│   │   │   └── reply-poller.md                 [NEW] checks Slack DMs + Gmail unread for replies
│   │   ├── pipeline.md                         [MOD] insert Phase 0-pre, notify hooks at terminal events
│   │   ├── final-report.md.template            [MOD] include "notifications sent" footer
│   │   └── qa-loop.md                          [MOD] dispatch notifier on QA pass / 5-cycle exhaustion
│   └── cto-loop/
│       ├── notify-setup.md                     [NEW] interactive variant: ask if not cached
│       ├── masterplan-gate.md                  [MOD] async path: notify → ScheduleWakeup → poll
│       ├── cto-review-gate.md                  [MOD] async path: same pattern
│       └── pipeline.md                         [MOD] overlay map shows Phase 0-pre, async gates
└── ... (manifest files updated for v1.7.0)
```

### State file

Path: `~/.config/sonzai/cto-runs/<run-id>.json`

Run ID format: `<YYYY-MM-DD>-<HHMM>-<random-4>`, e.g. `2026-05-13-1442-a8c1`.

Schema:

```json
{
  "run_id": "2026-05-13-1442-a8c1",
  "started_at": "2026-05-13T14:42:11Z",
  "base_sha": "abc1234",
  "mode": "cto-loop",
  "masterplan_path": "docs/cto-review/2026-05-13-masterplan.md",
  "current_phase": "gate_a",
  "recipients": {
    "gmail": "operator@example.com",
    "slack_user": "U07ABC123",
    "slack_handle": "@operator"
  },
  "notifications_sent": [
    { "event": "gate_a", "sent_at": "2026-05-13T14:42:15Z",
      "slack_message_ts": "1715607735.123456", "gmail_message_id": "<abc@mail.gmail.com>" }
  ],
  "last_poll_at": "2026-05-13T15:02:00Z",
  "poll_count": 1,
  "first_poll_at": "2026-05-13T14:42:15Z",
  "timeout_at": "2026-05-14T14:42:15Z",
  "paused": false
}
```

### Recipient config

Path: `~/.config/sonzai/cto.json` (per-user, not per-run).

Schema:

```json
{
  "gmail": "operator@example.com",
  "slack_handle": "@operator",
  "slack_user_id": "U07ABC123"
}
```

Precedence: env vars (`SONZAI_NOTIFY_GMAIL`, `SONZAI_NOTIFY_SLACK_HANDLE`, `SONZAI_NOTIFY_SLACK_USER_ID`) > this file > interactive ask. `cto-loop`'s notify-setup is the only path that asks; full-auto's notify-setup never asks (autonomous) — if neither env nor file is set, notifications are silently disabled with a one-line stderr note.

`slack_user_id` is resolved at first-run by calling the available Slack MCP's `users.lookupByEmail` or equivalent. Cached in the file so subsequent runs skip the lookup.

## MCP detection

Detection runs in `notify-setup.md` (both autonomous + interactive variants). The skill inspects its own tool list at runtime.

| Platform | Gmail tool prefix | Slack tool prefix | Auth status check |
|---|---|---|---|
| Claude Code (Anthropic built-in) | `mcp__claude_ai_gmail__*` | `mcp__plugin_slack_slack__*` | Call a no-op tool; if it returns "needs authentication", surface a one-line "/mcp → enable & authenticate Gmail" message |
| Codex (v0.117.0+ first-party) | `mcp__codex_gmail__*` | `mcp__codex_slack__*` | Same |
| Community fallback (this plugin's `.mcp.json`) | `mcp__gmail__*` | `mcp__slack__*` | Same |

Selection order: first-party > community. If neither path exposes Gmail tools, Gmail notifications are skipped. Same for Slack. If both fail, the skill falls back to terminal-sync mode and prints:

```
Notifications disabled — no Gmail or Slack MCP server is enabled.
To enable in Claude Code: /mcp → enable claude.ai Gmail and plugin:slack:slack.
To enable in Codex: codex mcp enable codex_gmail codex_slack.
Falling back to terminal-only gate prompts.
```

## `.mcp.json` for non-Claude-Code platforms

Path: `plugins/sonzai-sdk/.mcp.json` (plugin-root level, auto-loaded by Claude Code per [plugin docs](https://code.claude.com/docs/en/plugins)).

```json
{
  "mcpServers": {
    "gmail": {
      "command": "npx",
      "args": ["-y", "@gongrzhe/server-gmail-autoauth-mcp"]
    },
    "slack": {
      "command": "npx",
      "args": ["-y", "slack-mcp-server@latest"],
      "env": {
        "SLACK_MCP_XOXP_TOKEN": "${SLACK_MCP_XOXP_TOKEN}"
      }
    }
  }
}
```

**Always-search-current-state applies:** package names + invocation flags are subject to verification by `version-checker.md` at implementation time. The strings above are placeholders; the writing-plans phase MUST regenerate them via `npm view` lookup. No version pin (`@latest` or unpinned) so the always-search rule isn't violated by static config.

On Claude Code, if the operator has already enabled the Anthropic-shipped Gmail / Slack via `/mcp`, both paths are live simultaneously. Detection prefers the Anthropic path; the bundled community servers idle.

## Notification body formats

### Gate A (cto-loop, interactive)

**Slack DM:**
```
🚪 cto-loop Gate A — Masterplan ready for your review.

Run:     2026-05-13-1442-a8c1
File:    docs/cto-review/2026-05-13-masterplan.md
Mode:    greenfield
Stack:   TypeScript + Next.js + Postgres + Clerk
Scope:   companion archetype, memory-layer-sessions runtime

Reply with:
  approve
  reject <reason>
  edit                (then read the file, edit, reply 'approved')
  abort

(Waiting up to 24h. Polling every 20min.)
```

**Gmail subject:**
`[cto-loop] Gate A — Masterplan ready for review — run 2026-05-13-1442-a8c1`

**Gmail body:** same as Slack DM, in plain text.

### Gate B (cto-loop, interactive)

**Slack DM:**
```
🚪 cto-loop Gate B — Live app ready (cycle 1/5).

Run:        2026-05-13-1442-a8c1
Live URL:   http://localhost:3000
Commits:    abc1234..def5678 (12 commits)
QA status:  passing
Time taken: 18 minutes

What was built (3-para synthesis):
<...generated by Gate B prep step, same as terminal version...>

Reply with:
  approve
  feedback <text>
  abort

(Waiting up to 24h. Polling every 20min.)
```

### Final report (cto-loop, informational)

**Slack DM:**
```
✅ cto-loop complete (approved).

Run:        2026-05-13-1442-a8c1
Live URL:   http://localhost:3000
Report:     docs/cto-review/2026-05-13-final-report.md
Total cycles: 2

App is still running (docker compose). Push when ready.
```

### full-auto: Build complete

**Slack DM:**
```
✅ full-auto build complete (healthy).

Run:        2026-05-13-1442-a8c1
Live URL:   http://localhost:3000
Report:     docs/cto-review/2026-05-13-final-report.md
QA status:  passing
Time taken: 22 minutes

App is still running (docker compose). Push when ready.
```

### full-auto: Failing QA

**Slack DM:**
```
⚠️  full-auto build done with failing QA (5 fixer cycles exhausted).

Run:        2026-05-13-1442-a8c1
Live URL:   http://localhost:3000
Report:     docs/cto-review/2026-05-13-final-report.md
QA status:  FAILING on: auth-redirect, db-migration
Time taken: 31 minutes

App is up; some flows broken. See report for what's red.
```

### Security / PII rules for all notification bodies

1. **No transcript content.** Body is derived from masterplan + run state only.
2. **No customer / client / tenant name.** Phase 0a already strips these from synthesis; notification re-uses synthesis verbatim.
3. **No code snippets.** Path references only.
4. **No API keys.** Live URL is `localhost:*` — never points at a prod env.
5. **No PII (operator name, email, etc.) inside the body** beyond the recipient header.

## Async gate flow (cto-loop only)

```
Phase 1 (masterplan written)
  └─ Read masterplan_path into state file
  └─ Dispatch notifier(event=gate_a, recipient=<from config>)
       └─ Slack DM sent + Gmail sent
       └─ Save message_ts / message_id to state.notifications_sent[]
  └─ Print to terminal:
       "Gate A — masterplan at <path>. Notified via Slack + email.
        Reply here OR via Slack/email. I'll poll every 20min for up to 24h."
  └─ If ScheduleWakeup tool available:
       └─ Save state, ScheduleWakeup(1200s, prompt="<resume-sentinel>")
       └─ Exit this turn.
     Else:
       └─ Fall through to terminal-sync wait (current behavior).

(Wake-up after ~20min)
  └─ Re-read state file.
  └─ Dispatch reply-poller(run_id, recipient, last_seen_*)
       └─ Calls mcp_slack.unread_dms since state.last_poll_at, filtered by sender.
       └─ Calls mcp_gmail.search('is:unread from:<recipient.gmail> subject:cto-loop')
       └─ Parses first match against {approve, reject <txt>, edit, abort} grammar.
       └─ Returns: { action: <one of> | "none", source: "slack" | "gmail", raw: <text> }
  └─ If action != "none":
       └─ Mark replied messages as read (Slack + Gmail).
       └─ Process action exactly as terminal would.
       └─ Clear state.current_phase / advance.
     Else if (now - first_poll_at) >= 24h:
       └─ Set state.paused = true.
       └─ Notify operator: "Gate A timed out at 24h. Resume with /cto-loop resume <run-id>."
       └─ Exit.
     Else:
       └─ Update state.last_poll_at + state.poll_count++.
       └─ ScheduleWakeup(1200s, prompt="<resume-sentinel>").
       └─ Exit this turn.
```

Gate B follows the identical pattern; only the action grammar differs (`approve / feedback <txt> / abort`).

## Reply parsing grammar

Strict keyword. Case-insensitive. The grammar lives in `reply-poller.md` and matches the terminal grammar in `masterplan-gate.md` / `cto-review-gate.md`.

### Gate A grammar

| Keyword | Action | Notes |
|---|---|---|
| `approve` | Approve masterplan, proceed to Phase 2 | First word of body. Single word lines or "approve." or "approve!" all fine. |
| `reject <reason>` | Re-derive masterplan with feedback | `<reason>` is rest-of-message. Body must start with "reject" + whitespace + at least 4 chars of reason. |
| `edit` | Skill pauses, expects operator to edit file directly | Reply with just "edit" (or "edit." etc.). Skill re-polls for "approved" after edit. |
| `abort` | Stop cto-loop run, leave masterplan in place | Single word. |

### Gate B grammar

| Keyword | Action |
|---|---|
| `approve` | Write final report |
| `feedback <text>` | Dispatch fixer with feedback |
| `one more <text>` | Force-extra cycle past the 5-cycle limit (only valid at cycle 6) |
| `abort` | Stop, leave docker-compose running |

### Ambiguous / unparseable replies

Re-poll. Do not interpret "lgtm", "looks good", or any informal positive as approve. Send NO clarifying notification — that would create a feedback loop where every casual reply triggers another email. Just wait for the next poll cycle; operator will likely correct themselves when they see no response.

After the 3rd consecutive unparseable reply within 1h, send ONE clarifying note via the same channel:

```
Your reply didn't match the expected grammar. Reply with one of:
  approve / reject <reason> / edit / abort
```

## Idempotency

- **Slack:** track `last_seen_message_ts` per run. Replies older than this are ignored. After processing, advance ts to the processed message's ts.
- **Gmail:** track `last_seen_message_id` + `last_seen_internal_date`. Use Gmail's `is:unread` + `from:` filter, then sort by `internalDate` ascending. Mark processed messages as read so they drop out of the next unread query.
- **State writes:** atomic — write to `<file>.tmp`, fsync, rename. Avoid corrupt state on concurrent crashes.
- **Multiple cto-loop runs:** Each has its own run_id + state file. Subject-line `run <run_id>` discriminator lets reply-poller match a Gmail reply back to the right run. Slack DM threading — reply must be in the same thread as the notification, OR (fallback) a top-level DM that matches the latest pending gate's run_id.

## Resumability

If 24h timeout fires:
- State file is preserved (`paused: true`).
- A "paused" notification is sent: "Gate A timed out. Resume with `/cto-loop resume <run-id>`."
- Skill exits. Docker stack (if running) is left as-is.

Resume path: `/cto-loop resume <run-id>` re-loads state, re-emits the gate banner + notification, and re-enters the polling loop with a fresh 24h budget. State file's `poll_count` resets; `notifications_sent` is appended-to, not cleared.

If the operator never resumes, the state file lingers. Cleanup is a separate `/cto-loop cleanup` command (out of scope for v1.7.0). For now: documented in skill `notify-setup.md` as "if you have stale runs, `rm ~/.config/sonzai/cto-runs/<run-id>.json`".

## Failure modes

| Failure | Behavior |
|---|---|
| No `ScheduleWakeup` (not under `/loop`) | Fall back to terminal-sync. Notification still sent (push-only). Operator must come back to terminal. |
| Gmail MCP not enabled, Slack is | Send Slack DM only. Print one-line note in terminal: "Gmail not configured — Slack-only." |
| Slack MCP not enabled, Gmail is | Send Gmail only. Same note. |
| Neither MCP enabled | Print enablement instructions (see MCP detection section). Fall back to terminal-sync. |
| MCP enabled but auth missing ("needs authentication") | Treat as not-enabled. Print enablement instructions with auth note. |
| Notifier subagent crashes mid-send | Log to state file. Retry once. If still failing, skip and proceed (push-only notify is best-effort). |
| Reply-poller subagent times out | Re-schedule same wakeup. State file unchanged. |
| Both Slack DM and Gmail reply arrive between polls | First-seen wins. Process the earlier-by-timestamp action. Discard the other with a one-line note in terminal log. |
| Terminal reply arrives during async wait | Currently impossible — when ScheduleWakeup is in flight, the skill's not in stdin. After resume, the terminal banner gets re-printed; terminal reply still works. |
| Operator approves Gate A but masterplan checkbox in file says rejected | Same as terminal: ignore, re-ask via the same channel that replied. |

## Codex parity

`.codex-plugin/plugin.json` is updated to v1.7.0 with the same skill list. Codex's plugin system (v0.117.0+) supports bundled MCPs via the same `.mcp.json` mechanism. The same file at plugin root serves both Claude Code and Codex.

Codex first-party plugins (`codex_gmail`, `codex_slack`) are detected at runtime — same tool-name-prefix introspection. Operator enables via `codex mcp enable`.

Gemini CLI / Cursor users get the community fallbacks via `.mcp.json` auto-load on plugin enable.

## Hard rules

1. **Notifications are best-effort.** A failed send NEVER blocks the pipeline. Print one line, continue.
2. **Terminal stdin remains an override.** Async polling is additive — the terminal banner still says "Reply here OR via Slack/email."
3. **No tokens / signed messages in v1.7.0.** Trust sender identity. (Reconsider in v2 if abuse appears.)
4. **No transcript / customer / tenant content in any notification body.** Same hard rule as Phase 0a synthesis.
5. **No version pinning in `.mcp.json`.** Use `@latest` or unpinned. Per CLAUDE.md Rule 5 (always-search-current-state).
6. **Notifier + reply-poller live in `full-auto/subagent-prompts/`.** Per CLAUDE.md Rule 6 (shared core).
7. **State file is local-only.** `~/.config/sonzai/` is per-user. Never synced to git, never sent to a server.

## Testing strategy

Lightweight, matching v1.6.0's approach (see prior design doc §17).

- Sanity check: each new file passes a basic shape/lint review.
- Cross-references: every `../full-auto/<file>.md` in `cto-loop/` resolves.
- Manual smoke: operator runs `/cto-loop` against a tiny throwaway transcript on a local branch; verifies Slack DM + Gmail land and the reply gets parsed.
- No full TDD-with-pressure-scenarios in v1.7.0. Subagent-driven implementation will dispatch implementers per task; each implementer follows TDD on its own slice.

## Open items resolved

1. **Run-id format:** `<YYYY-MM-DD>-<HHMM>-<random-4>`. Human-readable + uniqueness without UUIDs.
2. **Slack channel choice:** DM only. No channel posts in v1.7.0.
3. **Email threading:** Use subject-line discriminator `run <run-id>`. Gmail's threading on subject does the rest. Reply-poller matches by `subject:cto-loop` + `from:<recipient>` + the run-id token.
4. **Operator never replies + 24h elapsed:** Pause + notify + exit. Resume command preserved.
5. **MCP "needs authentication" state:** Treat as not-enabled. Print enablement instructions.
6. **Version-checker for `.mcp.json`:** Must run at writing-plans phase to verify package names + invocation flags. No hardcoded versions.

## Out of scope (v1.7.0)

- Cleanup command (`/cto-loop cleanup`) — manual rm for now.
- Multiple recipients (single operator only).
- Slack channel posts.
- SMS / Discord / Teams.
- Phone push notifications.
- Hard-crash notifications.
- Reply tokens / HMAC signing.

These may be added in v1.8+ if usage shows demand.

## File-change manifest

(For the writing-plans phase. All paths relative to repo root.)

### New files

- `plugins/sonzai-sdk/.mcp.json`
- `plugins/sonzai-sdk/skills/full-auto/notify-setup.md`
- `plugins/sonzai-sdk/skills/full-auto/subagent-prompts/notifier.md`
- `plugins/sonzai-sdk/skills/full-auto/subagent-prompts/reply-poller.md`
- `plugins/sonzai-sdk/skills/cto-loop/notify-setup.md`

### Modified files

- `plugins/sonzai-sdk/skills/full-auto/SKILL.md` — add Phase 0-pre to pipeline overview, link notify-setup
- `plugins/sonzai-sdk/skills/full-auto/pipeline.md` — insert Phase 0-pre, notify hooks after qa-loop pass + fail-exhaustion
- `plugins/sonzai-sdk/skills/full-auto/qa-loop.md` — dispatch notifier on terminal events
- `plugins/sonzai-sdk/skills/full-auto/final-report.md.template` — note which notifications fired
- `plugins/sonzai-sdk/skills/cto-loop/SKILL.md` — add Phase 0-pre to overlay table
- `plugins/sonzai-sdk/skills/cto-loop/pipeline.md` — overlay map shows async gate flow
- `plugins/sonzai-sdk/skills/cto-loop/masterplan-gate.md` — async path + ScheduleWakeup branch
- `plugins/sonzai-sdk/skills/cto-loop/cto-review-gate.md` — async path + ScheduleWakeup branch
- `plugins/sonzai-sdk/skills/sonzai-sdk/SKILL.md` — skill-matrix mention of notifications
- `plugins/sonzai-sdk/.claude-plugin/plugin.json` — v1.7.0 + description tweak
- `plugins/sonzai-sdk/.codex-plugin/plugin.json` — v1.7.0 + description tweak
- `plugins/sonzai-internal-staff/.claude-plugin/plugin.json` — v1.7.0 lockstep
- `plugins/sonzai-internal-staff/.codex-plugin/plugin.json` — v1.7.0 lockstep
- `.claude-plugin/marketplace.json` — sonzai-sdk plugin description
- `package.json` — v1.7.0
- `CHANGELOG.md` — v1.7.0 entry
- `README.md` — notification feature mention
- `CLAUDE.md` — add Rule 7 (notification PII security), Rule 8 (always-search applies to `.mcp.json` too)

### Versioning

v1.7.0 (minor bump — additive). v1.6.x users keep working; notifications are opt-in via MCP enablement.
