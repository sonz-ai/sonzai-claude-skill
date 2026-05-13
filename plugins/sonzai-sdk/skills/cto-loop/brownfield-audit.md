# Brownfield audit (Phase 0c — brownfield, with operator confirmation)

**Wrapper around `../full-auto/brownfield-audit.md`.** Runs the autonomous auditor, then presents findings to the operator for row-by-row confirmation before the masterplan locks in.

Used when `cto-loop` is the active skill and project-type-detection returned `brownfield`.

## When to use

Reached from `project-type-detection.md` when the CWD has prior signals.

## Flow

1. **Dispatch the auditor subagent** (`../full-auto/subagent-prompts/auditor.md`) with the operator's CWD + transcript context. Same dispatch as `full-auto` does.
2. **Receive the YAML audit** with confidence ratings per layer.
3. **Print the findings as a table** for operator review (see format below).
4. **Loop on `fix <row>` / `details <row>` until operator types `confirm`.**
5. **Save the confirmed audit** to `docs/cto-review/<date>-brownfield-context.md`.
6. **Proceed to `../full-auto/answer-derivation.md`** (Phase 0d).

## Print

```
Brownfield audit:

  Layer        Detected                              Confidence
  ─────        ────────                              ──────────
  Backend      <fwk> v<X.Y> on <lang>                high
  Frontend     <fwk> v<X.Y>                          high
  Database     <db> <tag>                            high
  ORM          <orm> v<X.Y>                          medium
  Auth         <auth>                                LOW    ⚠️
  Tests        <test fwk>                            high
  Sonzai SDK   v<X.Y> (<integration>)                high

Notable patterns:
  - <pattern>
  - <pattern>

Risks:
  - <risk>

Reply with one of:
  confirm                              → all rows correct, proceed
  fix <row>: <new value>               → correct one row (e.g., "fix Auth: custom-jwt")
  details <row>                        → show what files the row was inferred from
```

## Action handling

### `confirm`

1. Mark in-memory state: `audit.confirmed = true`, `audit.confirmed_at = <date>`
2. Save the (possibly edited) audit to `docs/cto-review/<date>-brownfield-context.md`
3. Proceed to Phase 0d (`../full-auto/answer-derivation.md`)

### `fix <row>: <new value>`

1. Update that row in the in-memory audit
2. Set `confidence: high` on the corrected row (operator stated it explicitly)
3. Remove the corresponding `risks:` entry if there was one (the risk was that we'd misdetected, now overridden by operator)
4. Reprint the audit table
5. Re-ask: `confirm` / `fix` / `details`

### `details <row>`

1. Print which files the row was inferred from (the auditor's `notable_patterns` and `notes` for that layer)
2. Re-ask: `confirm` / `fix` / `details`

### Ambiguous response (e.g., "looks ok", "yes")

Re-ask. `confirm` is the explicit keyword.

## Why this is a separate file from `../full-auto/brownfield-audit.md`

`full-auto` runs the same auditor subagent but skips the operator confirmation step. The autonomous version writes findings directly to `brownfield-context.md` (with low-confidence rows turning into risks). The cto-loop variant adds the confirmation UX layer on top of that — same audit subagent, same detection logic, different presentation + handoff.

## Hard rules

1. **Same auditor.** Don't fork the auditor subagent — use `../full-auto/subagent-prompts/auditor.md` verbatim. Operator confirmation happens AFTER, not during, audit.
2. **`confirm` is explicit.** No interpreting positive-sounding messages as confirm.
3. **`fix` corrections become `confidence: high`.** Operator's explicit statement supersedes detection.
4. **Read-only.** Same as the underlying auditor — never edit operator's source files, only write `brownfield-context.md`.
5. **No tenant names in output.** Should already be redacted by auditor; verify before writing the file.
