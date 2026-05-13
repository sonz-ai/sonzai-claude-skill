---
name: decision-runtime-mode
description: Use when picking how to integrate Sonzai — full agent runtime (Sonzai owns the chat LLM call) vs memory-layer-only (you own the LLM, Sonzai handles memory). Resolves Q8 of the wizard. The biggest single architectural decision; affects which SDK methods you write and where prompts live.
---

# Decision: Runtime mode

## The rule

**Default: full-chat (mode A).** Use the Sonzai chat endpoints — Sonzai orchestrates context build → LLM call → response → memory consolidation. It's one SDK call per turn and you get the entire platform.

Switch modes only when you have a concrete reason:

- Need explicit session boundaries with per-session tools / deferred turns → **B**
- Already have your own LLM stack running, want Sonzai for memory only, want session boundaries → **C**
- Ingesting non-chat data (emails, doc reads, telemetry) into memory → **D**

**Production posture for A and B**: **use BYOK** (Bring Your Own Key). Platform credit is for development and evaluation; production should run through your own provider key for billing isolation, rate-limit isolation, and audit cleanliness. See `byok-vs-customllm.md`.

---

## The four modes

### A. Full Sonzai chat (recommended)

Sonzai owns the chat LLM call. You call one of the `agents.chat*` endpoints; Sonzai does context build → LLM → response → memory write in one round trip.

```python
# Python
client.agents.chat(
    agent_id=agent_id,
    messages=[{"role": "user", "content": "hi"}],
    user_id=user_id,
)
# stream:
for event in client.agents.chat_stream(agent_id, messages=[...], user_id=user_id):
    print(event.choices[0].delta.content, end="")
```

```typescript
// TypeScript
await client.agents.chat({
  agentId,
  messages: [{ role: "user", content: "hi" }],
  userId,
});
// stream:
for await (const event of client.agents.chatStream({ agentId, messages: [...], userId })) {
  process.stdout.write(event.choices?.[0]?.delta?.content ?? "");
}
```

```go
// Go
resp, err := client.Agents.Chat(ctx, sonzai.AgentChatParams{
    AgentID: agentID,
    ChatOptions: sonzai.ChatOptions{
        Messages: []sonzai.ChatMessage{{Role: "user", Content: "hi"}},
        UserID:   userID,
    },
})
// stream:
err = client.Agents.ChatStream(ctx, params, func(ev sonzai.ChatStreamEvent) error { ... })
```

**You get:** memory recall, personality injection, mood updates, fact extraction, optional tool calls — all in one POST.

**Variants:**
- `chat` — synchronous, returns the full reply
- `chatStream` — SSE streaming
- `chatAsync` — defer; returns processing ID; poll via `pollChatResult`

**Use when:** building any chat-shaped product (companion, coach, customer-support, enterprise assistant, guide-router specialists). This is the default for ~80% of builds.

### B. Full Sonzai chat with explicit sessions

Same chat orchestration as A, but wrapped in an explicit `sessions.start` / `sessions.end` lifecycle. Each turn goes through `sessions.turn` (which still calls the LLM via Sonzai).

```python
session = client.sessions.start(agent_id, user_id=user_id, provider="openai", model="gpt-5.5")
# (optionally) set tools scoped to this session:
client.sessions.set_tools(agent_id, session.session_id, tool_definitions=[...])
# each turn:
result = client.sessions.turn(
    agent_id, session.session_id,
    user_id=user_id,
    messages=[{"role": "user", "content": "hi"}],
)
# at end:
client.sessions.end(agent_id, user_id=user_id, session_id=session.session_id)
```

```typescript
const session = await client.sessions.start({ agentId, userId, provider: "openai", model: "gpt-5.5" });
const turn = await client.sessions.turn({
  agentId, sessionId: session.sessionId,
  userId, messages: [{ role: "user", content: "hi" }],
});
await client.sessions.end({ agentId, userId, sessionId: session.sessionId });
```

```go
session, _ := client.Sessions.Start(ctx, agentID, sonzai.SessionStartOptions{
    UserID: userID, Provider: "openai", Model: "gpt-5.5",
})
turn, _ := client.Sessions.Turn(ctx, agentID, session.SessionID, sonzai.TurnOptions{
    UserID: userID,
    Messages: []sonzai.TurnMessage{{Role: "user", Content: "hi"}},
})
_, _ = client.Sessions.End(ctx, agentID, sonzai.SessionEndOptions{
    UserID: userID, SessionID: session.SessionID,
})
```

**You get:** everything from A, plus session-scoped tool definitions, deferred-turn polling (`turnStatus`), and end-of-session consolidation hooks. Session boundaries make consolidation timing explicit instead of implicit.

**Use when:**
- You inject different tool definitions per session (game NPC swapping toolsets between quests)
- You want to bracket a multi-turn conversation as a discrete unit (therapy session, support ticket lifecycle)
- You need deferred-turn semantics for long-running operations
- You want guaranteed end-of-session consolidation to fire on a known boundary

### C. Memory-layer via sessions (BYO chat LLM, Sonzai owns memory)

You run your own LLM (Anthropic, OpenAI, internal, whatever). Sonzai never sees a chat completion. You bracket the conversation with sessions, then push the transcript via `/process` for memory extraction.

```python
session = client.sessions.start(agent_id, user_id=user_id)

# YOUR LLM, NOT SONZAI'S
# user_msg = "hi"
# reply = your_llm_call(user_msg)

# push the turn into Sonzai memory:
client.agents.process(
    agent_id,
    user_id=user_id,
    session_id=session.session_id,
    messages=[
        {"role": "user", "content": user_msg},
        {"role": "assistant", "content": reply},
    ],
)
# ... more turns, each pushed to /process ...
# at end:
client.sessions.end(agent_id, user_id=user_id, session_id=session.session_id)
```

```typescript
const session = await client.sessions.start({ agentId, userId });
// const reply = await yourLLM.chat({ messages });
await client.agents.process({
  agentId, userId, sessionId: session.sessionId,
  messages: [{ role: "user", content: userMsg }, { role: "assistant", content: reply }],
});
await client.sessions.end({ agentId, userId, sessionId: session.sessionId });
```

```go
session, _ := client.Sessions.Start(ctx, agentID, sonzai.SessionStartOptions{UserID: userID})
// reply := yourLLMCall(userMsg)
_, _ = client.Agents.Process(ctx, agentID, sonzai.ProcessOptions{
    UserID: userID, SessionID: session.SessionID,
    Messages: []sonzai.ChatMessage{
        {Role: "user", Content: userMsg},
        {Role: "assistant", Content: reply},
    },
})
_, _ = client.Sessions.End(ctx, agentID, sonzai.SessionEndOptions{UserID: userID, SessionID: session.SessionID})
```

**You get:** memory consolidation, fact extraction, mood updates, personality drift — all from your transcripts. No assistant reply from Sonzai. Sessions give you explicit consolidation timing.

**Use when:**
- You already have an LLM stack you don't want to change (existing Anthropic / OpenAI / vLLM infra)
- You need memory + personality but already wrote your own chat handler
- You want to A/B test Sonzai memory against another memory layer without rewriting chat
- Compliance: certain regimes prohibit any third-party touching chat inference, but memory storage is fine

### D. Memory-layer via /process only (BYO chat LLM, no sessions)

Like C but without session lifecycle. You push events to `/process` as they happen — no `start`/`end`. Memory accumulates from each call.

```python
# user has email read activity, observed by your backend
client.agents.process(
    agent_id,
    user_id=user_id,
    messages=[{"role": "user", "content": "Read email from CEO about Q3 priorities"}],
)
# or — a document the user uploaded
client.agents.process(
    agent_id, user_id=user_id,
    messages=[{"role": "user", "content": full_doc_text}],
)
```

```typescript
await client.agents.process({
  agentId, userId,
  messages: [{ role: "user", content: "Read email about Q3 priorities" }],
});
```

```go
_, _ = client.Agents.Process(ctx, agentID, sonzai.ProcessOptions{
    UserID: userID,
    Messages: []sonzai.ChatMessage{{Role: "user", Content: "Read email about Q3 priorities"}},
})
```

**You get:** raw memory ingestion. Sonzai extracts facts, observes habits, detects interests, updates relationship — all without ever participating in a chat.

**Use when:**
- Non-chat surfaces (email read events, doc reads, telemetry, calendar events)
- Periodic batch ingestion from another system
- Building memory FIRST and chat later — populate memory while you're still wiring up the chat side
- Ingestion-only verticals: "memory of what the user did this week", no conversational surface

---

## Quick picker

| Your situation | Mode |
|---|---|
| Building a chat product, want the platform to handle everything | **A** |
| Voice-first product (live conversation) | **A** with `memory_mode=async` and `skip_context_build` per turn |
| Game NPC that swaps tools between quests | **B** |
| Therapy / support ticket with explicit session lifecycle | **B** |
| Already running Anthropic / OpenAI / internal LLM, want Sonzai memory | **C** (if you have session boundaries) or **D** (if you don't) |
| Email/doc/telemetry ingestion, no chat | **D** |
| Pre-populating memory before chat is built | **D** |
| Mixing chat + non-chat ingestion | **A** for chat + **D** for the side channels (one agent, two flows) |

---

## Implications by mode

### Capabilities

| Capability | A | B | C | D |
|---|---|---|---|---|
| `memoryMode` (sync/async) | applies | applies | applies (to extraction) | applies |
| `wisdom` | applies | applies | applies | applies |
| `sharedMemory` | applies | applies | applies | applies |
| `imageGeneration` / `voiceGeneration` | through chat | through chat | N/A (you handle generation) | N/A |
| `webSearch` | through chat | through chat | N/A | N/A |
| `mcpEnabled` | through chat | through chat | partial (MCP server still exposes memory) | partial |
| Custom tools | declared in chat call | declared per session | N/A (you call your own tools) | N/A |

### Latency

| Mode | First-token latency | End-of-session latency |
|---|---|---|
| A | `chatStream` ≈ LLM TTFC (+context build if sync memory) | Extraction folded into stream tail |
| B | Same as A | `sessions.end` triggers consolidation; can be `wait=true` (blocking, slower) or async (returns immediately, consolidates after) |
| C | Your LLM's TTFC (Sonzai out of the path) | `/process` adds extraction latency; `sessions.end` adds consolidation |
| D | Your LLM's TTFC | `/process` extraction latency per call |

Modes C/D are typically faster for chat (Sonzai isn't in the synchronous path) but pay an out-of-band extraction cost.

### Billing posture

| Mode | What you pay Sonzai for | What you pay your provider for |
|---|---|---|
| A (platform credit) | Chat completion + memory operations | (nothing — dev/eval only) |
| A (BYOK / Custom LLM) **recommended for prod** | Memory operations + orchestration | Chat completion tokens |
| B (platform credit) | Chat completion + memory operations | (nothing — dev/eval only) |
| B (BYOK / Custom LLM) **recommended for prod** | Memory operations + orchestration | Chat completion tokens |
| C | Memory operations + extraction LLM calls | Chat completion tokens (entirely yours) |
| D | Memory operations + extraction LLM calls | Chat completion tokens (entirely yours) |

**Production rule:** modes A and B should use BYOK or Custom LLM in production. Platform credit is for development, evaluation, and prototyping — it routes LLM cost through Sonzai's billing, which is convenient for getting started but not how you should run at scale.

---

## Why these specific modes (and no others)

People often ask about combinations that aren't real modes:

- **"Full chat + memory-only at the same time"** — same agent, different flows. Use A for chat surfaces and D for ingestion surfaces; Sonzai stores them in the same memory graph. Not a separate mode; just two SDK call patterns on one agent.
- **"BYOK is its own mode"** — no. BYOK is a configuration of A or B (and only A or B; C and D don't call the chat LLM through Sonzai). See `byok-vs-customllm.md`.
- **"Sessions without start/end"** — no. The session lifecycle is the point of B; otherwise you're in A.
- **"/process with sessions but Sonzai also does chat"** — no. If Sonzai is doing chat, use B (which writes to memory natively). `/process` exists for the *other* side: BYO LLM.

---

## How to apply (wizard)

The wizard's Q8 returns one of: `full-chat` (A), `full-chat-sessions` (B), `memory-sessions` (C), `memory-process` (D).

When derived from transcript signals (full-auto):

| Transcript signal | Maps to |
|---|---|
| "we want a chatbot" / "AI talks to user" / silent | A (default) |
| "per-session tools" / "tool swap" / "deferred turn" / "long ticket lifecycle" | B |
| "we have our own LLM" / "already using GPT-4 / Claude" / "want memory layer" | C if "session" mentioned, else D |
| "ingest emails" / "doc memory" / "passive learning" / "no chat surface" | D |
| "voice-first" / "real-time conversation" | A (with async memory) |

The default when ambiguous: **A**. The wizard derivation records the assumption in `wizard-answers.md`.

---

## Exceptions

- **Live voice + sessions** is allowed (B with async memory) but uncommon — most voice products use A.
- **Switching modes mid-product** is technically allowed (the agent doesn't care which surface you call), but the memory shape differs — A/B generate richer extractions because Sonzai sees the assistant's reasoning; C/D extract from whatever transcript you push.
- **Mode mixing per agent** is allowed and common (A for chat + D for ingestion). Not the same as "switching modes" — these are concurrent flows.

---

## Cross-references

- `sessions-vs-conversations.md` — once you're in mode B, this aid covers session-level patterns
- `decisions/byok-vs-customllm.md` — BYOK sub-config for modes A and B; **read this for production posture**
- `decisions/memory-mode.md` — sync/async memory applies to all four modes; orthogonal decision
- `features/agent-insights.md` — `agents.process` is described here under the read API too
- `archetypes/companion.md` — defaults to **A**
- `archetypes/coach-therapist.md` — defaults to **A**, with D for diary ingestion if relevant
- `archetypes/customer-support.md` — defaults to **A** or **B** (per-ticket session)
- `archetypes/enterprise-assistant.md` — defaults to **A**, sometimes **C** when integrating with existing chat infra
- `archetypes/game-npc.md` — defaults to **B** (per-session tool swap)
- `archetypes/guide-router.md` — defaults to **A** (each specialist is a fresh chat agent)
- `archetypes/hybrid-custom.md` — depends on the strongest signal; document the choice explicitly
