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

## Step 0 — Drift check (REQUIRED)

The committed OpenAPI snapshot inside an installed SDK version can lag the live spec. Always run the drift check before writing non-trivial integration code.

→ Read `references/drift-detection.md`

## Step 1 — Run the wizard

The wizard diagnoses what you're building, prescribes the archetype + memory mode + capabilities, then drives you through spec → review → plan.

→ Read `intake.md`

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
