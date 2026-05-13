---
name: feature-webhooks
description: Use when registering Sonzai webhooks for events (agent.message.created, agent.created, etc.). HMAC-SHA256 signed deliveries. Includes delivery-attempts inspection and signing-secret rotation.
---

# Webhooks

## What it is

Server-to-server fanout. Sonzai POSTs JSON payloads to your registered URL on event types (chat messages, agent lifecycle, custom-tool fires). Every delivery is HMAC-SHA256 signed with a per-registration secret. Project-scoped variants exist for multi-project ops.

## When to use

- Audit trail to your SIEM / S3 / logging service
- Notification fanout to Slack / email / push services
- Backend integration with existing event-driven systems
- Custom-tool delivery to your backend

## When NOT to use

- Real-time UI updates (use SSE inline or polling)
- Tight loops where latency matters (webhooks are eventually-consistent)

## SDK surface

```python
# Register
resp = client.webhooks.register(
    event_type="agent.message.created",
    webhook_url="https://your-app/webhook/sonzai",
    auth_header="Bearer your-secret",     # optional — added to delivery requests
)
print(resp.signing_secret)                # save this — needed for HMAC verify

# List
hooks = client.webhooks.list()

# Inspect delivery attempts
attempts = client.webhooks.list_delivery_attempts("agent.message.created")
for a in attempts:
    print(a.timestamp, a.status_code, a.attempt_number)

# Rotate signing secret (quarterly recommended)
new = client.webhooks.rotate_secret("agent.message.created")
print(new.signing_secret)                  # update your verify helper

# Delete
client.webhooks.delete("agent.message.created")

# Project-scoped variants
client.webhooks.register_for_project("project-id", "agent.created", webhook_url=...)
client.webhooks.list_for_project("project-id")
client.webhooks.delete_for_project("project-id", "agent.created")
```

```typescript
const resp = await client.webhooks.register("agent.message.created", {
  webhookUrl: "https://your-app/webhook/sonzai",
  authHeader: "Bearer your-secret",
});
const secret = resp.signing_secret;
```

```go
resp, err := client.Webhooks.Register(ctx, "agent.message.created", sonzai.WebhookRegisterOptions{
    WebhookURL: "https://your-app/webhook/sonzai",
})
```

## Common event types

(Verify exact list via your dashboard or `client.webhooks.list_event_types` if available.)

- `agent.message.created` — every chat turn fires this; useful for audit
- `agent.created` — new agent registered
- `agent.deleted` — agent removed
- `agent.session.ended` — session lifecycle
- `agent.notification.fired` — proactive notification dispatched
- `agent.tool.called` — custom tool invoked
- (more — depends on your tier / project config)

## HMAC verification (mandatory)

Verify HMAC-SHA256 over the **raw request body** (not the parsed JSON), using the `signing_secret` from registration. Timing-safe compare.

```python
import hmac, hashlib

def verify_sonzai_webhook(payload_bytes: bytes, header_hex: str, secret: str) -> bool:
    expected = hmac.new(secret.encode(), payload_bytes, hashlib.sha256).hexdigest()
    return hmac.compare_digest(expected, header_hex)

# In your Flask/FastAPI/Express handler
raw_body = request.body                    # or request.get_data() depending on framework
sig_header = request.headers["X-Sonzai-Signature"]  # verify exact header name
if not verify_sonzai_webhook(raw_body, sig_header, signing_secret):
    return 401
# parse and process
```

```typescript
import { createHmac, timingSafeEqual } from "node:crypto";

function verify(payload: Buffer, header: string, secret: string) {
  const expected = createHmac("sha256", secret).update(payload).digest("hex");
  const a = Buffer.from(expected, "hex");
  const b = Buffer.from(header, "hex");
  return a.length === b.length && timingSafeEqual(a, b);
}
```

## Decisions linked

- `decisions/proactive-channel.md` — webhook is one of three delivery channels for notifications
- `archetypes/enterprise-assistant.md`, `archetypes/customer-support.md` — primary webhook users
- `features/proactive.md` — webhooks deliver scheduled / wakeup / event notifications

## Common gotchas

- **Verify over raw bytes** — parsing the JSON first changes byte representation (whitespace, key ordering) and breaks HMAC.
- **Timing-safe compare** — use `hmac.compare_digest` / `timingSafeEqual`; `==` leaks timing.
- **Retries** — Sonzai retries on 5xx / connection failures with exponential backoff. Your handler should be idempotent (dedupe by event ID).
- **Delivery attempts inspection** is your debugging tool — `list_delivery_attempts` shows status codes, retry count, timing.
- **Rotate signing secret quarterly** — set a rotation cadence in your runbook.
- **`authHeader`** is an optional credential you provide that Sonzai adds to delivery requests (helpful if your endpoint is behind a gateway requiring its own auth).
- **Project-scoped variants** for multi-project setups — different webhook per project.
- **Event ordering** — webhooks fire in approximately the order events occurred but no strict ordering guarantee.
- **Payload size limits** — typical limits around 1MB; truncated for very large chat transcripts.
