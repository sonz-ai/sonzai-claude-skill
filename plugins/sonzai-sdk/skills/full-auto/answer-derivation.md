# Wizard answer derivation (Phase 0d)

Derive the 8 `sonzai-sdk` wizard answers from the transcript analysis + (tech-stack intake OR brownfield audit). Run autonomously — no operator prompts in this phase; the operator reviews the result at Gate A.

## The 8 answers

Mirrors `sonzai-sdk/intake.md` Q1-Q8.

| # | Field | Possible values | How to derive |
|---|---|---|---|
| 1 | **archetype** | companion / guide-router / enterprise-assistant / customer-support / game-npc / coach-therapist / hybrid-custom | Pick from `transcript.archetype_hint`. If two candidates, pick the closer match by capabilities. |
| 2 | **integration_path** | sdk-direct / platform-rest / both | Default `sdk-direct`. Use `platform-rest` if transcript says "no app code, just call our REST" or if the vertical is a Zapier-style passthrough. |
| 3 | **runtime_mode** | full-chat / memory-layer-sessions / memory-layer-process | Default `memory-layer-sessions`. Use `full-chat` only if transcript says "streaming chat is primary UX" + low capability needs. Use `memory-layer-process` for write-heavy ingest paths (batch import of past conversations). |
| 4 | **capabilities** | comma-list from {memory, personality, mood, diary, kb, custom-tools, webhooks, proactive} | Union of: explicit transcript mentions + archetype defaults (companion → memory+personality+mood; enterprise-assistant → kb+webhooks; etc.) |
| 5 | **brand_persona** | one-sentence | Synthesize from transcript "client goal" + "product shape". Generic, no tenant names. |
| 6 | **proactive** | yes / no + conditions | `yes` if transcript mentions "reaches out", "reminds", "follows up"; else `no`. |
| 7 | **scope** | comma-list of deliverable surfaces | Map from product shape + frontend choice: e.g., "Web SaaS" → ["api", "web-ui", "auth", "db"]; "Telegram bot" → ["api", "bot-integration"] |
| 8 | **byok_posture** | prod / eval-only | Default `prod` (BYOK is the recommended production posture). Use `eval-only` if transcript says "demo / proof-of-concept / hackathon". |

## Derivation logic

Read the inputs:
- `transcript_analysis` (in-memory from Phase 0a)
- `tech_stack` (in-memory from Phase 0c-G) **OR** `audit` (from Phase 0c-B)

Produce the 8 answers as a structured block. Annotate each with a one-line "rationale" so the operator can trace decisions at Gate A.

Example output (in-memory, will be embedded in the masterplan):

```yaml
sonzai_wizard:
  archetype: companion
    rationale: "Transcript: '1:1 relationship', mood matters, persistent — archetype: companion"
  integration_path: sdk-direct
    rationale: "TS backend (Hono) calls SDK directly; no need for REST proxy"
  runtime_mode: memory-layer-sessions
    rationale: "Long sessions per user (hospice context, multi-day) — sessions over process"
  capabilities: [memory, personality, mood, diary]
    rationale: "Archetype: companion defaults + 'tracks how user feels day-to-day' → mood + diary"
  brand_persona: "An empathetic listening companion for end-of-life conversations"
    rationale: "Synthesized from goal + product shape; tenant name redacted"
  proactive: yes
    rationale: "Transcript: 'reaches out daily to check in'"
  scope: [api, web-ui, auth, db]
    rationale: "Web SaaS + Next frontend + Better-Auth + postgres"
  byok_posture: prod
    rationale: "Production deployment intended, not eval/demo"
```

## Cross-checks before proceeding

Before handing off to masterplan-assembly, verify:

1. **archetype × capabilities sanity.** E.g., archetype = `game-npc` but no `custom-tools` in capabilities is suspicious — re-check transcript. Either fix or flag in `risks`.
2. **runtime_mode × capabilities sanity.** E.g., `full-chat` mode + `diary` capability is incoherent (diary requires the memory layer). Either fix or flag.
3. **integration_path × tech-stack sanity.** E.g., `sdk-direct` requires the chosen backend to have an SDK. Check `sonzai-{python,typescript,go}` covers the chosen language. If not, fall back to `platform-rest`.
4. **byok_posture × deploy_target.** `prod` posture + `deploy_target = tbd` is OK at this phase but should be a risk in the masterplan.

## Open answers

If an answer genuinely cannot be derived (e.g., transcript says nothing about runtime mode and tech stack doesn't imply it), set the field to `unclear` and add it to `open_questions`. **Never invent.**

- **`full-auto` mode:** unclear answers fall back to documented defaults from `tech-stack-derivation.md` (or, for sonzai-specific answers, the per-archetype defaults listed in this file's table). Both the default value AND the `unclear` flag go into the masterplan, so the final report surfaces what was guessed.
- **`cto-loop` mode:** unclear answers surface to the operator at Gate A; they decide there before the builder runs.

## Handoff

The 8 derived answers + rationales + open_questions go into `masterplan-assembly.md` next.

## Hard rules

1. **No invention.** Unclear values stay unclear (full-auto falls back to documented defaults + flags it; cto-loop surfaces it at Gate A).
2. **Annotate rationale.** Every answer must have a one-line "why this value" — that's how Gate A review stays fast.
3. **Tenant names already redacted** (per `transcript-analysis.md`). Don't reintroduce them.
4. **BYOK default is prod** even if transcript is silent — that's the documented posture across all 3 skills.
