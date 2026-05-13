# CTO review gate (Gate B)

After Phase 3 deploy + QA, present the live app + diff + summary to the operator. **This gate is non-optional.**

Two modes (same selection as Gate A): **async** (when Phase 0-pre set `notify_enabled = true` AND `ScheduleWakeup` is available) and **sync** (terminal-only fallback).

## When to use

Reached from `../full-auto/qa-loop.md` (success path or after 5 fixer cycles).

## Mode selection

At Gate B entry:
1. Read in-memory state: `notify_enabled` true?
2. Tool list: `ScheduleWakeup` available?
3. BOTH true → **async mode**. Else → **sync mode**.

## Print (both modes)

```
─────────────────────────────────────────────────────
🚪 GATE B — CTO REVIEW (cycle ${CYCLE_N} of 5)

Live app:    ${APP_URL}
Commits:     ${BASE_SHA}..HEAD (${COMMIT_COUNT} commits)
Built files: ${FILE_COUNT}
QA status:   ${QA_STATUS}        # passing | failing on <checks>
Time taken:  ${ELAPSED}

What was built (synthesis):
  ${THREE_PARA_SUMMARY}

Reply with one of:
  approve         → final report, done
  feedback <txt>  → I dispatch fixer, re-deploy, return here
  abort           → stop, leave docker-compose running for inspection

[Async note (async mode only):]
  Also notified via Slack DM + Gmail. Reply through any channel.
  Polling every 20min for up to 24h.
─────────────────────────────────────────────────────
```

## Building the 3-paragraph synthesis

Paragraph 1: Mapping to masterplan. "Built per masterplan §X. The architecture matches the planned <X-layer-Y-stack>. Sonzai integration uses <runtime mode> against the <archetype> archetype with <capabilities>."

Paragraph 2: What's actually working. "End-to-end flow tested: <list of QA checks that passed>. The <key user flow> is functional at <URL/route>."

Paragraph 3: What's NOT working or partial. "Not yet implemented / failing / deferred: <list>. <Specific issues found in QA, if any>."

Be honest. If QA failed, paragraph 3 must surface that prominently.

## Capturing commits + files

```bash
# commits since this cto-loop run started:
git log ${BASE_SHA}..HEAD --oneline

# files changed:
git diff --name-only ${BASE_SHA}..HEAD | wc -l
```

`BASE_SHA` is captured at the start of Phase 2 (before any builder commits).

## Async mode

### Step 1: Notify

Dispatch `../full-auto/subagent-prompts/notifier.md`:

```yaml
event:        gate_b
run_id:       <run-id>
recipient:    <from state>
subject:      "[cto-loop] Gate B — Live app ready (cycle <N>) — run <run-id>"
body: |
  🚪 cto-loop Gate B — Live app ready (cycle <N>/5).

  Run:        <run-id>
  Live URL:   <APP_URL>
  Commits:    <BASE_SHA>..HEAD (<N> commits)
  Built files: <FILE_COUNT>
  QA status:  <QA_STATUS>
  Time taken: <ELAPSED>

  What was built:
  <THREE_PARA_SUMMARY>

  Reply with:
    approve              → final report, done
    feedback <text>      → I dispatch fixer, re-deploy, return here
    one more <text>      → only valid after cycle 5; force-extra cycle
    abort                → stop, leave docker compose running

  (Waiting up to 24h. Polling every 20min.)
interactive:  true
```

Record returned message IDs in run state.

### Step 2: Save state + schedule wakeup

Update run state file `~/.config/sonzai/cto-runs/<run-id>.json` (atomic write):

```json
{
  "run_id": "<run-id>",
  "current_phase": "gate_b",
  "cycle_n": <N>,
  "notifications_sent": [ ..., { event: gate_b, ... } ],
  "first_poll_at": "<iso8601>",
  "last_poll_at": "<iso8601>",
  "poll_count": 0,
  "timeout_at": "<iso8601 + 24h>",
  "last_seen": { "slack_ts": "0", "gmail_internal_date": 0 },
  "unparseable_count": 0
}
```

Call `ScheduleWakeup`:
```yaml
delaySeconds: 1200
reason:       "cto-loop Gate B cycle <N> waiting for reply"
prompt:       "<original /loop prompt>"
```

Exit this turn.

### Step 3 (on wakeup): Dispatch reply-poller

Dispatch `../full-auto/subagent-prompts/reply-poller.md`:
```yaml
run_id:    <run-id>
gate:      gate_b
recipient: <from state>
last_seen: <from state>
grammar:   gate_b
```

### Step 4: Process result

- `action: "approve"` → process as approve (see Action handling). Additionally dispatch the final-report notification (see below).
- `action: "feedback", text: "<txt>"` → process as feedback (append to log + dispatch fixer + re-enter Gate B). Reset `last_seen`, increment `cycle_n`.
- `action: "one_more", text: "<txt>"` → only valid when `cycle_n >= 5`. Bypass cycle limit once. Process as feedback.
- `action: "abort"` → dispatch abort notification (see below), process as abort.
- `action: "unparseable"` → increment `unparseable_count`. If `>= 3`: dispatch notifier with clarification body; re-schedule. Else: silently re-schedule.
- `result: "none"` → check timeout. If `now >= timeout_at`: pause (Step 5). Else: re-schedule wakeup.

### Step 5: Timeout / pause

If `now >= timeout_at`:

1. Dispatch notifier:
   ```yaml
   event:    gate_b_paused
   subject:  "[cto-loop] Gate B timed out — run <run-id>"
   body: |
     ⏸ cto-loop Gate B paused after 24h with no reply.
     Resume with /cto-loop resume <run-id> when ready.
   interactive: false
   ```
2. Set `state.paused = true`, save state.
3. Print: `Gate B timed out at 24h. Resume with /cto-loop resume <run-id>.`
4. Exit. Docker stack stays up.

## Sync mode

(v1.6.0 behavior — unchanged.) Print banner, wait for terminal input. Parse against `approve` / `feedback <txt>` / `abort` / `one more <txt>` (cycle 6 only). Same grammar as async.

## Action handling (used by BOTH modes)

### `approve`

1. Print `Final report: docs/cto-review/${DATE}-final-report.md`.
2. Write the final report using `../full-auto/final-report.md.template`.
3. (Async mode) Dispatch final-report notification:
   ```yaml
   event:        final_report
   subject:      "[cto-loop] Complete (approved) — run <run-id>"
   body: |
     ✅ cto-loop complete (approved).

     Run:          <run-id>
     Live URL:     <APP_URL>
     Report:       docs/cto-review/<date>-final-report.md
     Total cycles: <CYCLE_N>

     App is still running (docker compose). Push when ready.
   interactive:  false
   ```
4. Save state: `cto_loop_status = "approved"`, `paused = false`. (Async mode writes state file; sync mode in-memory only.)
5. Print: "cto-loop complete. App is running at ${APP_URL}. Push when ready (operator owns)."
6. Do NOT `docker compose down`. Leave the stack running so operator can keep exploring.
7. Exit.

### `feedback <txt>`

1. Append the feedback to `docs/cto-review/<date>-feedback-log.md` with cycle number + timestamp.
2. Read `feedback-iteration.md` next (Phase 4) — operator-feedback-driven fixer dispatch.

### `one more <txt>` (cycle 6 only)

Same handling as `feedback`. Mark cycle 6 as the final cycle — no more iterations after.

### `abort`

1. (Async mode) Dispatch abort notification:
   ```yaml
   event:        gate_b_aborted
   subject:      "[cto-loop] Aborted at Gate B — run <run-id>"
   body: |
     🛑 cto-loop aborted at Gate B.
     docker compose still running. Inspect or `docker compose down`.
   interactive:  false
   ```
2. Print: "Aborting cto-loop. Docker compose is still running (`docker compose down` to stop it). Commits stay on the current branch — push or reset as you prefer."
3. Save state: `cto_loop_status = "aborted"`.
4. Exit. NO final report. NO automatic cleanup.

### Ambiguous response

Re-ask. Do NOT interpret an informal positive as `approve`. The keywords `approve` / `feedback <txt>` / `abort` / `one more <txt>` are explicit for a reason — `approve` puts a checkbox-style stamp on the artifact, and that needs deliberate intent.

## Cycle counter

Track `CYCLE_N`. Increment by 1 each time you re-enter Gate B via `feedback`. Starts at 1.

When `CYCLE_N >= 5` AND the operator gives more feedback:

```
─────────────────────────────────────────────────────
⚠️  5 CYCLES REACHED

I've iterated 5 times. Continuing risks endless churn.

Reply with one of:
  approve            → I accept where it is and write the final report
  abort              → I stop; you take it from here
  one more <txt>     → genuinely-last cycle; after this I will not iterate again
─────────────────────────────────────────────────────
```

`one more <txt>` is the only way past 5. After that 6th cycle, Gate B reads:

```
🛑 6 CYCLES DONE — I will not iterate further.

  approve   → final report
  abort     → stop
```

No more `feedback` option. Operator decides.

## Hard rules

1. **`approve` is explicit in BOTH modes.** No interpreting "looks good" as approve.
2. **Live URL is part of the print.** The whole point of Gate B is the operator clicks the URL — don't omit it.
3. **QA status is honest.** If QA failed and you reached Gate B with 5 retries exhausted, say so prominently.
4. **No automatic teardown.** `docker compose down` is operator's call. Leaving it running supports continued exploration.
5. **Bounded cycles.** 5 + 1 = absolute max. After cycle 6, the operator forces a decision.
6. **Don't `git push` after approve.** Final report says "push when ready" — operator owns.
7. **Final-report notification fires on approve only.** Not on feedback / abort.
8. **State file is local-only** (same as Gate A).
