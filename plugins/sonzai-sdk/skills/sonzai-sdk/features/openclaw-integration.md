---
name: feature-openclaw-integration
description: Use when integrating Sonzai into an OpenClaw project. The @sonzai-labs/openclaw-context plugin registers Sonzai as the contextEngine slot — Sonzai handles memory, mood, personality while OpenClaw's chat loop and tool plugins stay intact.
---

# OpenClaw integration

## What it is

[OpenClaw](https://openclaw.ai) is an open-source framework for building conversational agents via a slot-based plugin system. The `contextEngine` slot decides what context goes into the system prompt on every turn. Installing `@sonzai-labs/openclaw-context` registers Sonzai under the name `"sonzai"` — assign it to the slot in `openclaw.json` and every conversation flows through the Mind Layer.

OpenClaw is JavaScript-only. The plugin works in any OpenClaw project regardless of what language you wrote your tools in.

## When to use

- You're already on OpenClaw (your team standardized on it)
- You want OpenClaw's existing chat loop, telemetry, and tool plugins to keep working — Sonzai swaps only the memory/personality layer
- You want a `<sonzai-context>` block auto-injected into every system prompt, priority-ordered and budget-trimmed

## When NOT to use

- Not on OpenClaw → use Pattern 1 (Managed Runtime — SDKs) or Pattern 4 (Standalone Realtime — SDK with `skip_context_build`)
- No real-time chat → use Pattern 5 (Standalone Batch)

## Setup

### Prerequisite: install OpenClaw

```bash
npm install -g openclaw@latest
openclaw onboard --install-daemon
```

(See [openclaw.ai docs](https://docs.openclaw.ai/start/getting-started) for full OpenClaw setup.)

### One-shot install

```bash
# Probes Sonzai backend health, runs `openclaw plugins install`,
# launches the interactive wizard, writes ~/.openclaw/openclaw.json
npx --yes @sonzai-labs/openclaw-context install
```

### Manual install

```bash
# 1. Install the plugin via the OpenClaw CLI
openclaw plugins install @sonzai-labs/openclaw-context

# 2. Run the Sonzai setup wizard (asks for API key + agent name/ID)
npx @sonzai-labs/openclaw-context setup

# 3. Restart the gateway
openclaw gateway restart
```

### openclaw.json config

The wizard writes this for you; shown here for hand-edits or CI:

```json
{
  "plugins": {
    "slots": {
      "contextEngine": "sonzai"
    },
    "entries": {
      "sonzai": {
        "enabled": true,
        "apiKey": "sk-your-api-key",
        "agentId": "your-agent-uuid"
      }
    }
  }
}
```

## Architecture

```
OpenClaw Runtime              SonzaiContextEngine            Sonzai Mind Layer
      |                                |                            |
      |-- bootstrap(sessionId) ------->|                            |
      |                                |-- resolve agent + session->|
      |                                |<-- session state ----------|
      |                                |                            |
      |-- assemble(messages, budget) ->|                            |
      |                                |-- fetch memory, mood,      |
      |                                |   personality, goals ----->|
      |                                |<-- ranked context blocks --|
      |<-- systemPromptAddition -------|   priority-ordered,        |
      |                                |   token-budget-trimmed     |
      |                                |                            |
      |  [LLM call w/ enriched prompt] |                            |
      |                                |                            |
      |-- afterTurn(sessionId) ------->|                            |
      |                                |-- send transcript -------->|
      |                                |   Mind Layer extracts      |
      |                                |   facts, updates mood,     |
      |                                |   evolves personality      |
      |                                |                            |
      |-- compact(sessionId) --------->|                            |
      |                                |-- merge short → long term->|
```

## B2B provisioning (programmatic — TS / Python / Go)

OpenClaw runtime is JS, but if you provision agents from a non-JS backend, derive the agent UUID deterministically:

```typescript
// TypeScript
import { v5 as uuidv5 } from "uuid";
import { setup } from "@sonzai-labs/openclaw-context";

const MY_NAMESPACE = "your-uuid-namespace-here";
const agentId = uuidv5("customer-acme-corp", MY_NAMESPACE);

await setup({
  apiKey: process.env.SONZAI_API_KEY!,
  agentId,
});
// Writes ~/.openclaw/openclaw.json with the right config
```

```python
# Python (B2B provisioning — derive UUID then write config)
import uuid, json, os

NAMESPACE = uuid.UUID("your-uuid-namespace-here")
agent_id = str(uuid.uuid5(NAMESPACE, "customer-acme-corp"))

# Create / update the agent in Sonzai first
from sonzai import Sonzai
client = Sonzai()
client.agents.create(agent_id=agent_id, name="Acme Companion", ...)

# Then write openclaw.json (or invoke the JS setup helper from your provisioning script)
openclaw_config_path = os.path.expanduser("~/.openclaw/openclaw.json")
config = {
    "plugins": {
        "slots": {"contextEngine": "sonzai"},
        "entries": {
            "sonzai": {
                "enabled": True,
                "apiKey": os.environ["SONZAI_API_KEY"],
                "agentId": agent_id,
            },
        },
    },
}
with open(openclaw_config_path, "w") as f:
    json.dump(config, f, indent=2)
```

```go
// Go (B2B provisioning)
import (
    "github.com/google/uuid"
    sonzai "github.com/sonz-ai/sonzai-go"
)

namespace := uuid.MustParse("your-uuid-namespace-here")
agentID := uuid.NewSHA1(namespace, []byte("customer-acme-corp")).String()

client, _ := sonzai.NewClient("")
client.Agents.Create(ctx, sonzai.CreateAgentOptions{AgentID: agentID, Name: "..."})
// Write openclaw.json — same shape as the Python example
```

## Trade-offs vs other patterns

| Concern | OpenClaw plugin | Managed Runtime (SDK) | MCP |
|---|---|---|---|
| OpenClaw chat loop, tools, telemetry | ✅ Reused as-is | ❌ Not used | ❌ Not used |
| Per-turn enriched context | ✅ Via plugin's `assemble` hook | ✅ Via `session.context` | Partial |
| Custom tool dispatch | OpenClaw's tool system | `side_effects.external_tool_calls` | MCP-mediated |
| Memory + personality + mood + drift | ✅ All on | ✅ All on | ✅ All on |
| Code required | Minimal (just config) | Full SDK code | None |
| Best for | OpenClaw-native projects | Greenfield product | In-IDE / in-chat |

## Decisions linked

- `archetypes/companion.md` and friends — same archetypes apply; you configure the agent first via SDK / dashboard, then point OpenClaw's `contextEngine` slot at it
- `features/capabilities.md` — capabilities are set on the agent via SDK / dashboard; the plugin reads them but doesn't toggle
- `features/mcp-integration.md` — alternative integration path

## Common gotchas

- **OpenClaw must be installed first.** The `npx ... install` wizard checks; if missing, run `npm install -g openclaw@latest` first.
- **Agent must exist in Sonzai before the slot connects.** Create via SDK or the wizard's interactive prompt.
- **The plugin is JS-only.** Non-JS backends provision the config; the runtime that consumes it is JS.
- **B2B provisioning** — use deterministic `uuid5` for agent IDs; safe to re-run.
- **`enabled: false`** in the entry disables the slot without removing the config — useful for A/B tests.
- **Single agent per OpenClaw config.** For multi-agent setups, you switch the `agentId` in the entry. Or run multiple OpenClaw projects.
- **`apiKey` in config file** — treat the openclaw.json as a secret (gitignore it; in production write it from secrets manager at boot).

## Cross-references

- `https://sonz.ai/docs/connect-openclaw` — public docs version
- `https://openclaw.ai` — OpenClaw itself
- `features/mcp-integration.md` — alternative integration path
- `references/auth-and-setup.md` — API key handling
