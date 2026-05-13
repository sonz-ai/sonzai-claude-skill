---
name: feature-mcp-integration
description: Use when integrating Sonzai into an MCP-aware client (Claude Code, Cursor, Claude Desktop, ChatGPT, VS Code) via the hosted MCP server at api.sonz.ai/mcp/memory/{agent_id}. No SDK code required — just client config.
---

# MCP integration

## What it is

Sonzai ships a hosted **Streamable HTTP** MCP server at `https://api.sonz.ai/mcp/memory/{agent_id}`. Any MCP-compatible client can point at it. **34 tools, 4 resources, 3 guided prompts.** No local binary, no SSE port, no Go toolchain — just client config + API key.

This is a **consumption** path. To author your own MCP server, that's a separate concern (see `superpowers:mcp-builder` if available).

## When to use

- User is already inside an MCP-aware client (Claude Code, Cursor, Claude Desktop, ChatGPT Developer Mode, VS Code)
- Driving Sonzai by conversation rather than by SDK code
- Prototyping — use the `mind-layer-setup` or `create-companion` guided prompts to spin up an agent without writing code
- Replacing per-MCP-call wiring with one config block

## When NOT to use

- Building your own product UI — use Pattern 1 (Managed Runtime: the SDKs covered everywhere else in this skill)
- You want Sonzai inside your own LLM loop, not exposed as a tool to someone else's — use Pattern 4 (Standalone Realtime — SDK with `skip_context_build` per call)
- Batch-only flows with no real-time chat — use Pattern 5 (Standalone Batch)

## Setup snippets (paste per client)

You need:
- **API key** from `https://platform.sonz.ai/dashboard/projects`
- **Agent ID** — either create one via SDK / dashboard first, or use a guided prompt to create one in-client

### Claude Code

One-liner:

```bash
claude mcp add --transport http sonzai \
  https://api.sonz.ai/mcp/memory/AGENT_ID \
  --header "Authorization: Bearer $SONZAI_API_KEY"
```

Then in a Claude Code session:
```
"Chat with agent 'Luna' and say 'I had a great day hiking today!'"
"Search Luna's memories about hiking adventures"
"Use mind-layer-setup with assistant_name 'Aria' ..."
```

### Cursor / VS Code

`~/.cursor/mcp.json` (Cursor) or `.vscode/mcp.json` (VS Code — note: VS Code uses top-level `"servers"` instead of `"mcpServers"`):

```json
{
  "mcpServers": {
    "sonzai": {
      "type": "http",
      "url": "https://api.sonz.ai/mcp/memory/AGENT_ID",
      "headers": {
        "Authorization": "Bearer YOUR_SONZAI_API_KEY"
      }
    }
  }
}
```

Restart Cursor / VS Code. Sonzai tools appear in @-mentions and the MCP picker.

### ChatGPT

Available on Plus / Pro / Business / Enterprise / Edu (Developer Mode beta).

1. Settings → Connectors → Advanced → **Developer Mode ON**
2. Add custom connector:
   - URL: `https://api.sonz.ai/mcp/memory/AGENT_ID`
   - Header: `Authorization: Bearer YOUR_API_KEY`

### Claude Desktop

The JSON config supports **stdio servers only** — for the hosted HTTP endpoint use the in-app Connectors UI:

- Settings → Connectors → Add custom connector
  - URL: `https://api.sonz.ai/mcp/memory/AGENT_ID`
  - Auth: `Authorization: Bearer YOUR_API_KEY`

### Local stdio binary (offline / air-gapped clients)

If you need a stdio server (Claude Desktop pre-Connectors, or offline use):

```bash
# Install (Go 1.25+):
go install github.com/sonz-ai/sonzai-mcp/cmd/mcp-server@latest

# Add to your stdio-only client config, e.g. macOS Claude Desktop at
# ~/Library/Application Support/Claude/claude_desktop_config.json:
{
  "mcpServers": {
    "sonzai": {
      "command": "sonzai-mcp",
      "env": { "SONZAI_API_KEY": "sk-your-api-key" }
    }
  }
}
```

## What you get (34 tools, sample list)

Approximate tool surface (verify against `https://api.sonz.ai/mcp/memory/{agent_id}` via the MCP `list_tools` call):

- `list_agents` / `create_agent` / `generate_character`
- `chat` / `start_session` / `end_session`
- `search_memories` / `list_facts`
- `get_personality` / `get_mood`
- `trigger_event` / `schedule_wakeup` / `list_notifications`
- `mind-layer-setup` (guided prompt) / `create-companion` (guided prompt)

## Trade-offs vs SDK

| Concern | MCP | SDK |
|---|---|---|
| Code required | None | Yes |
| Per-turn enriched context fetching | Limited (tool call) | Full (`session.context(query=...)`) |
| Streaming chat (SSE) | Yes via MCP tool | Yes natively |
| Custom tool dispatch shape | MCP-mediated | `side_effects.external_tool_calls` direct |
| Multi-archetype orchestration | One agent per MCP server endpoint | Full programmatic control |
| Latency | One extra hop via MCP client | Direct |
| Best for | In-IDE / in-chat prototyping | Production product |

## Decisions linked

- `decisions/sessions-vs-conversations.md` — MCP `chat` tool ≈ `agents.chat`; for explicit lifecycle use `start_session` / `end_session`
- All archetypes — same archetypes apply when consuming via MCP (companion / guide-router / etc.); you just talk to the agent through MCP instead of through code
- `features/capabilities.md` — capabilities still set via SDK or dashboard; MCP doesn't expose `update_capabilities`

## Common gotchas

- **Hosted MCP requires the agent to exist already.** Create the agent first via SDK / dashboard / guided prompt; then point MCP at it.
- **Replace `AGENT_ID` in the URL** with your actual agent's UUID. The URL is agent-scoped, not project-scoped.
- **API key auth via header** — `Authorization: Bearer sk-...`. Don't put the key in the URL.
- **34 tools is a lot** — MCP clients may load all of them into context per session. If your client has a tool budget, that matters.
- **Stdio is local-only** — the hosted HTTP endpoint is the recommended path for production. Use stdio only when forced (Claude Desktop pre-Connectors UI, air-gapped environments).
- **No per-turn context override** — MCP `chat` doesn't expose `compiled_system_prompt` per call. If you need that, use the SDK directly.
- **MCP tool retries** — the MCP client may retry on transient failures; ensure your downstream handling is idempotent.

## Cross-references

- `https://sonz.ai/docs/connect-mcp` — public docs version
- `features/openclaw-integration.md` — alternative integration path (slot-based plugin in OpenClaw)
- `references/auth-and-setup.md` — API key handling
- `intake.md` — wizard fork: if user is consuming via MCP-only, archetype playbooks still apply; just configure via this file instead of SDK code
