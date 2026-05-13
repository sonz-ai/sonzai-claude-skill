---
name: feature-capabilities
description: Use when reading or updating an agent's capabilities (memory_mode, web_search, knowledge_base, shared_memory, etc.). Foundational — every archetype touches this.
---

# Capabilities

## What it is

Agent-level feature flags. Enable/disable retrieval mode, KB access, web search, image generation, shared memory, custom integrations. Set at creation time (`tool_capabilities` on `agents.create`) or updated anytime via `update_capabilities` (PATCH-style).

## When to use

- After creating an agent — set its capability defaults per the archetype playbook
- Mid-life — flip async ↔ sync, enable a new feature, configure cascade KB scope
- Auditing — read current state via `get_capabilities`
- Tier-gated features — read-only check (voice, image, music, video unlocks)

## When NOT to use

- Per-call overrides (none exist — capabilities are agent-wide). For per-call control, use chat options (`skip_context_build`, `provider`, `model`).
- Setting personality drift (no flag exists — see `archetypes/customer-support.md` for brand-locked pattern via prompt shaping)

## SDK surface

```python
# Read
caps = client.agents.get_capabilities(agent_id)

# Update (PATCH — omitted fields unchanged)
client.agents.update_capabilities(
    agent_id,
    memory_mode="async",            # "sync" | "async"
    web_search=True,
    image_generation=True,           # tier-gated; check image_unlocked_at after
    inventory=True,
    knowledge_base=True,
    knowledge_base_write=True,
    knowledge_base_scope_mode="cascade",  # project_only | org_only | cascade | union
    remember_name=True,
    shared_memory=True,              # requires wisdom=True
    wisdom=True,
    skills=True,
    auto_learn_skills=True,          # requires skills=True
    composio=True,
    mcp_enabled=["mcp-id-1", "mcp-id-2"],
)
```

```typescript
const caps = await client.agents.getCapabilities(agentId);

await client.agents.updateCapabilities(agentId, {
  memoryMode: "async",
  webSearch: true,
  imageGeneration: true,
  knowledgeBase: true,
  knowledgeBaseWrite: true,
  knowledgeBaseScopeMode: "cascade",
  rememberName: true,
  sharedMemory: true,
  wisdom: true,
});
```

```go
caps, err := client.Agents.GetCapabilities(ctx, agentID)

err = client.Agents.UpdateCapabilities(ctx, agentID, sonzai.UpdateCapabilitiesOptions{
    MemoryMode:    sonzai.Ptr("async"),
    WebSearch:     sonzai.Ptr(true),
    KnowledgeBase: sonzai.Ptr(true),
    SharedMemory:  sonzai.Ptr(true),
    Wisdom:        sonzai.Ptr(true),
})
```

Go uses `sonzai.Ptr(...)` to distinguish "not set" from zero value — PATCH-style fields are `*bool` / `*string`.

## Capability reference

Read-only (in `AgentCapabilities`, NOT in `UpdateCapabilitiesInputBody`):
- `customTools` — list of registered custom tools
- `voiceGeneration`, `voiceId`, `voiceTier`, `voiceUnlockedAt` — voice tier
- `imageUnlockedAt`, `musicGeneration`, `musicUnlockedAt`, `videoGeneration`, `videoUnlockedAt` — media tiers
- `knowledgeBaseProjectId` — backing project
- `pendingCapabilities` — scheduled activations

Toggleable (in `UpdateCapabilitiesInputBody`):
See `decisions/capabilities-matrix.md` for the full grid + per-archetype defaults.

## Preconditions

| Capability | Requires |
|---|---|
| `shared_memory=true` | `wisdom=true` |
| `knowledge_base_write=true` | `knowledge_base=true` |
| `auto_learn_skills=true` | `skills=true` |

The server-side validator rejects mismatches.

## Decisions linked

- `decisions/capabilities-matrix.md` — per-archetype grid + which capabilities to enable
- `decisions/memory-mode.md` — `memory_mode` deep dive
- `decisions/sharedmemory-vs-wisdom.md` — `shared_memory` + `wisdom` preconditions
- `decisions/byok-vs-customllm.md` — separate provider-routing config

## Common gotchas

- **PATCH-style** — omitted fields are unchanged, **not reset to false**. To explicitly disable, pass `false`.
- **Tier-gated capabilities** are read-only via API. Setting `image_generation=true` on a tier without `imageUnlockedAt` still leaves the tier ungated — you'll see the flag set but `imageUnlockedAt` remains null.
- **`voiceGeneration` and tier-gated media** can only be enabled via tier upgrade / dashboard / admin action. Not via `update_capabilities`.
- **Capability changes apply to subsequent chats** — in-flight sessions complete with the prior config.
- **No `personality_drift_disabled` flag exists.** Drift runs server-side regardless. Control output via `personality_prompt` + `compiled_system_prompt`.
- **Composio integration** needs admin-connected apps in your dashboard. Setting `composio=true` doesn't connect apps — it just makes their tools available once connected.
