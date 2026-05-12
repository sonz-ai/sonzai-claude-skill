# Auth & setup

Same model in all three SDKs. The skill assumes the user is building a **server-side** application — SDK keys must never touch a browser, mobile, or desktop client bundle.

## API key

1. Sign in at [`https://platform.sonz.ai`](https://platform.sonz.ai)
2. **Projects → API Keys → Generate**
3. Copy the `sk-...` key (it's shown once)

Keys are scoped to a single project. Use one project per environment (dev/staging/prod).

## Where to put it

| Place | Status |
|---|---|
| Server env var `SONZAI_API_KEY` (default — read automatically by all 3 SDKs) | ✅ Recommended |
| Secrets manager (GCP Secret Manager, AWS Secrets Manager, Vault) injected at boot | ✅ Production |
| `.env.local` for local dev (gitignored) | ✅ Local only |
| Explicit constructor arg, sourced from a secret | ✅ Fine |
| `NEXT_PUBLIC_*` / `VITE_*` / `EXPO_PUBLIC_*` | ❌ Browser-exposed |
| Committed to the repo | ❌ Rotate it now |
| Frontend / mobile / desktop client | ❌ Server-side only |

## Base URL

Default: `https://api.sonz.ai`. Override with `SONZAI_BASE_URL` env var or the explicit `base_url` / `baseUrl` config field. The OpenAPI spec is at `https://api.sonz.ai/docs/openapi.json`.

## Per-language minimum init

```python
# Python
from sonzai import Sonzai
client = Sonzai()                                   # reads SONZAI_API_KEY
```

```ts
// TypeScript
import { Sonzai } from "@sonzai-labs/agents";
const client = new Sonzai();                        // reads SONZAI_API_KEY
```

```go
// Go
import sonzai "github.com/sonz-ai/sonzai-go"
client, err := sonzai.NewClient("")                 // reads SONZAI_API_KEY
```

## Auth header sent on every request

```
Authorization: Bearer sk-...
```

If you need to call an endpoint with raw HTTP (drift fallback), use that header.

## API key scopes

API keys carry scopes. For features beyond basic chat, the key needs the matching scope:

| Feature | Required scope(s) |
|---|---|
| BYOK list/get | `read:byok` |
| BYOK set/delete/setActive/test | `write:byok` |
| Webhooks management | `write:webhooks` |
| Knowledge base writes | `write:knowledge` |

If you get `403 Forbidden`, check the scopes on the key in the dashboard.

## Web app integration shape

Don't expose the SDK to browsers. Pattern:

```
Browser → your-app.com/api/* (your backend) → api.sonz.ai (Sonzai SDK)
```

In Next.js: use route handlers (`app/api/.../route.ts`) or server actions. Never `"use client"` + `new Sonzai()`.

In Expo / React Native: same — your own server proxies the call. The mobile app talks to your server, not Sonzai.
