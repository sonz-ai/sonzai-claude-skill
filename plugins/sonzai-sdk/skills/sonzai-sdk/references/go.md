# Go — github.com/sonz-ai/sonzai-go

Repo: [`github.com/sonz-ai/sonzai-go`](https://github.com/sonz-ai/sonzai-go) · Go 1.25+ · Standard library only (no external deps)

## Install

```bash
go get github.com/sonz-ai/sonzai-go@latest
```

## Client init

```go
import sonzai "github.com/sonz-ai/sonzai-go"

// Reads SONZAI_API_KEY from env when arg is ""
client, err := sonzai.NewClient("")
if err != nil { return err }

// Or explicit
client, err := sonzai.NewClient("sk-...")
```

`SONZAI_BASE_URL` overrides the default `https://api.sonz.ai`.

## Quick Start — chat once

```go
resp, err := client.Agents.Chat(ctx, sonzai.AgentChatParams{
    AgentID: "agent-id",
    ChatOptions: sonzai.ChatOptions{
        Messages: []sonzai.ChatMessage{{Role: "user", Content: "Hello!"}},
        UserID:   "user-123",
    },
})
```

## Quick Start — streaming (SSE)

```go
err := client.Agents.ChatStream(ctx, sonzai.AgentChatParams{
    AgentID: "agent-id",
    ChatOptions: sonzai.ChatOptions{
        Messages: []sonzai.ChatMessage{{Role: "user", Content: "Tell me a story"}},
    },
}, func(event sonzai.ChatStreamEvent) error {
    fmt.Print(event.Content())
    return nil
})
```

## Streaming from short-deadline contexts (NATS, Watermill, queue workers)

If the caller's `ctx` has a deadline shorter than an LLM generation (typical for message handlers), use the **`*Detached` variants** — `ChatDetached`, `ChatStreamDetached`, `ChatStreamChannelDetached`. They decouple the HTTP call from `ctx.Done()` while applying an SDK-managed timeout.

```go
err := client.Agents.ChatStreamDetached(ctx, params, sonzai.DetachOptions{
    Timeout: 120 * time.Second,
}, func(event sonzai.ChatStreamEvent) error {
    // ... handle
    return nil
})
```

Use the non-detached variants only when the caller's `ctx` is long-lived (HTTP request handlers with no aggressive timeout, background workers without per-message cancellation).

## Resources

```go
client.Agents               // chat, context engine, agent-scoped ops
client.Agents.Memory        // tree, search, facts, timeline
client.Agents.Personality
client.Agents.Sessions
client.Agents.Instances
client.Agents.Notifications
client.Agents.CustomState
client.Agents.Image
client.Agents.Priming
client.Knowledge            // project-scoped KB
client.Eval                 // sub-package: evaluation, simulation, benchmarking
client.Eval.Templates
client.Eval.Runs
client.BYOK                 // bring-your-own-key (v1.5.2+)
```

## Option structs use `Ptr[T]` for nullable booleans

For PATCH-style fields where omitted ≠ false (e.g. `UpdateCapabilitiesOptions.WebSearch`, `AgentToolCapabilities.KnowledgeBase`), wrap with `sonzai.Ptr`:

```go
opts := sonzai.UpdateCapabilitiesOptions{
    WebSearch: sonzai.Ptr(true),    // explicitly set to true
    // RememberName omitted → unchanged
}
```

## Sessions (recommended turn loop)

```go
session, err := client.Agents.Sessions.Start(ctx, sonzai.StartSessionParams{
    AgentID:   "agent-id",
    UserID:    "user-123",
    SessionID: "session-456",
    Provider:  "gemini",
    Model:     "gemini-3.1-flash-lite",
})

sessCtx, err := session.Context(ctx, sonzai.ContextOptions{Query: "..."})
// ... build prompt with sessCtx, call your LLM ...

result, err := session.Turn(ctx, sonzai.TurnOptions{
    Messages: []sonzai.TurnMessage{
        {Role: "user", Content: "..."},
        {Role: "assistant", Content: assistantReply},
    },
    FetchNextContext: &sonzai.ContextOptions{Query: "..."},
})

err = session.End(ctx, sonzai.EndSessionOptions{
    TotalMessages:   10,
    DurationSeconds: 300,
    Wait:            true,
})
```

## BYOK (v1.5.2+)

```go
keys, err := client.BYOK.List(ctx, "project-id")
key, err  := client.BYOK.Set(ctx, "project-id", sonzai.BYOKProviderOpenAI, "sk-...")
err       = client.BYOK.SetActive(ctx, "project-id", sonzai.BYOKProviderOpenAI, false)
result, err := client.BYOK.Test(ctx, "project-id", sonzai.BYOKProviderGemini)
err       = client.BYOK.Delete(ctx, "project-id", sonzai.BYOKProviderXAI)
```

Providers: `BYOKProviderOpenAI`, `BYOKProviderGemini`, `BYOKProviderXAI`, `BYOKProviderOpenRouter`.

## Error handling

```go
var authErr *sonzai.AuthenticationError
var rlErr   *sonzai.RateLimitError

resp, err := client.Agents.Chat(ctx, opts)
switch {
case errors.As(err, &authErr):
    // 401
case errors.As(err, &rlErr):
    // 429 — rlErr.RetryAfter
default:
    // other
}
```

## Common mistakes

| Mistake | Fix |
|---|---|
| Calling `ChatStream` from a NATS/Watermill handler with a short-deadline ctx | Use `ChatStreamDetached` — the caller ctx will cancel mid-generation otherwise. |
| Setting a boolean field directly when the struct uses `*bool` | Wrap with `sonzai.Ptr(true)`. |
| Importing as `import "sonzai-go"` | Module path is `github.com/sonz-ai/sonzai-go`. |
| Reusing one `*sonzai.Client` from many goroutines | Safe — the client is goroutine-safe. Do not re-init per request. |
| Discarding the `error` from `NewClient("")` | It returns an error when `SONZAI_API_KEY` is unset and no key was passed. |
