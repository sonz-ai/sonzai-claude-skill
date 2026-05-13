---
name: cto-loop
description: Use when given a meeting transcript or scoping document and the goal is a running, locally-deployed product with operator (CTO / tech lead) approval at two gates — masterplan validation before any build work, then live-app review after docker-compose boot. Triggers on phrases like "CTO in the loop", "tech-lead review build", "transcript with two gates", "build it and let me review the running app". Runs full-auto's pipeline with two operator gates inserted and an interactive tech-stack intake (7 questions) instead of autonomous defaults. For unattended autonomous builds, use full-auto. For wizard-only output (spec + plan, no build), use sonzai-sdk. Operator can ALSO reply to gates via Slack DM or Gmail when those MCPs are enabled — skill polls every 20min for up to 24h.
---

# cto-loop

**Semi-autonomous variant of `full-auto`** — same pipeline, plus two operator gates and an interactive tech-stack intake. Use when a tech lead is available to review the plan upfront and the running app at the end.

This skill is a **thin overlay on `full-auto`**. All shared logic (transcript analysis, project-type detection, audit, masterplan assembly, builder dispatch, version search, docker-compose deploy, QA loop, fixer subagent) lives in `../full-auto/` and is referenced from there. Only the gate flows + the interactive overlays live here.

## When to use

- A transcript + a tech lead who wants to review at two points
- Greenfield builds where the operator wants to choose the stack interactively (instead of accepting `full-auto`'s autonomous defaults)
- Brownfield builds where the operator wants to confirm the audit before the masterplan locks in

## When NOT to use

- No human in the loop → use `full-auto`
- One-line ask, no transcript → use `sonzai-sdk` wizard
- Spec + plan only, no build → use `sonzai-sdk` wizard

## Pipeline overlay

The pipeline mirrors `full-auto`'s exactly, with two gates inserted and two phases swapped for interactive variants:

| Phase | full-auto behavior | cto-loop overlay |
|---|---|---|
| **0-pre Notify-setup** | `../full-auto/notify-setup.md` (autonomous) | **`notify-setup.md` (interactive — asks if missing)** |
| Pre-flight | drift check, docker check, key check | identical |
| 0a Transcript | `../full-auto/transcript-analysis.md` | identical |
| 0b Project type | `../full-auto/project-type-detection.md` | identical |
| 0c-G Greenfield stack | `../full-auto/tech-stack-derivation.md` (autonomous) | **`tech-stack-intake.md` (interactive 7Q)** |
| 0c-B Brownfield audit | `../full-auto/brownfield-audit.md` (autonomous) | **`brownfield-audit.md` (detect-and-confirm)** |
| 0d Wizard answers | `../full-auto/answer-derivation.md` | identical |
| 1 Masterplan | `../full-auto/masterplan-assembly.md` | identical |
| **🚪 Gate A** | (none) | **`masterplan-gate.md`** |
| 2 Build | `../full-auto/builder-dispatch.md` + subagent prompts in `../full-auto/subagent-prompts/` | identical |
| 3 Deploy + QA | `../full-auto/local-deploy.md` + `../full-auto/qa-loop.md` | identical |
| **🚪 Gate B** | (none — auto-fixer instead) | **`cto-review-gate.md`** |
| 4 Iterate | `../full-auto/feedback-iteration.md` (auto-fixer on QA failure) | **`feedback-iteration.md`** (operator feedback drives fixer) |
| 5 Final report | `../full-auto/final-report.md.template` | identical (with operator approval reflected) |

## Hard rules (overlay-specific)

1. **Two gates are non-optional.** No skip-flag. If you want autonomous, use `full-auto`.
2. **Interactive overrides override autonomous.** When `cto-loop` is active, Phase 0c-G uses `tech-stack-intake.md` (interactive 7Q) instead of full-auto's `tech-stack-derivation.md`; brownfield-audit prints findings for row-by-row confirmation; feedback-iteration loops on operator feedback rather than QA-failure auto-fixing.

(All other hard rules — always-search, no remote git/deploy, no tenant names, etc. — are inherited from `../full-auto/SKILL.md`. They apply identically in this mode.)

## Files (cto-loop-specific only)

| File | Purpose |
|---|---|
| `notify-setup.md` | Phase 0-pre: interactive ask for Gmail/Slack if not cached |
| `pipeline.md` | Overlay pipeline with gates inserted |
| `tech-stack-intake.md` | Interactive 7-question variant of tech-stack-derivation |
| `brownfield-audit.md` | Detect-and-confirm wrapper around full-auto's audit |
| `masterplan-gate.md` | Gate A flow (approve / edit / reject the masterplan) |
| `cto-review-gate.md` | Gate B flow (live app review) |
| `feedback-iteration.md` | Operator-feedback-driven fixer loop (bounded 5 cycles) |

Everything else is referenced via `../full-auto/<file>.md`.

## Versioning

Ships with the `sonzai-sdk` plugin in `sonzai-claude-skill`. See repo CHANGELOG.
