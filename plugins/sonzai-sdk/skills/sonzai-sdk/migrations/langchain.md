---
name: migration-langchain
description: Use when migrating from LangChain (ConversationBufferMemory, VectorStoreRetrieverMemory, chains) to Sonzai. LangChain's memory classes map to Sonzai sessions; retrievers map to client.knowledge.search.
---

# Migrating from LangChain

## What LangChain provides

LangChain is a Python/TS framework for building LLM apps. Memory classes wrap conversation buffers and vector-store retrievers. Chains compose prompts + LLMs + memory.

## Field mapping

| LangChain concept | Sonzai equivalent |
|---|---|
| `ConversationBufferMemory` | Sonzai sessions (`agents.sessions.start` + `session.turn`) — automatic conversation history |
| `ConversationSummaryMemory` | Sonzai sessions + automatic summarization in agent insights |
| `VectorStoreRetrieverMemory` | `client.knowledge.search(project_id, query, ...)` |
| Custom retriever | `client.knowledge.search` or move to `agents.memory.search` |
| `ConversationChain` | `agents.chat` or `agents.sessions.start + session.turn` |
| `LLMChain` with system prompt | Sonzai agent with `personality_prompt` + chat |
| LangChain tools | Sonzai custom tools (`agents.create_custom_tool`) |
| AgentExecutor with tool routing | Sonzai's built-in tool-calling via `agents.chat` |
| `ChatPromptTemplate` | `compiled_system_prompt` per chat call |
| `RunnableLambda` | Move to your own server-side logic (Sonzai doesn't replace orchestration) |

## Migration order (strangler)

1. **Identify which LangChain memory class is in use.** Different ones map differently.
2. **Replace ConversationBufferMemory** with `agents.sessions.start` + `session.turn`. Sessions automatically track history.
3. **Replace VectorStoreRetrieverMemory** with `client.knowledge.search` (project KB). Upload your existing vector store content to KB.
4. **Replace ConversationChain / LLMChain** with `agents.chat`. The system prompt becomes `personality_prompt` (at create) + `compiled_system_prompt` (per call).
5. **Move tools** — LangChain `Tool` definitions translate to `create_custom_tool` parameters. Tool-call handling moves from AgentExecutor to your handler reading `side_effects.external_tool_calls`.
6. **Decommission chain wrappers** — Sonzai's chat/session API is enough.

## Code shape before → after

**Before (LangChain Python):**
```python
from langchain.memory import ConversationBufferMemory
from langchain.chains import ConversationChain
from langchain.llms import OpenAI

memory = ConversationBufferMemory()
chain = ConversationChain(llm=OpenAI(), memory=memory, prompt=CUSTOM_PROMPT)

response = chain.predict(input="Hello")
```

**After (Sonzai):**
```python
from sonzai import Sonzai
client = Sonzai()

# Once: create the agent
agent = client.agents.create(name="Assistant", personality_prompt=CUSTOM_PROMPT)

# Per conversation:
session = client.agents.sessions.start(agent.agent_id, user_id="u1", session_id="s1")
result = session.turn(messages=[{"role": "user", "content": "Hello"}])
print(result.response)
```

**Before (LangChain VectorStoreRetrieverMemory):**
```python
from langchain.memory import VectorStoreRetrieverMemory
from langchain.vectorstores import Pinecone

retriever = Pinecone(...).as_retriever()
memory = VectorStoreRetrieverMemory(retriever=retriever)
relevant = memory.load_memory_variables({"input": "..."})
```

**After (Sonzai):**
```python
# Upload your vector store content to Sonzai KB first (migration step)
# Then search:
results = client.knowledge.search("project-id", query="...", limit=10)
```

## Gotchas specific to LangChain

- **LangChain agents ≠ Sonzai agents.** LangChain "agents" are tool-using LLM loops. Sonzai "agents" are persistent personalities with memory. Don't confuse the two.
- **Custom prompts** — `ChatPromptTemplate` with variables → use `compiled_system_prompt` per call, substituting variables in your code before passing.
- **Chain composition** — LangChain's `Runnable` composition doesn't have a direct equivalent in Sonzai (which is by design more opinionated). Move orchestration logic to your own code.
- **Multiple LLMs in one chain** — Sonzai's `agents.chat` uses one LLM per call. For multi-LLM flows (e.g. one model classifies, another responds), make two separate `agents.chat` calls.
- **LangChain tracing (LangSmith)** — Sonzai has its own observability surface; LangSmith integration is not built-in. Audit via webhooks instead.
- **Retriever score thresholds** — LangChain uses cosine similarity; Sonzai search returns scores too but threshold semantics may differ. Re-tune your threshold.

## Verification

- Sample 30 conversations from production. Re-run through Sonzai with the same inputs; compare response quality + retrieval quality.
- Check that tool-calling parity is intact — LangChain's tool descriptions may be more permissive than Sonzai's; tighten as needed.

## Cross-references

- `features/custom-tools.md` — tool migration
- `features/knowledge-base.md` — retriever → KB
- `archetypes/companion.md` / `archetypes/customer-support.md` — typical destinations
- `migrations/overview.md`
