---
name: archetype-customer-support
description: Use when building an external-facing AI customer support agent. KB-backed answers, custom tools for ticket creation and order lookup, webhook fanout to ticketing systems (Zendesk, Intercom, Jira) and escalation channels (Slack, PagerDuty).
---

# Customer support archetype

A customer-facing support agent grounded in product/policy knowledge. Answers from the KB, creates tickets via custom tools, escalates urgent issues via webhooks. Brand-locked voice. Audited.

## 1. When this archetype fits

**Strong signals:**
- External customers asking product / billing / policy questions
- You have FAQ / product docs to upload as KB
- Ticketing system integration needed (Zendesk / Intercom / Jira / custom)
- Escalation paths exist (urgent → Slack #incidents / PagerDuty)
- CS team learns from each other's tickets → shared memory pattern

**Anti-signals:**
- Internal team bot → `archetypes/enterprise-assistant.md`
- 1:1 companion / no KB → `archetypes/companion.md`
- Pure FAQ with no ticket creation → consider just a KB search endpoint without an agent

## 2. Prescribed stack

| Decision | Value | Why |
|---|---|---|
| `memory_mode` | `sync` | Compliance — every retrieval in record same turn |
| Personality | brand-locked via prompt | Consistent CS voice; no per-customer drift |
| `shared_memory` | `on` | CS team learns from each other's tickets |
| `wisdom` | `on` (required for shared_memory) | K-anonymized cross-customer patterns |
| `knowledge_base` | `on` | FAQ + product docs |
| `knowledge_base_write` | optional | If you want the agent to auto-record CS notes |
| `knowledge_base_scope_mode` | `project_only` typically | One project per product/region |
| `web_search` | **off** typically | Hallucination risk on policy answers; KB is authority |
| `remember_name` | `on` | Customers expect personalization |
| `image_generation` | off | Off-topic risk |
| `inventory` | off | Use orders/tickets via custom tools instead |
| Custom tools | `create_ticket`, `escalate`, `lookup_order`, etc. | Backend integration |
| Webhooks | `agent.message.created` for audit; tool-call webhooks for ticket/escalation | Server-to-server fanout |

## 3. Required SDK functions (in order)

### Step 1 — Create the agent (brand-locked)

```python
from sonzai import Sonzai
client = Sonzai()

agent = client.agents.create(
    name="SupportAgent",
    personality_prompt=(
        "You are a customer support agent for <Product>. Voice: warm, "
        "professional, never sycophantic. Answer from the knowledge base; "
        "if the answer isn't there, say so and offer to create a ticket. "
        "Never invent product features, prices, or policies. Verify order "
        "details via lookup_order before discussing specifics. For urgent "
        "issues (downtime, data loss, payment failure), call escalate()."
    ),
    language="en",
)
```

### Step 2 — Set capabilities (KB-grounded, no web_search)

```python
client.agents.update_capabilities(
    agent.agent_id,
    memory_mode="sync",
    shared_memory=True,
    wisdom=True,
    knowledge_base=True,
    knowledge_base_scope_mode="project_only",
    web_search=False,                    # KB is authority
    remember_name=True,
)
```

### Step 3 — Upload knowledge base

```python
import os
for fname in os.listdir("kb/"):
    with open(f"kb/{fname}", "rb") as f:
        client.knowledge.upload_document(
            project_id="your-project-id",
            file=f,
            filename=fname,
        )
```

Re-upload after content changes (or use delta uploads if your tier supports them).

### Step 4 — Register custom tools

```python
# Ticket creation
client.agents.create_custom_tool(
    agent.agent_id,
    name="create_ticket",
    description="Create a support ticket when the issue requires human follow-up.",
    parameters={
        "type": "object",
        "properties": {
            "subject": {"type": "string"},
            "description": {"type": "string"},
            "priority": {"type": "string", "enum": ["low", "normal", "high", "urgent"]},
            "category": {"type": "string"},
        },
        "required": ["subject", "description", "priority"],
    },
)

# Order lookup (read-only)
client.agents.create_custom_tool(
    agent.agent_id,
    name="lookup_order",
    description="Look up an order's details by ID. Use before discussing order-specific topics.",
    parameters={
        "type": "object",
        "properties": {"order_id": {"type": "string"}},
        "required": ["order_id"],
    },
)

# Escalation
client.agents.create_custom_tool(
    agent.agent_id,
    name="escalate",
    description="Escalate an urgent issue (downtime, data loss, payment failure). Notifies the on-call channel.",
    parameters={
        "type": "object",
        "properties": {
            "reason": {"type": "string"},
            "severity": {"type": "string", "enum": ["sev1", "sev2", "sev3"]},
        },
        "required": ["reason", "severity"],
    },
)
```

### Step 5 — Chat handler + tool-call dispatch

```python
def handle_support_message(customer_id, message_text):
    session = client.agents.sessions.start(
        agent.agent_id,
        user_id=customer_id,
        session_id=f"support-{customer_id}",
    )
    result = session.turn(
        messages=[{"role": "user", "content": message_text}],
    )

    # Dispatch any tool calls the agent fired
    for tool_call in (result.side_effects.external_tool_calls or []):
        if tool_call.name == "create_ticket":
            create_zendesk_ticket(customer_id, **tool_call.arguments)
        elif tool_call.name == "lookup_order":
            order = order_db.get(tool_call.arguments["order_id"])
            # Feed result back via session.turn with tool_call_id (next loop iteration)
        elif tool_call.name == "escalate":
            slack_alert("#cs-escalations", **tool_call.arguments)

    return result.response
```

### Step 6 — Register audit webhook

```python
resp = client.webhooks.register(
    event_type="agent.message.created",
    webhook_url="https://your-app/webhook/sonzai-audit",
    auth_header="Bearer YOUR_SECRET",
)
signing_secret = resp.signing_secret
```

Verify HMAC-SHA256 on every delivery (see `features/webhooks.md`).

### Step 7 — (Optional) Per-tenant scoping

If you serve multiple business customers (B2B SaaS for support), use separate **projects** per tenant — not instances. See `decisions/instances-vs-multitenant.md`.

## 4. Archetype-specific wizard intake questions

After Q2 picks `customer-support`, the wizard asks:

1. **Ticketing system?** Zendesk / Intercom / Jira / Freshdesk / HubSpot / custom backend.
2. **Escalation channels?** Slack channel ID / PagerDuty service / email distribution list.
3. **KB documents to upload at deploy?** [list of file paths / URLs]
4. **Tool authentication strategy?** Direct backend call (your handler dispatches tool calls) / webhook (Sonzai delivers tool fire to a registered endpoint) / both.
5. **Multi-tenant?** No (one product) / Yes (B2B SaaS — separate project per tenant). Affects KB scoping and webhook routing.
6. **Voice / phone IVR variant?** Off / yes (tier-gated, see `features/voice.md`).

## 5. Spec template fields

```markdown
## Agent

- name, personality_prompt (brand-locked CS voice)
- Capabilities: memory_mode=sync, shared_memory+wisdom, KB+optional write, web_search=off

## Knowledge base

- Document corpus: <list>
- Refresh cadence: <on every doc change | weekly | manual>
- KB scope: project_only

## Custom tools

- create_ticket (ticketing-system fields per Step 4)
- lookup_order (read-only)
- escalate (severity + reason)
- (additional tools per your backend)

## Tool dispatch

- Handler: <your server endpoint that processes side_effects.external_tool_calls>
- Authentication: HMAC verify on every tool-fire delivery

## Webhooks

- agent.message.created → audit destination
- (optional) tool-call delivery webhook
- Signing secret rotation: quarterly

## Multi-tenancy (if applicable)

- One Sonzai project per business customer
- KB scoped per project
- API keys per project
```

## 6. Plan template

```markdown
1. Create agent with brand-locked personality_prompt. Verify exists.
2. Set capabilities (sync, shared_memory, wisdom, KB, no web_search). Verify get_capabilities.
3. Upload KB documents (loop over kb/ directory). Verify knowledge.listDocuments.
4. Register custom tools: create_ticket, lookup_order, escalate. Verify each appears in agents.get_capabilities().customTools.
5. Implement chat handler in your server, with tool-call dispatch. Verify a "where is my order" message triggers lookup_order.
6. Implement HMAC verify on the agent.message.created webhook. Verify timing-safe compare.
7. Register the audit webhook. Test delivery; check delivery attempts in dashboard or via webhooks.list_delivery_attempts.
8. Implement escalation handler (Slack/PagerDuty fanout). Verify a sev1 escalate() fires the right channel.
9. Integration test: customer asks about an order, agent calls lookup_order, returns formatted answer. Verify side_effects captured.
10. Integration test: customer asks an off-KB question, agent says "I don't know, want me to file a ticket?" Verify hallucination is suppressed.
11. Privacy test: shared_memory floor categories prevent cross-customer leakage. Verify with a test query.
12. Production checklist: secret rotation cadence documented, KB re-upload runbook, BYOK if billing isolation required.
```

## 7. Anti-patterns

| Anti-pattern | Why it's wrong | Fix |
|---|---|---|
| Letting `web_search=true` for KB-grounded support | Hallucination risk on policy — agent invents external sources | Keep web_search off; if a customer asks about external products, route to a different surface |
| Skipping HMAC verification on tool callbacks | Backend trusts unauthenticated requests claiming to be Sonzai | Always verify with the `signing_secret` from webhook registration; timing-safe compare |
| Forgetting webhook delivery failures | Lost tickets, lost escalations | Inspect `webhooks.list_delivery_attempts`; alert on repeated failure for the same event |
| Enabling `image_generation` | Customers using agent for off-topic image gen drives cost + brand risk | Off by default; only enable if there's a specific use case |
| Sharing `compiled_system_prompt` updates without cache invalidation | Inconsistent voice across sessions during rollout | Use rolling deploy + explicit cache clear |
| Calling `escalate` for non-urgent issues | PagerDuty noise → on-call fatigue → real escalations missed | Tool description should be specific: "sev1=service down, sev2=major degradation"; agent over-escalation is a sign the prompt is too permissive |
| Sharing `shared_memory` across distinct B2B tenants | Customer A's data leaks to customer B's agent | Separate Sonzai project per tenant; never share KB or shared_memory across tenants |

## Cross-references

- `decisions/memory-mode.md` — why sync for compliance
- `decisions/sharedmemory-vs-wisdom.md` — privacy floor for CS team learning
- `decisions/instances-vs-multitenant.md` — B2B SaaS tenant isolation
- `decisions/capabilities-matrix.md` — full capability grid
- `features/custom-tools.md` — tool registration + dispatch + tool-call shape
- `features/webhooks.md` — HMAC verify pattern, delivery attempts inspection
- `features/knowledge-base.md` — KB upload + search + audit
- `features/shared-memory.md` — privacy floor + disclosure audit
- `migrations/openai-assistants.md` — common starting point for migrating from Assistants API
- `archetypes/enterprise-assistant.md` — similar but internal-facing; overlap in stack
