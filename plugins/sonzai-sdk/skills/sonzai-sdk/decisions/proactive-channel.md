---
name: decision-proactive-channel
description: Use when picking how to deliver proactive notifications (scheduled reminders, wakeups, backend events) to the user — SSE, polling, or webhooks.
---

# Decision: proactive channel (SSE vs polling vs webhooks)

## The rule

**SSE if user is active in your app.** Lowest latency; requires an open connection.

**Polling if mobile / web with no persistent server connection.** Works anywhere; minor latency.

**Webhooks for server-to-server fanout.** Notifications dispatched to your backend infrastructure (Slack, email service, push notification service).

**Mix them.** Channels don't conflict — the same notification can fire to all three.

## How to apply

| Your runtime | Recommended channel(s) |
|---|---|
| Web app, user has chat window open | SSE inline (notification surfaces in the chat stream) |
| Web app, user has app open but not chat | Polling (`agents.notifications.list(status="pending")`) every 10-30s |
| Mobile app | Polling on app foreground + webhook → APNs/FCM for background |
| Slack / Teams / Discord bot | Webhook → your bot relays to channel |
| Email-based companion | Webhook → your email service |
| SMS-based reminder | Webhook → Twilio/equivalent |
| Backend-to-backend integration (no user-facing UI) | Webhook |

## Why

- **SSE (inline):** Notifications surface in the user's active chat stream. Zero added latency. Requires the user to have an open connection — Cloudflare / ALB / Vercel Edge typically cut at ~100s, so SSE alone is fragile.

- **Polling:** Your client fetches pending notifications periodically via `agents.notifications.list(...)`. Works everywhere. Notifications are queued server-side until consumed (`agents.notifications.consume(notification_id)`). Polling cadence (10-30s) is the latency floor.

- **Webhooks:** Sonzai dispatches a signed HTTP POST to your registered URL when a notification fires. Best for fanout — one webhook → your backend → Slack/email/push. You verify HMAC-SHA256 on the raw bytes (see `features/webhooks.md`). Webhooks survive when the user is offline.

## Notification sources

Three sources fire notifications; the channel is independent of the source:

| Source | API | Triggered by |
|---|---|---|
| Scheduled reminders | `client.schedules.create` (recurring) | Cron/cadence at the configured time |
| Wakeups | `client.agents.schedule_wakeup` (one-off) | Single delayed fire |
| Backend events | `client.agents.trigger_backend_event` | Your server pushes an event |

All three queue notifications. Your delivery channel(s) decide how the user sees them.

## Exceptions

- **Push notifications via APNs / FCM** — wrap a webhook fanout to your push service.
- **Email** — webhook to your email service (SendGrid, Postmark, SES).
- **SMS** — webhook to Twilio / Vonage.
- **Multi-channel** — register the same agent with multiple webhooks (one per event_type); notifications fan out to all relevant channels.

## Cross-references

- `features/proactive.md` — full surface (schedules, wakeups, events, notifications)
- `features/webhooks.md` — registration, HMAC verify, delivery attempts
- `references/streaming-chat.md` — SSE vs polling chat (related but distinct from notification delivery)
- `archetypes/companion.md`, `archetypes/coach-therapist.md` — common proactive users
