---
name: full-auto
description: Use when given a meeting/sales/product transcript (file path or pasted text) and asked to ship a working Sonzai SDK integration end-to-end with zero human prompts. Reads transcript, derives wizard answers, dispatches a builder subagent that runs the sonzai-sdk skill, then exercises the built app (API + UI) and re-dispatches the builder with concrete failure lists until the app passes. Bounded loop, fully autonomous, no questions back to the operator.
---

# full-auto

**Closed-loop autonomous Sonzai implementer.** Transcript in → working repo out. Zero prompts to the operator.

## When to use

- A meeting / sales / product / discovery transcript exists (file path, pasted text, or sitting in the prior message)
- The operator wants end-to-end build without being interrupted for wizard questions
- The output should be a *running, exercised* app — not just a spec

## When NOT to use

- The user can answer questions interactively → use `sonzai-sdk` (wizard) instead
- One-line ask, no transcript → use `sonzai-sdk`
- Transcript clearly describes something the Sonzai SDK doesn't do (e.g., trading bots, video editing, payment processing) → halt with a written reason. Do not improvise an unrelated build.

## Triggers

The user invoked this skill if any of these match:

- Operator pastes a transcript and says "build this" / "go full auto" / "run full-auto" / "no questions, just build it"
- Operator runs `/full-auto <path>` (file path arg) or `/full-auto` with transcript inline
- Operator's prompt mentions "transcript" + "build" / "implement" / "ship"
- Operator's prompt says "from this meeting we just had" + Sonzai mention

## Hard rules

1. **Never ask the operator a question.** All ambiguity resolves to a documented assumption in the spec.
2. **Bounded fix-loop.** Maximum 5 fix cycles. If still failing, stop and write `.full-auto/BLOCKED.md` with what's failing and why.
3. **Local commits only.** Never `git push`, never `gh pr create`. That decision belongs to the operator.
4. **Drift check first.** Same as `sonzai-sdk` Step 0 — fetch live OpenAPI before anything else.
5. **No invented SDK symbols.** Every Sonzai call must verify against the live OpenAPI at `https://api.sonz.ai/docs/openapi.json` or the public SDK source.
6. **No mock SDK calls.** Use the real SDK against a real (or operator-provided) `SONZAI_API_KEY`. If absent, write `.env.example` and document.

## Pipeline (6 phases)

→ Full detail: `pipeline.md`

| # | Phase | Output |
|---|---|---|
| 0 | Drift check | `.full-auto/openapi.live.json` |
| 1 | Ingest transcript | `.full-auto/transcript.txt` |
| 2 | Signal extraction → `transcript-analysis.md` | `.full-auto/signals.md` |
| 3 | Wizard-answer derivation → `answer-derivation.md` | `.full-auto/wizard-answers.md` |
| 4 | Builder subagent dispatch → `builder-dispatch.md` | spec + plan + impl, local commits |
| 5 | QA loop (test, fix, repeat) → `qa-loop.md` | `.full-auto/qa-cycle-N.md` × N |
| 6 | Final report → `final-report.md.template` | `.full-auto/REPORT.md` |

## Required cross-skill dependencies

- `sonzai-sdk` (sibling skill in this plugin) — archetype playbooks, feature references, spec templates
- `superpowers:brainstorming` (used by builder subagent for spec self-review)
- `superpowers:writing-plans` (builder subagent)
- `superpowers:subagent-driven-development` (builder subagent for plan execution)

## Optional dependencies (degrades gracefully if absent)

- Browser MCP for UI testing: `mcp__chrome-devtools__*` OR `mcp__playwright__*`
  - If neither is loaded → UI tests fall back to "API contract exercised only; UI not driven"
  - If present → drives the UI in a headless browser, captures screenshots + console errors

## Red flags (stop and halt)

- Transcript names a competitor / non-Sonzai product as the target stack → halt, do not build
- Transcript asks for game state / billing / payments / video editing / trading → halt, out of SDK scope
- Operator's prompt contains "don't use Sonzai" or "build without the SDK" → halt
- Builder subagent returns BLOCKED → do not re-dispatch, write `.full-auto/BLOCKED.md`, exit
- Cycle 5 reached with failures → write `.full-auto/BLOCKED.md`, exit

## What success looks like

Operator gets:
- A new (or modified) repo with working code, local commits, no remote push
- A spec document under `docs/superpowers/specs/`
- A plan document under `docs/superpowers/plans/`
- A `.full-auto/REPORT.md` summarizing what was built, what was tested, what was assumed, and what's left for the operator
- Optionally: `.full-auto/BLOCKED.md` if something couldn't be resolved within 5 cycles

Operator does NOT get bothered with a single prompt during the run.
