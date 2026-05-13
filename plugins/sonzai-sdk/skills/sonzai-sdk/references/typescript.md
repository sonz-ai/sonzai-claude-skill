# TypeScript — @sonzai-labs/agents

npm: [`@sonzai-labs/agents`](https://www.npmjs.com/package/@sonzai-labs/agents) · Repo: [`github.com/sonz-ai/sonzai-typescript`](https://github.com/sonz-ai/sonzai-typescript) · Node ≥18, Bun ≥1.0, Deno ≥1.28 · Zero runtime deps · ESM + CJS

## Install

```bash
npm install @sonzai-labs/agents          # or: bun add / pnpm add / yarn add
```

Deno:
```ts
import { Sonzai } from "npm:@sonzai-labs/agents";
```

## Client init

```ts
import { Sonzai } from "@sonzai-labs/agents";

// Reads SONZAI_API_KEY from env by default
const client = new Sonzai();

// Or explicit
const client = new Sonzai({
  apiKey: "sk-...",                  // or SONZAI_API_KEY
  baseUrl: "https://api.sonz.ai",     // or SONZAI_BASE_URL
  timeout: 30_000,
  maxRetries: 2,
  defaultHeaders: { "X-My-Header": "value" },
  customFetch: fetch,                 // swap in undici / a mock / a wrapper
});
```

Idempotent requests (GET, DELETE) retry with exponential backoff. Mutating requests (POST/PATCH/PUT) do not retry.

## Quick Start — chat once

```ts
const response = await client.agents.chat({
  agent: "agent-id",
  messages: [{ role: "user", content: "Hello!" }],
  userId: "user-123",
});
console.log(response.content, response.usage?.totalTokens);
```

## Quick Start — streaming (SSE)

```ts
for await (const event of client.agents.chatStream({
  agent: "agent-id",
  messages: [{ role: "user", content: "Tell me a story" }],
})) {
  process.stdout.write(event.choices?.[0]?.delta?.content ?? "");
}
```

`chatStream` returns `AsyncGenerator<ChatStreamEvent>`. The final frame may carry `usage`.

## Quick Start — async polling

```ts
const queued = await client.agents.chatAsync({
  agent: "agent-id",
  messages: [{ role: "user", content: "Plan my week." }],
  userId: "user-123",
});

// Or just call the helper that does the poll loop for you:
const result = await client.agents.chatAsyncBlocking(
  {
    agent: "agent-id",
    messages: [{ role: "user", content: "Plan my week." }],
    userId: "user-123",
  },
  { pollIntervalMs: 1000, maxPollIntervalMs: 5000, timeoutMs: 600_000 },
);
```

## Sessions (recommended turn loop)

```ts
const session = await client.agents.sessions.start("agent-id", {
  userId: "user-123",
  sessionId: "session-456",
  provider: "gemini",
  model: "gemini-3.1-flash-lite",
});

const ctx = await session.context({ query: "what's the user about to say?" });
// ... build prompt with ctx, call your LLM, get assistantReply ...

const result = await session.turn({
  messages: [
    { role: "user", content: "what did we talk about last week?" },
    { role: "assistant", content: assistantReply },
  ],
  fetchNextContext: { query: "anticipated next user message" },
});

await session.end({ totalMessages: 10, durationSeconds: 300, wait: true });
```

## Resources

```ts
client.agents            // chat, CRUD, context engine data
client.agents.memory     // tree, search, facts, timeline
client.agents.personality
client.agents.sessions
client.agents.instances
client.agents.notifications
client.agents.customStates
client.agents.voice      // TTS / STT / live WebSocket
client.agents.generation // generate from prompt
client.agents.priming
client.agents.inventory
client.agents.schedules
client.knowledge
client.evalTemplates
client.evalRuns
client.voices
client.webhooks
client.projects
client.userPersonas
client.analytics
client.customLLM
client.projectConfig
client.accountConfig
client.byok
```

## Memory

```ts
const memory = await client.agents.memory.list("agent-id", {
  userId: "user-123",
  includeContents: true,
});

const results = await client.agents.memory.search("agent-id", {
  query: "favorite food",
  user_id: "user-123",
  limit: 20,
});

await client.agents.memory.bulkCreateFacts("agent-id", {
  userId: "user-123",
  facts: [
    { content: "prefers espresso" },
    { content: "based in Singapore", factType: "location" },
  ],
});

const ctx = await client.agents.getContext("agent-id", {
  userId: "user-123",
  query: "what did we discuss about espresso?",
});
```

## Voice (live bidirectional)

```ts
const token = await client.agents.voice.getToken("agent-id", {
  voiceName: "Kore", language: "en-US", userId: "user-123",
});
const stream = await client.agents.voice.stream(token);

stream.sendText("Hello!");
for await (const event of stream) {
  if (event.type === "input_transcript")  console.log("User:",  event.text);
  if (event.type === "output_transcript") console.log("Agent:", event.text);
  if (event.type === "audio") playPCM(event.audio);   // 24 kHz PCM
  if (event.type === "session_ended") break;
}
stream.close();
```

## BYOK

```ts
await client.byok.set("project-id", "openai", "sk-...");
await client.byok.list("project-id");
await client.byok.setActive("project-id", "openai", false);
await client.byok.test("project-id", "gemini");
await client.byok.delete("project-id", "xai");
```

## Webhook verification (HMAC-SHA256)

```ts
import { createHmac, timingSafeEqual } from "node:crypto";

function verify(payload: Buffer, header: string, secret: string) {
  const expected = createHmac("sha256", secret).update(payload).digest("hex");
  const a = Buffer.from(expected, "hex");
  const b = Buffer.from(header, "hex");
  return a.length === b.length && timingSafeEqual(a, b);
}
```

## Error types

```ts
import {
  SonzaiError, AuthenticationError, PermissionDeniedError,
  NotFoundError, BadRequestError, RateLimitError,
  InternalServerError, APIError, StreamError,
} from "@sonzai-labs/agents";

try {
  await client.agents.chat({ agent, messages });
} catch (err) {
  if (err instanceof RateLimitError) {
    // err.retryAfter is in ms
  } else if (err instanceof SonzaiError) {
    // base class
  }
}
```

## Common mistakes

| Mistake | Fix |
|---|---|
| Calling from a React client component / `"use client"` page | Server-side only. Move to a route handler / server action. |
| Putting key in `NEXT_PUBLIC_*` / `VITE_*` / `EXPO_PUBLIC_*` | Same — these get bundled to the browser. |
| Awaiting `chatStream` then iterating | Iterate directly with `for await`. It's an `AsyncGenerator`, not a `Promise`. |
| Using `setTimeout` without backoff in `chatAsync` poll | Use `chatAsyncBlocking` or implement 1→2→4→5s. |
| Confusing `chat` (text-only) and `session.turn` (function-calling) message types | `TurnMessage` carries `tool_call_id` / `tool_calls`; `ChatMessage` does not. |
