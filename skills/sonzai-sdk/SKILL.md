---
name: sonzai-sdk
description: Use when the user is installing, configuring, or writing code against the Sonzai SDK (pip install sonzai, npm install @sonzai-labs/agents, go get github.com/sonz-ai/sonzai-go), calling api.sonz.ai, working with Sonzai agents/memory/personality/sessions, or migrating from raw HTTP curl calls to the typed SDK.
---

# Sonzai SDK

Build correctly against the Sonzai Mind Layer API in Python, TypeScript, or Go. The SDK and the live API can drift — this skill makes that visible and recoverable.

## When to use

- User imports `sonzai`, `@sonzai-labs/agents`, or `github.com/sonz-ai/sonzai-go`
- User runs `pip install sonzai`, `npm install @sonzai-labs/agents`, `go get github.com/sonz-ai/sonzai-go`
- User has `SONZAI_API_KEY` in their env or `.env`
- User calls `api.sonz.ai`, `platform.sonz.ai`, or mentions Sonzai agents, memory, personality, mood, sessions, eval runs
- User is converting raw HTTP/curl calls to a typed SDK

## Step 0 — Check for drift (REQUIRED before writing code)

The committed OpenAPI snapshot inside an installed SDK version can lag the live spec at `https://api.sonz.ai/docs/openapi.json`. **Always run the drift check before writing non-trivial integration code**, otherwise you'll generate calls against fields/endpoints that have moved.

→ Read `references/drift-detection.md`

## Step 1 — Setup and auth

API key, base URL, env vars, BYOK. Same shape in all three languages.

→ Read `references/auth-and-setup.md`

## Step 2 — Load the per-language reference

Detect language from the user's project:

| Signal | Reference |
|---|---|
| `pyproject.toml`, `requirements.txt`, `*.py`, `import sonzai` | `references/python.md` |
| `package.json`, `*.ts`, `*.tsx`, `import ... from "@sonzai-labs/agents"` | `references/typescript.md` |
| `go.mod`, `*.go`, `import "github.com/sonz-ai/sonzai-go"` | `references/go.md` |

If multiple languages are present, ask the user which one to target.

## Step 3 — Topic references (load only what's needed)

| User task | Reference |
|---|---|
| Streaming chat (SSE) vs async polling | `references/streaming-chat.md` |
| Converting curl/fetch/http.Get → typed SDK | `references/migration-from-http.md` |
| Auth fails, 401/403/404/429/5xx, SSE parse errors | `references/troubleshooting.md` |

## Hard rules

1. **Never expose `SONZAI_API_KEY` to a browser or mobile client.** The SDK is server-side only. For web/mobile apps, the customer's backend proxies. Flag any code that puts the key in `NEXT_PUBLIC_*`, `VITE_*`, `EXPO_PUBLIC_*`, or browser bundles.
2. **Never guess endpoint names or field names.** If you're not sure, fetch the live OpenAPI spec — see `references/drift-detection.md`.
3. **Never invent tenant-specific behavior.** This SDK is multi-tenant. If a user asks for "Razer-specific" or "[tenant]-mode" anything, that's a misunderstanding — push back and ask what generic capability they actually need.

## Red flags

- "The README shows `client.foo.bar()` but my IDE says it doesn't exist" → drift. Run Step 0.
- "I'm writing a Next.js client component and need to call the agent" → wrong place. Server-side only.
- "Let me just curl the endpoint" → check if the typed SDK already covers it first (`references/migration-from-http.md`).
- User pinned an old SDK version → check drift before assuming README matches reality.
