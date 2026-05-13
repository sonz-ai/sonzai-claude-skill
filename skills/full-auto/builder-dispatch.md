# Builder dispatch

Phase 4: dispatch one Agent subagent that does the actual build, and keep its name reachable for the fix-loop in Phase 5.

## One subagent, kept alive across cycles

We name the subagent `sonzai-builder` so Phase 5 can `SendMessage` it for fixes without losing context. Building from scratch each cycle wastes tokens.

**Agent call (Phase 4):**

```
Agent({
  name: "sonzai-builder",
  description: "Sonzai SDK end-to-end builder",
  subagent_type: "general-purpose",
  model: "sonnet",  // or "opus" — see Model selection
  prompt: <contents of subagent-prompts/builder-prompt.md.template with placeholders filled>
})
```

**Fix-cycle re-dispatch (Phase 5):**

```
SendMessage({
  to: "sonzai-builder",
  message: <contents of subagent-prompts/fixer-prompt.md.template with QA report inlined>
})
```

## Model selection

| Archetype | Model | Reason |
|---|---|---|
| companion, coach-therapist | sonnet | Mostly mechanical wiring |
| customer-support, game-npc | sonnet | Standard SDK use |
| enterprise-assistant | opus | Multi-user coordination, KB scoping require judgment |
| guide-router | opus | Routing logic + N specialists need design taste |
| hybrid-custom | opus | Highest design surface |

Never haiku (per `feedback_subagent_models`).

## Prompt template

→ Read `subagent-prompts/builder-prompt.md.template`

Placeholders to fill from `wizard-answers.md`:

| Placeholder | Source |
|---|---|
| `{{WIZARD_ANSWERS_PATH}}` | always `.full-auto/wizard-answers.md` |
| `{{TRANSCRIPT_PATH}}` | always `.full-auto/transcript.txt` |
| `{{DRIFT_ARTIFACT_PATH}}` | always `.full-auto/openapi.live.json` |
| `{{TARGET_REPO_PATH}}` | from wizard-answers Target section |
| `{{LANG}}` | from wizard-answers Target section |
| `{{ARCHETYPE}}` | from wizard-answers Q2 |
| `{{INTEGRATION_PATH}}` | from wizard-answers Q7 |
| `{{ACCEPTANCE_CHECKLIST}}` | full copy of wizard-answers Acceptance Checklist |
| `{{SKILL_REPO_PATH}}` | absolute path to this repo so the builder can read sonzai-sdk archetypes/features. Either the operator's clone, or the plugin install dir |

The builder needs to find `skills/sonzai-sdk/archetypes/{{ARCHETYPE}}.md` etc. Resolution order:

1. If running inside the operator's clone of `sonzai-claude-skill`, use that absolute path
2. Else use the plugin cache (typically `~/.claude*/plugins/cache/.../sonzai-claude-skill/skills/sonzai-sdk/`)
3. Pass the resolved absolute path to the builder via `{{SKILL_REPO_PATH}}` — don't make the builder guess

## What the builder does

The builder prompt encodes this workflow (read the template for the exact instructions):

1. `cd` to target repo (or `git init` if greenfield)
2. Read `{{WIZARD_ANSWERS_PATH}}` end to end
3. Read `{{SKILL_REPO_PATH}}/skills/sonzai-sdk/archetypes/{{ARCHETYPE}}.md` end to end
4. Read every feature file referenced in that archetype's Section 5
5. Read the spec template matching Q1 (greenfield vs existing-codebase)
6. Fill the spec template → save to `docs/superpowers/specs/YYYY-MM-DD-{{ARCHETYPE}}-design.md`. Commit.
7. Invoke `superpowers:writing-plans` to produce `docs/superpowers/plans/YYYY-MM-DD-{{ARCHETYPE}}-plan.md`. Commit.
8. Invoke `superpowers:subagent-driven-development` to execute the plan task-by-task. Commit per task.
9. Smoke-build locally (language-appropriate command)
10. Return the JSON contract below

## Return contract (builder → full-auto)

The builder's final message MUST be a single fenced JSON block (no prose around it):

```json
{
  "status": "DONE",
  "commits": ["<sha>", "<sha>", "..."],
  "spec_path": "docs/superpowers/specs/2026-05-13-companion-design.md",
  "plan_path": "docs/superpowers/plans/2026-05-13-companion-plan.md",
  "entrypoints": {
    "backend": "go run ./cmd/server",
    "frontend": null,
    "ports": {"backend": 8080, "frontend": null},
    "env_required": ["SONZAI_API_KEY"]
  },
  "smoke_build": {"ok": true, "command": "go build ./...", "stdout_tail": "..."},
  "blockers": []
}
```

`status` is `"DONE"` or `"BLOCKED"`. If `BLOCKED`, populate `blockers` with one string per blocker.

`entrypoints.frontend` is `null` if no frontend was built. `ports.frontend` is null if frontend is null.

`env_required` lists env vars the operator needs to set before Phase 5 testing.

Save the parsed JSON to `.full-auto/build-summary.json`.

## Failure handling

| Builder return | full-auto action |
|---|---|
| `status: DONE`, valid JSON | proceed to Phase 5 |
| `status: BLOCKED` | write `.full-auto/BLOCKED.md` with builder's `blockers`; exit |
| Malformed JSON / non-JSON return | retry once with explicit "Respond with ONLY the JSON block, no prose." If still bad → BLOCKED with reason "Builder failed to produce machine-readable return after retry" |
| Builder times out / errors | BLOCKED; do not re-dispatch (something's wrong with the env, not the prompt) |

## Pre-flight check before dispatching

Before the Agent call:

1. `.full-auto/wizard-answers.md` exists and is non-empty
2. `.full-auto/openapi.live.json` exists and parses as JSON
3. `.full-auto/transcript.txt` exists
4. `{{SKILL_REPO_PATH}}/skills/sonzai-sdk/archetypes/{{ARCHETYPE}}.md` is readable
5. `{{TARGET_REPO_PATH}}` either doesn't exist (greenfield, builder will mkdir) OR is a git repo OR is empty

If any check fails, halt — write the missing item to `.full-auto/BLOCKED.md`. Do NOT dispatch a builder that can't possibly succeed.
