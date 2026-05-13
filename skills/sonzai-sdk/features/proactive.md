---
name: feature-proactive
description: Use when adding proactive outreach to an agent — scheduled reminders (recurring), wakeups (one-off), backend events (server-pushed). Three sources of notifications; delivery via SSE, polling, or webhooks.
---

# Proactive (schedules + wakeups + events + notifications)

## What it is

Four surfaces working together:

- **Schedules** (`client.schedules`) — **recurring** reminders. Cadence is cron or simple-rrule with timezone. Active window for quiet hours.
- **Wakeups** (`client.agents.schedule_wakeup`) — **one-off** delayed fires. Hours-from-now.
- **Backend events** (`client.agents.trigger_backend_event`) — **server-pushed** triggers ("player leveled up", "order shipped"). Soft context for the next chat.
- **Notifications** (`client.agents.notifications`) — the **delivery queue**. List pending, consume, history.

Three independent sources, one queue, three delivery channels (SSE / polling / webhooks).

## When to use

| Goal | Surface |
|---|---|
| "Check in on the user every Sunday evening" | Schedules (recurring) |
| "Remind me about the doctor appointment in 24h" | Wakeup (one-off) |
| "Player just defeated the boss — react in character" | Backend event |
| "Send a daily morning prompt" | Schedule |
| "Fire-once after onboarding completes" | Wakeup |

## When NOT to use

- One-time messages the user already has on their screen — just send them inline
- Cron jobs unrelated to a user/agent pair — use your own scheduler

## SDK surface

### Schedules (recurring)

```python
# Mount: client.schedules (top-level)
schedule = client.schedules.create(
    agent_id=agent_id,
    user_id="user-123",
    cadence={"cron": "0 18 * * SUN", "timezone": "America/New_York"},
    check_type="general",                  # taxonomy varies; verify in SDK
    intent="Gentle Sunday-evening prompt to reflect on the week.",
    active_window={"hours": "08:00-22:00", "timezone": "America/New_York"},  # optional quiet hours
    starts_at="2026-05-13T00:00:00Z",      # optional
    ends_at="2026-12-31T00:00:00Z",         # optional
)

# List / delete / patch
schedules = client.schedules.list(agent_id=agent_id, user_id="user-123")
client.schedules.delete(schedule.schedule_id)
```

Cadence forms (verify exact shapes in your SDK):
- `{"cron": "<cron-expr>", "timezone": "..."}` — full cron
- `{"simple": {"interval": "daily" | "weekly" | ...}, "timezone": "..."}` — sugar

### Wakeups (one-off)

```python
# Mount: client.agents (since it's agent-scoped)
client.agents.schedule_wakeup(
    agent_id,
    check_type="reminder",                 # taxonomy
    delay_hours=24,
    intent="Remind about the doctor appointment tomorrow.",
    user_id="user-123",
)
```

### Backend events (server-pushed)

```python
client.agents.trigger_backend_event(
    agent_id,
    user_id="user-123",
    event_type="level_up",
    event_description="Player just reached level 25",
    metadata={"new_level": "25"},
    instance_id="us-east",                # optional
)
```

### Notifications (delivery queue)

```python
# List pending for a user
pending = client.agents.notifications.list(
    agent_id,
    status="pending",
    user_id="user-123",
    limit=20,
)
for n in pending.notifications:
    print(f"[{n.check_type}] {n.generated_message}")

# Consume one
client.agents.notifications.consume(agent_id, pending.notifications[0].message_id)

# Full history (regardless of status)
history = client.agents.notifications.history(agent_id, user_id="user-123", limit=50)
```

## Delivery channels

See `decisions/proactive-channel.md`. Same agent can use all three:

| Channel | When |
|---|---|
| SSE inline | User actively chatting in your app |
| Polling | Mobile / web with no persistent connection |
| Webhooks | Server-to-server fanout (Slack, push notifications, email) |

## Decisions linked

- `decisions/proactive-channel.md` — picking delivery channel
- `archetypes/companion.md`, `archetypes/coach-therapist.md` — common proactive users
- `archetypes/game-npc.md` — events (level_up, boss_defeated)

## Common gotchas

- **Schedules vs wakeups** — recurring vs one-off. Don't try to fake recurring with a self-rescheduling wakeup.
- **Timezone is required** for schedules; otherwise the cron runs in UTC and surprises users.
- **Active window filters quiet hours** — set it for user-respecting cadence.
- **`check_type` taxonomy** — verify the allowed values in your SDK / OpenAPI.
- **Notifications queue server-side** — they don't auto-deliver. You poll, subscribe to SSE, or register a webhook.
- **Consuming a notification** is required for client-side reconciliation (so it doesn't reappear on next poll).
- **Inventory linkage** — reminders can read live `inventory.query` results in their trigger context (e.g. "remind to take medication X that they have remaining" — only fires if inventory still has the item).
- **Event metadata is soft context** — informational; the agent will reference it but won't act on it unless prompted.
