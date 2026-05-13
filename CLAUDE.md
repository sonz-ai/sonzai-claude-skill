# Maintaining sonzai-claude-skill

Rules for AI agents (and humans) working inside this repo.

## What this repo is

A multi-plugin Claude Code / Codex / Gemini CLI repo that helps other AI agents implement the Sonzai SDK correctly. Two plugins ship from this repo:

- **`plugins/sonzai-sdk/`** — PUBLIC plugin. Anyone can install. Distributed via the `sonz-ai` plugin marketplace and direct symlink. Strictly tenant-agnostic.
- **`plugins/sonzai-internal-staff/`** — INTERNAL plugin. Same repo, separate plugin manifest, install-time gated. Sonzai staff opt in with a second `/plugin install` command. Layers monolith / workspace awareness onto the public plugin.

## Hard rules

### 1. No platform internals in `plugins/sonzai-sdk/` — ever

The public plugin runs in users' editors at customer sites. Inside `plugins/sonzai-sdk/skills/` and any future public-plugin skill, never reference:

- `services/contextengine/` source paths or internal Go modules
- `services/ai-service/` (TypeScript AI middleware) internals
- `platform/api/internal/` handlers or schemas
- Tenant names (Razer, Eragon, PocketSouls, etc.)
- Internal infrastructure (Caddy config, VM layout, deploy scripts)
- BigQuery / DuckLake table names, internal metrics, billing math
- Anything in the `sonzai-ai-monolith-ts` repo

**Source of truth for what the public plugin teaches:** only the public surface area, i.e.:
- `github.com/sonz-ai/sonzai-python` (pip: `sonzai`)
- `github.com/sonz-ai/sonzai-typescript` (npm: `@sonzai-labs/agents`)
- `github.com/sonz-ai/sonzai-go`
- The live OpenAPI spec at `https://api.sonz.ai/docs/openapi.json`
- The public docs at `https://sonz.ai/docs`

If a contributor PR adds anything from the internal surface to `plugins/sonzai-sdk/`, reject it. Tenant names are forbidden in **both** plugins regardless — staff still don't leak names.

### 1b. Internal pointers live ONLY in `plugins/sonzai-internal-staff/`

Monolith paths (`sonzai-ai-monolith-ts/...`), workspace-resolution rules, server-log tailing instructions, and similar staff-only material belong inside `plugins/sonzai-internal-staff/skills/sonzai-internal-staff/` and nowhere else. The plugin is install-time gated by the `sonz-ai` marketplace (`category: internal`), so external users never receive these files on disk. The skill itself still needs to be safe to read aloud — no secrets, no tenant names, no API keys.

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

1. Update `plugins/sonzai-sdk/skills/sonzai-sdk/references/{lang}.md` Quick Start to match
2. Bump version in lockstep: `package.json`, `plugins/sonzai-sdk/.claude-plugin/plugin.json`, `plugins/sonzai-sdk/.codex-plugin/plugin.json`, `plugins/sonzai-internal-staff/.claude-plugin/plugin.json`, `plugins/sonzai-internal-staff/.codex-plugin/plugin.json` (semver: minor for additions, major for breaks)
3. Re-run the drift-detection guidance against the new live spec
4. Commit with `feat(skill):` / `fix(skill):` prefix

## When to add a new reference

Only when one of these is true:

- An agent observed in real usage repeatedly fumbles a specific pattern
- A new SDK feature ships that doesn't fit into an existing reference
- A user files an issue with a reproducible failure mode

Don't pre-write references for features that haven't shipped or problems that haven't happened.
