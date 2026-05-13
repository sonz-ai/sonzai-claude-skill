# Transcript analysis (Phase 0a)

Phase 0a of the autonomous transcript-driven pipeline. Output feeds Phase 0d (`answer-derivation.md`) and informs Phase 0b (`project-type-detection.md`).

This file is the canonical analysis logic for both `full-auto` and `cto-loop` — extraction is identical in both modes; only later phases differ in whether they ask the operator.

## Input

One of:
- A pasted transcript block in the operator's chat message
- A path to a transcript file (`.txt`, `.md`, meeting notes, JIRA-formatted requirements, an email thread)
- A loose paragraph of "build me a thing that does X"

## Extraction targets

Read the transcript once. Produce a structured summary covering:

| Field | What to extract | Example |
|---|---|---|
| **Client goal** | The 1-sentence "why this exists" | "A companion app for hospice patients to talk to" |
| **Product shape** | What kind of artifact is being asked for | "Web SaaS with chat UI", "Telegram bot", "Mobile app backend", "Slack app" |
| **Archetype hint** | Closest match to a `sonzai-sdk` archetype (see list below) | `companion`, `guide-router`, `enterprise-assistant`, `customer-support`, `game-npc`, `coach-therapist`, `hybrid-custom` |
| **Scale hint** | Expected user count / message volume / latency requirement | "100 users in beta", "low latency for voice", "throughput not critical" |
| **Capabilities mentioned** | Specific sonzai SDK features called out | memory, personality, mood, diary, KB, custom tools, webhooks, proactive |
| **Auth requirement** | Whether the vertical needs its own user auth | "users log in with Google", "no auth, anonymous" |
| **Frontend/backend split** | Explicit mentions of language / framework | "Python backend, React frontend", "TypeScript everything", "no mention" |
| **Database hint** | Whether business state needs storage | "store conversations" (yes), "stateless webhook" (no), "no mention" (assume yes) |
| **Constraints** | Deadlines, integrations, compliance, on-prem, BYOK | "GDPR", "must work on customer's own AWS", "by Friday" |
| **Known stakeholders** | "the team", "Alice", "the CEO" — generic, no real names | |
| **Open questions** | Things the transcript leaves ambiguous that the operator may need to clarify | |

## Output

Write a brief synthesis (~10 lines) to in-memory state. Do NOT write to a file yet — that happens at masterplan-assembly. Print the synthesis in chat:

```
Transcript analysis:
  Client goal:       <one line>
  Product shape:     <one line>
  Archetype hint:    <one of 7>
  Scale hint:        <one line>
  Capabilities:      <comma-separated>
  Auth requirement:  <one line>
  Stack mentioned:   <one line | none>
  Database hint:     <yes | no | unclear>
  Constraints:       <one line>
  Open questions:    <list, may be empty>
```

## Archetype mapping cheatsheet

(Mirrors `sonzai-sdk/archetypes/`.)

- **companion** — 1:1 persistent relationship, mood + memory matter, often Replika-shaped
- **guide-router** — MBTI / personality-routed intake guide that fans out to specialists
- **enterprise-assistant** — team-shared, KB-heavy, audited, RBAC
- **customer-support** — KB + custom tools + webhooks, ticket-shaped
- **game-npc** — inventory, custom state, dialogue trees, events
- **coach-therapist** — long sessions, sync memory, diary, mood
- **hybrid-custom** — combinations / multi-tenant / "none of the above"

If the transcript is genuinely ambiguous between two archetypes, note both in the synthesis. The Phase 0d answer-derivation will pick one with reasoning.

## Hard rules

1. **No tenant names.** If the transcript mentions a specific company / customer / brand name (e.g., the client's own name, a Sonzai customer's name, or a competitor's name), redact it. Replace with "the client" or "this project". The skill's outputs may end up in public-facing places (commits, masterplan doc, final report).
2. **No PII in synthesis.** If the transcript contains personal info (names, emails, phone numbers), strip them. The masterplan is a build artifact that may be committed.
3. **Don't hallucinate.** If the transcript doesn't mention a field, write `none` or `unclear` — never invent a value.
