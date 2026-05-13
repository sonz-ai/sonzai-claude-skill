# Python — sonzai

PyPI: [`sonzai`](https://pypi.org/project/sonzai/) · Repo: [`github.com/sonz-ai/sonzai-python`](https://github.com/sonz-ai/sonzai-python) · Python 3.11+

## Install

```bash
pip install sonzai                # or: uv add sonzai / poetry add sonzai
```

## Client init

```python
from sonzai import Sonzai, AsyncSonzai

# Reads SONZAI_API_KEY from env by default
client = Sonzai()

# Or explicit
client = Sonzai(
    api_key="sk-...",                  # or SONZAI_API_KEY
    base_url="https://api.sonz.ai",     # or SONZAI_BASE_URL
    timeout=30.0,
    max_retries=2,
)

# Always close when done (or use the context manager pattern via AsyncSonzai)
client.close()
```

Async variant:

```python
import asyncio
from sonzai import AsyncSonzai

async def main():
    async with AsyncSonzai() as client:
        response = await client.agents.chat(
            "agent-id",
            messages=[{"role": "user", "content": "Hello"}],
        )
        print(response.content)

asyncio.run(main())
```

## Quick Start — chat once

```python
response = client.agents.chat(
    "agent-id",
    messages=[{"role": "user", "content": "Hello!"}],
    user_id="user-123",
)
print(response.content)
print(response.usage.total_tokens)
```

## Quick Start — streaming (SSE)

```python
for event in client.agents.chat(
    "agent-id",
    messages=[{"role": "user", "content": "Tell me a story"}],
    stream=True,
):
    print(event.content, end="", flush=True)
```

## Quick Start — async polling (long chats)

Use when generation may exceed ~100s (Cloudflare/LB SSE cutoff). Cancelling locally does **not** cancel the server task — re-poll the same `processing_id` later.

```python
queued = client.agents.chat_async(
    "agent-id",
    messages=[{"role": "user", "content": "Plan my week."}],
    user_id="user-123",
)
processing_id = queued["processing_id"]

import time
delay = 1.0
while True:
    result = client.agents.poll_chat_result("agent-id", processing_id)
    if result["status"] in ("complete", "failed"):
        break
    time.sleep(delay)
    delay = min(delay * 2, 5.0)
print(result["response"])
```

Or one-shot:

```python
result = client.agents.chat_async_blocking(
    "agent-id",
    messages=[{"role": "user", "content": "Plan my week."}],
    user_id="user-123",
    timeout_seconds=600.0,
)
```

## Sessions (the recommended turn loop)

`sessions.start()` returns a `Session` handle carrying the identity tuple `(agent_id, user_id, session_id, instance_id)`. Each `session.turn(...)` fetches a fresh enriched context, so mood + memory + recent turns stay current.

```python
session = client.agents.sessions.start(
    "agent-id",
    user_id="user-123",
    session_id="session-456",
    provider="gemini",
    model="gemini-3.1-flash-lite",
)

ctx = session.context(query="what's the user about to say?")
# ... build prompt with ctx, call your LLM, get assistant_reply ...

result = session.turn(
    messages=[
        {"role": "user", "content": "what did we talk about last week?"},
        {"role": "assistant", "content": assistant_reply},
    ],
    fetch_next_context={"query": "anticipated next user message"},
)
print(result.mood, result.extraction_id, result.next_context)

session.end(total_messages=10, duration_seconds=300, wait=True)
```

`wait=True` on `end()` forces the consolidation pipeline to run synchronously — use it in benchmarks/tests that query memory immediately after.

## Memory

```python
# Tree
memory = client.agents.memory.list("agent-id", user_id="user-123")

# Semantic search
results = client.agents.memory.search("agent-id", query="favorite food")

# Timeline
timeline = client.agents.memory.timeline(
    "agent-id", user_id="user-123", start="2026-01-01", end="2026-03-01",
)

# Bulk create up to 1000 manual facts (no LLM extraction)
client.agents.memory.bulk_create_facts(
    "agent-id",
    user_id="user-123",
    facts=[{"content": "prefers espresso"}, {"content": "based in Singapore", "fact_type": "location"}],
)

# One-call enriched context (mood + facts + recent turns)
ctx = client.agents.get_context(
    "agent-id", user_id="user-123", query="what did we discuss about espresso?",
)
```

## BYOK — Bring Your Own (LLM) Key

Register your own provider keys per project. Token billing falls on your provider account, not Sonzai's. Keys are encrypted at rest and never returned (only prefix + health).

```python
client.byok.set("project-id", "openai", api_key="sk-...")
client.byok.list("project-id")
client.byok.set_active("project-id", "openai", is_active=False)
client.byok.test("project-id", "gemini")
client.byok.delete("project-id", "xai")
```

Providers: `"openai" | "gemini" | "xai" | "openrouter"`. Requires `read:byok` / `write:byok` scopes on the API key.

## Error types

```python
from sonzai import (
    Sonzai,
    AuthenticationError,    # 401
    NotFoundError,          # 404
    BadRequestError,        # 400
    RateLimitError,         # 429
    InternalServerError,    # 5xx
    SonzaiError,            # base
)

try:
    response = client.agents.chat("agent-id", messages=[...])
except AuthenticationError:
    ...
except RateLimitError:
    ...
except SonzaiError as e:
    ...
```

## Common mistakes

| Mistake | Fix |
|---|---|
| Forgetting `client.close()` | Use `with` or `AsyncSonzai`'s `async with` |
| Putting API key in a frontend bundle | Server-side only. Proxy through the customer's backend. |
| Polling with no backoff | Use 1 → 2 → 4 → 5s capped. The README's `chat_async_blocking` does it for you. |
| Calling `chat(..., stream=True)` and getting back a single object | Iterate the result — it's a generator, not a coroutine result. |
| Mixing sync `Sonzai()` inside an `async def` | Use `AsyncSonzai` in async contexts. |

## Patterns the SDK does NOT cover yet

If your installed version is behind the live spec — see `drift-detection.md`. Fall back to raw HTTP via `httpx` using `client._http_client` URL/headers as a template, or build a fresh `httpx.Client` reading `SONZAI_API_KEY` / `SONZAI_BASE_URL` yourself.
