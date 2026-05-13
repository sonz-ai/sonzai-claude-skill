---
name: migration-openai-assistants
description: Use when migrating from the OpenAI Assistants API to Sonzai. Assistants → agents; threads → sessions; instructions → personality_prompt; file_search → KB upload; function calling → custom tools.
---

# Migrating from OpenAI Assistants API

## What the OpenAI Assistants API provides

OpenAI's Assistants v1/v2 API: stateful threads with persistent messages, attached assistants with system instructions, tools (function_calling, file_search, code_interpreter), and runs that manage execution.

## Field mapping

| Assistants concept | Sonzai equivalent |
|---|---|
| Assistant | Sonzai agent (`agents.create` or `agents.generation.generate_and_create`) |
| Assistant `instructions` | `personality_prompt` |
| Thread | Sonzai session (`agents.sessions.start` + `session.turn`) |
| Thread messages | Automatic via `session.turn` |
| `file_search` tool | `client.knowledge` (project KB) |
| Custom function tools | `agents.create_custom_tool` |
| Run / step / tool_calls | `session.turn` returns `side_effects.external_tool_calls` |
| `assistants.create(model=...)` | `agents.chat(..., provider="openai", model="...")` per call |
| `thread.metadata` | `priming.metadata` (per user) or `custom_states` |

## Migration order

1. **Replicate the assistant as a Sonzai agent** with matching instructions:
   ```python
   agent = client.agents.create(
       name=openai_assistant.name,
       personality_prompt=openai_assistant.instructions,
   )
   ```
2. **Set capabilities** to match Assistants' tools:
   ```python
   client.agents.update_capabilities(
       agent.agent_id,
       knowledge_base=True if has_file_search else False,
       # web_search if you'd previously used a custom web tool
   )
   ```
3. **Upload Assistants' files to Sonzai KB** if `file_search` is used:
   ```python
   for file in assistants_files:
       client.knowledge.upload_document(project_id, file=file_bytes, filename=file.name)
   ```
4. **Re-register function tools** as Sonzai custom tools:
   ```python
   for fn in openai_assistant.tools:
       if fn.type == "function":
           client.agents.create_custom_tool(
               agent.agent_id,
               name=fn.function.name,
               description=fn.function.description,
               parameters=fn.function.parameters,
           )
   ```
5. **Migrate threads to sessions.** Map thread_id → session_id. Existing thread messages → priming content_blocks (or replay through Sonzai for fact extraction).
6. **Switch run loops** from OpenAI's `runs.create + runs.retrieve` polling to Sonzai's `session.turn` (returns result directly).

## Code shape before → after

**Before (OpenAI Assistants v2):**
```python
from openai import OpenAI
oai = OpenAI()

assistant = oai.beta.assistants.create(
    name="Helper",
    instructions="You are a coding helper",
    model="gpt-5",
    tools=[{"type": "function", "function": {...}}],
)
thread = oai.beta.threads.create()
oai.beta.threads.messages.create(thread.id, role="user", content="Hello")
run = oai.beta.threads.runs.create(thread.id, assistant_id=assistant.id)
# poll run.status until completed
# tool_calls in run.required_action.submit_tool_outputs
```

**After (Sonzai):**
```python
from sonzai import Sonzai
client = Sonzai()

agent = client.agents.create(name="Helper", personality_prompt="You are a coding helper")
client.agents.create_custom_tool(agent.agent_id, name="...", description="...", parameters={...})

session = client.agents.sessions.start(agent.agent_id, user_id="u1", session_id="s1")
result = session.turn(messages=[{"role": "user", "content": "Hello"}])

# Tool calls land in result.side_effects.external_tool_calls — dispatch synchronously
```

## Gotchas specific to OpenAI Assistants

- **Assistants don't have personality drift.** Sonzai's compounding is new behavior — your existing app will improve over time once on Sonzai. Mention this in changelog/release notes if users care.
- **Thread → session mapping** — both are conceptually conversation buckets. Use `thread.id` as `session.id` for traceability (no automatic mapping; you set it).
- **`code_interpreter` tool** — not directly supported by Sonzai. If you need code execution, build your own custom tool that runs code in your backend (with sandboxing).
- **`file_search`** → upload files to `client.knowledge`. Search happens automatically during chat when `knowledge_base=true`.
- **Polling `runs.retrieve`** is gone. Sonzai's `session.turn` returns directly. For long chats use `chat_async_blocking` (see `references/streaming-chat.md`).
- **Vector store IDs** — OpenAI's vector store IDs don't map. Re-upload files to Sonzai KB; vector indexing happens server-side.
- **Tool output submission** — OpenAI requires `submit_tool_outputs`; Sonzai just feeds tool results in the next `session.turn` via a `tool` role message.

## Verification

- Sample 20 threads. Re-run through Sonzai with the same input messages; compare assistant responses.
- File search parity: same query against KB should return comparable results.

## Cross-references

- `features/custom-tools.md` — function migration
- `features/knowledge-base.md` — file_search migration
- `features/priming.md` — thread history migration
- `archetypes/customer-support.md`, `archetypes/enterprise-assistant.md` — common destinations
- `migrations/overview.md`
