# Pipeline

Six phases, sequential. Phase 5 is the loop. Each phase reads the previous phase's output.

All artifacts live under `.full-auto/` in the current working directory (gitignored — add `.full-auto/` to `.gitignore` at the start of Phase 1).

---

## Phase 0 — Drift check (REQUIRED)

Fetch the live OpenAPI spec before doing anything else. The committed snapshot in any installed SDK version can lag.

```bash
mkdir -p .full-auto
curl -sSfL https://api.sonz.ai/docs/openapi.json -o .full-auto/openapi.live.json
```

If the fetch fails (network, 4xx, 5xx):
- Retry once after 5s
- If still failing → halt, write `.full-auto/BLOCKED.md` with "Could not fetch live OpenAPI — drift check is required, cannot proceed safely"

Path the builder subagent reads later: `.full-auto/openapi.live.json`.

---

## Phase 1 — Ingest transcript

Order of resolution:

1. **Arg given** — operator ran `/full-auto path/to/transcript.txt` → read that file
2. **Prior message** — last operator message contains a multi-line block that looks like a transcript (timestamps, speaker labels, or "client said... we said..." patterns) → use that
3. **Inline in invocation** — operator's current prompt contains the transcript inline → use that
4. **None of the above** → HALT — write a one-line reason and exit. Do not improvise.

Persist (copy or write) the transcript to `.full-auto/transcript.txt` so the builder subagent and fix-cycle subagents can re-read it from a stable path.

Add `.full-auto/` to `.gitignore` if not already there.

---

## Phase 2 — Transcript analysis

→ Read `transcript-analysis.md` for signal extraction rules.

Produces `.full-auto/signals.md` containing:
- Direct-quote evidence for each signal (archetype, integration path, latency, capabilities, brand/persona, scope, deadlines)
- Inferred Q1-Q7 answers (preview)
- Ambiguities list (where the transcript is silent)

This is purely analytic — no decisions get finalized here. Phase 3 turns these signals into deterministic answers.

---

## Phase 3 — Wizard-answer derivation

→ Read `answer-derivation.md`.

Maps signals → exact answers for the 7 wizard questions in `../sonzai-sdk/intake.md` + archetype-specific follow-ups. For every silent signal, picks the lowest-risk default and **explicitly documents the assumption**.

Output: `.full-auto/wizard-answers.md`

This file is what the builder subagent reads. It contains:
- Q1-Q7 answers
- Archetype-specific follow-up answers
- Capabilities resolved to UpdateCapabilitiesInputBody flag list
- Target repo path (where the impl will land)
- Documented assumptions
- **Acceptance checklist** — what Phase 5 will test against

---

## Phase 4 — Builder dispatch

→ Read `builder-dispatch.md`.

Dispatch ONE Agent subagent with:
- `subagent_type: general-purpose`
- `model: sonnet` (default) or `opus` for guide-router / enterprise / hybrid-custom (more reasoning)
- `name: "sonzai-builder"` (so we can SendMessage it for fix cycles)
- Prompt from `subagent-prompts/builder-prompt.md.template`, filled with paths and acceptance checklist

The builder reads `wizard-answers.md`, picks the matching archetype from `sonzai-sdk/archetypes/`, reads referenced feature files, fills the spec template, writes the plan, executes the plan via `superpowers:subagent-driven-development`, and commits locally.

The builder returns a **structured JSON summary** (see `builder-dispatch.md` for the schema).

If the builder returns `status: BLOCKED`:
- Write `.full-auto/BLOCKED.md` with the builder's blocker list
- Do NOT try to fix what the builder couldn't — exit

If `status: DONE`:
- Save the JSON to `.full-auto/build-summary.json`
- Continue to Phase 5

---

## Phase 5 — QA loop (test → fix → repeat)

→ Read `qa-loop.md`.

The outer session (this session) becomes the tester:

1. **Read entrypoints** from `.full-auto/build-summary.json`
2. **Start backend** (and frontend if present) in background via `Bash run_in_background=true`
3. **Wait for readiness** — Monitor stdout until "listening" / "ready" / port-bound (cap 60s)
4. **Exercise backend** — curl every endpoint in the acceptance checklist
5. **Exercise frontend** (if browser MCP available) — drive UI, capture console errors, screenshot key states
6. **Aggregate failures** to `.full-auto/qa-cycle-N.md`

Decision:
- All pass → Phase 6
- Failures AND cycle < 5 → SendMessage to "sonzai-builder" with `subagent-prompts/fixer-prompt.md.template` filled with the QA report. Await new commit SHAs. Re-enter Phase 5 step 2 (servers may need restart).
- Failures AND cycle == 5 → write `.full-auto/BLOCKED.md`, exit

Cycle counter starts at 1 and increments on each fix iteration.

---

## Phase 6 — Final report

→ Use `final-report.md.template`.

Writes `.full-auto/REPORT.md` summarizing:
- Archetype built, integration path, memory mode, capabilities enabled
- Backend endpoints exercised (passing / total)
- Frontend flows exercised (passing / total, or "API contract only" if no browser MCP)
- Number of QA cycles run
- Documented assumptions (from `wizard-answers.md`)
- Final commit SHA
- Operator's next steps (review, set API key, push, PR)

Then exit. The operator decides if/when to `git push` and `gh pr create`.

---

## Failure surfaces (where things can halt)

| Phase | Halt condition | Output |
|---|---|---|
| 0 | OpenAPI unreachable | `.full-auto/BLOCKED.md` |
| 1 | No transcript anywhere | one-line halt message |
| 2 | Transcript is non-Sonzai-shaped (trading bot, payments, etc.) | `.full-auto/BLOCKED.md` with scope reason |
| 4 | Builder returns BLOCKED | `.full-auto/BLOCKED.md` with builder's blockers |
| 5 | Cycle 5 reached, still failing | `.full-auto/BLOCKED.md` with last QA report |
| 5 | Server entrypoint missing from builder summary | `.full-auto/BLOCKED.md` ("builder did not report entrypoints") |

In every halt case: do NOT modify the operator's git history beyond local commits already made. Leave the working tree in its current state.
