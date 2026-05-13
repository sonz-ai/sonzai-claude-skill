---
name: decision-byok-vs-customllm
description: Use when picking how to route LLM inference for an agent — platform default, BYOK (your provider key), or Custom LLM (your own OpenAI-compatible endpoint).
---

# Decision: BYOK vs Custom LLM vs platform default

## The rule

**Production posture: BYOK or Custom LLM. Platform default is for development and evaluation only.**

Three paths:

1. **BYOK (Bring Your Own Key)** — **recommended for production.** Register your own provider key (OpenAI / Gemini / xAI / OpenRouter). LLM token cost routes through your provider account. Best for billing isolation, rate-limit isolation, audit cleanliness, and provider-region compliance.
2. **Custom LLM (BYOM)** — point at your own OpenAI-compatible endpoint. You control the model entirely. Best for fine-tuned models, on-prem / self-hosted stacks (vLLM, llama.cpp, internal inference), or compliance regimes that prohibit external inference.
3. **Platform default** — Sonzai picks model + provider; you pay through Sonzai. **Dev / eval only.** Convenient for getting started, prototyping, and running internal evals — but you give up rate-limit isolation and route all token cost through Sonzai's billing. Don't ship production traffic this way.

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
| **Production traffic** (any volume) | **BYOK** (or Custom LLM) — not platform default |
| Prototyping, getting started, internal demos | Platform default (cheapest setup time) |
| Running evals before launch | Platform default for eval; flip to BYOK before going live |
| You want token billing on your own provider account (cost isolation, audit) | **BYOK** |
| You need rate-limit isolation per project | **BYOK** |
| Your compliance regime requires data flow through your provider's region/entity | **BYOK** with the right provider region |
| You have a fine-tuned model | **Custom LLM** |
| Self-hosted inference (vLLM, llama.cpp, on-prem) | **Custom LLM** |
| Internal model with no public API | **Custom LLM** with a private endpoint |
| You want to test multiple providers during development | Platform default (let the fallback chain handle it); switch to BYOK once you pick a winner |

## Why

- **BYOK (production recommended):** You keep using Sonzai's provider integrations (OpenAI / Gemini / xAI / OpenRouter) but billing routes through your key. Keys are encrypted at rest server-side and never returned (only `api_key_prefix` and health metadata are exposed). Per-project scope. **In production this is the default posture** — keeps your provider relationship intact, gives you per-project rate limits, isolates failures from other Sonzai tenants, and produces cleaner audit logs.
- **Custom LLM:** Maximum control. You bring an OpenAI-compatible endpoint. Sonzai sends requests as if it's calling OpenAI; your endpoint can be anything (fine-tuned, self-hosted, internal). Cost of complexity: you operate the inference layer.
- **Platform default (dev/eval only):** Sonzai's routing handles fallbacks on 429 (e.g. Gemini Flash → Gemini Pro). Cheap setup, no extra wiring — great for prototypes and evaluation runs. **Not for production** because all token cost routes through Sonzai's billing and you share rate limits with platform-wide traffic. Use it to get started, then flip to BYOK before going live.

## Exceptions

- **Compliance prohibits sending data through Sonzai's billing path** → BYOK with strict provider region.
- **Compliance prohibits external inference entirely** → Custom LLM with an internal endpoint.
- **You want to enforce a specific cheap model for cost reasons** → BYOK with rate-limited key, or pin model per chat.

## Cross-references

- `runtime-mode.md` — BYOK and Custom LLM only apply to runtime modes A (full chat) and B (full chat with sessions); modes C/D run your own LLM independently
- `features/models.md` — full models surface (providers, BYOK, Custom LLM, post-processing model map)
- `decisions/post-processing-model.md` — separate decision about which model handles extraction/mood/personality
- `archetypes/enterprise-assistant.md` — BYOK is common here for compliance
- `archetypes/customer-support.md` — BYOK if customer billing isolation matters
