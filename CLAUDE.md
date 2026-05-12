# Maintaining sonzai-claude-skill

Rules for AI agents (and humans) working inside this repo.

## What this repo is

A public Claude Code / Codex / Gemini CLI skill that helps other AI agents implement the Sonzai SDK correctly. Distributed via plugin marketplaces and direct symlink.

## Hard rules

### 1. No platform internals — ever

This skill is **public**. It runs in users' editors at customer sites. It must never reference:

- `services/contextengine/` source paths or internal Go modules
- `services/ai-service/` (TypeScript AI middleware) internals
- `platform/api/internal/` handlers or schemas
- Tenant names (Razer, Eragon, PocketSouls, etc.)
- Internal infrastructure (Caddy config, VM layout, deploy scripts)
- BigQuery / DuckLake table names, internal metrics, billing math
- Anything in the `sonzai-ai-monolith-ts` repo

**Source of truth for what the skill teaches:** only the public surface area, i.e.:
- `github.com/sonz-ai/sonzai-python` (pip: `sonzai`)
- `github.com/sonz-ai/sonzai-typescript` (npm: `@sonzai-labs/agents`)
- `github.com/sonz-ai/sonzai-go`
- The live OpenAPI spec at `https://api.sonz.ai/docs/openapi.json`
- The public docs at `https://sonz.ai/docs`

If a contributor PR adds anything from the internal surface, reject it.

### 2. Live spec is the ground truth

The SDK READMEs are reference material, but the **live OpenAPI spec at `https://api.sonz.ai/docs/openapi.json` always wins** when there's a conflict. The whole point of the `drift-detection.md` reference is that READMEs/snapshots can lag.

When updating `references/`, do not invent endpoint names, parameter names, or response fields. If unsure, point the agent to the live spec rather than guessing.

### 3. Skills are TDD documentation

This repo follows the superpowers `writing-skills` discipline:

- Description in `SKILL.md` frontmatter must describe **WHEN to use** the skill, never WHAT it does. If you summarise the workflow there, agents skip the body.
- Each reference must address a real failure mode observed in a baseline test (dispatch a subagent without the skill, watch it fumble, document the rationalisation, then write the reference). Don't add references for hypothetical problems.
- Keep `SKILL.md` under 200 words. Heavy content goes in `references/` and only loads on demand.

### 4. Cross-platform tool-name compatibility

The skill uses Claude Code tool names (`Read`, `Bash`, etc.). When mentioning a tool, name it once; do not write platform-specific branches inside the same reference. If platform divergence becomes necessary, add a `references/copilot-tools.md` / `references/codex-tools.md` style shim, matching how the superpowers plugin handles it.

## When updating the SDK

After a breaking change ships in any of the three SDKs:

1. Update `references/{lang}.md` Quick Start to match
2. Bump `package.json` version (semver: minor for additions, major for breaks)
3. Re-run the drift-detection guidance against the new live spec
4. Commit with `feat(skill):` / `fix(skill):` prefix

## When to add a new reference

Only when one of these is true:

- An agent observed in real usage repeatedly fumbles a specific pattern
- A new SDK feature ships that doesn't fit into an existing reference
- A user files an issue with a reproducible failure mode

Don't pre-write references for features that haven't shipped or problems that haven't happened.
