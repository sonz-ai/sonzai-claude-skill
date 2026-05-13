---
name: feature-shared-memory
description: Use when enabling cross-user attributed memory on a multi-user agent (enterprise assistant, customer support). Requires wisdom precondition and privacy floor. Disclosure-audited.
---

# Shared memory

## What it is

When `shared_memory=true`, the agent can store and surface **attributed cross-user facts** ("Alice owns the auth domain"). Agent gains tools: `sonzai_wisdom_set`, `sonzai_wisdom_update`, `sonzai_wisdom_delete`, `sonzai_wisdom_relate`. Server-side **privacy floor** validates every surfaced fact; **disclosure audit** logs every cross-user surfacing.

Distinct from `wisdom` (default-on, k-anonymized, unattributed patterns). See `decisions/sharedmemory-vs-wisdom.md`.

## When to use

- Enterprise assistant where the team learns from each other
- Customer support where the CS team builds institutional memory
- Team-coordination bots (project tracker, on-call rotation)

## When NOT to use

- 1:1 companion → off (privacy bug)
- Multi-tenant SaaS where tenants must be isolated → use separate projects, never shared_memory across tenants
- Coach/therapist → off (per-user privacy)
- Any flow where users have an expectation that what they say stays private to them

## SDK surface

```python
# Enable on the agent
client.agents.update_capabilities(
    agent_id,
    shared_memory=True,
    wisdom=True,                          # required precondition
)

# Configure privacy floor — exact API varies by your SDK; may be project-config
# Common categories to block: compensation, health, performance_review, PII
# Verify mechanism in your tier (dashboard or per-call option)
```

Once enabled, the agent uses built-in `sonzai_wisdom_*` tools automatically during chat. You don't directly call these — they're surfaced as tool invocations the agent makes when it learns or surfaces attributed facts.

### Reading the disclosure audit

```python
# Verify exact API path in your SDK
audit = client.agents.get_disclosure_audit(
    agent_id,
    user_id="user-123",
    limit=100,
)
for entry in audit.entries:
    print(entry.timestamp, entry.fact, entry.attributed_to, entry.surfaced_to)
```

The audit shows: every cross-user fact surfaced, attribution, surfaced-to-whom, why.

## Privacy floor

Server-side validator that **refuses to surface facts in protected categories** even if the agent would otherwise reveal them. Common floor categories:

- **compensation** — salaries, bonuses, equity
- **health** — medical, mental health, disabilities
- **performance_review** — internal performance feedback
- **PII** — SSN, addresses, personal identifiers beyond display name

The floor is server-enforced — you don't have to filter in your handler. But you do have to **configure** it; the default categories may not match your regulatory needs.

## Decisions linked

- `decisions/sharedmemory-vs-wisdom.md` — wisdom vs shared_memory
- `archetypes/enterprise-assistant.md`, `archetypes/customer-support.md` — primary users
- `features/capabilities.md` — the toggle flag
- `features/multiplayer-memory.md` — broader cross-user / cross-agent model

## Common gotchas

- **`wisdom=true` is required** — server-side rejects `shared_memory=true` without it.
- **Privacy floor MUST be configured** before turning on shared_memory in production. Default categories may be too permissive for your regime.
- **Disclosure audit grows fast** — implement retention; review weekly.
- **Per-user privacy expectations** — even with shared_memory on, *some* facts are still per-user by default (the agent decides what to share via the wisdom tools). The floor is a safety net, not the only line.
- **Don't share across distinct tenants** — separate projects per tenant for B2B isolation.
- **Wisdom (k-anonymized) is unaffected** — it stays on regardless; shared_memory adds attribution on top.
- **Tool reservation** — `sonzai_wisdom_*` are platform-managed; you can't override them with custom tools.
