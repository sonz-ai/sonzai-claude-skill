---
name: feature-models
description: Use when configuring LLM provider routing — platform default, BYOK (your provider key), Custom LLM (your OpenAI-compatible endpoint), or the post-processing model map.
---

# Models (providers, BYOK, Custom LLM, post-processing)

## What it is

Four related surfaces:

1. **Providers** — Sonzai's supported LLM providers (Gemini, OpenAI, xAI, OpenRouter) with fallback chains
2. **BYOK** — register your own provider key for billing isolation
3. **Custom LLM (BYOM)** — point Sonzai at your own OpenAI-compatible endpoint
4. **Post-processing model map** — choose cheap fast tier for extraction/mood/personality

See `decisions/byok-vs-customllm.md` for the picking-which decision.

## When to use

| Goal | Surface |
|---|---|
| Pick model per call | `provider` + `model` in chat options |
| Discover available models | `client.list_models()` |
| Billing isolation | BYOK (`client.byok`) |
| Self-hosted / fine-tuned | Custom LLM (`client.custom_llm`) |
| Cheap post-processing | post-processing model map (project config) |

## SDK surface

### List models

```python
result = client.list_models()
for prov in result.providers:
    print(prov.name, prov.models)
print(result.default_model)
```

### Per-call provider + model

```python
client.agents.chat(
    agent_id,
    messages=[{"role": "user", "content": "..."}],
    provider="openai",                    # gemini | openai | xai | openrouter | custom
    model="gpt-5.5",
)
```

Or via constants:

```typescript
import { Sonzai, providers } from "@sonzai-labs/agents";
await client.agents.chat({
  agent: agentId,
  messages: [...],
  provider: providers.GEMINI,
  model: providers.models.gemini.FLASH_LITE,
});
```

### BYOK

```python
# Set / replace (validates against the provider before saving)
key = client.byok.set("project-id", "openai", api_key="sk-...")
print(key.api_key_prefix)             # e.g. "sk-...abc" (only prefix returned)

# List configured providers
keys = client.byok.list("project-id")
for k in keys:
    print(k.provider, k.health_status, k.is_active)

# Enable / disable without rotating
client.byok.set_active("project-id", "openai", is_active=False)

# Re-run health check
result = client.byok.test("project-id", "gemini")
print(result.health_status)           # healthy | invalid | unknown

# Remove
client.byok.delete("project-id", "xai")
```

```typescript
await client.byok.set("project-id", "openai", "sk-...");
await client.byok.list("project-id");
```

```go
key, err := client.BYOK.Set(ctx, "project-id", sonzai.BYOKProviderOpenAI, "sk-...")
```

Providers: `"openai" | "gemini" | "xai" | "openrouter"`.

Requires API key scopes: `read:byok` / `write:byok`.

### Custom LLM (BYOM)

```python
# Point at your own OpenAI-compatible endpoint — verify exact API shape in your SDK
client.custom_llm.set(
    project_id="project-id",
    endpoint_url="https://your-endpoint.example.com/v1",
    model_name="your-finetuned-model",
    api_key="your-endpoint-token",        # if your endpoint requires auth
)

# Toggle on/off
client.custom_llm.set_active("project-id", is_active=True)

# Remove
client.custom_llm.delete("project-id")

# Then use it
client.agents.chat(agent_id, messages=[...], provider="custom", model="your-finetuned-model")
```

Your endpoint must accept OpenAI's `/v1/chat/completions` schema. Sonzai sends requests in that format.

### Post-processing model map

```python
# Per-project — verify exact API path
client.project_config.set(
    project_id="project-id",
    post_processing_model_map={
        "*": "gemini-3.1-flash-lite",     # wildcard fallback
        "gpt-5.5": "gemini-3.1-flash-lite",
        "specialized-medical": "gpt-5.5",  # upgrade for high-stakes
    },
)
```

See `decisions/post-processing-model.md` for the picking rule.

## Decisions linked

- `decisions/byok-vs-customllm.md` — the picking rule
- `decisions/post-processing-model.md` — model map choice
- `archetypes/enterprise-assistant.md` — common BYOK user

## Common gotchas

- **BYOK keys are encrypted at rest** — server never returns key material, only `api_key_prefix` and health.
- **Fallback chains on 429** — Sonzai falls back to next-tier model on rate limits. Configure your project's fallback preferences in dashboard.
- **Custom LLM must be OpenAI-compatible** — `/v1/chat/completions` format. Most local servers (vLLM, llama.cpp) support this.
- **Per-call `provider`/`model` overrides** — session-level + per-call shadowing. Verify precedence in your SDK.
- **Post-processing wildcard** — `"*"` matches any chat model not explicitly mapped.
- **BYOK + Custom LLM combined** — uncommon but possible (BYOK for chat models, Custom LLM for one specific model name).
- **Scopes** — BYOK requires `read:byok` / `write:byok` on the API key making the configuration call.
- **Multi-region keys** — if your compliance requires data flow through a specific provider region, set the BYOK key for that region's endpoint.
