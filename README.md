# sonzai-claude-skill

A multi-plugin skill repo for AI coding agents (Claude Code, Codex, Gemini CLI, Copilot CLI) building on the **Sonzai SDK**. Two plugins ship from this repo:

| Plugin | Skills | Audience |
|---|---|---|
| **`sonzai-sdk`** (public) | `sonzai-sdk` (wizard) + `full-auto` (autonomous closed-loop) | Any developer using the Sonzai SDK |
| **`sonzai-internal-staff`** (internal-only) | `sonzai-internal-staff` | Sonzai staff with read access to private monolith repos |

The `sonzai-internal-staff` plugin is **install-time gated** — it lives in a separate plugin manifest, so it is never copied to a non-staff disk just because they installed `sonzai-sdk`.

---

## What the skills do

### `sonzai-sdk` skill (the wizard)

When invoked:

1. **Step 0 — drift check.** Compares the installed SDK version's committed OpenAPI snapshot against the live spec at `https://api.sonz.ai/docs/openapi.json`. Catches stale code generation.
2. **Step 1 — runs the wizard** (`intake.md`). 8-question diagnostic interview (Q1–Q8: archetype, integration path, latency, capabilities, brand/persona, proactive, scope, runtime mode). Skips any answered by workspace inference. Forks on greenfield vs existing-codebase.
3. **Loads the archetype playbook** that matches the developer's intent — one of:
   - `companion` — 1:1 persistent companion (Replika-shaped)
   - `guide-router` — MBTI / personality-routed (intake guide + N specialists)
   - `enterprise-assistant` — team-shared, KB-heavy, audited
   - `customer-support` — KB + custom tools + webhooks
   - `game-npc` — inventory + custom states + dialogue + events
   - `coach-therapist` — long sessions, sync memory, diary, mood
   - `hybrid-custom` — combinations / multi-tenant
4. **Drives spec → review → plan** via the playbook's templates. Output is a fully-specified implementation plan ready to execute via `superpowers:subagent-driven-development` or `superpowers:executing-plans`.

For developers who just want a syntax lookup, a skip-wizard path falls through to the per-language references (`python.md`, `typescript.md`, `go.md`).

### `full-auto` skill (autonomous closed-loop)

Take a meeting transcript or a paste of client requirements, no operator prompts:

1. **Phase 0** — drift check (same as wizard) + transcript analysis (`transcript-analysis.md`)
2. **Phase 1** — derives all 8 wizard answers from the transcript (`answer-derivation.md`)
3. **Phase 2** — dispatches a builder subagent that runs the `sonzai-sdk` wizard end-to-end (`builder-dispatch.md`)
4. **Phase 3** — receives JSON contract back: commits, spec path, plan path, entrypoints, smoke-build
5. **Phase 4** — boots the built app (backend + frontend if any) and exercises it (`qa-loop.md`)
6. **Phase 5** — on QA failure, re-dispatches the builder subagent via `SendMessage` for fixes (bounded 5-cycle loop)

Operator owns the push decision — `full-auto` only commits locally.

### `sonzai-internal-staff` skill (internal-only)

Layers monolith + workspace awareness onto the public skills. Augments `full-auto`'s drift check with the monolith's generated OpenAPI, tails server logs during QA cycles, verifies wizard capability questions against `services/contextengine/domain/entity/agent.go`. Useless without read access to the private monolith — see install section.

---

## Install

### Claude Code

Recommended (marketplace):

```bash
/plugin marketplace add sonz-ai/sonzai-claude-skill
/plugin install sonzai-sdk@sonz-ai
```

That installs **only the public plugin**. The two public skills (`sonzai-sdk`, `full-auto`) become available; the internal-staff skill is NOT copied.

Sonzai internal staff (additional, optional):

```bash
/plugin install sonzai-internal-staff@sonz-ai
```

Set `SONZAI_WORKSPACE` to the dir containing `sonzai-sdk/` and `sonzai-ai-monolith-ts/` (or run from inside that workspace; the skill will walk up to find it).

Manual install (no marketplace):

```bash
# Public plugin
git clone https://github.com/sonz-ai/sonzai-claude-skill ~/sonzai-claude-skill
ln -s ~/sonzai-claude-skill/plugins/sonzai-sdk/skills/sonzai-sdk      ~/.claude/skills/sonzai-sdk
ln -s ~/sonzai-claude-skill/plugins/sonzai-sdk/skills/full-auto       ~/.claude/skills/full-auto

# Internal staff (optional, requires monolith access to be useful)
ln -s ~/sonzai-claude-skill/plugins/sonzai-internal-staff/skills/sonzai-internal-staff \
      ~/.claude/skills/sonzai-internal-staff
```

### Codex

Recommended (plugin):

```bash
codex plugin add sonz-ai/sonzai-claude-skill         # registers marketplace
codex plugin install sonzai-sdk@sonz-ai
# Internal staff (optional):
codex plugin install sonzai-internal-staff@sonz-ai
```

Codex reads `.codex-plugin/plugin.json` from each plugin folder. Same `skills/` layout as Claude Code.

Manual:

```bash
git clone https://github.com/sonz-ai/sonzai-claude-skill ~/sonzai-claude-skill
ln -s ~/sonzai-claude-skill/plugins/sonzai-sdk/skills/sonzai-sdk      ~/.agents/skills/sonzai-sdk
ln -s ~/sonzai-claude-skill/plugins/sonzai-sdk/skills/full-auto       ~/.agents/skills/full-auto
# Internal staff (optional):
ln -s ~/sonzai-claude-skill/plugins/sonzai-internal-staff/skills/sonzai-internal-staff \
      ~/.agents/skills/sonzai-internal-staff
```

### Gemini CLI / Copilot CLI

Symlink each skill directory into your platform's skills root. See [agentskills.io/specification](https://agentskills.io/specification) for portability details.

Per-skill source paths (public):

```
plugins/sonzai-sdk/skills/sonzai-sdk/
plugins/sonzai-sdk/skills/full-auto/
```

Internal-staff (optional):

```
plugins/sonzai-internal-staff/skills/sonzai-internal-staff/
```

---

## Repo layout

```
sonzai-claude-skill/
├── .claude-plugin/
│   └── marketplace.json                      # marketplace listing for sonz-ai
├── plugins/
│   ├── sonzai-sdk/                           # PUBLIC plugin (auto-installed for everyone)
│   │   ├── .claude-plugin/plugin.json        # Claude Code manifest
│   │   ├── .codex-plugin/plugin.json         # Codex manifest
│   │   └── skills/
│   │       ├── sonzai-sdk/                   # wizard skill
│   │       │   ├── SKILL.md                  # router (always-loaded; <200 words)
│   │       │   ├── intake.md                 # 8-question wizard
│   │       │   ├── existing-codebase-audit.md
│   │       │   ├── archetypes/               # 7 vertical playbooks
│   │       │   ├── features/                 # 20 SDK-surface refs
│   │       │   ├── decisions/                # 11 decision aids (incl. runtime-mode)
│   │       │   ├── migrations/               # 9 from-X playbooks
│   │       │   ├── spec-templates/           # wizard output templates
│   │       │   └── references/               # syntax lookup
│   │       └── full-auto/                    # autonomous closed-loop skill
│   │           ├── SKILL.md, pipeline.md, transcript-analysis.md,
│   │           ├── answer-derivation.md, builder-dispatch.md, qa-loop.md,
│   │           ├── subagent-prompts/         # builder + fixer prompt templates
│   │           └── final-report.md.template
│   └── sonzai-internal-staff/                # INTERNAL plugin (opt-in install only)
│       ├── .claude-plugin/plugin.json
│       ├── .codex-plugin/plugin.json
│       └── skills/
│           └── sonzai-internal-staff/
│               ├── SKILL.md
│               └── workspace-pointers.md
├── README.md
├── CHANGELOG.md
├── CLAUDE.md                                 # maintenance rules
├── LICENSE
└── package.json
```

---

## Why two plugins instead of a runtime gate?

Earlier versions had a single plugin with the internal skill gated at runtime by `SONZAI_INTERNAL_STAFF=1` env var. That worked but still **copied the internal SKILL.md into every public user's disk**, which is wasteful and conceptually messy.

Splitting into two plugins moves the gate to **install time** (`/plugin install` is the decision point), which is exactly the plugin marketplace model Anthropic's own `claude-plugins-official` marketplace uses (200+ plugins in one repo, each installed by name). Public users literally never touch the internal-staff files.

---

## Source-of-truth discipline

Every endpoint name, parameter name, response field, and capability flag referenced in the **public** skills is verified against either:

- A current public SDK repo (`sonzai-python`, `sonzai-typescript`, `sonzai-go`)
- The live OpenAPI spec at `https://api.sonz.ai/docs/openapi.json`

Skills never invent symbols. When the SDK drifts, `references/drift-detection.md` is the agent's first stop.

The **internal-staff** skill additionally references monolith paths (`sonzai-ai-monolith-ts/services/contextengine/`, etc.). Those references are useless to external users — that's the point. The plugin is install-time gated for that reason.

---

## Privacy & safety

The **public** plugin (`sonzai-sdk`) only references the public sonz.ai API surface: the three public SDK repos, the live OpenAPI spec, and the developer docs at `https://sonz.ai/docs`. Zero references to platform internals (context engine, AI service, billing, infrastructure, tenant names). See `CLAUDE.md` for the maintenance rule that enforces this.

The **internal-staff** plugin references private monolith paths by name (`sonzai-ai-monolith-ts/...`). The repo itself is public, but those paths only resolve on a staff machine with the private repos cloned. **No secrets, API keys, or tenant data are ever embedded** in either plugin.

---

## Contributing

PRs welcome. Before submitting:

1. **Verify symbols** — grep every method/field name against the SDK source. The skill's value collapses if it points at things that don't exist.
2. **For new archetypes:** dispatch a baseline subagent (per `superpowers:writing-skills` Iron Law) without the archetype playbook; document failures; write the playbook to address them; re-run.
3. **Keep `SKILL.md` under 200 words narrative.** It loads into every conversation; every token counts.
4. **No platform internals in `plugins/sonzai-sdk/`.** Internal pointers go in `plugins/sonzai-internal-staff/` only. See `CLAUDE.md`.
5. **Bump both `plugin.json` files in lockstep** with the repo's `package.json` version.

---

## Architecture decisions captured here

See `docs/design/2026-05-13-skill-v1-design.md` for the original v1 design and `docs/design/2026-05-13-skill-v1-plan.md` for the implementation plan. Later versions (v1.2 full-auto, v1.3 internal-staff gate, v1.4 runtime mode, v1.5 two-plugin split) are documented in `CHANGELOG.md`.

---

## License

MIT
