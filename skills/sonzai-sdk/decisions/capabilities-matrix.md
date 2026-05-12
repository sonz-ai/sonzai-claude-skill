---
name: capabilities-matrix
description: Use when picking which agent capabilities to enable for a given archetype, or when an archetype playbook calls for a specific capabilities config.
---

# Decision: capabilities matrix

The canonical mapping of archetype × capability. Used by `intake.md` and all `archetypes/*.md` playbooks.

## The rule

Capabilities default OFF unless explicitly enabled. Enable only what the archetype needs. `update_capabilities` is PATCH-style — omitted fields are unchanged.

---

## Capability categories

The Sonzai API exposes three categories of capability fields. The wizard must know which category each belongs to:

### A. User-toggleable via `update_capabilities`

These appear in the `UpdateCapabilitiesInputBody` schema and can be flipped at any time via `client.agents.update_capabilities(agent_id, ...)` or at creation via `agents.create(... tool_capabilities=...)`.

| Field (snake_case / camelCase) | What it does |
|---|---|
| `memory_mode` / `memoryMode` | Supplementary memory recall timing. `sync` (default) blocks context build; `async` races a deadline. |
| `web_search` / `webSearch` | Agent can search the web. |
| `image_generation` / `imageGeneration` | Agent can generate images. Requires tier unlock (see B). |
| `inventory` | Inventory subsystem available. |
| `knowledge_base` / `knowledgeBase` | Agent reads from project KB. |
| `knowledge_base_write` / `knowledgeBaseWrite` | Agent writes to KB autonomously. Requires `knowledge_base=true`. |
| `knowledge_base_scope_mode` / `knowledgeBaseScopeMode` | Read scope: `project_only` (default) / `org_only` / `cascade` / `union`. |
| `remember_name` / `rememberName` | Agent retains user names across sessions. |
| `shared_memory` / `sharedMemory` | Person/entity-attributed memory across users of this agent. Requires `wisdom=true`. |
| `wisdom` | K-anonymized cross-user pattern learning. Precondition for `shared_memory`. |
| `skills` | Project-library skill loading (auto-includes skills index + `sonzai_load_skill` tool). |
| `auto_learn_skills` / `autoLearnSkills` | Agent-authored skills via `sonzai_create_skill` / `sonzai_update_skill`. Requires `skills=true`. |
| `composio` | Per-agent Composio SaaS integrations (Gmail, Calendar, Slack, GitHub, Linear, etc.). |
| `mcp_enabled` / `mcpEnabled` | Array of MCP catalog entry IDs the agent uses. |

### B. Tier-gated / unlock-required (read-only via API)

These appear in the `AgentCapabilities` read schema but **NOT** in `UpdateCapabilitiesInputBody`. They're enabled by billing tier or admin action via the dashboard; you cannot toggle them via the standard API.

| Field | Meaning |
|---|---|
| `voiceGeneration` | Voice (TTS/STT/live) available. Paired with `voiceId`, `voiceTier`, `voiceUnlockedAt`. |
| `musicGeneration` + `musicUnlockedAt` | Music generation tier-gate. |
| `videoGeneration` + `videoUnlockedAt` | Video generation tier-gate. |
| `imageUnlockedAt` | When image generation was tier-unlocked (the toggle `imageGeneration` is still required). |

**How to check:** read with `client.agents.get_capabilities(agent_id)` after creation. If `voiceGeneration=true`, the agent has voice; otherwise contact your project admin / upgrade tier.

### C. Server-managed state (read-only)

| Field | Meaning |
|---|---|
| `customTools` | List of registered custom tools (set via `agents.createCustomTool`, not via update_capabilities). |
| `knowledgeBaseProjectId` | Backing project for KB reads. |
| `pendingCapabilities` | Capabilities scheduled to activate (e.g. after billing change). |

---

## Archetype × capability grid

Defaults the wizard should prescribe. Read across the row for an archetype to know which capabilities to enable, leave default, or check tier.

Legend: **on** = set to `true`; **off** = set to `false` (or leave default-false); **opt** = optional, depends on follow-up Q; **tier** = tier-gated, check via `get_capabilities`; **N/A** = capability is not relevant.

| Capability | Companion | Guide (intake) | Specialist | Enterprise | Customer Support | Game NPC | Coach |
|---|---|---|---|---|---|---|---|
| `memory_mode` | async | async | sync | sync (or async if voice) | sync | async | sync |
| `web_search` | off | off | on (typically) | on | off (KB-grounded) | off | off |
| `image_generation` | opt (tier) | off | opt (tier) | off | off | opt (tier) | off |
| `inventory` | off | off | off | off | off | on | off |
| `knowledge_base` | off | off | opt | on | on | opt (lore) | opt |
| `knowledge_base_write` | off | off | opt | on (audited) | opt | off | off |
| `knowledge_base_scope_mode` | project_only | project_only | project_only | cascade | project_only or cascade | project_only | project_only |
| `remember_name` | on | on | on | on | on | on (per player) | on |
| `shared_memory` | off | off | off | on | on | off | off |
| `wisdom` | on (default) | on (default) | on (default) | on | on | on (default) | on (default) |
| `skills` | opt | off | opt | opt | off | opt | off |
| `auto_learn_skills` | off | off | off | off | off | off | off |
| `composio` | off | off | off | opt (Slack/GitHub) | opt (Zendesk/Jira) | off | off |
| `mcp_enabled` | off | off | off | opt | off | off | off |

### Personality drift — not a capability

There is no `personality_drift_disabled` field. Drift runs server-side regardless. To "brand-lock" personality:
- Write a strict `personality_prompt` at agent creation
- Pass a directive `compiled_system_prompt` on every chat call
- Monitor with `agents.personality.get_recent_shifts(agent_id)` and react if drift exceeds tolerance

### Voice (tier-gated)

If an archetype calls for voice (companion w/ voice add-on, customer support phone IVR, coach voice mode):
- Check `client.agents.get_capabilities(agent_id).voiceGeneration` — must be `true`
- If `false`: contact project admin or upgrade billing tier in the dashboard
- `voiceId` selects the voice from `client.voices.list()`
- See `features/voice.md`

---

## How to apply

1. Pick the archetype from `intake.md` Q2.
2. Read the column for that archetype.
3. For guide-router specifically: read **both** the Guide and Specialist columns; call `update_capabilities` once per agent (guide separately from each specialist).
4. Make the API call:

   ```python
   # Python — companion archetype example
   client.agents.update_capabilities(
       agent_id,
       memory_mode="async",
       remember_name=True,
       image_generation=True,         # only if tier-unlocked
       # all other fields omitted = unchanged
   )
   ```

   ```typescript
   // TypeScript — enterprise archetype example
   await client.agents.updateCapabilities(agentId, {
     memoryMode: "sync",
     sharedMemory: true,
     wisdom: true,                    // precondition for sharedMemory
     knowledgeBase: true,
     knowledgeBaseWrite: true,
     knowledgeBaseScopeMode: "cascade",
     webSearch: true,
     rememberName: true,
   });
   ```

5. Verify with a follow-up read:
   ```python
   caps = client.agents.get_capabilities(agent_id)
   assert caps.memory_mode == "async"
   ```

---

## Why these defaults

- **Async memory** anywhere voice is on (TTFC budget) or interactivity is critical
- **Sync memory** anywhere compliance / audit / coaching matters (every fact lands this turn)
- **KB writes always require audit** (regulatory)
- **`shared_memory` requires `wisdom`** as a precondition (server-side rule)
- **`knowledge_base_write` requires `knowledge_base`** as a precondition
- **`auto_learn_skills` requires `skills`** as a precondition

---

## Exceptions

- If your latency budget is generous (>2s TTFC), sync memory is fine even for game NPCs.
- If your compliance regime requires logging every retrieval, sync + audit log even for companions.
- If a user explicitly asks for an off-stack combination (e.g. companion + shared_memory), redirect them to `archetypes/hybrid-custom.md` and document the override in the spec's "Out of scope / Known risks" section.

---

## Cross-references

- `decisions/memory-mode.md` — the sync/async decision in depth
- `decisions/sharedmemory-vs-wisdom.md` — when to enable shared_memory + privacy floor
- `decisions/byok-vs-customllm.md` — provider-side capability concerns (separate from agent capabilities)
- `features/capabilities.md` — the SDK surface (`get_capabilities`, `update_capabilities`) + code examples
- `features/voice.md` — voice tier checks
- `features/shared-memory.md` — privacy floor configuration
- `features/knowledge-base.md` + `features/org-knowledge-base.md` — KB scope mode details
- `archetypes/*.md` — each archetype's prescribed stack (Section 2 of each playbook)
