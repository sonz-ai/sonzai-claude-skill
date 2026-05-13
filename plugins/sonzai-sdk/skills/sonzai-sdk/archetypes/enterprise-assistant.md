---
name: archetype-enterprise-assistant
description: Use when building a team-shared AI assistant serving multiple employees (engineering team, sales team, customer-success team). Heavy on knowledge-base, shared memory across team members, audit trail for compliance, sync memory mode.
---

# Enterprise / employee assistant archetype

One agent serving N team members. Learns from each person's interactions, surfaces attributed cross-team facts ("Alice owns the auth domain"), and grounds answers in uploaded policy/product docs. Audit trail is mandatory.

## 1. When this archetype fits

**Strong signals:**
- Internal Slack / Teams / web bot for an engineering / sales / CS team
- 5-500 team members sharing one agent
- Policy / product / runbook docs need to be queryable
- Compliance regime (SOC2, HIPAA-adjacent, internal audit) requires logged retrievals
- "I want the agent to know who owns what" / "agent should remember team context"

**Anti-signals:**
- 1:1 personal companion → `archetypes/companion.md`
- Customer-facing support → `archetypes/customer-support.md` (similar but different prescriptions)
- Multi-tenant SaaS where tenants must be isolated → use separate projects per tenant; see `decisions/instances-vs-multitenant.md`

## 2. Prescribed stack

| Decision | Value | Why |
|---|---|---|
| `memory_mode` | `sync` (default) | Compliance — every retrieval lands in record same turn |
| Runtime mode (Q8) | **A** (default) or **C** (existing internal chat infra) | Most enterprises start at A. Pick C (memory-layer via sessions, with your own LLM) when you already have an internal LLM stack — Anthropic/OpenAI contracts, on-prem inference — that you don't want to replace. See `decisions/runtime-mode.md`. |
| LLM provider (production) | **BYOK** (often required) or Custom LLM | Compliance regimes typically mandate provider-region isolation; BYOK with the right provider region is the answer. Custom LLM with an internal endpoint if external inference is prohibited. See `decisions/byok-vs-customllm.md`. |
| Personality | brand-locked via prompt shaping | Consistent voice across team; no per-user drift surprise |
| `shared_memory` | `on` | Cross-user attributed facts (the whole point) |
| `wisdom` | `on` (required precondition for `shared_memory`) | K-anonymized cross-user patterns |
| `knowledge_base` | `on` | Policy/product/runbook docs |
| `knowledge_base_write` | `on` (audited) | Agent can record context autonomously (e.g. CS notes) |
| `knowledge_base_scope_mode` | `cascade` | Reads project KB + org KB; project wins on collision |
| `web_search` | `on` typically | External lookups (docs sites, GitHub issues) |
| `remember_name` | `on` | Names matter in team context |
| `image_generation` | usually off | Off-topic risk |
| `voice_generation` | optional (tier-gated) | If you're building a voice-bot variant |
| `inventory` | off | Not relevant |
| `composio` | optional | Slack/GitHub/Linear/Calendar integration |

## 3. Required SDK functions (in order)

### Step 1 — Create the agent (explicit personality, brand-locked)

```python
from sonzai import Sonzai
client = Sonzai()

agent = client.agents.create(
    name="EngBot",
    personality_prompt=(
        "You are EngBot, the engineering team's internal assistant. "
        "Voice: direct, professional, never sycophantic. Cite sources when "
        "answering policy/process questions. Defer to the team's documented "
        "decisions; flag conflicts. Never invent code paths — verify with KB "
        "first. Default tone: brief unless asked for detail."
    ),
    language="en",
)
```

Brand-locked is enforced via this prompt + `compiled_system_prompt` on every chat call.

### Step 2 — Set capabilities (full enterprise stack)

```python
client.agents.update_capabilities(
    agent.agent_id,
    memory_mode="sync",
    shared_memory=True,
    wisdom=True,                          # required for shared_memory
    knowledge_base=True,
    knowledge_base_write=True,            # audited
    knowledge_base_scope_mode="cascade",
    web_search=True,
    remember_name=True,
)
```

Verify:
```python
caps = client.agents.get_capabilities(agent.agent_id)
assert caps.shared_memory and caps.wisdom and caps.knowledge_base
```

### Step 3 — Upload knowledge base documents

```python
# Project-scoped (team-specific) policy docs
for doc_path in ["policies/sec.pdf", "runbooks/incident.md", "onboarding.md"]:
    with open(doc_path, "rb") as f:
        client.knowledge.upload_document(
            project_id="your-project-id",
            file=f,
            filename=doc_path.split("/")[-1],
        )

# Optionally insert structured org-level facts (visible across all projects in the tenant)
client.knowledge.create_org_node(
    type="Policy",
    label="Code review policy",
    properties={"approvers_required": 2, "blocking_for": ["security/*"]},
)
```

For wider org docs (cross-team policies, brand guidelines), see `features/org-knowledge-base.md`.

### Step 4 — Set up the per-user chat handler

Each team member's chat call uses their stable user_id (Slack `user_id`, Teams `aadObjectId`, SSO sub, etc.).

```python
# Slack handler example
def on_slack_message(slack_event):
    user_id = slack_event["user"]
    session = client.agents.sessions.start(
        agent.agent_id,
        user_id=user_id,
        session_id=f"slack-{user_id}-{slack_event['channel']}",
        provider="gemini",
    )
    result = session.turn(
        messages=[{"role": "user", "content": slack_event["text"]}],
        compiled_system_prompt=(
            "[reinforced brand voice — same as personality_prompt]"
        ),
    )
    post_to_slack(result.response)
```

The `compiled_system_prompt` on every turn keeps the voice consistent across team members despite server-side drift.

### Step 5 — Configure privacy floor (when `shared_memory` is on)

The privacy floor prevents the agent from surfacing protected categories cross-user. Configure at the project/account level — verify the exact mechanism in your tier (dashboard setting or per-call option). The categories typically include: compensation, health, performance reviews, PII.

See `features/shared-memory.md` for the disclosure-audit mechanism that logs every surfaced cross-user fact.

### Step 6 — Register the audit webhook

```python
resp = client.webhooks.register(
    event_type="agent.message.created",
    webhook_url="https://your.audit-sink.com/webhook",
    auth_header="Bearer YOUR_SECRET",
)
signing_secret = resp.signing_secret
# Save signing_secret in your secret manager — HMAC verify every delivery
```

Every chat turn fires this webhook, giving your SIEM / audit-log destination a copy.

### Step 7 — (Optional) Composio integration

For Slack/GitHub/Linear/Calendar/Gmail access via the agent:

```python
client.agents.update_capabilities(
    agent.agent_id,
    composio=True,
)
# Then connect specific apps via your admin dashboard. The agent gets the
# matching tools dynamically based on what's connected.
```

## 4. Archetype-specific wizard intake questions

After Q2 picks `enterprise-assistant`, the wizard asks:

1. **Team size?** Affects KB write quota planning and rate-limit headroom.
2. **KB scope mode?** project_only (just this team's docs) / org_only / **cascade** (recommended — project wins, org fills defaults) / union.
3. **Privacy floor categories?** Compensation / health / performance / PII / custom. Defaults are conservative.
4. **Audit destination?** Slack channel / SIEM endpoint / S3 bucket / multiple webhooks.
5. **Integration channels?** Slack, Teams, Web UI, internal CLI, multiple.
6. **Composio apps?** Slack / GitHub / Linear / Gmail / Calendar / Jira / none.
7. **Voice variant?** Off (default) / yes (tier-gated).

## 5. Spec template fields

```markdown
## Agent

- Strategy: explicit agents.create (brand-locked personality)
- name: "<your team's bot name>"
- personality_prompt: <50-200 words, directive voice>
- compiled_system_prompt strategy: same as personality_prompt, reinforced per call

## Capabilities

- memory_mode: sync
- shared_memory: true
- wisdom: true
- knowledge_base: true
- knowledge_base_write: true (audited)
- knowledge_base_scope_mode: cascade
- web_search: true
- remember_name: true
- composio: <true if connecting SaaS apps>

## Knowledge base

- Documents to upload at deploy: [list of files/URLs]
- Org-level facts (cascade mode): [structured facts]
- Write policy: <which agents/users can author>

## Team identification

- User-id source: <Slack user_id | Teams aadObjectId | SSO sub | custom>
- Channel scoping: <one agent per channel | one agent globally>

## Privacy floor

- Protected categories: [compensation, health, performance, PII, custom]
- Disclosure audit destination: <Slack | SIEM | S3>

## Audit webhook

- Endpoint: https://...
- Signing secret rotation cadence: quarterly
- Retention: <N days/years per compliance>

## Composio (if enabled)

- Connected apps: [Slack, GitHub, Linear, ...]
- Per-app scopes: [...]
```

## 6. Plan template (typical step breakdown)

```markdown
1. Create agent via agents.create with explicit personality_prompt. Verify: agent.agent_id stable; appears in agents.list().
2. Set full capabilities config (sync, shared_memory, wisdom, KB+write, cascade scope, web_search). Verify: get_capabilities returns expected.
3. Upload KB documents (loop). Verify: knowledge.listDocuments shows uploads.
4. (Optional) Insert org-level facts via knowledge.createOrgNode. Verify: queryable cross-project.
5. Configure privacy floor categories at project level. Verify: protected-category test query returns refusal.
6. Implement team-channel chat handler. Verify: user_id maps to team member identity.
7. Register audit webhook with HMAC verification. Verify: test delivery, signing_secret rotation works.
8. (Optional) Composio: enable capability, connect SaaS apps in dashboard. Verify: tools available in agent context.
9. Integration test: 3 team members share one agent. User A says "I own auth"; user B asks "who owns auth?" → agent surfaces A's claim WITH attribution. Verify: disclosure audit logs the surfacing.
10. Privacy test: user A mentions compensation; user B asks "what does A make?" → agent refuses, citing privacy floor. Verify: refusal is logged.
11. Production checklist: secret rotation runbook, SIEM ingestion verified, runbook for capability changes, BYOK if billing isolation required (see decisions/byok-vs-customllm.md).
```

## 7. Anti-patterns

| Anti-pattern | Why it's wrong | Fix |
|---|---|---|
| Enabling `shared_memory=true` without `wisdom=true` | Server-side rule: shared_memory requires wisdom. Update will reject. | Always set `wisdom=true` alongside `shared_memory=true`. |
| Enabling `shared_memory` without privacy floor categories configured | Regulatory risk — agent will surface protected categories | Configure privacy floor before enabling shared_memory; don't ship the capability turn-on without the floor turn-on |
| BYOM to a non-logging custom LLM and claiming compliance | Loses retrieval audit trail | Use BYOK with a logging provider OR use a Custom LLM endpoint that logs to your SIEM |
| Async memory + audit regime | Facts spill to next turn; "in record same turn" requirement fails | Use sync memory for compliance archetypes |
| Sharing `custom_states` across users (assuming it's automatic) | custom_states are per-user when scope="user"; cross-user requires explicit promotion via shared_memory facts | Use shared_memory for cross-user attributed facts, custom_states for per-user state only |
| Omitting `Authorization` header on custom-tool callbacks | Backend gets unauthenticated requests claiming to be Sonzai | Verify HMAC on every webhook delivery; use the `signing_secret` from registration |
| One agent across distinct customer tenants (real multi-tenancy) | Wisdom and shared_memory bleed across customers | Use separate projects per customer; instances within one project are not strict multi-tenancy. See `decisions/instances-vs-multitenant.md`. |
| Showing the audit log to all team members | Disclosure audit reveals private cross-user facts | Audit destination is admin-only; team members see surfaced facts only when relevant to their query |

## Cross-references

- `decisions/memory-mode.md` — why sync is right for compliance
- `decisions/sharedmemory-vs-wisdom.md` — privacy floor + the wisdom precondition
- `decisions/capabilities-matrix.md` — full enterprise capability grid
- `decisions/byok-vs-customllm.md` — billing isolation choices
- `decisions/instances-vs-multitenant.md` — when to use separate projects
- `features/shared-memory.md` — disclosure audit + privacy validator
- `features/knowledge-base.md` + `features/org-knowledge-base.md` — KB ingestion
- `features/webhooks.md` — audit webhook details + HMAC verify
- `features/custom-tools.md` — Composio + custom tool integration
- `migrations/openai-assistants.md` — common migration source for enterprise bots
