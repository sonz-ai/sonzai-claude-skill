---
name: archetype-hybrid-custom
description: Use when no single archetype fits — the app combines multiple archetypes (companion + enterprise, game-npc + customer-support, multi-tenant SaaS spanning archetypes per tenant). Walks the developer through assembling a custom stack from features and decisions.
---

# Hybrid / custom archetype

When none of the 6 named archetypes fits cleanly. Common cases:

- B2B SaaS where each customer-tenant gets a different archetype
- Companion app that also handles team interactions in a workspace mode
- Game NPC that also acts as customer support for in-game purchases
- Onboarding funnel with a guide-router phase, then a long-term companion phase

This playbook is a **recipe assembly guide**, not a fixed stack. Read it alongside the named archetypes you're combining.

## 1. When this archetype fits

**Strong signals:**
- "It's like a companion + enterprise" / "MBTI router followed by long-term companion"
- Multi-tenant SaaS where archetype varies per customer
- Multi-phase user journey with distinct archetype shapes
- Existing archetype's anti-pattern table flags your combination

**Anti-signal:** if you can describe your app in one of the 6 named archetypes without forcing it, use that archetype instead. Hybrid is for genuinely combinatorial cases.

## 2. How to assemble your stack

Pick the **strongest signal** first; layer in features from other archetypes as needed.

### Step 1 — Identify the dominant pattern

| Your situation | Start from this archetype |
|---|---|
| Mostly 1:1 + some team features | `archetypes/companion.md`, add selected `archetypes/enterprise-assistant.md` capabilities |
| Mostly team + occasional individual | `archetypes/enterprise-assistant.md`, soften some restrictions |
| Routing required at start | `archetypes/guide-router.md`, extend specialists with companion or game-npc traits |
| Multi-tenant SaaS where archetype varies per tenant | Separate Sonzai project per tenant; each project uses one named archetype; see `decisions/instances-vs-multitenant.md` |
| Game with social / market NPCs | `archetypes/game-npc.md` plus selected dialogue / customer-support features |
| Onboarding flow → companion / coach | `archetypes/guide-router.md` for intake; transition to `archetypes/companion.md` or `archetypes/coach-therapist.md` for ongoing |
| Multi-phase journey | Compose two named archetypes; track the phase in `custom_states` |

### Step 2 — Borrow capabilities by phase or user-type

Walk through each named archetype's Section 2 (Prescribed stack) and pull rows that apply to your use case. Document them in your spec:

```markdown
## Capability matrix (custom)

| Phase / User type | Source archetype | Capabilities |
|---|---|---|
| Onboarding (guide phase) | guide-router | memory_mode=async, web_search=off, drift consistent via prompt |
| Ongoing (after routing) | companion | memory_mode=async, drift on, optional voice, optional image |
| Power-user mode | enterprise-assistant | upgraded to sync memory + KB access |
```

### Step 3 — Resolve conflicts

When two source archetypes disagree on a capability:

| Conflict | Resolution rule |
|---|---|
| Voice on + sync memory | Voice wins → async memory always when voice is in use |
| Shared memory on + per-user privacy concerns | If privacy floor can't isolate, fall back to wisdom-only (default-on) and disable shared_memory |
| Drift on + brand-locked compliance | Brand-lock via prompt shaping (no flag exists); accept that server-side drift still runs |
| Custom_states vs inventory for the same data type | Schema-validated items → inventory; primitive flags → custom_states (see `decisions/state-vs-inventory.md`) |
| Sync memory + game-NPC pattern | If you need both, raise the latency budget; otherwise concede async and accept fact-spill |

Document each resolved conflict in your spec's "Trade-offs" section so future maintainers know why.

## 3. Required SDK functions

There is no fixed list. Reference the relevant `features/*.md` for each surface you use:

| You need | See feature |
|---|---|
| Agent creation | `features/generation.md` |
| Capabilities | `features/capabilities.md` |
| Memory | `features/multiplayer-memory.md` (if shared) or each archetype's section 3 |
| Inventory | `features/inventory.md` |
| Custom states | `features/custom-states.md` |
| Custom tools | `features/custom-tools.md` |
| Knowledge base | `features/knowledge-base.md`, `features/org-knowledge-base.md` |
| Proactive | `features/proactive.md` |
| Webhooks | `features/webhooks.md` |
| Voice | `features/voice.md` |
| Events / dialogue | `features/events-and-dialogue.md` |
| Eval | `features/eval-and-simulation.md` |
| Instances | `features/instances.md` |
| Models / BYOK / Custom LLM | `features/models.md` |
| Priming / personas | `features/priming.md`, `features/personas.md` |

## 4. Archetype-specific wizard intake questions

After Q2 picks `hybrid-or-other`, the wizard asks:

1. **What's the strongest signal?** 1:1 companion / team-shared / personality routing / multi-tenant / game / multi-phase / other (describe).
2. **Which other named archetypes are you borrowing from?** None / specific named ones.
3. **Phases over time?** Single phase / multi-phase (describe transitions).
4. **Compliance regime?** None / SOC2 / HIPAA-adjacent / GDPR-strict / other. Drives sync-vs-async and audit choices.
5. **Multi-tenant?** No / yes — if yes, confirm separate-project approach (`decisions/instances-vs-multitenant.md`).
6. **Latency budget?** Same Q6 as standard wizard.
7. **Voice?** Off / yes (tier-gated).

## 5. Spec template fields

```markdown
## Pattern summary

- Strongest signal: <description>
- Source archetypes (in priority order): <list>
- Phases (if multi-phase): <list with capability deltas per phase>
- Multi-tenant?: <no | yes — N tenants, separate projects per tenant>

## Per-phase / per-tenant agents

| Phase / tenant | Archetype source | Agent count | Capabilities |
|---|---|---|---|
| ... | ... | ... | ... |

## Transitions (if multi-phase)

- Phase A → Phase B trigger: <e.g. custom_states["phase"]="post-onboarding">
- Routing logic: <where in your code>

## Trade-off resolutions

Document each conflict from Step 3 above with the resolution.

## Out of scope (named archetypes' features you're NOT using)

- [e.g. "not using shared_memory because we're 1:1"]
```

## 6. Plan template

Plan structure depends on which archetypes you're combining. Generally:

```markdown
1. Per-phase / per-tenant: set up agents per the source archetype's section 3. Verify each independently.
2. Implement phase-transition logic (or tenant routing). Verify the right agent gets the right user.
3. Reconcile conflicting capability defaults; explicitly call update_capabilities with the resolved values. Verify get_capabilities.
4. Integration test: full user journey across phases (or full multi-tenant request flow). Verify state and routing.
5. Per-archetype integration tests (companion-side, enterprise-side, etc.).
6. Compliance review for the combined stack.
```

## 7. Anti-patterns

| Anti-pattern | Why it's wrong | Fix |
|---|---|---|
| Enabling everything ("kitchen sink agent") | Slow, expensive, behaviorally inconsistent — features fight each other | Pick a primary archetype; layer only what's needed |
| Conflating instances with multi-tenancy | Instances share personality + memory globally; only custom_states isolate. Real tenants need full data isolation. | Separate Sonzai project per tenant; see `decisions/instances-vs-multitenant.md` |
| `shared_memory=on` AND `personality_drift` strict via prompt shaping (lose both compounding *and* attribution benefits) | Investing in shared_memory tooling for an agent whose voice is fixed — you've paid for cross-user attribution but disabled the evolution that gives it value | Pick one: either shared+evolving (enterprise default) or 1:1+fixed (custom support default) |
| Using a guide-router pattern for tenants instead of for personality types | Confuses two different problems (multi-tenant routing vs intra-tenant intake) | Tenant routing is at the project layer; guide-router is intra-project |
| One agent handling all phases without `custom_states["phase"]` tracking | Agent doesn't know what phase the user is in; behavior drifts unpredictably | Track phase explicitly in `custom_states`; condition behavior on it |
| Mixing sync (for one phase) and async (for another) on the SAME agent | Capability is agent-wide, not per-phase | If you need both, use two agents (handoff via `custom_states["phase"]`) |
| Choosing hybrid when one named archetype would fit | Hybrid is more spec/code per feature; named archetypes have validated plans | Audit your situation against each named archetype's section 1 anti-signals before declaring hybrid |

## Cross-references

- All 6 named archetypes (`archetypes/companion.md`, `guide-router.md`, `enterprise-assistant.md`, `customer-support.md`, `game-npc.md`, `coach-therapist.md`)
- `decisions/instances-vs-multitenant.md` — the multi-tenant question
- `decisions/capabilities-matrix.md` — the canonical capability grid (column per archetype)
- `decisions/memory-mode.md` — sync vs async resolution rules
- `decisions/sessions-vs-conversations.md` — when to use chat vs sessions vs dialogue
- `intake.md` — the wizard that loaded this archetype after Q2="hybrid-or-other"
