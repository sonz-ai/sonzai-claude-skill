# sonzai-claude-skill

A skill for AI coding agents (Claude Code, Codex, Gemini CLI, Copilot CLI) that **diagnoses what the developer is building** with the Sonzai SDK, **prescribes** the right archetype + memory mode + capabilities, and **drives them through spec → review → plan** using the superpowers brainstorming/writing-plans discipline.

## What this skill does

When invoked, the skill:

1. **Step 0 — drift check.** Compares the installed SDK version's committed OpenAPI snapshot against the live spec at `https://api.sonz.ai/docs/openapi.json`. Catches stale code generation.
2. **Step 1 — runs the wizard** (`intake.md`). 7-question diagnostic interview (skipping any answered by workspace inference). Forks on greenfield vs existing-codebase.
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

## Install

### Claude Code (manual — current)

```bash
git clone https://github.com/sonz-ai/sonzai-claude-skill ~/sonzai-claude-skill
ln -s ~/sonzai-claude-skill/skills/sonzai-sdk ~/.claude/skills/sonzai-sdk
```

### Claude Code (plugin marketplace, once published)

```bash
/plugin install sonz-ai/sonzai-claude-skill
```

### Codex

```bash
git clone https://github.com/sonz-ai/sonzai-claude-skill ~/sonzai-claude-skill
ln -s ~/sonzai-claude-skill/skills/sonzai-sdk ~/.agents/skills/sonzai-sdk
```

### Gemini CLI / Copilot CLI

Symlink `skills/sonzai-sdk/` into your platform's skills directory. See [agentskills.io/specification](https://agentskills.io/specification) for portability details.

## What's inside

```
skills/sonzai-sdk/
├── SKILL.md                              # router (always-loaded; <200 words)
├── intake.md                             # wizard interview (7 questions)
├── existing-codebase-audit.md            # 8-step insertion-point checklist
├── archetypes/                           # 7 vertical playbooks
│   ├── companion.md
│   ├── guide-router.md                   # MBTI / personality-routed
│   ├── enterprise-assistant.md
│   ├── customer-support.md
│   ├── game-npc.md
│   ├── coach-therapist.md
│   └── hybrid-custom.md
├── features/                             # 20 SDK-surface refs
│   ├── generation.md inventory.md custom-tools.md custom-states.md
│   ├── capabilities.md voice.md knowledge-base.md org-knowledge-base.md
│   ├── priming.md personas.md proactive.md shared-memory.md
│   ├── multiplayer-memory.md instances.md events-and-dialogue.md
│   ├── agent-insights.md self-improvement.md models.md
│   ├── eval-and-simulation.md webhooks.md
├── decisions/                            # 10 decision aids
│   ├── memory-mode.md state-vs-inventory.md capabilities-matrix.md
│   ├── sharedmemory-vs-wisdom.md byok-vs-customllm.md
│   ├── instances-vs-multitenant.md sessions-vs-conversations.md
│   ├── proactive-channel.md post-processing-model.md
│   └── generation-vs-manual-create.md
├── migrations/                           # 9 from-X playbooks
│   ├── overview.md mem0.md langchain.md letta.md zep.md
│   ├── openai-assistants.md character-ai.md crm-csv.md raw-json.md
├── spec-templates/                       # wizard output templates
│   ├── archetype-spec.md.template archetype-plan.md.template
│   └── existing-codebase-spec.md.template migration-spec.md.template
└── references/                           # syntax lookup (kept from v0)
    ├── drift-detection.md auth-and-setup.md
    ├── python.md typescript.md go.md
    ├── streaming-chat.md migration-from-http.md troubleshooting.md
```

**60 files; ~43,000 words.** `SKILL.md` is the only always-loaded file (under 200 words narrative); everything else loads on demand based on the wizard's routing.

## Source-of-truth discipline

Every endpoint name, parameter name, response field, and capability flag referenced in the skill is verified against either:

- A current public SDK repo (`sonzai-python`, `sonzai-typescript`, `sonzai-go`)
- The live OpenAPI spec at `https://api.sonz.ai/docs/openapi.json`

Skills never invent symbols. When the SDK drifts, `references/drift-detection.md` is the agent's first stop.

## Privacy & safety

This skill **only references the public sonz.ai API surface**: the three public SDK repos, the live OpenAPI spec, and the developer docs at `https://sonz.ai/docs`. Zero references to platform internals (context engine, AI service, billing, infrastructure, tenant names). See `CLAUDE.md` for the maintenance rule that enforces this.

## Contributing

PRs welcome. Before submitting:

1. **Verify symbols** — grep every method/field name against the SDK source. The skill's value collapses if it points at things that don't exist.
2. **For new archetypes:** dispatch a baseline subagent (per `superpowers:writing-skills` Iron Law) without the archetype playbook; document failures; write the playbook to address them; re-run.
3. **Keep `SKILL.md` under 200 words narrative.** It loads into every conversation; every token counts.
4. **No platform internals.** See `CLAUDE.md`.

## Architecture decisions captured here

See `docs/design/2026-05-13-skill-v1-design.md` for the full design rationale (the audit of v0, the goals, what's deliberately out of scope, the risks). See `docs/design/2026-05-13-skill-v1-plan.md` for the implementation plan (58 atomic tasks across 8 phases).

## License

MIT
