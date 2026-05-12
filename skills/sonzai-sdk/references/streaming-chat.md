# Streaming chat: SSE vs async polling

Two ways to receive a chat response. Pick based on **how long the generation might take** and **how patient your transport is**.

## Quick decision

| Situation | Use |
|---|---|
| Interactive UI, response < ~60s | **SSE streaming** (`chat(stream=True)` / `chatStream` / `ChatStream`) |
| Behind Cloudflare / AWS ALB / GCP LB with default ~100s idle timeout | **Async polling** (`chat_async` / `chatAsync` / detached variants) |
| Long planning/agent calls expected to exceed 100s | **Async polling** |
| NATS/Watermill/queue-worker handler with short caller deadline | **Async polling** OR Go `*Detached` variants |
| One-shot script, latency irrelevant | Plain non-streaming `chat()` is simplest |

## Pattern 1 — SSE streaming

The server holds the connection open and pushes events as tokens arrive. Lowest first-byte latency, but the connection must stay open for the entire generation.

**Python:**
```python
for event in client.agents.chat(
    "agent-id",
    messages=[{"role": "user", "content": "Tell me a story"}],
    stream=True,
):
    print(event.content, end="", flush=True)
```

**TypeScript:**
```ts
for await (const event of client.agents.chatStream({
  agent: "agent-id",
  messages: [{ role: "user", content: "Tell me a story" }],
})) {
  process.stdout.write(event.choices?.[0]?.delta?.content ?? "");
}
```

**Go:**
```go
err := client.Agents.ChatStream(ctx, params, func(event sonzai.ChatStreamEvent) error {
    fmt.Print(event.Content())
    return nil
})
```

**SSE event shape:** events arrive as `data: <json>\n\n` frames. Each frame is a `ChatStreamEvent` with a `delta` (text chunk) and optional `usage` on the final frame. New event types may appear over time — log+ignore unknown `type` values rather than crashing.

## Pattern 2 — Async polling

Queue the request, get a `processing_id` back immediately, poll the result endpoint. Survives connection drops. Cancelling the poll locally **does not** cancel the server-side task; re-poll the same id later if needed.

**Python:**
```python
queued = client.agents.chat_async(
    "agent-id",
    messages=[{"role": "user", "content": "Plan my week."}],
    user_id="user-123",
)

import time
delay = 1.0
while True:
    result = client.agents.poll_chat_result("agent-id", queued["processing_id"])
    if result["status"] in ("complete", "failed"):
        break
    time.sleep(delay)
    delay = min(delay * 2, 5.0)
```

Or the one-shot helper:
```python
result = client.agents.chat_async_blocking(
    "agent-id",
    messages=[{"role": "user", "content": "Plan my week."}],
    user_id="user-123",
    timeout_seconds=600.0,    # matches server-side CE_AGENT_CHAT_DEADLINE_MS
)
```

**TypeScript:**
```ts
const result = await client.agents.chatAsyncBlocking(
  {
    agent: "agent-id",
    messages: [{ role: "user", content: "Plan my week." }],
    userId: "user-123",
  },
  { pollIntervalMs: 1000, maxPollIntervalMs: 5000, timeoutMs: 600_000 },
);
```

**Status values:** `queued | running | complete | failed`. While `running`, `response` carries partial assistant text and `phase` / `tool` reflect the latest progressive-elaboration event — useful for showing "thinking…" UI without true streaming.

**Recommended backoff:** 1s → 2s → 4s, capped at 5s.

## Pattern 3 — Go `*Detached` (short-deadline callers)

When you're inside a handler whose `ctx` will be cancelled before the LLM finishes (NATS subscriber, Watermill consumer, short-lived queue worker), the regular `ChatStream` will be killed mid-generation. Use the detached variants:

```go
err := client.Agents.ChatStreamDetached(ctx, params, sonzai.DetachOptions{
    Timeout: 120 * time.Second,
}, func(event sonzai.ChatStreamEvent) error {
    // handle
    return nil
})
```

Available: `ChatDetached`, `ChatStreamDetached`, `ChatStreamChannelDetached`.

## Common pitfalls

| Symptom | Cause | Fix |
|---|---|---|
| Stream cuts at ~60s / 100s in browsers behind CDN | Cloudflare / ALB / GCP LB idle timeout | Switch to async polling |
| Cancelled the poll but the server kept billing | Server-side cancellation is independent | Re-poll the same `processing_id` if you want the result; otherwise accept the cost |
| Streaming events arrive but `event.content` is empty | Old SDK doesn't know a new event `type` | Upgrade SDK (see `drift-detection.md`) or default-branch and ignore |
| Polling loop hammers the API every 100ms | No backoff | Use 1→2→4→5s capped |
| Python `for event in chat(...)` returns one item | Forgot `stream=True` | Pass `stream=True` |
| TS `await chatStream(...)` returns AsyncGenerator | `chatStream` is not a Promise | Iterate with `for await`, don't `await` it |
