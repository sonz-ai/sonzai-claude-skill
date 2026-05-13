---
name: migration-character-ai
description: Use when migrating Character.AI personas + chat history to Sonzai. Character.AI persona descriptions → generate_and_create; chat history → priming content_blocks.
---

# Migrating from Character.AI

## What Character.AI provides

Character.AI is a consumer chat platform with user-created characters (persona + greeting). Each character has a name, description, greeting, and per-user chat history. Users export their data via account settings.

## Field mapping

| Character.AI concept | Sonzai equivalent |
|---|---|
| Character (the persona) | Sonzai agent (`agents.generation.generate_and_create` from description) |
| Character description | `description` (for generate_and_create) or `personality_prompt` |
| Character greeting | First-turn `compiled_system_prompt` or seed memory (priming text block) |
| User chat history (per character × per user) | `priming.prime_user` content_blocks (type=chat) |
| User account → Sonzai user_id | derive stable UUID from user identifier |
| Definition / "definition" advanced field | `personality_prompt` |

## Migration order

1. **Export user data from Character.AI** (account settings → request data export). Wait for the email; download the archive.
2. **Parse the export.** Typical structure: characters/, chats/, users.json. Each character has metadata + chats per user.
3. **Create Sonzai agents per character:**
   ```python
   for char in character_ai_characters:
       agent = client.agents.generation.generate_and_create(
           name=char.name,
           description=char.description,         # often a paragraph; works great with generate_and_create
           language="en",                         # or detect from chats
       )
       # Map old character_id → new agent_id in your migration table
   ```
4. **Prime each user with their character history:**
   ```python
   for (char_id, user_id), chat in character_ai_chats.items():
       client.priming.prime_user(
           agent_id=migration_table[char_id],
           user_id=user_id,
           metadata={"display_name": user_display_names.get(user_id, "User")},
           content_blocks=[
               {"type": "chat", "content": [
                   {"role": m.role, "content": m.text} for m in chat.messages
               ]},
           ],
       )
   ```
5. **Set capabilities for companion archetype** (Character.AI shape is closest to companion):
   ```python
   client.agents.update_capabilities(
       agent.agent_id,
       memory_mode="async",
       remember_name=True,
   )
   ```
6. **Switch your app's chat handler** to Sonzai. The character now has Big5 personality, memory, mood, and drift — features Character.AI lacks.

## Code shape before → after

**Before (Character.AI usage, no direct API):**
```
# Character.AI doesn't have a direct dev API for chat;
# users interact through their app/web. Your migration starts
# from the exported data.
```

**After (Sonzai chat):**
```python
session = client.agents.sessions.start(agent.agent_id, user_id="u1", session_id="s1")
result = session.turn(messages=[{"role": "user", "content": "Hi Luna!"}])
print(result.response)
```

## Gotchas specific to Character.AI

- **Personas are short text** — Sonzai's `generate_and_create` benefits from 50-200 words; if Character.AI descriptions are very short (under 50 words), the auto-derivation may be generic. Consider hand-augmenting descriptions before migration.
- **Chat history quality** — Character.AI conversations can be very long and have explicit content; review/filter before priming to avoid spurious facts.
- **User identity mapping** — Character.AI usernames → your stable user_id; mint deterministic UUIDs via `uuid5(YOUR_NAMESPACE, character_ai_username)`.
- **Greeting handling** — first-turn greeting in Character.AI is a system-injected message. In Sonzai, you can either pre-pend it to the user's first chat message or embed it in personality_prompt.
- **Multi-language users** — Character.AI users converse in many languages; preserve the language per agent (use `generate_and_create(..., language=...)` per detected language).
- **NSFW / adult content** — Character.AI permits adult content; Sonzai's default models may refuse. Set provider/model accordingly or use Custom LLM with a model that handles it (per your compliance / TOS).
- **No "character_id" continuity** — your migration table maps Character.AI character_id → Sonzai agent_id; record this mapping.

## Verification

- Sample 10 character × user pairs. Prime, then chat. Compare response feel to Character.AI samples.
- Memory recall test: insert facts from old chat; ask about them later; verify recall.

## Cross-references

- `features/generation.md` — `generate_and_create` for personality
- `features/priming.md` — chat history migration
- `archetypes/companion.md` — Character.AI shape → companion archetype
- `migrations/overview.md`
