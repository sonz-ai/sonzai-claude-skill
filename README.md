# sonzai-claude-skill

A Claude Code skill that teaches AI coding agents (Claude Code, Codex, Gemini CLI, Copilot CLI) how to implement the [Sonzai SDK](https://sonz.ai/docs) correctly in Python, TypeScript, or Go — and how to handle SDK/API drift gracefully.

## What this skill does

When invoked, the skill:

1. Detects which language the user is working in (from imports, lockfiles, project structure)
2. **Checks for SDK/API drift** before writing code — compares the installed SDK version's committed OpenAPI snapshot against the live spec at `https://api.sonz.ai/docs/openapi.json`
3. Loads the matching per-language reference (`references/python.md`, `references/typescript.md`, `references/go.md`)
4. Guides setup, auth, common patterns (chat streaming, memory, sessions), and migration from raw HTTP
5. Falls back to raw HTTP only for endpoints the installed SDK doesn't cover yet

## Install

### Claude Code (plugin marketplace, once published)

```bash
/plugin install sonz-ai/sonzai-claude-skill
```

### Claude Code (manual)

```bash
git clone https://github.com/sonz-ai/sonzai-claude-skill ~/sonzai-claude-skill
ln -s ~/sonzai-claude-skill/skills/sonzai-sdk ~/.claude/skills/sonzai-sdk
```

### Codex

```bash
git clone https://github.com/sonz-ai/sonzai-claude-skill ~/sonzai-claude-skill
ln -s ~/sonzai-claude-skill/skills/sonzai-sdk ~/.agents/skills/sonzai-sdk
```

### Gemini CLI / Copilot CLI

Same pattern — symlink the `skills/sonzai-sdk/` folder into your platform's skills directory. See the [agentskills.io specification](https://agentskills.io/specification) for portability details.

## What's inside

```
skills/sonzai-sdk/
  SKILL.md                       # router; loaded into context when relevant
  references/
    drift-detection.md           # SDK ↔ live API drift check (run this first)
    auth-and-setup.md            # SONZAI_API_KEY, base URL, config
    python.md                    # Python-specific patterns
    typescript.md                # TypeScript-specific patterns
    go.md                        # Go-specific patterns
    streaming-chat.md            # SSE vs polling
    migration-from-http.md       # raw curl → typed SDK
    troubleshooting.md           # common errors + fixes
```

## Privacy & safety

This skill **only references the public sonz.ai API surface**: the three public SDK repos and the live OpenAPI spec at `https://api.sonz.ai/docs/openapi.json`. It contains zero references to platform internals (context engine, AI service, billing, infrastructure). See `CLAUDE.md` for the maintenance rule that keeps it that way.

## Contributing

PRs welcome. Before submitting, please:

1. Run a baseline test: dispatch a fresh subagent **without** the skill and watch how it fumbles. Document the failure.
2. Add or update the relevant reference so the agent now succeeds.
3. Keep references narrow — one topic per file, no copy-paste of the entire SDK README.

## License

MIT
