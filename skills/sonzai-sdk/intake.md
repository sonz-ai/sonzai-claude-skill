---
name: sonzai-intake
description: Use when the user is starting a new Sonzai integration (greenfield or existing codebase) and needs a recommendation on archetype, memory mode, capabilities, and implementation order. Use BEFORE writing any non-trivial Sonzai code.
---

# Sonzai integration wizard

Diagnose what the developer is building. Prescribe archetype + memory mode + capabilities. Drive them through `superpowers:brainstorming` (spec) → self-review → `superpowers:writing-plans` (plan).

**Do not skip ahead to writing code.** The wizard's output is what tells you which feature files, decision aids, and archetype playbook to read next. Skipping it produces wrong stacks (wrong memory mode for the latency budget, wrong capabilities, wrong agent topology).

---

## Section 1 — Pre-question inference

Before asking anything, scan the workspace for signals. If a signal is present, **confirm in one line** ("Detected existing TS project — yes?") rather than asking the question cold. Confirmation costs less than the wrong inference.

| Signal | Inferred answer |
|---|---|
| `pyproject.toml` / `requirements.txt` + source files | `language=Python`, `project=existing` |
| `package.json` with `"@sonzai-labs/agents"` in deps | `language=TS`, `project=existing`, Sonzai partially wired |
| `package.json` without Sonzai deps + has source | `language=TS`, `project=existing`, no Sonzai yet |
| `go.mod` + `.go` files | `language=Go`, `project=existing` |
| Workspace empty / README-only | `project=greenfield` |
| User prompt contains "companion / Replika-like / 1:1 chat / personality evolves" | `archetype hint=companion` (confirm) |
| User prompt contains "MBTI / personality router / matchmaker / route to specialist / N specialists" | `archetype hint=guide-router` (confirm) |
| User prompt contains "team / shared / employees / enterprise / Slack bot / Teams bot" | `archetype hint=enterprise-assistant` (confirm) |
| User prompt contains "NPC / character / game / inventory / quests" | `archetype hint=game-npc` (confirm) |
| User prompt contains "customer support / ticket / help desk / Zendesk / Intercom" | `archetype hint=customer-support` (confirm) |
| User prompt contains "coach / therapist / journal / wellness / mental health" | `archetype hint=coach-therapist` (confirm) |
| User prompt contains "Claude Desktop / Cursor / ChatGPT / Claude Code MCP / VS Code MCP" | `install path = MCP` — load `features/mcp-integration.md` ahead of wizard |
| User prompt contains "OpenClaw / openclaw / `@sonzai-labs/openclaw-context`" | `install path = OpenClaw` — load `features/openclaw-integration.md` ahead of wizard |
| `~/.cursor/mcp.json` or `.vscode/mcp.json` or `~/.openclaw/openclaw.json` present | install path inferred per above |

---

## Section 2 — The 7 wizard questions

Ask in order, **skipping any answered by inference**. Each question's options are fixed — present them to the user (use the host platform's structured selector when available, plain prompt otherwise).

### Q1 — Project state

> "Existing codebase or starting fresh?"

- **existing** → Load `existing-codebase-audit.md`, run its 8-step audit, then resume at Q2.
- **greenfield** → Continue to Q2.

### Q2 — Archetype

> "What are you building?"

- **companion** — 1:1 persistent companion (Replika-shaped); personality evolves with each user.
- **guide-router** — Intake guide agent routes user to one of N specialists by personality (MBTI-style).
- **enterprise-assistant** — Team-shared agent serving multiple employees; KB-heavy; audit trail.
- **customer-support** — KB-backed support agent with custom tools and webhook fanout.
- **game-npc** — Game NPC / character with inventory, custom states, dialogue, and events.
- **coach-therapist** — Long-session coach / therapist / journaler.
- **hybrid or other** → Load `archetypes/hybrid-custom.md`.

### Q3 — User identification model

> "Who are your users?"

- **anonymous-at-intake** — User identified later (typical for guide-router, signup funnels).
- **stable-user-id** — Stable identifier from your auth from day one (companion, enterprise).
- **team-shared** — Multiple users share one agent (enterprise default).

### Q4 — Personality behavior

> "How fixed is the agent's personality?"

- **drift on** — Evolves with each user. Default and required for compounding behavior (the SOTOPIA s30 lift, the rapport rebuild). No flag needed.
- **overlays only** — Stable per agent, varies per user via automatic per-user overlays. Default behavior; works alongside drift.
- **brand-locked** — Tight tone control via prompt shaping. There is **no capability flag** that disables drift. To brand-lock: (1) write a directive `personality_prompt` at agent creation, (2) pass a strict `compiled_system_prompt` on every chat that overrides emergent style, (3) review `agents.personality.get_recent_shifts` in production to monitor drift if needed. Server-side drift still runs; you just don't show it.

### Q5 — Proactive features

> "Does the agent reach out unprompted?"

- **none** — Reactive chat only.
- **scheduled-reminders** — Recurring (`client.schedules.create`).
- **backend-events** — React to backend triggers (`agents.triggerBackendEvent`).
- **both** — Both of the above.

### Q6 — Latency budget for first token

> "How fast must first-token feel?"

- **<500ms** (live voice, sub-second chat widgets) → forces `memory_mode=async`, possibly `skip_context_build` per call.
- **500ms–2s** (interactive chat, web/mobile UIs) → `memory_mode=async` is recommended.
- **2s+** (patient flows, batch, server-to-server) → `memory_mode=sync` is fine (default — facts always land same turn).

### Q7 — Integration path

> "How are you integrating?"

- **Python SDK** (`pip install sonzai`) — build/run server-side in Python
- **TypeScript SDK** (`npm install @sonzai-labs/agents`) — build/run server-side in Node / Bun / Deno
- **Go SDK** (`go get github.com/sonz-ai/sonzai-go`) — build/run server-side in Go
- **MCP-only** (Claude Code / Cursor / ChatGPT / Claude Desktop / VS Code) — no SDK code; client config + hosted MCP server. Load `features/mcp-integration.md` for the config pattern. The archetype playbook still applies (companion / guide-router / etc.) — you just create/configure the agent via MCP guided prompts or dashboard instead of SDK code.
- **OpenClaw plugin** (`@sonzai-labs/openclaw-context`) — Sonzai as the `contextEngine` slot. Load `features/openclaw-integration.md`. Same archetypes; agent provisioning via wizard or B2B SDK.

Skip if inferred from workspace.

---

## Section 3 — Archetype-specific follow-ups

After Q2 picks an archetype, ask 2-4 more questions from that archetype's section 4 (defined in each `archetypes/*.md`). Cheat sheet:

| Archetype | Extra questions |
|---|---|
| **companion** | Voice needed? Image generation? Scheduled check-ins (cadence)? Per-user agent or one-agent-serves-all? |
| **guide-router** | Number of specialists? Framework (MBTI=16 / Big5=5 / OCEAN / custom)? Pre-defined or auto-generated specialist personalities? Re-assessment allowed? |
| **enterprise-assistant** | Number of users sharing one agent? KB scope (project_only / org_only / cascade)? Privacy floor categories? Audit destination? |
| **customer-support** | Ticketing system (Zendesk / Intercom / Jira / custom)? Escalation channels (Slack / PagerDuty / email)? KB docs to upload? |
| **game-npc** | Single NPC or cast of N? Inventory schema (typed items)? Player-vs-shared NPC state? Sharding (instances per region)? Events triggered? |
| **coach-therapist** | Session length (typical)? Diary visibility (user-facing vs internal)? Intake assessment shape (PHQ-9, custom)? Reminder cadence? |
| **hybrid-custom** | Strongest signal (1:1 / team / routing / multi-tenant / game)? Which features from other archetypes are needed? Compliance constraints? |

---

## Section 4 — Answer-to-stack mapping

This is a routing table. After collecting answers, run the mapping. The canonical version of the capabilities grid lives in `decisions/capabilities-matrix.md` — consult it for the full per-archetype × per-capability picture.

### Memory mode mapping (Q2 + Q6)

| Archetype (Q2) | Q6 latency | memory_mode |
|---|---|---|
| companion | <500ms or 500ms-2s | async |
| companion | 2s+ | sync |
| guide-router | any | **async on guide**, **sync on specialists** |
| enterprise-assistant | <500ms or 500ms-2s | async |
| enterprise-assistant | 2s+ | sync |
| customer-support | any | sync (compliance / audit) |
| game-npc | any | async |
| coach-therapist | any | sync (every fact matters) |

### Shared memory mapping (Q3)

| Q3 | shared_memory |
|---|---|
| anonymous-at-intake | off |
| stable-user-id | off |
| team-shared | on (requires privacy floor — see `decisions/sharedmemory-vs-wisdom.md`) |

### Personality drift mapping (Q4)

| Q4 | Implementation |
|---|---|
| drift on | leave defaults (drift is automatic, no flag) |
| overlays only | leave defaults (overlays are automatic) |
| brand-locked | strict `personality_prompt` at creation + `compiled_system_prompt` on every chat call. There is no flag to disable server-side drift; you control output via prompt shaping. See `archetypes/customer-support.md` for an example. |

### Proactive mapping (Q5)

| Q5 | Channels needed |
|---|---|
| none | — |
| scheduled-reminders | `client.schedules` + see `decisions/proactive-channel.md` for SSE vs polling vs webhooks |
| backend-events | `agents.triggerBackendEvent` + receiver registration |
| both | combine the above |

### Edge cases

If the combination doesn't match a row above (e.g. enterprise + per-user drift + voice + scheduled), tell the user: *"This combination isn't in the standard playbook. I'll assemble the stack from the feature references — confirm?"* and route to `archetypes/hybrid-custom.md`.

---

## Section 5 — Output handoff

After all answers are collected (Q1-7 + archetype-specific extras), produce a structured recommendation:

```
RECOMMENDATION
==============
Archetype:      <name>
Language:       <python|typescript|go>
Memory mode:    <sync|async|per-agent override>
Shared memory:  <on|off>
Capabilities:
  - <agent role 1>: <flags>
  - <agent role 2>: <flags>
Proactive:      <none|scheduled|events|both>

NEXT
====
1. Load archetypes/<chosen>.md — this is the implementation playbook
2. Playbook drives:
   (a) writing sonzai-implementation-spec.md (use superpowers:brainstorming)
   (b) self-review of the spec
   (c) writing sonzai-implementation-plan.md (use superpowers:writing-plans)
3. Execute the plan via superpowers:subagent-driven-development or
   superpowers:executing-plans.
```

Then load `archetypes/<chosen>.md` and hand control to that playbook.

---

## Section 6 — Skip / escape hatches

The wizard is not always wanted. Recognize and route out:

| User signal | Action |
|---|---|
| "Skip wizard" / "I know what I want" | Fall through to reference mode (see `SKILL.md` skip-wizard table). |
| Q2 answer "none of these" / explicit "hybrid" / "custom" | Load `archetypes/hybrid-custom.md`. |
| Existing Sonzai integration code detected in open file (active SDK calls) | Short-circuit; load `references/troubleshooting.md` if it looks like debugging, else the matching `features/*.md`. |
| User has a syntax-level question only ("how do I call chat?") | Skip the wizard; load the matching `references/{language}.md`. |
| User is mid-migration with a known incumbent | Skip wizard if archetype is already clear; load the matching `migrations/{source}.md`. |

---

## Section 7 — Anti-patterns the wizard must reject

The wizard's job is not to comply with every request — it's to prescribe the right stack. If the user's answer combination violates a rule, **push back with explanation**, do not silently comply.

| User combination | Reject because | Suggested fix |
|---|---|---|
| `memory_mode=sync` + voice on Q-followup | Sync blocks the audio loop; first-token latency goes over budget | Force `memory_mode=async` and remind voice always needs async |
| Companion archetype (Q2) + team-shared users (Q3) | Companion is 1:1; sharing across users is a privacy leak | Suggest enterprise or hybrid; if the user insists, route to hybrid-custom |
| API key in `NEXT_PUBLIC_*` / `VITE_*` / `EXPO_PUBLIC_*` | Browser-exposed secret | Refuse; require server-side proxy |
| guide-router (Q2) + framework=MBTI + N specialists ≠ 16 | MBTI has exactly 16 types; mismatch | Confirm intent; either keep MBTI=16 or switch framework |
| game-npc + drift off (Q4) without justification | NPCs benefit from drift; flat NPCs feel dead | Flag for confirmation; allow override |
| customer-support + `web_search=true` without justification | Hallucination risk on policy questions | Default `web_search=off`; only enable if explicitly needed |
| coach-therapist + `memory_mode=async` | Coaching can't afford to lose a fact mid-conversation | Force `memory_mode=sync` |
| User on Cloudflare Workers / Vercel Edge + SSE streaming | Edge runtime has no long-lived connections | Auto-pick async polling (`chat_async` / `chatAsync`) |
| `shared_memory=on` without privacy floor configured | Regulatory risk; surfaces protected categories | Require privacy floor before enabling |

If the user insists after pushback, document the exception in the spec (`Out of scope`, `Known risks`) and proceed.

---

## How to use this wizard (operationally)

1. Read this file and `SKILL.md`.
2. Run Section 1 inference against the workspace.
3. Confirm inferred answers in one line each; ask only the un-inferred Q1-Q7.
4. After Q2, fetch archetype-specific extra questions from `archetypes/<chosen>.md` section 4 and ask those.
5. Run Section 4 mapping to produce the recommendation.
6. Check Section 7 — does any combination violate an anti-pattern? If yes, push back.
7. Emit the Section 5 output block to the user.
8. Load `archetypes/<chosen>.md` and follow its section 5 (spec template fields) and section 6 (plan template).
9. Drive the user through `superpowers:brainstorming` to write the spec, then `superpowers:writing-plans` to write the implementation plan.

---

## Cross-references

- `SKILL.md` — router that loads this file
- `existing-codebase-audit.md` — for Q1=existing
- `archetypes/*.md` — playbooks loaded after Q2
- `decisions/capabilities-matrix.md` — canonical capability grid
- `decisions/memory-mode.md` — sync/async decision rule
- `decisions/proactive-channel.md` — SSE vs polling vs webhooks
- `decisions/sharedmemory-vs-wisdom.md` — privacy floor rules
