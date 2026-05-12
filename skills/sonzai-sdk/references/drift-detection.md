# Drift detection — SDK ↔ live API

The Sonzai backend ships continuously. Each language SDK commits a **snapshot** of the OpenAPI spec (`openapi.json`) at the time of release. If the user's installed SDK version is older than the current backend, the SDK's typed surface will be missing endpoints, fields, or enum values that exist in production. This is "drift."

**Do this check before writing any non-trivial integration code.** Five minutes here saves an hour of debugging mysteriously-missing methods.

## What to check

| Question | How |
|---|---|
| What SDK version is installed? | `pip show sonzai` / `npm list @sonzai-labs/agents` / `go list -m github.com/sonz-ai/sonzai-go` |
| What's the latest published SDK? | `pip index versions sonzai` / `npm view @sonzai-labs/agents version` / `go list -m -versions github.com/sonz-ai/sonzai-go` |
| What's the live API spec? | `curl -s https://api.sonz.ai/docs/openapi.json` |
| What spec did the installed SDK commit against? | Located inside the installed package — see paths below |

## Step 1 — Find the installed SDK version

### Python
```bash
pip show sonzai | grep -i version
# or:
python -c "import sonzai; print(sonzai.__version__)"
```

### TypeScript
```bash
npm list @sonzai-labs/agents 2>/dev/null | grep agents
# or check package.json directly:
jq '.dependencies["@sonzai-labs/agents"] // .devDependencies["@sonzai-labs/agents"]' package.json
```

### Go
```bash
go list -m github.com/sonz-ai/sonzai-go
# or grep the go.mod:
grep sonzai-go go.mod
```

## Step 2 — Pull the live spec

```bash
curl -sSfL https://api.sonz.ai/docs/openapi.json -o /tmp/live.openapi.json
jq '.info.version' /tmp/live.openapi.json
```

The `info.version` field gives you the API server's build identifier. If you cannot reach `api.sonz.ai` (no network, sandbox, air-gapped CI), stop and tell the user — silently falling back to the SDK's snapshot risks generating stale code.

## Step 3 — Find the committed snapshot in the installed package

The snapshot ships **with the source repos** (not necessarily inside the installed package). The fastest way to compare is to clone the SDK repo at the matching version tag:

```bash
# Python
git clone --depth 1 --branch v$(pip show sonzai | awk '/Version:/ {print $2}') \
  https://github.com/sonz-ai/sonzai-python /tmp/sdk
jq '.info.version' /tmp/sdk/openapi.json

# TypeScript
git clone --depth 1 --branch v$(npm view @sonzai-labs/agents@$(jq -r '.dependencies["@sonzai-labs/agents"] // .devDependencies["@sonzai-labs/agents"]' package.json | tr -d '^~') version) \
  https://github.com/sonz-ai/sonzai-typescript /tmp/sdk
jq '.info.version' /tmp/sdk/openapi.json

# Go
go env GOPATH
ls "$(go env GOPATH)/pkg/mod/github.com/sonz-ai/sonzai-go@$(go list -m -f '{{.Version}}' github.com/sonz-ai/sonzai-go)/openapi.json" 2>/dev/null || echo "fallback: clone repo"
```

(If those tag patterns don't resolve cleanly — the repo may not tag every release — fall back to `git clone main` and bisect, or skip the local snapshot and rely on the live spec alone.)

## Step 4 — Diff

Quick check: do the operationIds and paths the user needs to call exist in the live spec?

```bash
# List every operationId in the live spec
jq -r '.paths | to_entries[] | .key as $path | .value | to_entries[] | "\(.key | ascii_upcase) \($path) — \(.value.operationId // "no-opid")"' /tmp/live.openapi.json | sort | less
```

Targeted check — does `agents_chat_async` exist?
```bash
jq -r '.paths | to_entries[] | .value | to_entries[] | .value.operationId' /tmp/live.openapi.json | grep -i async
```

Field-level check — what params does `POST /api/v1/agents` accept right now?
```bash
jq '.paths["/api/v1/agents"].post.requestBody.content["application/json"].schema' /tmp/live.openapi.json
```

## Step 5 — Decide

| State | Action |
|---|---|
| Live spec version == installed SDK's snapshot | Proceed with typed SDK; you're in sync. |
| Live spec is newer, but the endpoint you need exists in both | Proceed with typed SDK. |
| Live spec has an endpoint/field the SDK lacks | Two options — see below |
| Live spec is **older** than the SDK | Unusual. Tell the user; production may have rolled back. |
| Cannot reach `api.sonz.ai` | Stop. Don't guess. Tell the user. |

### When the SDK is missing what you need

**Option A: upgrade the SDK** (preferred)
```bash
pip install -U sonzai
npm install @sonzai-labs/agents@latest
go get -u github.com/sonz-ai/sonzai-go@latest
```
Then verify the symbol now exists, e.g. `python -c "from sonzai import Sonzai; help(Sonzai().agents.chat_async)"`.

**Option B: fall back to raw HTTP for just that endpoint** — keep the typed SDK for everything else, drop to HTTP for the missing surface:

```python
# Python — reuse the SDK's auth + base URL
import httpx, os
r = httpx.post(
    f"{os.getenv('SONZAI_BASE_URL', 'https://api.sonz.ai')}/api/v1/.../new-endpoint",
    headers={"Authorization": f"Bearer {os.environ['SONZAI_API_KEY']}"},
    json={...},
    timeout=30.0,
)
r.raise_for_status()
```

```typescript
// TypeScript — same idea
const r = await fetch(`${process.env.SONZAI_BASE_URL ?? "https://api.sonz.ai"}/api/v1/.../new-endpoint`, {
  method: "POST",
  headers: {
    Authorization: `Bearer ${process.env.SONZAI_API_KEY}`,
    "Content-Type": "application/json",
  },
  body: JSON.stringify({...}),
});
if (!r.ok) throw new Error(`${r.status}: ${await r.text()}`);
```

```go
// Go — same idea using net/http
req, _ := http.NewRequestWithContext(ctx, "POST",
    os.Getenv("SONZAI_BASE_URL")+"/api/v1/.../new-endpoint",
    bytes.NewReader(body))
req.Header.Set("Authorization", "Bearer "+os.Getenv("SONZAI_API_KEY"))
req.Header.Set("Content-Type", "application/json")
resp, err := http.DefaultClient.Do(req)
```

**Mark the fallback clearly in code** so it gets removed once the SDK catches up:
```python
# TODO(sonzai-sdk-drift): drop raw HTTP once sonzai >= X.Y.Z covers this endpoint
```

## Common drift symptoms (and what they mean)

| Symptom | Likely cause |
|---|---|
| `AttributeError: 'Agents' object has no attribute 'foo'` (Python) | SDK is older than the feature. Upgrade or fall back. |
| `Property 'foo' does not exist on type ...` (TypeScript) | Same. Upgrade or fall back. |
| `404 Not Found` on an endpoint the README shows | README is from a newer SDK version than what's installed. Check installed version. |
| `400 Bad Request: unknown field 'bar'` | Server has tightened validation. Field was removed/renamed. Check live spec. |
| Streaming events arrive with unexpected `type` values | New event types added. Add a default branch that logs+ignores. |

## When to NOT bother with the drift check

- One-off script, throwaway code, prototype
- User explicitly said "just use what's in the SDK, I don't care about new endpoints"
- You only need a single, stable, well-known endpoint (`agents.chat`, `agents.get` — these don't drift in shape)

For production integration work, always check.
