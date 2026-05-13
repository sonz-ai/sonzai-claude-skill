# cto-loop v1.6.0 Implementation Plan

> Companion to `2026-05-13-cto-loop-design.md`. Sequenced file authoring + version bumps + release.

**Goal:** Ship a third skill `cto-loop` (transcript-driven, supervised, deploys locally, 2 gates) alongside `sonzai-sdk` and `full-auto`.

**Architecture:** See design §4. 5 phases + 2 gates + bounded feedback loop.

**Tech stack:** Markdown skill content + JSON manifests. No executable code in the skill itself.

---

## Files to create

```
plugins/sonzai-sdk/skills/cto-loop/
├── SKILL.md                                   (NEW)
├── pipeline.md                                (NEW)
├── transcript-analysis.md                     (NEW)
├── project-type-detection.md                  (NEW)
├── tech-stack-intake.md                       (NEW)
├── brownfield-audit.md                        (NEW)
├── answer-derivation.md                       (NEW)
├── masterplan-assembly.md                     (NEW)
├── masterplan-gate.md                         (NEW)
├── builder-dispatch.md                        (NEW)
├── version-search.md                          (NEW)
├── local-deploy.md                            (NEW)
├── qa-loop.md                                 (NEW)
├── cto-review-gate.md                         (NEW)
├── feedback-iteration.md                      (NEW)
├── subagent-prompts/
│   ├── builder.md                             (NEW)
│   ├── fixer.md                               (NEW)
│   ├── auditor.md                             (NEW)
│   └── version-checker.md                     (NEW)
├── templates/
│   ├── masterplan.md.template                 (NEW)
│   ├── dockerfile-ts.template                 (NEW)
│   ├── dockerfile-py.template                 (NEW)
│   ├── dockerfile-go.template                 (NEW)
│   ├── docker-compose-postgres.template       (NEW)
│   ├── docker-compose-postgres-redis.template (NEW)
│   └── docker-compose-stateless.template      (NEW)
└── final-report.md.template                   (NEW)
```

## Files to modify

```
README.md                                                       (matrix update)
CHANGELOG.md                                                    (v1.6.0 entry)
CLAUDE.md                                                       (no structural change; mention cto-loop in checklist)
package.json                                                    (1.5.0 → 1.6.0)
plugins/sonzai-sdk/.claude-plugin/plugin.json                   (1.5.0 → 1.6.0)
plugins/sonzai-sdk/.codex-plugin/plugin.json                    (1.5.0 → 1.6.0)
plugins/sonzai-internal-staff/.claude-plugin/plugin.json        (1.5.0 → 1.6.0 lockstep)
plugins/sonzai-internal-staff/.codex-plugin/plugin.json         (1.5.0 → 1.6.0 lockstep)
plugins/sonzai-sdk/skills/sonzai-sdk/SKILL.md                   (cross-ref to cto-loop)
plugins/sonzai-sdk/skills/full-auto/SKILL.md                    (cross-ref to cto-loop)
plugins/sonzai-sdk/skills/sonzai-sdk/references/version-search.md  (NEW — back-port)
```

---

## Tasks

### Task 1 — Skill router + pipeline + project type
Create:
- `SKILL.md` (frontmatter: `name`, `description` per design §3 audience; body <200 words; lists phases and references)
- `pipeline.md` (full Phase 0-4 + Gates A/B flowchart, no business logic, just routing)
- `project-type-detection.md` (decision: is operator's CWD an existing repo or empty? short logic)

### Task 2 — Phase 0 intake references
Create:
- `tech-stack-intake.md` (the 7 questions per design §6 + how to fetch options at runtime + when to skip a question because transcript pre-fills it)
- `brownfield-audit.md` (file list to read per design §7 + detect-and-confirm UX + output schema)
- `transcript-analysis.md` (forked from `full-auto/transcript-analysis.md` — same goals, scale, archetype-hint extraction)
- `answer-derivation.md` (derive 8 wizard answers from transcript + tech-stack/audit context)

### Task 3 — Masterplan + Gate A
Create:
- `masterplan-assembly.md` (combine outputs into single doc; uses `templates/masterplan.md.template`)
- `masterplan-gate.md` (approve / edit-then-approve / reject UX per design §8)
- `templates/masterplan.md.template` (the doc skeleton)

### Task 4 — Build phase
Create:
- `builder-dispatch.md` (how to dispatch builder subagent with masterplan as context)
- `subagent-prompts/builder.md` (builder system prompt — masterplan-aware, always-search-aware)
- `subagent-prompts/fixer.md` (fixer system prompt — feedback or QA-failure aware)
- `subagent-prompts/auditor.md` (brownfield audit subagent)

### Task 5 — Always-search rule
Create:
- `version-search.md` (full rule + commands table per design §9)
- `subagent-prompts/version-checker.md` (version-checker subagent — given list of (ecosystem, package), return current versions + citations)

### Task 6 — Local deploy templates
Create:
- `local-deploy.md` (Dockerfile + compose generation logic; resolves placeholder vars via version-checker; `docker compose up -d --wait` flow with retry on failure)
- `templates/dockerfile-ts.template`
- `templates/dockerfile-py.template`
- `templates/dockerfile-go.template`
- `templates/docker-compose-postgres.template`
- `templates/docker-compose-postgres-redis.template`
- `templates/docker-compose-stateless.template`

### Task 7 — QA + Gate B
Create:
- `qa-loop.md` (smoke + archetype-specific functional checks against running app)
- `cto-review-gate.md` (Gate B UX: print URL + diff + summary + collect feedback)
- `feedback-iteration.md` (fixer loop, bounded 5 cycles)
- `final-report.md.template` (final report skeleton)

### Task 8 — Back-port always-search to wizard
- Create `plugins/sonzai-sdk/skills/sonzai-sdk/references/version-search.md` (smaller version of the rule)
- Add 1-line reference from the wizard SKILL.md or relevant references that mention package install

### Task 9 — Cross-references in sibling skills
- Edit `plugins/sonzai-sdk/skills/sonzai-sdk/SKILL.md`: add line "For transcript-driven supervised builds, see `cto-loop`."
- Edit `plugins/sonzai-sdk/skills/full-auto/SKILL.md`: add line "For transcript-driven builds with operator review at 2 gates, see `cto-loop`."

### Task 10 — README skill matrix + CHANGELOG
- Edit `README.md`: add "When to use which skill" matrix per design §16
- Edit `CHANGELOG.md`: prepend v1.6.0 entry

### Task 11 — Version bumps (5 files in lockstep)
- `package.json`: 1.5.0 → 1.6.0 + description mentions cto-loop
- `plugins/sonzai-sdk/.claude-plugin/plugin.json`: version + description
- `plugins/sonzai-sdk/.codex-plugin/plugin.json`: version + description
- `plugins/sonzai-internal-staff/.claude-plugin/plugin.json`: version (lockstep, no functional change)
- `plugins/sonzai-internal-staff/.codex-plugin/plugin.json`: version (lockstep, no functional change)

### Task 12 — CLAUDE.md
Add a brief note about cto-loop existing alongside the other two skills (in the "what this repo is" section). No structural change to version-bump checklist.

### Task 13 — Sanity check (lightweight, not full TDD)
- Verify SKILL.md frontmatter is valid YAML
- Verify all referenced files exist
- JSON-validate all 4 plugin manifests
- Spot-check no platform internals / tenant names leaked into `cto-loop/`

(Full pressure-test against a baseline subagent is a follow-up task — see acceptance criteria in design §17.)

### Task 14 — Ship
- `git add` all changed/new files
- Commit with `feat(skill): add cto-loop skill — transcript-driven supervised build (v1.6.0)`
- `git push origin main`
- `git tag -a v1.6.0`
- `git push origin v1.6.0`
- `gh release create v1.6.0` with full notes

---

## Execution order

Tasks 1-7 produce the cto-loop skill itself, in dependency order. Tasks 8-9 ripple changes to siblings. Tasks 10-12 update docs and version metadata. Task 13 sanity-checks. Task 14 ships.

Tasks 1-7 file writes are mostly independent; will be batched in parallel Write calls where possible.

---

## Out of scope for this PR

- Full baseline-subagent pressure test per writing-skills TDD discipline (deferred — would take hours to run E2E with a real transcript and docker-compose boot)
- Podman / non-Docker deploy support (v0 requires Docker)
- Multi-tenant scaffolding (one app per run)
- Browser-based review dashboard (v0 is text + URL)
