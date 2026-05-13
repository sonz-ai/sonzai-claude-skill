---
name: feature-custom-tools
description: Use when the agent needs to call functions in your backend (create_ticket, give_item, spend_currency, lookup_order). Tool calls surface in chat response side_effects; your handler dispatches them.
---

# Custom tools

## What it is

Tool definitions you register on an agent. The agent can call them during chat. Tool invocations surface in the chat response's `side_effects.external_tool_calls`. Your backend dispatches them; results can be fed back via the next `session.turn`.

## When to use

- Backend integrations: tickets, orders, inventory mutations, escalations
- Game mechanics: spend_currency, give_item, award_xp
- External APIs: lookup_order, send_email, post_message
- Anything the agent decides to do that has a side effect outside the conversation

## When NOT to use

- Read-only data the agent needs *every turn* — put it in priming metadata or `custom_states`
- Information available via Sonzai's built-in capabilities (KB, inventory, web search) — use those
- Per-turn conditionals (the agent decides whether to call) — that's exactly what tools are for; **DO** use them

## SDK surface

```python
# Register a tool (agent-level — persists across sessions)
client.agents.create_custom_tool(
    agent_id,
    name="create_ticket",
    description=(
        "Create a customer support ticket when the issue needs human follow-up. "
        "Use for complex problems, refund requests, or escalations."
    ),
    parameters={                          # JSON Schema for tool args
        "type": "object",
        "properties": {
            "subject": {"type": "string"},
            "description": {"type": "string"},
            "priority": {"type": "string", "enum": ["low", "normal", "high", "urgent"]},
        },
        "required": ["subject", "description", "priority"],
    },
)

# Update / delete (verify exact surface in your SDK)
client.agents.update_custom_tool(agent_id, tool_id, ...)
client.agents.delete_custom_tool(agent_id, tool_id)
```

```typescript
await client.agents.createCustomTool(agentId, {
  name: "create_ticket",
  description: "...",
  parameters: { type: "object", properties: { ... }, required: [...] },
});
```

## Session-level tools (per-call instead of persistent)

```python
# Inject tools just for this session
session = client.agents.sessions.start(agent_id, user_id="...", session_id="...")
session.set_tools(tool_definitions=[
    {"name": "ad_hoc_tool", "description": "...", "parameters": {...}},
])
```

Use session-level tools for short-lived flows or A/B tests.

## Dispatching tool calls

```python
result = session.turn(messages=[{"role": "user", "content": "..."}])

for tool_call in (result.side_effects.external_tool_calls or []):
    if tool_call.name == "create_ticket":
        ticket_id = create_in_zendesk(**tool_call.arguments)
        # Optional: feed result back as tool message in next turn
        session.turn(messages=[
            {"role": "tool", "tool_call_id": tool_call.id, "content": f"Created ticket {ticket_id}"},
        ])
    elif tool_call.name == "spend_currency":
        game_db.deduct_gold(user_id, tool_call.arguments["amount"])
```

## Reserved prefix

Tool names starting with `sonzai_` are platform-managed (e.g. `sonzai_wisdom_set`, `sonzai_load_skill`, `sonzai_create_skill`). **You cannot register tools with this prefix.** Pick names that don't collide.

## Decisions linked

- `archetypes/customer-support.md` — heavy custom-tool usage
- `archetypes/game-npc.md` — `give_item`, `spend_currency`, `award_xp`
- `features/webhooks.md` — webhook-delivered tool fire (server-to-server)
- `features/capabilities.md` — `composio` capability flag for SaaS-integration tools (Gmail, Calendar, Slack, GitHub, Linear)

## Common gotchas

- **Tool descriptions matter.** Vague descriptions = agent over-calls or under-calls. Be specific: "Use for X. Don't use for Y."
- **Idempotency** — Sonzai may retry tool calls on transient failures. Your handler should be idempotent (use a deduplication key from `tool_call.id`).
- **Tool callbacks via webhooks** — if you register a webhook for tool delivery, **HMAC-verify** every request. See `features/webhooks.md`.
- **Reserved prefix** — don't use `sonzai_` as a prefix.
- **Tool errors** — return errors in the `tool` message; the agent will read them and adapt next turn.
- **Argument validation** — Sonzai validates against your `parameters` JSON Schema before invoking, but your handler should still defensively validate (don't trust upstream).
- **Sensitive tool args** — tool arguments are logged in audit trails; don't put secrets in argument names/values.
