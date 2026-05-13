# Transcript analysis

Extract structured signals from raw transcript text. Output: `.full-auto/signals.md`.

This is the **diagnostic** step. No decisions get committed here — Phase 3 (`answer-derivation.md`) turns these signals into final answers.

## Strategy

Read the transcript end-to-end first. Then scan for signals in seven categories. For each signal, record the **strongest matching direct quote** (with line number or timestamp if available).

If the same category has multiple matching quotes, take the most specific. If they conflict, list both and mark as ambiguous — Phase 3 will pick a default.

## Signal categories

### 1. Archetype

| Trigger phrase (case-insensitive) | Maps to |
|---|---|
| "companion", "Replika-like", "1:1 chat", "personality evolves", "remembers them" | `companion` |
| "MBTI", "Big Five", "personality test", "matchmaker", "route to specialist", "N agents", "16 agents", "personality router" | `guide-router` |
| "team", "shared", "Slack bot", "Teams bot", "enterprise", "employees", "everyone uses", "internal tool" | `enterprise-assistant` |
| "customer support", "ticket", "Zendesk", "Intercom", "help desk", "support agent" | `customer-support` |
| "NPC", "character", "game", "quests", "inventory", "world", "player" | `game-npc` |
| "coach", "therapist", "journal", "wellness", "mental health", "mood tracker", "self-improvement" | `coach-therapist` |
| Multi-match OR no clean match | `hybrid-custom` |

### 2. Integration path

| Trigger | Maps to |
|---|---|
| "Python", "FastAPI", "Django", "Flask", "pip" | `python-sdk` |
| "TypeScript", "Node", "Next.js", "Bun", "Deno", "Elysia", "Hono", "Express", "npm" | `typescript-sdk` |
| "Go", "golang", "Echo", "Gin", "Chi", "go mod" | `go-sdk` |
| "Claude Desktop", "Cursor", "ChatGPT", "Claude Code MCP", "VS Code MCP", "MCP server" | `mcp` |
| "OpenClaw", "openclaw" | `openclaw` |
| No signal | default → `typescript-sdk` (broadest reach) |

### 3. Latency budget

| Trigger | Maps to |
|---|---|
| "real-time", "voice", "sub-second", "live", "<1s", "instant" | `<500ms` → forces async memory |
| "interactive", "fast chat", "snappy", "<2s", "feels fast" | `500ms-2s` → async recommended |
| "batch", "async", "weekly", "nightly", "no rush", "non-interactive" | `2s+` → sync ok |
| No signal | default → `500ms-2s` |

### 4. Capabilities (Sonzai SDK feature flags)

For each, look for explicit mention. If silent, leave OFF — capabilities are paid surfaces; never enable speculatively.

| Trigger | Maps to UpdateCapabilitiesInputBody flag |
|---|---|
| "image generation", "generate art", "portrait", "avatar generation" | `imageGeneration: true` |
| "voice", "TTS", "speech", "speak", "audio replies" | `voiceGeneration: true` (note: tier-gated, see decisions/capabilities-matrix.md) |
| "knowledge base", "KB", "docs", "RAG", "search documents", "upload docs" | `knowledgeBase: true` |
| "web search", "browse", "look up online", "search the web" | `webSearch: true` |
| "shared memory", "team memory", "everyone sees", "common context" | `sharedMemory: true` + `wisdom: true` (precondition) |
| "long-term memory", "facts", "preferences", "remembers across sessions" | `wisdom: true` |
| "MCP", "external tools via MCP" | `mcpEnabled: true` |
| "skills", "custom skills", "agent learns" | `skills: true` |
| "auto-learn", "improves over time", "self-improving" | `autoLearnSkills: true` |
| "Composio", "integrations", "Composio tools" | `composio: true` |
| "remember name", "calls me by name" | `rememberName: true` |

Tools (custom function calling) is NOT a capability flag — it's a feature. Look for "function calling", "tools", "actions", "execute code" and note it.

### 5. Brand / persona constraints

| Trigger | Maps to |
|---|---|
| "stay on brand", "strict tone", "no off-script", "exact wording", "brand voice" | Q4 = `brand-locked` (prompt-shaping approach) |
| "human-like", "authentic", "unique per user", "personality grows", "no robot speak" | Q4 = `drift on` |
| Persona description ("our character is..." / "she's a..." / "his name is...") | Capture persona text for spec; doesn't change Q4 |

### 6. Proactive features

| Trigger | Maps to Q5 |
|---|---|
| "daily check-in", "reminder", "schedule", "every morning", "cadence" | `scheduled-reminders` |
| "webhook", "trigger from backend", "react to events", "when X happens" | `backend-events` |
| Both above | `both` |
| Neither | `none` |

### 7. Scope / deadlines

| Trigger | Capture as constraint |
|---|---|
| "MVP", "prototype", "demo", "pilot", "proof of concept" | scope = minimal (only essential capabilities) |
| "production", "enterprise", "SOC2", "compliance", "audit" | scope = production (enable audit, privacy floor) |
| Explicit date ("by Friday", "next week", "Q3") | deadline = capture verbatim (don't enforce, just record) |
| "single user demo", "just me to start" | scale = single-user (don't over-provision multi-tenancy) |
| "thousands of users", "scale" | scale = production |

## Output format

Write to `.full-auto/signals.md`:

```markdown
# Transcript signals

**Transcript**: `.full-auto/transcript.txt` ({{N_LINES}} lines)
**Analyzed**: {{ISO_TIMESTAMP}}

## Direct-quote evidence

### Archetype
- "we want each of our users to chat with their own AI companion that gets to know them" (line 14) → **companion**

### Integration path
- "our backend is Go and we're running on GCP" (line 22) → **go-sdk**

### Latency
- "this needs to feel instant — it's voice-first" (line 31) → **<500ms** → forces async memory

### Capabilities
- "we'll need voice replies" (line 33) → **voiceGeneration** (note: tier-gated, may need uplift)
- "the agent should remember each user's preferences across sessions" (line 41) → **wisdom**
- "no image gen, no web search for v1" (line 48) → explicitly OFF

### Brand / persona
- "she's named Aria, she's curious and warm" (line 12) → persona captured for spec; Q4 = drift on (default)

### Proactive
- "daily morning check-ins" (line 51) → **scheduled-reminders**

### Scope
- "MVP for 50 beta users by Q3 2026" (line 8) → scope=minimal, scale=small, deadline=Q3 2026

## Inferred answers (preview — Phase 3 will finalize)
- Q1: greenfield (no existing repo mentioned)
- Q2: companion
- Q3: stable-user-id (companion default)
- Q4: drift on
- Q5: scheduled-reminders
- Q6: <500ms
- Q7: go-sdk

## Ambiguities (Phase 3 will pick safe defaults)
- Number of agents per user: silent → assume 1 (per-user agent, since drift+companion)
- KB scope: silent → assume project_only (lowest-friction default)
- Voice provider: silent → ElevenLabs (platform default)
- Reminder cadence specifics: "daily morning" — assume 09:00 user-local
- Auth provider: silent → assume external (caller passes stable user_id)

## Out-of-scope mentions (record but skip)
- "payment integration" (line 62) — out of Sonzai SDK scope; flag for operator follow-up
```

## Halt conditions

If after analysis the transcript contains NO Sonzai-buildable signal — i.e., the meeting was about something else entirely (trading bot, payment processor, video editor, data warehouse) — write `.full-auto/BLOCKED.md`:

```markdown
# BLOCKED at Phase 2

The transcript does not describe a Sonzai SDK-shaped product.

## Strongest signal found
"{{quote that suggested out-of-scope work}}"

## Why this is out of scope
The Sonzai SDK provides: agent personality, memory, mood, generation (image/video/audio), inventory, custom tools, capabilities, MCP integration.

It does NOT provide: payment processing, trading, video editing pipelines, low-level infra primitives.

## What the operator should do
Re-check the transcript. If a Sonzai surface was discussed but I missed it, re-run with a more specific subset of the transcript.
```
