---
name: full-auto
description: Use when given a meeting transcript or scoping doc and asked to ship a working Sonzai SDK integration end-to-end with zero human prompts. Reads transcript, derives tech stack autonomously (greenfield) or audits existing repo (brownfield), assembles a masterplan, dispatches a builder subagent that always-searches current package versions before writing any manifest, then generates Dockerfile + docker-compose, boots the app locally with `docker compose up -d --wait`, runs functional QA against the running app, and auto-fixes via a bounded fixer loop. Bounded, fully autonomous, no operator prompts.
---

# full-auto

**Closed-loop autonomous transcript → running-app pipeline.** Transcript in → running local docker-compose stack out. Zero prompts to the operator. Bounded fix loop.

## When to use

- A transcript exists (file path, pasted text, or sitting in the prior message)
- Operator wants end-to-end build without being interrupted
- Output should be a *running, locally-deployed app* — not just a spec

## When NOT to use

- Operator can answer questions interactively, wants reviews mid-build → use `cto-loop` (semi-auto with 2 gates) or `sonzai-sdk` wizard (spec + plan, no build)
- One-line ask, no transcript → use `sonzai-sdk` wizard
- Transcript describes something Sonzai SDK doesn't do (trading, payments, video editing) → halt; do not improvise

## Triggers

- Operator pastes a transcript and says "build this" / "go full auto" / "ship it autonomously"
- Operator runs `/full-auto <path>` with a transcript file path
- Operator's prompt mentions "transcript" + "build" / "implement" / "ship" + no requested review

## Hard rules

1. **Never ask the operator a question.** All ambiguity resolves to a documented default. The masterplan + final report surface what got guessed.
2. **Always-search-current-state.** No package version, install command, docker image tag, or library API from training memory. Verify at runtime. See `version-search.md`.
3. **Bounded fix loop.** Max 5 fixer cycles. If still failing, write `.full-auto/BLOCKED.md` and exit.
4. **Local commits only.** No `git push`, no `gh pr create`, no remote deploys (Fly / Cloud Run / etc.). Operator owns those decisions.
5. **No mock SDK calls.** Use the real SDK against a real `SONZAI_API_KEY`. If absent, abort with a clear message (autonomous pipeline cannot prompt for a key).
6. **Drift check first.** Fetch live OpenAPI before anything else (`https://api.sonz.ai/docs/openapi.json`).
7. **No invented symbols.** Every Sonzai call verifies against live OpenAPI or public SDK source.
8. **No tenant names.** Anywhere — code, docs, commits.
9. **postgres default ≠ postgres mandatory.** Stateless apps skip the database.
10. **Brownfield: integrate, never recreate.** Edit the existing repo; don't scaffold alongside.

## Pipeline (Phases 0-5)

→ Full detail: `pipeline.md`

| # | Phase | Reference | Output |
|---|---|---|---|
| 0a | Transcript analysis | `transcript-analysis.md` | in-memory synthesis |
| 0b | Project type detection | `project-type-detection.md` | `greenfield` or `brownfield` |
| 0c-G | Tech-stack derivation (greenfield) | `tech-stack-derivation.md` | `tech_stack` |
| 0c-B | Brownfield audit | `brownfield-audit.md` + `subagent-prompts/auditor.md` | `brownfield-context.md` |
| 0d | Wizard answer derivation | `answer-derivation.md` | 8 sonzai wizard answers + rationales |
| 1 | Masterplan assembly | `masterplan-assembly.md` + `templates/masterplan.md.template` | `docs/cto-review/<date>-masterplan.md` |
| 2 | Build | `builder-dispatch.md` + `subagent-prompts/{builder,version-checker}.md` | local commits, files in CWD |
| 3 | Local deploy + QA | `local-deploy.md` + `qa-loop.md` + `templates/` | running app + QA report |
| 4 | Feedback iteration (auto-fixer) | `feedback-iteration.md` + `subagent-prompts/fixer.md` | bounded retry loop |
| 5 | Final report | `final-report.md.template` | `docs/cto-review/<date>-final-report.md` |

## Required cross-skill dependencies (within this plugin)

- `sonzai-sdk` (sibling) — archetype playbooks, feature references, spec templates, drift-detection guide
- `cto-loop` (sibling) — only if operator opted into the semi-auto path; otherwise unused

## Optional cross-skill (degrades gracefully if absent)

- `sonzai-internal-staff` (sibling, install-time gated) — adds workspace awareness when present; full-auto runs fine without it on public sources alone

## Optional MCPs (degrades gracefully)

- Browser MCP for UI testing: `mcp__chrome-devtools__*` or `mcp__playwright__*`
  - If neither is loaded → UI tests fall back to "API contract exercised only; frontend not driven by browser"
  - If present → drives the UI in a headless browser, captures screenshots + console errors

## Red flags (halt and write BLOCKED.md)

- Transcript names a non-Sonzai product as the target stack
- Transcript asks for game state / billing / payments / video editing / trading
- Operator's prompt contains "don't use Sonzai" or "build without the SDK"
- Builder subagent returns `status: blocked` — do NOT re-dispatch; exit with the blocker
- Auto-fixer cycle 5 reached with QA still failing — write BLOCKED.md, leave compose running, exit
- Drift check failed twice — write BLOCKED.md, exit
- `SONZAI_API_KEY` missing from environment AND from .env — halt with "set the key, then re-run"

## What success looks like

Operator gets:

- A new or modified repo with working code, local commits, no remote push
- `docs/cto-review/<date>-masterplan.md` (the autonomous masterplan; approval section emitted but not marked)
- `docs/cto-review/<date>-final-report.md` summarizing what was built, what was tested, what defaults were used, and what's left for the operator
- A running `docker compose` stack on localhost — operator can `docker compose down` when ready
- Optionally: `.full-auto/BLOCKED.md` if something couldn't be resolved within 5 cycles

Operator does NOT get prompted at all during the run.

## Versioning

Ships with the `sonzai-sdk` plugin in `sonzai-claude-skill` repo. See repo CHANGELOG for release history.
