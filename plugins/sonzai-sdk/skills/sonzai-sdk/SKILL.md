---
name: sonzai-sdk
description: Use when the user is installing, configuring, or writing code against the Sonzai SDK (pip install sonzai, npm install @sonzai-labs/agents, go get github.com/sonz-ai/sonzai-go), calling api.sonz.ai, working with Sonzai agents/memory/personality/sessions, building a companion / matchmaker / personality-routed app / enterprise assistant / game NPC / coach / customer support agent, or migrating from raw HTTP curl calls to the typed SDK.
---

# Sonzai SDK

Wizard-driven implementation guide for the Sonzai Mind Layer API across Python, TypeScript, and Go.

## When to use

- User imports `sonzai`, `@sonzai-labs/agents`, or `github.com/sonz-ai/sonzai-go`
- User runs `pip install sonzai`, `npm install @sonzai-labs/agents`, `go get github.com/sonz-ai/sonzai-go`
- User has `SONZAI_API_KEY` in their env or `.env`
- User calls `api.sonz.ai`, mentions Sonzai agents, memory, personality, mood, sessions
- User is converting raw HTTP/curl calls to a typed SDK
- User mentions building a companion / matchmaker / personality-routed app / enterprise assistant / game NPC / coach / customer support agent
- User mentions integrating Sonzai via MCP (Claude Code / Cursor / ChatGPT / Claude Desktop / VS Code) — load `features/mcp-integration.md`
- User mentions OpenClaw integration (`@sonzai-labs/openclaw-context`) — load `features/openclaw-integration.md`

## Step 0 — Drift check (REQUIRED)

The committed OpenAPI snapshot inside an installed SDK version can lag the live spec. Always run the drift check before writing non-trivial integration code.

→ Read `references/drift-detection.md`

## Step 1 — Run the wizard

The wizard diagnoses what you're building, prescribes the archetype + memory mode + capabilities, then drives you through spec → review → plan.

→ Read `intake.md`

## Going beyond the wizard

| Operator has... | Wants... | Use |
|---|---|---|
| A scoping conversation with stakeholders | Interactive wizard → spec → plan (no build) | this skill (`sonzai-sdk`) |
| A meeting transcript + no human in loop | Unattended autonomous build → running app. Sends Slack/Gmail notification on completion (build done healthy / build done failing QA) when MCPs are enabled. | `full-auto` (sibling) |
| A meeting transcript + tech-lead in loop | Supervised build with 2 gates (masterplan + live-app review). Operator can reply to gates from Slack DM or Gmail (with MCPs enabled) — terminal isn't required. | `cto-loop` (sibling) |

Both `full-auto` and `cto-loop` deploy the built app locally with `docker compose` and run functional QA against the running stack. `full-auto` auto-fixes; `cto-loop` lets the tech-lead steer the fix loop via free-text feedback. Both share the same core machinery (tech-stack derivation/intake, brownfield audit, version-search hard rule, builder + fixer subagents). Both auto-dispatch a `notifier` subagent at terminal events when Gmail + Slack MCPs are enabled. `cto-loop` additionally polls for replies via Slack DM + Gmail unread (24h budget, paused-resumable).

Trigger phrases:
- `full-auto`: "build from this transcript" / "go full auto" / "run full-auto" / "/full-auto"
- `cto-loop`: "build it and let me review" / "CTO in the loop" / "transcript with two gates"

## Sonzai internal staff (install-time gated, optional)

If the `sonzai-internal-staff` sibling skill is also installed in this plugin install, load it for additional workspace awareness when the operator is Sonzai staff. The skill is install-time gated by the `sonz-ai` plugin marketplace (`category: internal`), so external users never have it on disk. See that skill's own `SKILL.md` for what it adds.

## Skip the wizard

If the user explicitly says "skip wizard" / "I know what I want" / they're mid-implementation and just need a syntax lookup:

| User task | Reference |
|---|---|
| Streaming chat (SSE) vs async polling | `references/streaming-chat.md` |
| Auth, env vars, base URL | `references/auth-and-setup.md` |
| Python syntax | `references/python.md` |
| TypeScript syntax | `references/typescript.md` |
| Go syntax | `references/go.md` |
| Raw HTTP → typed SDK | `references/migration-from-http.md` |
| Errors, 4xx/5xx | `references/troubleshooting.md` |
| Package version recommendations (any) | `references/version-search.md` |

## Hard rules

1. **Never expose `SONZAI_API_KEY` to a browser or mobile client.** Server-side only.
2. **Never guess endpoint names or field names.** Fetch the live OpenAPI spec — see `references/drift-detection.md`.
3. **Never invent tenant-specific behavior.** This SDK is multi-tenant.
4. **Always search for current package versions before recommending them.** No `pip install sonzai==0.5.0`, `next@14`, `postgres:14`, or any other version pulled from training memory. See `references/version-search.md` for what to verify and how.

## Red flags

- "The README shows `client.foo.bar()` but my IDE says it doesn't exist" → drift. Step 0.
- "I'm writing a Next.js client component and need to call the agent" → wrong place. Server-side only.
- "Let me just curl the endpoint" → check the typed SDK first (`references/migration-from-http.md`).
- User pinned an old SDK version → check drift before assuming README matches reality.
