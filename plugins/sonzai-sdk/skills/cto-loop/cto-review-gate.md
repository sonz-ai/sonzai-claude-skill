# CTO review gate (Gate B)

After Phase 3 deploy + QA, present the live app + diff + summary to the operator. **This gate is non-optional.**

## When to use

Reached from `qa-loop.md` (success path or after 5 fixer cycles).

## Print

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

## Action handling

### `approve`

1. Print `Final report: docs/cto-review/${DATE}-final-report.md`
2. Write the final report using `../full-auto/final-report.md.template`
3. Save state: `cto_loop_status = "approved"`
4. Print: "cto-loop complete. App is running at ${APP_URL}. Push when ready (operator owns)."
5. Do NOT `docker compose down`. Leave the stack running so operator can keep exploring.
6. Exit.

### `feedback <txt>`

1. Append the feedback to `docs/cto-review/<date>-feedback-log.md` with cycle number + timestamp
2. Read `feedback-iteration.md` next (Phase 4)

### `abort`

1. Print: "Aborting cto-loop. Docker compose is still running (`docker compose down` to stop it). Commits stay on the current branch — push or reset as you prefer."
2. Save state: `cto_loop_status = "aborted"`
3. Exit. NO final report. NO automatic cleanup.

### Ambiguous response (e.g., "looks good" or "thanks")

Re-ask. Do NOT interpret an informal positive as `approve`. The keywords `approve` / `feedback` / `abort` are explicit for a reason — `approve` puts a checkbox-style stamp on the artifact, and that needs deliberate intent.

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

1. **`approve` is explicit.** No interpreting "looks good" as approve.
2. **Live URL is part of the print.** The whole point of Gate B is the operator clicks the URL — don't omit it.
3. **QA status is honest.** If QA failed and you reached Gate B with 5 retries exhausted, say so prominently.
4. **No automatic teardown.** `docker compose down` is operator's call. Leaving it running supports continued exploration.
5. **Bounded cycles.** 5 + 1 = absolute max. After cycle 6, the operator forces a decision.
6. **Don't `git push` after approve.** Final report says "push when ready" — operator owns.
