# Pipeline (semi-autonomous)

Overlay on `../full-auto/pipeline.md`. Same phases, two gates inserted, two phases swapped for interactive variants.

When `cto-loop` is active, read this file for the routing — it tells you when to use a cto-loop-local file vs `../full-auto/<file>.md`.

## Phase 0-pre: Notify-setup

→ Read `notify-setup.md`

Interactive overlay on `../full-auto/notify-setup.md`. Detects Gmail + Slack MCP availability. If recipient config missing AND at least one MCP available, asks operator once for Gmail address + Slack handle; caches to `~/.config/sonzai/cto.json`. Used by Gate A and Gate B for async reply support.

If no MCPs available OR operator says `skip`: cto-loop runs in terminal-only sync mode (same as full-auto with notify disabled).

## Phase map

```
┌──────────────────────────────────────────────────────────────────────────┐
│ Pre-flight: drift check + docker check + SONZAI_API_KEY check            │
│   → ../full-auto/pipeline.md "Pre-flight" section                        │
├──────────────────────────────────────────────────────────────────────────┤
│ PHASE 0a — Transcript analysis  (autonomous, identical in both modes)    │
│   → ../full-auto/transcript-analysis.md                                  │
├──────────────────────────────────────────────────────────────────────────┤
│ PHASE 0b — Project type detection  (autonomous, identical)               │
│   → ../full-auto/project-type-detection.md                               │
├──────────────────────────────────────────────────────────────────────────┤
│ PHASE 0c — Tech stack                                                    │
│   Greenfield: tech-stack-intake.md           ← OVERRIDE (interactive 7Q) │
│   Brownfield: brownfield-audit.md            ← OVERRIDE (detect+confirm) │
│                                                                          │
│   Brownfield internally still calls:                                     │
│     → ../full-auto/brownfield-audit.md       (the autonomous detection)  │
│     → ../full-auto/subagent-prompts/auditor.md                           │
│   This skill's brownfield-audit.md adds the confirm UX on top.           │
├──────────────────────────────────────────────────────────────────────────┤
│ PHASE 0d — Wizard answer derivation  (autonomous, identical)             │
│   → ../full-auto/answer-derivation.md                                    │
├──────────────────────────────────────────────────────────────────────────┤
│ PHASE 1 — Masterplan assembly  (autonomous, identical)                   │
│   → ../full-auto/masterplan-assembly.md                                  │
│   → ../full-auto/templates/masterplan.md.template                        │
├──────────────────────────────────────────────────────────────────────────┤
│ 🚪 GATE A — Operator approves masterplan                                 │
│   → masterplan-gate.md                       ← cto-loop-only             │
│   approve / edit / reject (bounded 3 reject cycles)                      │
├──────────────────────────────────────────────────────────────────────────┤
│ PHASE 2 — Build  (autonomous, identical)                                 │
│   → ../full-auto/builder-dispatch.md                                     │
│   → ../full-auto/subagent-prompts/builder.md                             │
│   → ../full-auto/subagent-prompts/version-checker.md                     │
├──────────────────────────────────────────────────────────────────────────┤
│ PHASE 3 — Local deploy + QA  (autonomous, identical)                     │
│   → ../full-auto/local-deploy.md                                         │
│   → ../full-auto/qa-loop.md                                              │
│   → ../full-auto/templates/dockerfile-*.template                         │
│   → ../full-auto/templates/docker-compose-*.template                     │
├──────────────────────────────────────────────────────────────────────────┤
│ 🚪 GATE B — CTO review of running app                                    │
│   → cto-review-gate.md                       ← cto-loop-only             │
│   approve / feedback <txt> / abort                                       │
├──────────────────────────────────────────────────────────────────────────┤
│ PHASE 4 — Feedback iteration (operator-driven, bounded 5 cycles)         │
│   → feedback-iteration.md                    ← cto-loop-only             │
│   Each cycle: operator feedback → fixer subagent → re-deploy → Gate B    │
│   Fixer is the same ../full-auto/subagent-prompts/fixer.md               │
├──────────────────────────────────────────────────────────────────────────┤
│ PHASE 5 — Final report                                                   │
│   → ../full-auto/final-report.md.template                                │
│   Includes operator's approval + final cycle count + feedback log        │
└──────────────────────────────────────────────────────────────────────────┘
```

## Override semantics

When this skill (`cto-loop`) is active, **the cto-loop-local file replaces the corresponding `../full-auto/` file at that phase.** For example:

- `cto-loop/tech-stack-intake.md` REPLACES `../full-auto/tech-stack-derivation.md` for Phase 0c-G
- `cto-loop/brownfield-audit.md` WRAPS `../full-auto/brownfield-audit.md` (calls into it for detection, adds confirm UX on top)
- `cto-loop/masterplan-gate.md` INSERTS between Phase 1 and Phase 2 (full-auto has nothing between them)
- `cto-loop/cto-review-gate.md` INSERTS after Phase 3 (full-auto goes straight from QA-pass to Phase 5; cto-loop interposes the gate)
- `cto-loop/feedback-iteration.md` REPLACES `../full-auto/feedback-iteration.md` for Phase 4 (auto-fixer becomes operator-feedback-driven fixer)

Everything else is inherited from `../full-auto/`.

## Phase boundaries (banners)

The skill prints clear banners at each gate:

```
─────────────────────────────────────────────────────
🚪 GATE A — MASTERPLAN APPROVAL
File:    docs/cto-review/<date>-masterplan.md
Action:  approve / edit / reject <txt>
─────────────────────────────────────────────────────
```

```
─────────────────────────────────────────────────────
🚪 GATE B — CTO REVIEW (cycle N of 5)
Live app:  http://localhost:<port>
Action:    approve / feedback <txt> / abort
─────────────────────────────────────────────────────
```

Non-gate phases run without banners; they print short status updates ("Building... done — 12 commits") so the operator knows what's happening.

## Pre-flight differences

`cto-loop`'s pre-flight is the same as `full-auto`'s, with ONE difference:

- **`SONZAI_API_KEY` check:** in `full-auto`, missing key → halt. In `cto-loop`, the operator IS available, so missing-key prompts inline: "Paste the key (goes into `.env`, not committed) or type `skip` to halt." See `../full-auto/local-deploy.md` §Pre-flight for the unified logic — it handles both modes.

## Where each file lives — quick lookup

| Need to do | File to read |
|---|---|
| **0-pre Notify-setup** | `../full-auto/notify-setup.md` (autonomous) | **`notify-setup.md`** (interactive 1Q ask if no cache) |
| Analyze transcript | `../full-auto/transcript-analysis.md` |
| Decide greenfield vs brownfield | `../full-auto/project-type-detection.md` |
| Greenfield: ask the 7 questions | `tech-stack-intake.md` (this skill) |
| Brownfield: audit + confirm | `brownfield-audit.md` (this skill) → calls `../full-auto/brownfield-audit.md` |
| Derive 8 sonzai wizard answers | `../full-auto/answer-derivation.md` |
| Write the masterplan doc | `../full-auto/masterplan-assembly.md` |
| Gate A flow | `masterplan-gate.md` (this skill) |
| Dispatch builder subagent | `../full-auto/builder-dispatch.md` |
| Version search rule | `../full-auto/version-search.md` |
| Generate Dockerfile + compose, boot | `../full-auto/local-deploy.md` |
| QA against running app | `../full-auto/qa-loop.md` |
| Gate B flow | `cto-review-gate.md` (this skill) |
| Operator-feedback fixer loop | `feedback-iteration.md` (this skill) |
| Final report | `../full-auto/final-report.md.template` |

## Hard rules

All inherited from `../full-auto/SKILL.md` plus the two cto-loop-specific rules in this skill's `SKILL.md`:

1. **Two gates are non-optional.**
2. **Interactive overrides override autonomous.**

## Async gates

When `notify_enabled` is true AND `ScheduleWakeup` is available, Gates A and B run in async mode:

1. Print gate banner (same as terminal).
2. Dispatch `../full-auto/subagent-prompts/notifier.md` to send Slack DM + Gmail.
3. Save run state to `~/.config/sonzai/cto-runs/<run-id>.json`.
4. `ScheduleWakeup(1200s)`, exit turn.
5. On wakeup: dispatch `../full-auto/subagent-prompts/reply-poller.md`, process result.
6. Loop steps 4-5 until reply OR 24h timeout (then pause + resumable).

Terminal input remains an override path even in async mode.

State file at `~/.config/sonzai/cto-runs/<run-id>.json` enables resume after timeout via `/cto-loop resume <run-id>`.
