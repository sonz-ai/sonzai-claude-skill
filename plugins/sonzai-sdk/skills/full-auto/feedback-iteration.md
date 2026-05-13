# Feedback iteration (Phase 4 — autonomous, full-auto)

When QA fails in Phase 3, dispatch a fixer subagent to address the failure, rebuild, and re-run QA. Bounded loop.

This file describes the AUTONOMOUS variant. For operator-feedback-driven iteration (Gate B → fixer), see `../cto-loop/feedback-iteration.md`.

## When to use

Reached from `qa-loop.md` when `overall: fail` after smoke or archetype-specific checks.

## Loop structure

```
cycle = 0
while qa.overall == fail and cycle < 5:
  dispatch fixer (subagent-prompts/fixer.md) with:
    - masterplan path
    - QA report (especially failing_checks)
    - relevant logs (docker compose logs --tail 100)
    - cycle counter
  fixer commits its changes
  docker compose up -d --build
  re-run qa-loop.md
  cycle += 1

if qa.overall == pass:
  proceed to final-report
else:
  proceed to final-report with status: qa-failing-after-5-fixes
```

## Cycle budget

5 fixer cycles, hard cap. Tracked in in-memory state as `auto_fix_cycles`.

After 5: STOP. Don't keep iterating. The final-report explicitly lists which QA checks remain failing.

## Fixer dispatch prompt

```
You are the fixer subagent for cto-loop / full-auto. The QA loop just failed.

Masterplan:        ${MASTERPLAN_PATH}
Failing checks:    ${FAILING_CHECKS}
Recent logs (last 100 lines of docker compose logs):
${LOGS}

QA report YAML:
${QA_REPORT_YAML}

Cycle: ${CYCLE} of 5

Diagnose the failure. Apply the minimum fix. Commit. Rebuild won't restart for you — the parent will run `docker compose up -d --build` after you return.

Hard rules:
1. Same as builder: never write versions from memory; if you change a manifest, run version-checker first
2. Don't introduce new features. Minimum fix for the failing check.
3. If the failure is a missing env var, do NOT auto-generate a value — leave the field empty in .env.example AND document the requirement in commit message
4. If the failure looks fundamental (e.g., the masterplan asked for an impossible combination), return status: blocked with explanation

Return JSON:
{
  "status": "fixed" | "blocked",
  "commits": [...],
  "files_touched": [...],
  "fix_description": "one line of what you changed and why",
  "blocker": "<if blocked>"
}
```

Use a Sonnet-class model for the fixer. Opus is overkill for narrow bug fixes; Haiku is under-powered.

## Auto-rebuild

After fixer returns `status: fixed`:

```bash
docker compose down
docker compose up -d --build --wait
```

Then re-run `qa-loop.md`. The healthcheck `--wait` flag is critical here — without it, QA may race the boot.

If the rebuild itself fails (compose exits non-zero), treat as a `fixed` that didn't actually fix. Increment cycle counter, dispatch fixer again with the new failure logs.

## Stopping conditions

- `qa.overall == pass` → exit loop, proceed to final-report
- `cycle >= 5` → exit loop with status `qa-failing-after-5-fixes`
- Fixer returns `status: blocked` → exit loop with status `fixer-blocked` (don't keep dispatching)

## Final-report inputs

When the loop exits, save to in-memory state:

```yaml
fix_loop:
  cycles_used:       <0..5>
  fixes_applied:     [<list of fix_description from each cycle>]
  final_status:      passed | qa-failing-after-5-fixes | fixer-blocked
  remaining_failing: [<list of QA checks still failing>]
  blocker:           <if fixer-blocked>
```

Final-report references this section prominently.

## Hard rules

1. **5 cycles, hard cap.** No "one more try". After 5, stop.
2. **Always-search propagates.** Fixer follows the same rule as builder — no manifest writes from memory.
3. **Minimum fix.** Fixer addresses ONLY the failing checks, not unrelated cleanup.
4. **No secrets generated.** If a missing env var is the cause, fixer documents the requirement; operator fills `.env` post-run.
5. **No new features.** Fixer is a band-aid, not a re-spec.
6. **Failure is documented, not hidden.** When the loop gives up at cycle 5, the final-report says so prominently.
