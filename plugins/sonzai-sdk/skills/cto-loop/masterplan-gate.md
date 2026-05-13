# Masterplan gate (Gate A)

Operator approval of the masterplan doc before any build work. **This gate is non-optional. No skip-flag exists.**

## When to use

Reached after `masterplan-assembly.md` has written the masterplan file.

## Print

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
─────────────────────────────────────────────────────
```

## Action handling

### `approve`

1. Mark the masterplan's approval checkbox: replace `- [ ] approved` with `- [x] approved`.
2. Save the masterplan path to in-memory state.
3. Proceed to `../full-auto/builder-dispatch.md` (Phase 2 — shared with full-auto).

### `edit`

1. Print: "Paused. Edit `docs/cto-review/<date>-masterplan.md` directly. Type `done` when ready. I will re-read and ask again."
2. Wait for `done`. Do not poll, do not preempt.
3. On `done`: re-read the file. Re-print the summary banner. Re-ask the approve / edit / reject question.
4. Loop until `approve` or `reject`.

Alternative within `edit`: operator may say "I want section X changed to Y" — then YOU edit the file (using Edit tool), print the diff, and re-ask. Don't make changes the operator didn't authorize.

### `reject <txt>`

1. Capture the rejection text into in-memory state as `gate_a_rejection`.
2. **Do NOT loop forever.** Go back to `../full-auto/answer-derivation.md` with the rejection text added to the inputs.
3. Re-derive the 8 wizard answers (which may change based on the feedback).
4. Re-run `../full-auto/masterplan-assembly.md` — overwriting the same masterplan file.
5. Return here (Gate A) for re-approval.
6. Bound: 3 rejection cycles. After 3, escalate: "Three rejections — the masterplan I produce is not matching what you want. Could you draft the masterplan yourself, then type `done` so I can read it and build from your version?"

## Detecting tampering

If the operator says `approve` but the approval checkbox at the bottom of the file already reads `- [x] rejected`, ignore and re-ask. The file's state of record is the approval section; the chat instruction must agree with it.

If the operator edited the file but did NOT update the approval line, ask: "I see you edited the file but the approval section still reads `- [ ] approved`. Mark it approved or tell me what to do?"

## Hard rules

1. **No skip-flag.** No `cto_loop_skip_gate_a` env var, no `auto-approve` mode. Gate A is what makes this skill `cto-loop` and not `full-auto`.
2. **`approve` is explicit.** Pressing enter on an empty line, or saying "ok" or "ya" — does NOT count as approve. Re-ask if the response is ambiguous.
3. **Edit happens in the file**, not in chat. If operator dictates changes in chat, YOU edit the file (with explicit Edit calls the operator can see), then re-ask.
4. **Reject loops are bounded.** 3 cycles then escalate. Don't grind forever.
5. **Never modify the masterplan's "approval" section yourself** except to flip the checkbox from `[ ]` to `[x]` after `approve`. The operator's review notes in section 9 are theirs.
