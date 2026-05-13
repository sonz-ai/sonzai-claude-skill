# Feedback iteration (Phase 4 — cto-loop, operator-driven)

**Operator-driven variant of `../full-auto/feedback-iteration.md`.** In full-auto, the fixer loop runs autonomously on QA failures. In cto-loop, the fixer loop runs on **operator free-text feedback** captured at Gate B.

## When to use

Reached from `cto-review-gate.md` when the operator replies `feedback <txt>` instead of `approve` or `abort`.

## Loop structure

```
cycle = 0
while operator_action == "feedback":
  cycle += 1
  if cycle > 5:
    print "5 cycles done — operator must decide"
    re-ask at Gate B with constrained options (approve / abort / one more)
  dispatch fixer (../full-auto/subagent-prompts/fixer.md) in cto-feedback-fixer mode with:
    - operator feedback text (from Gate B)
    - masterplan path
    - current diff (git log <base>..HEAD)
    - live app URL
    - cycle counter
  fixer commits its changes
  docker compose up -d --build --wait
  re-run ../full-auto/qa-loop.md (smoke + functional)
  return to cto-review-gate.md (Gate B) with updated state
```

## Cycle budget

5 fixer cycles, hard cap. Tracked as `cto_loop_cycles`.

After 5: STOP the normal feedback loop. Gate B re-asks with constrained options — see `cto-review-gate.md`.

After 6 (operator typed `one more` at cycle 5): hard stop. No further `feedback` option.

## Fixer dispatch — cto-feedback-fixer mode

The fixer is the same subagent as full-auto's. The `Mode:` field in the dispatch prompt distinguishes:

```
You are the fixer subagent. Mode: cto-feedback-fixer.

Operator feedback (free-text):
  ${FEEDBACK_TEXT}

Masterplan:        ${MASTERPLAN_PATH}
Live app URL:      ${APP_URL}
Recent commits:    ${COMMIT_LIST}
QA status:         ${QA_STATUS}
Cycle:             ${N} of 5

Apply the minimum fix that addresses the operator's feedback. If the feedback is ambiguous OR requests something out-of-scope of the masterplan, return status: blocked with "operator requested <X> — Gate B + new masterplan iteration recommended". Don't ship a wrong-intent fix.

Return JSON per the fixer's standard schema (see ../full-auto/subagent-prompts/fixer.md).
```

## What the fixer does NOT do

- Add new features beyond the masterplan's scope — that's a Gate A re-iteration, not a fix
- Touch unrelated code — minimum fix only
- Generate secrets — env vars are documented, operator fills `.env`
- `git push` — local commits only

## Auto-rebuild after fixer returns

```bash
docker compose down       # NO -v flag — preserve volumes (postgres data)
docker compose up -d --build --wait
```

Then re-run QA. Then re-enter Gate B for the operator's next decision.

## Bounded cycles + escalation

At cycle 5 with operator still typing `feedback`:

Print the warning banner from `cto-review-gate.md` ("5 cycles reached") and constrain options to `approve` / `abort` / `one more <txt>`. If `one more`: do exactly one more cycle, then HARD STOP — no `feedback` option offered after.

If operator types `abort` at any cycle: leave the app running, write a brief final report with `cto_loop_status: aborted-by-operator` + the feedback log, exit.

## Final-report inputs

When the loop exits (via `approve` or `abort` or hard stop):

```yaml
operator_feedback_loop:
  cycles_used:      <0..6>
  feedback_log:     [<per cycle: operator feedback text + fixer fix_description>]
  final_status:     approved | aborted-by-operator | hard-stop-at-6
  approval_cycle:   <which cycle was approved, if any>
```

Goes into the final report alongside the autonomous run details.

## Hard rules

1. **5 cycles + 1 explicit `one more` = absolute max.** No "just one more" past cycle 6.
2. **Always-search propagates.** Same as the builder rule — no manifest writes from memory.
3. **Minimum fix.** Don't expand scope. If operator's feedback implies new scope, return `blocked` and let Gate B re-route to a new masterplan iteration.
4. **No automatic teardown.** Docker compose stays running across cycles (volumes preserved). Operator decides when to `down`.
5. **Feedback is per-cycle.** Each cycle gets its OWN feedback text — don't aggregate feedback across cycles.
6. **Don't `git push`.** Same as full-auto — operator owns the push.
