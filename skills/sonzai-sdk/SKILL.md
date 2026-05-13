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

## Full-auto (no human in the loop)

If the operator has a meeting transcript and wants a working app shipped end-to-end without being prompted for wizard answers, use the **`full-auto` skill** (sibling skill in this plugin). It reads the transcript, derives the wizard answers itself, dispatches a builder subagent that runs this skill, exercises the built app, and feeds failures back until the app works. Bounded 5-cycle loop, zero operator prompts.

Trigger phrases: "build from this transcript" / "go full auto" / "run full-auto" / "/full-auto"

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

## Hard rules

1. **Never expose `SONZAI_API_KEY` to a browser or mobile client.** Server-side only.
2. **Never guess endpoint names or field names.** Fetch the live OpenAPI spec — see `references/drift-detection.md`.
3. **Never invent tenant-specific behavior.** This SDK is multi-tenant.

## Red flags

- "The README shows `client.foo.bar()` but my IDE says it doesn't exist" → drift. Step 0.
- "I'm writing a Next.js client component and need to call the agent" → wrong place. Server-side only.
- "Let me just curl the endpoint" → check the typed SDK first (`references/migration-from-http.md`).
- User pinned an old SDK version → check drift before assuming README matches reality.
