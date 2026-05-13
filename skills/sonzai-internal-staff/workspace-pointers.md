# Workspace pointers

Where to read when working with `sonzai-sdk` or `full-auto` as Sonzai internal staff. Staff workspace lives under `$SONZAI_WORKSPACE`.

## Resolving `$SONZAI_WORKSPACE`

The path varies per machine. **Never hardcode a specific path** (e.g. don't assume `~/code/sonzai/` exists). Resolve in this order:

1. **Env var**: read `$SONZAI_WORKSPACE`. If set and exists, use it.
2. **CWD ancestor**: walk up from the operator's current working directory; the first ancestor directory containing BOTH `sonzai-sdk/` and `sonzai-ai-monolith-ts/` is the workspace root.
3. **Common candidates** (try in order, use the first that exists and contains both required subdirs):
   - `$HOME/code/sonzai/`
   - `$HOME/work/sonzai/`
   - `$HOME/dev/sonzai/`
   - `$HOME/src/sonzai/`
4. **If still unresolved**: prompt-back is forbidden by the public skills' rules, but this is the internal-staff skill — print a one-line message: "Set `SONZAI_WORKSPACE` to the directory that contains `sonzai-sdk/` and `sonzai-ai-monolith-ts/`." and exit.

Every path in this file is rooted at `$SONZAI_WORKSPACE` (referenced as `${WORKSPACE}` below for brevity). Never embed a literal absolute path in code, commits, or docs you write — `${WORKSPACE}` only.

---

## Top-level workspace map

```
$SONZAI_WORKSPACE/
├── sonzai-claude-skill/       # this repo — the public skill (where sonzai-internal-staff also lives)
├── sonzai-sdk/                # SDK source — Python, TypeScript, Go, OpenClaw, vertical-evals
├── sonzai-ai-monolith-ts/     # the monolith — Context Engine, Platform API, AI Service, deploy
├── sonzai-landing/            # public landing + sonz.ai/docs source
├── sonzai-admin-backend/      # admin backend (internal-only)
├── sonzai-windmill/           # workflow orchestration (Windmill scripts)
├── sonzai-openclaw/           # OpenClaw integration host (duplicated under sonzai-sdk/ in some setups)
└── ...                        # other tenant verticals (out of scope for this skill)
```

For SDK-correctness and full-auto QA, the **three primary read targets** are:

1. `sonzai-sdk/` — what callers see
2. `sonzai-ai-monolith-ts/` — what the server actually does
3. `sonzai-landing/src/app/docs/` — what's published to developers at sonz.ai/docs

The rest are tenant projects; ignore unless the operator explicitly references them.

---

## SDK source — `sonzai-sdk/`

```
sonzai-sdk/
├── CLAUDE.md                  # SDK-wide rules (read this BEFORE editing any SDK)
├── sonzai-python/             # Python SDK (pip install sonzai)
│   ├── src/sonzai/            # package source
│   ├── benchmarks/, demos/, docs/
│   ├── CHANGELOG.md, DEPLOY.md
├── sonzai-typescript/         # TypeScript SDK (npm install @sonzai-labs/agents)
│   └── src/
│       ├── client.ts, errors.ts, http.ts, index.ts, types.ts, providers.ts
│       ├── resources/         # per-resource clients (agents, chat, schedules, etc.)
│       ├── generated/         # auto-generated from OpenAPI
│       └── post-processing-model.ts
├── sonzai-go/                 # Go SDK (go get github.com/sonz-ai/sonzai-go)
│   ├── client.go, agents.go, chat_*.go, custom_llm.go, byok.go, ...
│   ├── api.md                 # API surface doc
│   └── CONTRIBUTING.md
├── sonzai-openclaw/           # OpenClaw plugin source (@sonzai-labs/openclaw-context)
├── sonzai-vertical-evals/     # evaluation harness for SDK-built verticals
├── mempalace/                 # memory-pipeline experiments
└── EverOS/                    # research / experimental
```

**When to read here:**

- Wizard or full-auto needs to verify a method signature → grep the matching language directory under `src/` (TS), `src/sonzai/` (Py), or top-level `.go` files (Go)
- Drift question — does a flag exist in the SDK yet? → grep the generated bindings (`sonzai-typescript/src/generated/`, the Python equivalent, the Go top-level files)
- Demo / fixture needed → `sonzai-python/demos/`, similar in other SDKs
- API surface doc → `sonzai-go/api.md` (Go's manual reference); generated for TS/Py
- Evaluation runs (when changing SDK behavior) → `sonzai-vertical-evals/`

**Source-of-truth ordering for SDK behavior** (highest to lowest priority):

1. Go SDK source (`sonzai-go/*.go`) — usually the first SDK to get new endpoints
2. TypeScript SDK source (`sonzai-typescript/src/`) — close behind, sometimes ahead for streaming
3. Python SDK source (`sonzai-python/src/sonzai/`)
4. Generated bindings inside each SDK — derived; if disagrees with hand-written, the hand-written wins
5. `sonzai-go/api.md` and per-SDK `docs/` — narrative; lags source

If three SDKs disagree on a method shape, the **server's OpenAPI** (`services/platform/api/docs/` in the monolith) is the tiebreaker — see "OpenAPI: live vs source" below.

---

## Monolith — `sonzai-ai-monolith-ts/`

```
sonzai-ai-monolith-ts/
├── CLAUDE.md                          # project rules (read this BEFORE editing anything in the monolith)
├── ARCHITECTURE.md                    # high-level architecture
├── go.work                            # Go workspace
├── services/
│   ├── contextengine/                 # Go — stateful agent intelligence
│   ├── platform/api/                  # Go — REST/SSE API gateway
│   ├── platform/app/                  # TS — Sonzai dashboard
│   ├── ai-character-service/          # TS (Bun/Elysia) — stateless LLM + media middleware
│   ├── observability-agent/           # Go — telemetry agent
│   ├── demo/                          # TS — demo containers (general-demo, razer-demo)
│   ├── legacy/                        # frozen — do not touch
│   └── internal-docs/                 # served at staging
├── deploy/                            # shell scripts + Caddy config
├── ops/                               # operational tooling
└── docs/                              # internal engineering docs
```

### Context Engine — `services/contextengine/`

The **stateful brain**. Owns memory, personality, mood, behavioral processors, jobs. Tenant-agnostic.

| Subdir | Purpose |
|---|---|
| `cmd/` | binaries (engine server, CLI tools) |
| `domain/entity/` | core types (`agent.go`, `mood.go`, `memory.go`) — source of truth for SDK-visible schemas |
| `domain/repo/` | repository interfaces |
| `cestore/` | persistence (ScyllaDB, Postgres) |
| `behavior/` | processors (diary, mood drift, personality evolution) |
| `consolidation/` | memory consolidation pipelines |
| `constellation/` | relationship/topic graphs |
| `context/` | per-turn context builder (`Builder.FetchParallel`, `Builder.AssembleEnriched`) |
| `activity/`, `crossref/`, `dedup/`, `autotune/`, `cost/` | supporting subsystems |

### Platform API — `services/platform/api/`

The **REST/SSE gateway**. Tenant backends call here; this service calls context engine + ai-character-service.

| Subdir | Purpose |
|---|---|
| `cmd/` | HTTP server binary |
| `internal/handlers/` | per-resource handlers |
| `internal/middleware/` | auth, logging, etc. |
| `docs/` | generated OpenAPI artifacts |

### AI Character Service — `services/ai-character-service/`

The **stateless LLM + media middleware**. No DB, no session, no tenant logic — strictest rules.

Read its own `CLAUDE.md` before any edits. Owns:
- LLM provider routing (Gemini, OpenRouter, etc.)
- DashScope Wan API (image/video)
- ElevenLabs (TTS, music, SFX)

### Deploy — `deploy/`

| File | Purpose |
|---|---|
| `deploy-staging.sh` | full platform deploy to staging |
| `deploy-production.sh` | production deploy |
| `deploy-demos-staging.sh` | one-time staging demo setup |
| `update.sh staging` | ongoing platform updates to staging |
| `Caddyfile.staging`, `Caddyfile.production` | reverse-proxy config |

---

## Public docs — `sonzai-landing/`

```
sonzai-landing/
├── CLAUDE.md                          # landing-specific rules
├── src/app/docs/                      # developer docs served at sonz.ai/docs
│   └── content/{en,zh,ja}/            # localized markdown
```

When a public-skill recommendation cites "sonz.ai/docs", the source lives here. If a doc page is wrong, fix it in `sonzai-landing/src/app/docs/content/<locale>/...`, not in the skill itself.

---

## SDK ↔ monolith request mapping

When a public-skill user reports a bug, the request flow is:

```
caller code (using sonzai-sdk SDK)
   │
   │  HTTPS → api.sonz.ai
   ▼
services/platform/api/internal/handlers/<resource>*   (Go)
   │
   ├── calls → services/contextengine/...              (Go, in-VM)
   │            (memory, personality, mood, jobs)
   │
   └── calls → services/ai-character-service/...       (TS, in-VM)
                (LLM, image, audio gen)
```

To trace an SDK call end to end:

| SDK surface | Lives at |
|---|---|
| `client.chat.completions.create` | platform-api `handlers/chat*` → AI-character-service for LLM + contextengine for memory |
| `client.agents.create` / `update` / `delete` | platform-api `handlers/agents*` → contextengine `cestore/` |
| Capability flags (`UpdateCapabilitiesInputBody`) | source-of-truth: contextengine `domain/entity/agent.go` |
| Memory mode (sync/async) | contextengine `context/` (sync read) + `consolidation/` (async write) |
| Personality drift | contextengine `behavior/` + `domain/entity/personality*` |
| Mood | contextengine `domain/entity/mood.go` + processors under `behavior/` |
| Inventory + custom_states | contextengine `domain/entity/` + platform-api `handlers/` |
| Schedules / proactive | platform-api `handlers/schedules*` |
| Webhooks | platform-api `handlers/webhooks*` |
| KB / RAG | contextengine (search) + platform-api `handlers/kb*` |

---

## OpenAPI: live vs source

The public skill fetches OpenAPI from `https://api.sonz.ai/docs/openapi.json`. For internal dogfooding the **most current** schema may not be deployed yet.

Source-of-truth ordering (highest first):

1. **Go struct tags in `sonzai-ai-monolith-ts/services/contextengine/domain/entity/`** — earliest signal
2. **Generated OpenAPI in `sonzai-ai-monolith-ts/services/platform/api/docs/`** — built from Huma annotations
3. **Live OpenAPI at api.sonz.ai** — what external SDK users see; lags by one deploy
4. **SDK generated bindings** (`sonzai-sdk/*/generated/`) — built from #3; lags by one SDK release

When the four disagree, the upper layer is "the future"; the live API is "what's currently shipped." External users care about #3; internal staff debugging "why doesn't my new field work" should check #1 first.

---

## Server-log tailing during full-auto QA

When `full-auto` QA cycle fails with a 5xx and internal staff want server-side detail:

- **Local**: services start with `cmd/` Go binaries; stdout is the log. `tail -f` or `docker logs -f <container>` if Dockerized
- **Staging**: ssh to the staging VM; tail Caddy + service logs (paths in `deploy/Caddyfile.staging`)
- **Production**: do NOT tail production logs from a skill-driven session — humans-only

This is **read-only diagnostic**. The skill never recommends editing platform code mid-QA — that's a separate task with proper review.

---

## What this file does NOT provide

- Database schemas (read directly from `contextengine/cestore/`)
- API endpoint list (read from `platform/api/internal/handlers/` routing)
- Deployment credentials (handled by `deploy/` + ops)
- Tenant-specific data (never — no tenant names in skill output even internally)
- Live secrets / API keys (operator handles their own `.env`)

For anything not listed, read each repo's `CLAUDE.md` first — they're the authoritative project guidance:

- `sonzai-ai-monolith-ts/CLAUDE.md` — monolith rules
- `sonzai-ai-monolith-ts/services/ai-character-service/CLAUDE.md` — strictest layer (stateless)
- `sonzai-sdk/CLAUDE.md` — SDK rules
- `sonzai-landing/CLAUDE.md` — docs site rules
- `sonzai-claude-skill/CLAUDE.md` — skill repo rules (if present)
