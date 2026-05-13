# Troubleshooting

Match the symptom, apply the fix, then verify.

## Auth

### `401 Unauthorized` / `AuthenticationError`

| Cause | Fix |
|---|---|
| `SONZAI_API_KEY` not set or empty | `echo $SONZAI_API_KEY` — if empty, set it. In Node, `process.env.SONZAI_API_KEY`; in Go, `os.Getenv("SONZAI_API_KEY")`. |
| Key has whitespace / newline | Trim it. Common when copy-pasting from a CI variable. |
| Key was rotated, app didn't reload | Restart the app / redeploy. |
| Using a key from a different project | Each project has its own keys. Check the dashboard. |
| Pointing at `https://platform.sonz.ai` instead of `https://api.sonz.ai` | `SONZAI_BASE_URL` should be the **API**, not the dashboard. |

### `403 Forbidden` / `PermissionDeniedError`

The key is valid but lacks the scope. See `auth-and-setup.md` for scope requirements (`read:byok`, `write:byok`, `write:webhooks`, etc.). Regenerate the key with the right scopes in the dashboard.

## Not found

### `404 Not Found` on an endpoint the README documents

Probable drift. The installed SDK is newer than the API, or vice-versa.

```bash
# Quick check: does the operationId exist in the live spec?
curl -s https://api.sonz.ai/docs/openapi.json \
  | jq -r '.. | objects | select(.operationId?) | .operationId' \
  | grep -i <part_of_method_name>
```

See `drift-detection.md` for the full workflow.

### `404 Not Found` on agent or user

The agent ID or user ID doesn't exist in this project. Common when:
- Using the wrong project's API key
- Agent was deleted
- Typo in the agent UUID

```python
client.agents.list()    # confirm the agent exists in this project
```

## Bad request

### `400 Bad Request: unknown field 'foo'`

The server has tightened validation. Field was renamed or removed. Check the live spec:

```bash
curl -s https://api.sonz.ai/docs/openapi.json \
  | jq '.paths["/api/v1/agents"].post.requestBody.content["application/json"].schema'
```

### `400 Bad Request: invalid UUID`

Agent IDs are UUIDs. If you're deriving them from your own entity IDs, use `uuid.uuid5` (Python) / `uuidv5` (TS) / `uuid.NewSHA1` (Go) — see the **Idempotent by Design** callout in the SDK READMEs.

## Rate limiting

### `429 Too Many Requests` / `RateLimitError`

The error carries `retryAfter` (ms in TS, seconds in Python `Retry-After`-style, structured `RetryAfter` in Go). Honor it:

```python
except RateLimitError as e:
    time.sleep(getattr(e, "retry_after", 1.0))
    # then retry
```

If you hit 429s steadily during normal operation, raise the rate limit in the dashboard or batch your calls.

## Streaming

### Stream cuts at ~60s or ~100s

CDN / load balancer idle timeout. Switch to async polling (`chat_async` / `chatAsync`). See `streaming-chat.md`.

### `StreamError` / "unexpected end of stream"

Connection dropped mid-generation. The server-side task is still running — re-poll if you used `chat_async`. If you used `chatStream`, retry the request (idempotent for read-shaped chats; not for tool-call writes).

### Streaming events arrive but `content` is empty

The SDK is older than the new event types. Either upgrade the SDK or default-branch the event handler to log and skip unknown `type` values.

## Sessions

### Memory queries right after `session.end()` return stale data

The consolidation pipeline runs async by default. For tests / benchmarks that query memory immediately:

```python
session.end(total_messages=10, duration_seconds=300, wait=True)
```

`wait=True` blocks until consolidation finishes.

### `extraction_id` status never reaches `done`

Check `session.status(extraction_id)`. If `failed`, look at the error field. If stuck `running` for >5 minutes, it's a real issue — surface it; don't silently retry.

## Webhooks

### Signature verification fails

- Verify over the **raw bytes** of the request body, not the parsed JSON
- Use `timingSafeEqual` / constant-time comparison, not `==`
- The header is hex-encoded HMAC-SHA256

See the TypeScript reference for a working verify helper.

## Network / TLS

### `ConnectionError` to `api.sonz.ai`

- Corporate proxy or air-gapped CI — set `HTTPS_PROXY` / configure the SDK's `customFetch` (TS) / `http.Transport` (Go).
- DNS not resolving — `nslookup api.sonz.ai` to confirm.
- TLS handshake fails — check system clock (TLS validates date) and CA bundle.

## When to escalate to support

- Repeated 5xx errors with no rate-limit headers — `support@sonz.ai`, include the `x-request-id` header from a failed response
- `wait=True` consolidation hangs > 5 minutes — same, include agent ID and approximate time
- Streaming events with new `type` values that aren't documented anywhere — same, include sample frame

For everything else, the symptom is almost always either drift (`drift-detection.md`), auth (above), or transport (CDN/LB timeouts → switch to polling).
