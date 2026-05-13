---
name: decision-memory-mode
description: Use when picking sync vs async for an agent's memory_mode capability, especially when latency and fact-completeness trade off.
---

# Decision: memory mode (sync vs async)

## The rule

**Default sync.** Switch to async iff (a) you need first-token latency under your turn budget AND (b) you can tolerate facts spilling into the next turn.

## How to apply

Set via `agents.update_capabilities(agent_id, memory_mode="sync"|"async")` (PATCH-style — no rebuild needed; switch anytime).

| Situation | Pick |
|---|---|
| Live voice (`voice_generation=true`) | **async** — sync blocks the audio loop, blows TTFC budget |
| Interactive chat with sub-2s TTFC SLA | **async** |
| Mobile / web chat widget, latency-sensitive | **async** |
| Game NPC (player expects fast reply) | **async** |
| Customer support (compliance, audit) | **sync** |
| Wellness coach / journaler (every fact matters) | **sync** |
| Enterprise compliance (every retrieval same-turn in record) | **sync** |
| Batch / server-to-server flows, latency irrelevant | **sync** (default) |
| Don't know yet, haven't measured latency | **sync** — switch later, no rebuild required |

## Why

- **Sync** blocks the context build until supplementary memory recall returns. Every recalled fact lands in the **current** turn. First-token latency includes the recall round-trip.
- **Async** lets recall race a deadline. Fast hits land in-turn; slow hits **spill to the next turn**. First-token latency is lower, but some facts are missed in the current turn.

The default is sync because the SDK assumes the developer hasn't measured their turn budget yet. Flipping after measurement is one PATCH call.

## Exceptions

- **Compliance regimes that require every retrieval in-record same-turn:** sync, always, no exception. Async creates an audit gap (retrieval logged after the turn it's missing from).
- **Latency budget > 2s TTFC:** sync is fine even for "interactive" archetypes. The slowest hit will fit.
- **Mixed-mode flows:** if you need both behaviors at different times, use **two agents** (e.g. an `async` companion + a `sync` reflection-summarizer); capability is agent-wide, not per-call.
- **Voice ALWAYS:** voice + sync is incompatible. Voice forces async.

## Cross-references

- `features/capabilities.md` — the `update_capabilities` API
- `decisions/capabilities-matrix.md` — per-archetype memory_mode column
- `archetypes/companion.md` (async), `archetypes/game-npc.md` (async), `archetypes/coach-therapist.md` (sync, mandatory), `archetypes/customer-support.md` (sync), `archetypes/enterprise-assistant.md` (sync)
- `features/voice.md` — voice forces async
