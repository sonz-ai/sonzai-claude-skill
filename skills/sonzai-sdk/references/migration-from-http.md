# Migrating from raw HTTP to the typed SDK

If the user has `curl`, `fetch`, `requests`, `httpx`, or `http.Client` calls against `api.sonz.ai`, replacing them with the typed SDK gets you: retries on idempotent calls, typed responses, automatic SSE parsing, schema-validated request bodies, and free updates when the API evolves.

## When NOT to migrate

- The endpoint is not yet in the installed SDK version (drift — see `drift-detection.md`). Keep the raw HTTP call, mark with a `TODO(sonzai-sdk-drift)` comment.
- Internal one-off probe/healthcheck script. Raw HTTP is fine.
- The user explicitly wants no SDK dependency (tiny edge function, cold-start sensitive).

## Migration workflow

1. **Find every call site.** Grep for the base URL and obvious patterns:
   ```bash
   rg -n 'api\.sonz\.ai|SONZAI_BASE_URL|api/v1/agents|api/v1/projects'
   ```
2. **Map each raw call to its operationId** in the live spec:
   ```bash
   curl -s https://api.sonz.ai/docs/openapi.json \
     | jq -r '.paths | to_entries[] | .key as $p | .value | to_entries[] | "\(.key|ascii_upcase) \($p) — \(.value.operationId)"'
   ```
3. **Find the typed method** in the SDK that matches that operationId. Naming convention: snake_case for Python, camelCase for TS, PascalCase grouped on resource for Go.
4. **Replace the call.** Keep the same variable names; the typed response will often have the same field names (snake_case in Python, camelCase or snake_case mixed in TS depending on field, PascalCase in Go).
5. **Drop the manual error handling and retry code** — the SDK handles it for idempotent requests. Keep error handling only for cases you explicitly care about.

## Common before/after mappings

### Create an agent

**Before — Python `requests`:**
```python
import os, requests
r = requests.post(
    "https://api.sonz.ai/api/v1/agents",
    headers={"Authorization": f"Bearer {os.environ['SONZAI_API_KEY']}"},
    json={"name": "Luna", "language": "en", "big5": {"openness": 0.75}},
    timeout=30,
)
r.raise_for_status()
agent = r.json()
```

**After:**
```python
from sonzai import Sonzai
client = Sonzai()
agent = client.agents.create(name="Luna", language="en", big5={"openness": 0.75})
```

### Chat (streaming)

**Before — TypeScript raw `fetch` + manual SSE parse:**
```ts
const r = await fetch("https://api.sonz.ai/api/v1/agents/agent-id/chat", {
  method: "POST",
  headers: {
    Authorization: `Bearer ${process.env.SONZAI_API_KEY}`,
    "Content-Type": "application/json",
    Accept: "text/event-stream",
  },
  body: JSON.stringify({ messages: [...], stream: true }),
});
// ...manual SSE buffer parsing, line splitting, data: stripping...
```

**After:**
```ts
import { Sonzai } from "@sonzai-labs/agents";
const client = new Sonzai();
for await (const event of client.agents.chatStream({ agent: "agent-id", messages: [...] })) {
  process.stdout.write(event.choices?.[0]?.delta?.content ?? "");
}
```

### Chat (non-streaming, Go)

**Before — `net/http`:**
```go
body, _ := json.Marshal(map[string]any{"messages": msgs, "user_id": "user-123"})
req, _ := http.NewRequestWithContext(ctx, "POST",
    "https://api.sonz.ai/api/v1/agents/agent-id/chat",
    bytes.NewReader(body))
req.Header.Set("Authorization", "Bearer "+os.Getenv("SONZAI_API_KEY"))
req.Header.Set("Content-Type", "application/json")
resp, err := http.DefaultClient.Do(req)
// ... read, json.Unmarshal into ad-hoc struct
```

**After:**
```go
import sonzai "github.com/sonz-ai/sonzai-go"
client, _ := sonzai.NewClient("")
resp, err := client.Agents.Chat(ctx, sonzai.AgentChatParams{
    AgentID: "agent-id",
    ChatOptions: sonzai.ChatOptions{
        Messages: []sonzai.ChatMessage{{Role: "user", Content: "Hello"}},
        UserID:   "user-123",
    },
})
```

## Hybrid pattern: SDK + raw HTTP for missing endpoints

When the typed SDK covers 90% of what you need but lacks one new endpoint, **don't** drop the SDK. Use it for everything it covers, drop to raw HTTP for the gap, mark the gap.

```python
from sonzai import Sonzai
import httpx, os

client = Sonzai()                              # typed for everything covered

# Drift gap — typed method doesn't exist yet in installed sonzai version
def _raw_new_endpoint(payload: dict) -> dict:
    """TODO(sonzai-sdk-drift): replace with client.agents.foo() when sonzai >= X.Y.Z."""
    r = httpx.post(
        f"{os.getenv('SONZAI_BASE_URL', 'https://api.sonz.ai')}/api/v1/.../new-endpoint",
        headers={"Authorization": f"Bearer {os.environ['SONZAI_API_KEY']}"},
        json=payload, timeout=30.0,
    )
    r.raise_for_status()
    return r.json()
```

When the SDK gains the typed method, search for `TODO(sonzai-sdk-drift)` and replace.

## Verification checklist

After migration, before merging:

- [ ] Same logical behavior — the integration test that hit the raw HTTP path still passes against the SDK
- [ ] Same error semantics — code that branched on `r.status_code == 429` now branches on `RateLimitError`
- [ ] Same auth — `SONZAI_API_KEY` (or explicit `api_key=` arg) is the only auth path; no hardcoded `Authorization` headers slipped in
- [ ] Cleaned up imports — `import requests` / `import httpx` removed if no longer used
- [ ] Timeouts preserved — if the old code had a custom timeout, set it on the SDK client (`timeout=`)
- [ ] Retries are intentional — SDK retries idempotent calls by default; if the old code disabled retries deliberately, set `max_retries=0`
