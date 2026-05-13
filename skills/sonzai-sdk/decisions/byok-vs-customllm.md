---
name: decision-byok-vs-customllm
description: Use when picking how to route LLM inference for an agent — platform default, BYOK (your provider key), or Custom LLM (your own OpenAI-compatible endpoint).
---

# Decision: BYOK vs Custom LLM vs platform default

## The rule

Three paths, increasing in control and complexity:

1. **Platform default** — Sonzai picks model + provider; you pay through Sonzai. Simplest. Use unless you have a specific reason.
2. **BYOK (Bring Your Own Key)** — Register your own provider key (OpenAI / Gemini / xAI / OpenRouter). LLM token cost routes through your provider account. Best for billing isolation, rate-limit isolation, and audit-friendly routing.
3. **Custom LLM (BYOM)** — Point at your own OpenAI-compatible endpoint. You control the model entirely. Best for fine-tuned models, on-prem requirements, or self-hosted stacks (vLLM, llama.cpp, internal inference).

## How to apply

```python
# Platform default — nothing to configure
client.agents.chat(agent_id, messages=[...])  # uses platform routing

# BYOK — register your provider key
client.byok.set("project-id", "openai", api_key="sk-...")
# Subsequent chats on this project will route through your OpenAI key
client.agents.chat(agent_id, messages=[...], provider="openai", model="gpt-5.5")

# Custom LLM — point at your endpoint
client.custom_llm.set(
    project_id="project-id",
    endpoint_url="https://your-endpoint.example.com/v1",
    model_name="your-finetuned-model",
    api_key="your-endpoint-token",
)
client.agents.chat(agent_id, messages=[...], provider="custom", model="your-finetuned-model")
```

Verify exact `custom_llm.set` shape against your installed SDK — the surface may differ slightly.

## Pick the path

| Your situation | Recommended |
|---|---|
| Default — you want to ship fast | Platform default |
| You want token billing on your own provider account (cost isolation, audit) | **BYOK** |
| You need rate-limit isolation per project | BYOK |
| Your compliance regime requires data flow through your provider's region/entity | BYOK with the right provider region |
| You have a fine-tuned model | **Custom LLM** |
| Self-hosted inference (vLLM, llama.cpp, on-prem) | Custom LLM |
| Internal model with no public API | Custom LLM with a private endpoint |
| You want to test multiple providers | Platform default (let the fallback chain handle it) |

## Why

- **Platform default:** Sonzai's routing handles fallbacks on 429 (e.g. Gemini Flash → Gemini Pro). Cheap, simple, no extra wiring. You give up control over which provider/model serves any specific request.
- **BYOK:** You keep using Sonzai's provider integrations (one of the 4 supported: OpenAI, Gemini, xAI, OpenRouter) but billing routes through your key. Keys are encrypted at rest server-side and never returned (only `api_key_prefix` and health metadata are exposed). Per-project scope.
- **Custom LLM:** Maximum control. You bring an OpenAI-compatible endpoint. Sonzai sends requests as if it's calling OpenAI; your endpoint can be anything (fine-tuned, self-hosted, internal). Cost of complexity: you operate the inference layer.

## Exceptions

- **Compliance prohibits sending data through Sonzai's billing path** → BYOK with strict provider region.
- **Compliance prohibits external inference entirely** → Custom LLM with an internal endpoint.
- **You want to enforce a specific cheap model for cost reasons** → BYOK with rate-limited key, or pin model per chat.

## Cross-references

- `features/models.md` — full models surface (providers, BYOK, Custom LLM, post-processing model map)
- `decisions/post-processing-model.md` — separate decision about which model handles extraction/mood/personality
- `archetypes/enterprise-assistant.md` — BYOK is common here for compliance
- `archetypes/customer-support.md` — BYOK if customer billing isolation matters
