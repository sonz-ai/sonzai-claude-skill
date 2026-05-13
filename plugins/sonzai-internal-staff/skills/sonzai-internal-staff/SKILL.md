---
name: sonzai-internal-staff
description: Use when working as Sonzai internal staff alongside sonzai-sdk or full-auto. Layers awareness of the Sonzai monolith (sonzai-ai-monolith-ts — contextengine, platform/api, ai-character-service, deploy) and SDK source workspace ($SONZAI_WORKSPACE/sonzai-sdk/) onto the public skills so wizard recommendations and autonomous builds can be cross-checked against server source instead of just the OpenAPI snapshot, and QA cycles can trace SDK calls into server handlers.
---

# sonzai-internal-staff

Internal-only companion to `sonzai-sdk` and `full-auto`. Layers awareness of the monolith repo (`sonzai-ai-monolith-ts`) and the broader Sonzai workspace so internal staff can dogfood the public skills with full server-side context.

**This plugin is install-time gated.** The marketplace lists it as `category: internal` with a description that flags it as useless without monolith access. If you somehow have this skill loaded but are NOT Sonzai staff with read access to the private monolith repo, uninstall: `/plugin uninstall sonzai-internal-staff@sonz-ai`.

The public skills (`sonzai-sdk`, `full-auto`) MUST remain tenant-agnostic and free of platform-internals references. This skill is the safe place to put internal pointers.

---

## Purpose

This skill exists so Sonzai internal staff can:

1. **Dogfood `full-auto`** against dev/staging/production with real API keys, real workspace context, and the ability to trace SDK calls into server handlers when QA cycles fail
2. **Dogfood `sonzai-sdk` (wizard)** while cross-checking recommendations against the **live SDK source** at `$SONZAI_WORKSPACE/sonzai-sdk/` AND the **server source** at `$SONZAI_WORKSPACE/sonzai-ai-monolith-ts/services/contextengine/` instead of just the OpenAPI snapshot
3. **Debug SDK behavior from both sides** — when a public-skill user reports an issue, internal staff can reproduce locally and trace from SDK call → platform API → context engine
4. **Catch drift between layers** — SDK source / generated bindings / live OpenAPI / contextengine source frequently disagree by one deploy or release. Internal staff need the full chain.

**Staff workspace layout** (where you read from — exact path varies per machine):

```
$SONZAI_WORKSPACE/             # set this env var; never hardcode a path
├── sonzai-claude-skill/       # this skill repo
├── sonzai-sdk/                # SDK source (sonzai-python, sonzai-typescript, sonzai-go, sonzai-openclaw, ...)
├── sonzai-ai-monolith-ts/     # monolith (contextengine, platform/api, ai-character-service, deploy)
├── sonzai-landing/            # public docs source (sonz.ai/docs)
└── ...                        # tenant projects — out of scope
```

Resolving `$SONZAI_WORKSPACE` (in priority order):

1. Env var `$SONZAI_WORKSPACE` if set and exists
2. Walk up from the operator's CWD; pick the first ancestor containing both `sonzai-sdk/` and `sonzai-ai-monolith-ts/`
3. Try `$HOME/code/sonzai/`, `$HOME/work/sonzai/`, `$HOME/dev/sonzai/`, `$HOME/src/sonzai/`
4. None of the above → print "Set `SONZAI_WORKSPACE` to the directory containing `sonzai-sdk/` and `sonzai-ai-monolith-ts/`." and exit

Read all three of `sonzai-sdk/`, `sonzai-ai-monolith-ts/`, and the public skill (`sonzai-sdk` / `full-auto` under `sonzai-claude-skill/plugins/sonzai-sdk/skills/`) when doing internal work. See `workspace-pointers.md` for the full layout.

---

## What this skill does

1. **Augments `full-auto` Phase 0 (drift check)**: in addition to the live `https://api.sonz.ai/docs/openapi.json`, ALSO read:
   - Generated OpenAPI at `$SONZAI_WORKSPACE/sonzai-ai-monolith-ts/services/platform/api/docs/` for the **most current** schema (live may be one deploy behind)
   - SDK source under `$SONZAI_WORKSPACE/sonzai-sdk/sonzai-{python,typescript,go}/` to confirm the SDK actually exposes the symbol
2. **Augments `full-auto` Phase 5 (QA loop)**: when a fix cycle fails, optionally tail server logs from the local monolith to diagnose. Stop short of editing platform code — that's a separate task.
3. **Augments `sonzai-sdk` wizard**: when wizard derivation hits a capability question, verify against:
   - Canonical Go source in `$SONZAI_WORKSPACE/sonzai-ai-monolith-ts/services/contextengine/domain/entity/agent.go` (server truth)
   - SDK bindings under `$SONZAI_WORKSPACE/sonzai-sdk/sonzai-{python,typescript,go}/` (caller truth)
   - rather than guessing from the public OpenAPI snapshot.

---

## When NOT to use this skill

- Operator wants to **edit platform/api or contextengine** — that's not a skill-driven task; just open the file
- Operator wants to **release the public skill** — keep changes in `plugins/sonzai-sdk/skills/`; don't drag internal pointers into them
- Operator is **deploying to production** — use `deploy/` scripts directly; this skill doesn't gate deployment
- A **tenant-named example** would help — never; even internal, no tenant names in code or docs you write

---

## Cross-references

- `workspace-pointers.md` — where things live in the broader Sonzai workspace: SDK source (`sonzai-sdk/`), monolith (`sonzai-ai-monolith-ts/`), public docs (`sonzai-landing/`). Paths, key files, source-of-truth ordering.
- (Future: `sdk-server-side-debugging.md`, `staging-workflow.md`, `dogfooding-full-auto.md` — add when friction demands)

---

## Public-skill cross-refs

If you also have `sonzai-sdk` or `full-auto` loaded (the normal case for internal dogfooding):

- Read the public skill first as the source of truth for SDK semantics
- Read this skill second to layer in monolith context
- If they conflict, the **public skill wins for SDK behavior** (the API is what callers see) but **this skill wins for "where to look"** (the monolith is the implementation truth)

---

## Hard rules

1. **NO tenant names** in code, docs, commit messages, or anything you write. The skill repo is public. Don't leak Razer / Eragon / PocketSouls names even though staff know them.
2. **NO API keys, secrets, internal URLs not already public.** The monolith's `.env` files and secrets are out of bounds for this skill's output.
3. **NO modifying public-skill files** to add internal pointers. Internal stuff goes in `plugins/sonzai-internal-staff/skills/sonzai-internal-staff/` only.
4. **NO `git push` from this skill's context** to the monolith repo. (`full-auto` already enforces this for target repos; this rule extends it to the monolith.)
5. **No public-user fallback needed.** This plugin is install-time gated by the marketplace; if you're loaded, you're staff.

---

## Versioning

This plugin ships in lockstep with `sonzai-sdk` from the same repo (`github.com/sonz-ai/sonzai-claude-skill`). See the repo's `CHANGELOG.md` for release history.
