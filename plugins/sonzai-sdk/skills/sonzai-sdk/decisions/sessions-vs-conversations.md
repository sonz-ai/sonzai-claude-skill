---
name: decision-sessions-vs-conversations
description: Use when choosing between the chat API (stateless single-turn) and the sessions API (multi-turn with explicit lifecycle) for an interaction loop.
---

# Decision: sessions vs conversations (chat)

## The rule

**Use `agents.chat` (or `chatStream`)** for stateless single-turn calls.

**Use `agents.sessions.start` + `session.turn`** for multi-turn loops needing fresh enriched context per turn, tool calls, or explicit lifecycle management.

## How to apply

| Pattern | Use |
|---|---|
| One-off question/answer ("translate this to French") | `agents.chat` |
| Auto-session multi-turn (you don't care about session_id) | `agents.chat` with auto-created session — Sonzai handles it |
| Multi-turn chat with tool calls expected | `agents.sessions.start` (better tool-call lifecycle) |
| You need `session.context(query=...)` per-turn fresh context | `agents.sessions.start` |
| You want `fetchNextContext` prefetch for next turn | `agents.sessions.start` |
| Long-lived conversation with explicit `session.end(wait=...)` for consolidation control | `agents.sessions.start` |
| Coach/therapist session that must consolidate fully on end | `agents.sessions.start` with `session.end(wait=True)` |
| Voice live duplex | `agents.voice.stream` — neither chat nor sessions; voice has its own loop |

## Why

- `agents.chat` is the simplest surface. Each call is independent (though Sonzai still tracks `session_id` for memory continuity — auto-created if omitted). Best for stateless ops or when you don't need per-turn context control.

- `agents.sessions.start` returns a `Session` handle that bundles the identity tuple (`agent_id, user_id, session_id, instance_id`) plus session-level defaults. You get:
  - `session.context(query=...)` — fresh enriched context (mood + facts + recent turns) per turn
  - `session.turn(...)` — typed turn with tool-call return shape (`side_effects.external_tool_calls`)
  - `fetch_next_context` in a turn — prefetches next-turn context in the same round-trip
  - `session.end(wait=True|False)` — explicit lifecycle, controls consolidation
  - `session.status(extraction_id)` — poll deferred memory writes

- Sessions are not more expensive than chat — they're a different abstraction. Use them when the abstraction matches your need.

## Exceptions

- **Function-calling-heavy flows** without an obvious session lifecycle: still use sessions; the typed tool-call shape is cleaner than parsing chat responses.
- **Stateless API endpoints** (translation, summarization, classification): always chat; sessions add no value.
- **Voice live duplex:** use `agents.voice.stream` — it's its own surface, not chat/sessions.
- **Async polling** (long-running chats > 100s, behind Cloudflare/LB): `agents.chat_async` / `agents.chat_async_blocking`. Sessions surface doesn't have a polling variant; use chat_async if your hosting environment cuts SSE.

## Cross-references

- `references/streaming-chat.md` — SSE vs polling chat patterns (covers chat + chat_async)
- `references/python.md` / `references/typescript.md` / `references/go.md` — per-language signatures
- `features/voice.md` — voice has its own stream API
- All archetypes use `sessions.start` for their main loop; chat is for one-off lookups
