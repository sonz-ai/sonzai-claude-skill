# Masterplan gate (Gate A)

Operator approval of the masterplan doc before any build work. **This gate is non-optional.**

The gate has TWO modes depending on whether async notify is set up:

- **Async mode** — default when Phase 0-pre set `notify_enabled = true` AND the `ScheduleWakeup` tool is available. Print banner, dispatch notifier, save run state, ScheduleWakeup, exit turn. On wakeup, dispatch reply-poller; resume on match or reschedule.
- **Sync mode** — fallback. Print banner, wait for terminal input only. Original v1.6.0 behavior.

## When to use

Reached after `../full-auto/masterplan-assembly.md` has written the masterplan file.

## Print (both modes)

```
─────────────────────────────────────────────────────
🚪 GATE A — MASTERPLAN APPROVAL

File:    docs/cto-review/<YYYY-MM-DD>-masterplan.md
Mode:    <greenfield | brownfield>
Sonzai:  <archetype>, <runtime-mode>, [<capabilities>]
Stack:   <backend> + <frontend> + <db> + <auth>
Scope:   <comma-list>
Risks:   <count> identified
Verified: <date>

Reply with one of:
  approve         → I proceed to build (Phase 2)
  edit            → I pause; you edit the file; type 'done'
  reject <txt>    → I re-derive with your feedback and re-emit
  abort           → stop, leave masterplan in place

[Async note (only printed in async mode):]
  Also notified via Slack DM + Gmail. Reply through any channel.
  Polling every 20min for up to 24h.
─────────────────────────────────────────────────────
```

## Mode selection

At Gate A entry:

1. Read in-memory state: is `notify_enabled` true (from Phase 0-pre)?
2. Check tool list: is `ScheduleWakeup` available?
3. If BOTH true → **async mode**. Else → **sync mode**.

## Async mode

### Step 1: Notify

Dispatch `../full-auto/subagent-prompts/notifier.md`:

```yaml
event:        gate_a
run_id:       <run-id>
recipient:    <from state>
subject:      "[cto-loop] Gate A — Masterplan ready — run <run-id>"
body: |
  🚪 cto-loop Gate A — Masterplan ready for your review.

  Run:     <run-id>
  File:    <masterplan path>
  Mode:    <greenfield|brownfield>
  Stack:   <stack summary>
  Scope:   <one-line>

  Reply with:
    approve
    reject <reason>
    edit             (edit the file directly, then reply 'approve')
    abort

  (Waiting up to 24h. Polling every 20min.)
interactive:  true
```

Record returned IDs in state.

### Step 2: Save state + schedule wakeup

Save run state to `~/.config/sonzai/cto-runs/<run-id>.json` (atomic write: `.tmp` + rename):

```json
{
  "run_id": "<run-id>",
  "started_at": "<iso8601>",
  "base_sha": "<sha>",
  "mode": "cto-loop",
  "masterplan_path": "<path>",
  "current_phase": "gate_a",
  "recipients": { "gmail": "<...>", "slack_user_id": "<...>" },
  "notifications_sent": [ /* from notifier */ ],
  "first_poll_at": "<iso8601>",
  "last_poll_at": "<iso8601>",
  "poll_count": 0,
  "timeout_at": "<iso8601 + 24h>",
  "last_seen": { "slack_ts": "0", "gmail_internal_date": 0 },
  "unparseable_count": 0,
  "reject_count": 0,
  "paused": false
}
```

Call `ScheduleWakeup`:

```yaml
delaySeconds: 1200
reason:       "cto-loop Gate A waiting for reply"
prompt:       "<original /loop prompt — re-fires cto-loop, which detects state file and resumes>"
```

Exit this turn.

### Step 3 (on wakeup): Dispatch reply-poller

When the agent re-enters via wakeup, it should detect the existing state file at `~/.config/sonzai/cto-runs/<run-id>.json` with `current_phase = gate_a`, and dispatch `../full-auto/subagent-prompts/reply-poller.md`:

```yaml
run_id:    <run-id>
gate:      gate_a
recipient: <from state>
last_seen: <from state.last_seen>
grammar:   gate_a
```

### Step 4: Process result

Based on poller's return:

- `result: "matched", action: "approve"` → process as "approve" (see Action handling below). Clear `current_phase`, advance to Phase 2.
- `result: "matched", action: "reject", text: "<reason>"` → process as "reject <reason>" (re-derive).
- `result: "matched", action: "edit"` → print banner "Operator chose to edit. Pause until file is approved." Re-schedule wakeup at 600s (10min, faster cadence for edit mode); on next wake, just re-poll for `approve` / `abort` / next `edit`.
- `result: "matched", action: "abort"` → process as "abort".
- `result: "matched", action: "unparseable"` → increment `state.unparseable_count`. If `unparseable_count >= 3` (third consecutive unparseable): dispatch notifier with a clarification body:
  ```
  Your reply didn't match the expected grammar. Reply with one of:
    approve / reject <reason> / edit / abort
  ```
  Then re-schedule. Else: silently re-schedule.
- `result: "none"` → check timeout. If `now >= timeout_at`: pause (Step 5). Else: re-schedule wakeup, update `last_poll_at` + increment `poll_count`.

### Step 5: Timeout / pause

If `now >= timeout_at`:

1. Dispatch notifier:
   ```yaml
   event:    gate_a_paused
   subject:  "[cto-loop] Gate A timed out — run <run-id>"
   body: |
     ⏸ cto-loop Gate A paused after 24h with no reply.
     Resume with /cto-loop resume <run-id> when ready.
   interactive: false
   ```
2. Set `state.paused = true`, save state (atomic write).
3. Print to terminal: `Gate A timed out at 24h. Resume with /cto-loop resume <run-id>.`
4. Exit. Do NOT delete state.

## Sync mode

(v1.6.0 behavior — unchanged.)

Print banner, then wait for terminal input. Parse against the same grammar (approve / edit / reject `<txt>` / abort). Proceed per Action handling below.

## Action handling (used by BOTH modes)

### `approve`

1. Mark the masterplan's approval checkbox: replace `- [ ] approved` with `- [x] approved` in the masterplan file.
2. Save the masterplan path to in-memory state.
3. Clear async state file's `current_phase` (move on). For sync mode, no state file to clear.
4. Proceed to `../full-auto/builder-dispatch.md` (Phase 2 — shared with full-auto).

### `edit`

1. (Sync mode) Print: "Paused. Edit `<file>` directly. Type `done` when ready."
   (Async mode) Print: "Pause for edit; re-poll cadence reduced to 10min until you reply `approve`."
2. Wait / re-poll until `approve` or `reject` or `abort`.
3. On `approve`: re-read file, re-print banner, re-ask. Loop until terminal action.

### `reject <txt>`

1. Capture rejection text into state as `gate_a_rejection`.
2. Increment `state.reject_count`.
3. Go back to `../full-auto/answer-derivation.md` with rejection text added to inputs.
4. Re-run `../full-auto/masterplan-assembly.md` — overwrite same masterplan file.
5. Return to Gate A for re-approval.
6. **Bounded: 3 rejection cycles.** After 3, escalate: "Three rejections — the masterplan I produce is not matching what you want. Could you draft the masterplan yourself, then type `done` so I can read it and build from your version?"

### `abort`

1. Print: "Aborting. Masterplan left at `<path>`. No build performed."
2. (Async mode) Set `state.paused = true, aborted = true`, save state.
3. Exit.

## Detecting tampering (carried forward from v1.6.0)

If the operator says `approve` but the approval checkbox at the bottom of the masterplan file already reads `- [x] rejected`, ignore and re-ask. The file's state of record is the approval section; the chat instruction must agree with it.

If the operator edited the file but did NOT update the approval line, ask: "I see you edited the file but the approval section still reads `- [ ] approved`. Mark it approved or tell me what to do?"

## Hard rules

1. **Mode auto-selected, not user-selected.** Async is on whenever the prerequisites are met. No flag.
2. **No skip-flag.** No `cto_loop_skip_gate_a` env var, no `auto-approve` mode. Gate A is what makes this skill `cto-loop` and not `full-auto`.
3. **`approve` is explicit** in BOTH modes. No "lgtm" / "looks good" auto-approve. Same for async replies (strict keyword grammar in `reply-poller.md`).
4. **Edit happens in the file**, not in chat. If operator dictates changes in chat, YOU edit the file (with explicit Edit calls the operator can see), then re-ask.
5. **Bounded reject cycles.** 3 max, same as terminal mode. Don't grind forever.
6. **Terminal input is always honored in BOTH modes.** Even in async mode, terminal stdin during a wakeup interval works as override.
7. **Never modify the masterplan's "approval" section yourself** except to flip the checkbox from `[ ]` to `[x]` after `approve`. The operator's review notes are theirs.
8. **State file is local-only.** `~/.config/sonzai/cto-runs/` is per-user. Never committed.
